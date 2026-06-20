# Claude Code — Skill 系统深度解析

> 本文以 `commit` skill 为贯穿全文的例子，完整追踪一个 Skill 从磁盘文件到注入 system prompt、再到被 SkillTool 展开执行的全部过程。

---

## 一、Skill 文件格式

Skill 是一个带有 YAML frontmatter 的 Markdown 文件（`.md`）。以 `commit` skill 为例：

```markdown
---
description: Create a git commit with conventional commit format
allowed-tools:
  - Bash
when_to_use: When the user wants to commit changes
argument-hint: "message"
---

# Git Commit

Create a git commit. Current status:
!`git status --short`

Commit message: $message

Run: `git add -A && git commit -m "$message"`
```

**Frontmatter 字段一览：**

| 字段 | 作用 | 示例 |
|------|------|------|
| `description` | 展示给模型的能力描述（写入 system-reminder） | "Create a git commit..." |
| `allowed-tools` | 该 skill 允许使用的工具（自动授权，无需用户确认） | `[Bash]` |
| `when_to_use` | 模型判断何时应调用该 skill 的提示语 | "When the user wants to commit..." |
| `argument-hint` | 声明具名参数名称，供 `$variable` 替换使用 | `"message"` |
| `model` | 覆盖默认模型（可选） | `"opus"` |
| `context` | 执行模式：`inline`（默认）或 `fork`（子 agent） | `"fork"` |
| `shell` | 用于 `!`...`` 执行的 shell 类型 | `"bash"` / `"powershell"` |
| `disable-model-invocation` | 设为 true 则该 skill 不出现在 system-reminder 中 | `true` |

---

## 二、Skill 加载流程（启动时，一次性）

### 2.1 扫描目录

`src/skills/loadSkillsDir.ts` 的 `getSkillDirCommands()` 函数在启动时扫描多个目录（**结果被 memoize 缓存**）：

```
优先级（从高到低）:
  1. ${getManagedFilePath()}/.claude/skills/     ← 企业管控 skills
  2. ~/.claude/skills/                           ← 用户全局 skills
  3. .claude/skills/  （向上遍历目录树）          ← 项目级 skills
  4. .claude/commands/ 和 commands/              ← 旧版路径（向后兼容）
```

每个目录下的 `.md` 文件都会被逐个加载。同名 skill 按上面优先级覆盖（项目级 > 用户级 > 管控级）。通过 `realpath()` 解析符号链接后去重，避免同一个文件被重复加载。

### 2.2 解析 Frontmatter

```typescript
// src/utils/frontmatterParser.ts
const FRONTMATTER_REGEX = /^---\s*\n([\s\S]*?)---\s*\n?/

export function parseFrontmatter(markdown: string): ParsedMarkdown {
  const match = markdown.match(FRONTMATTER_REGEX)
  const frontmatter = parseYaml(match[1])   // js-yaml 解析
  const content = markdown.slice(match[0].length)
  return { frontmatter, content }
}
```

解析结果（针对 commit.md）：

```javascript
frontmatter = {
  description: "Create a git commit with conventional commit format",
  "allowed-tools": ["Bash"],
  when_to_use: "When the user wants to commit changes",
  "argument-hint": "message"
}

content = "# Git Commit\n\n...!`git status --short`...$message..."
```

### 2.3 创建 Command 对象

`createSkillCommand()` 基于解析结果构建一个 `Command` 对象，其核心是一个**闭包方法** `getPromptForCommand()`，它在调用时才真正展开内容：

```typescript
// src/skills/loadSkillsDir.ts
const command: Command = {
  name: "commit",
  type: "prompt",
  description: "Create a git commit with conventional commit format",
  whenToUse: "When the user wants to commit changes",
  allowedTools: ["Bash"],        // 来自 frontmatter['allowed-tools']
  argumentNames: ["message"],   // 来自 frontmatter['argument-hint'] 解析

  async getPromptForCommand(args, toolUseContext) {
    // 实际展开逻辑见第四章
  }
}
```

**设计要点**：Frontmatter 解析发生在**加载时**（一次），内容展开（变量替换、Shell 执行）发生在**调用时**（每次）。

---

## 三、Skills 注入 System Prompt（每轮第一次）

### 3.1 总体机制

Skills **不是**写死在静态 system prompt 里的。它们通过 **`<system-reminder>` 动态注入**，在会话的第一轮作为附件（attachment）随用户消息一起发送给模型。

```
用户输入（第一轮）
    │
    ▼
processUserInput.ts
    │  shouldExtractAttachments = true（非 slash command）
    ▼
getAttachmentMessages()              [attachments.ts]
    │
    ├── getSkillListingAttachments()  ← skill 列表在这里生成
    │
    ▼
normalizeAttachmentForAPI()          [messages.ts]
    │
    ▼
<system-reminder>
The following skills are available for use with the Skill tool:
- commit: ...
- review-pr: ...
</system-reminder>
```

### 3.2 `shouldExtractAttachments` 的关键判断

```typescript
// src/utils/processUserInput/processUserInput.ts:496
const shouldExtractAttachments =
  !skipAttachments &&
  inputString !== null &&
  (mode !== 'prompt' || effectiveSkipSlash || !inputString.startsWith('/'))
```

| 输入 | 结果 | 原因 |
|------|------|------|
| `"帮我解释这段代码"` | ✅ 提取 attachment | 不以 `/` 开头 |
| `"/commit feat: add feature"` | ❌ 跳过提取 | 以 `/` 开头，走 slash command 独立路径 |

当用户直接输入 `/commit` 时，系统走 `processSlashCommand()` 分支，由其内部自行处理 attachments，避免双重提取。

### 3.3 `getSkillListingAttachments()` 的核心逻辑

```typescript
// src/utils/attachments.ts:2661
async function getSkillListingAttachments(toolUseContext) {
  // 前置：没有 SkillTool 可用则直接返回
  if (!toolPool.has(SKILL_TOOL_NAME)) return []

  const allCommands = [
    ...await getSkillToolCommands(cwd),  // 过滤后的本地 skills
    ...getMcpSkillCommands(),            // MCP 来源的 skills
  ]

  // per-agent 的已发送记录（主线程和子 agent 各自独立）
  const agentKey = toolUseContext.agentId ?? ''
  let sent = sentSkillNames.get(agentKey) ?? new Set()

  // 仅发送"新增"的 skills（首次全部是新的）
  const newSkills = allCommands.filter(cmd => !sent.has(cmd.name))
  if (newSkills.length === 0) return []   // Turn 1+ 大多走这里

  newSkills.forEach(cmd => sent.add(cmd.name))

  // 格式化，受 token 预算约束
  const content = formatCommandsWithinBudget(newSkills, contextWindowTokens)
  return [{ type: 'skill_listing', content }]
}
```

**`getSkillToolCommands()` 的过滤条件**（`src/commands.ts`）：
- `type === 'prompt'`
- **没有** `disable-model-invocation: true`
- 来自 `/skills/` 目录，或者拥有 `description` / `whenToUse` 字段

### 3.4 Token 预算约束

```typescript
// src/tools/SkillTool/prompt.ts
const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 最多占用 1% context window
const MAX_LISTING_DESC_CHARS = 250          // 每条描述最多 250 字符
```

格式化后的列表示例：
```
- commit: Create a git commit with conventional commit format - When the user wants to commit changes
- review-pr: Review a PR across GitHub or ADO with deep repo knowledge
```

Bundled（内置）skills 的描述优先保留，用户自定义 skills 超出预算时会被截断。

### 3.5 最终注入的消息格式

```typescript
// src/utils/messages.ts:3728
case 'skill_listing': {
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `The following skills are available for use with the Skill tool:\n\n${attachment.content}`,
      isMeta: true,
    }),
  ])
}
```

模型实际收到：

```xml
<system-reminder>
The following skills are available for use with the Skill tool:

- commit: Create a git commit with conventional commit format - When the user wants to commit changes
- review-pr: Review a PR across GitHub or ADO with deep repo knowledge
- ...
</system-reminder>
```

### 3.6 多轮 Loop 中的更新行为

| 轮次 | `sentSkillNames` 状态 | 注入行为 |
|------|----------------------|---------|
| Turn 0（首轮） | 空 Set | ✅ 发送全量 skill 列表 |
| Turn 1, 2, ...（后续轮） | 已包含所有 skill 名 | ❌ 返回 `[]`，不注入 |
| 插件重载 / skill 文件变更 | `resetSentSkillNames()` 清空 | ✅ 重新发送新增部分 |
| `--resume` 恢复会话 | `suppressNextSkillListing()` 标记 | ❌ 跳过（历史中已有） |

**Loop 中的调用位置**（`query.ts:1580`）：每次工具调用完毕后，系统都会调用一次 `getAttachmentMessages(null, ...)`，但因为 `sentSkillNames` 缓存，Turn 1+ 几乎零开销。

**为什么不写死在静态 system prompt 里？**
静态 system prompt 会被 prompt cache 缓存（按前缀匹配）。如果 skill 列表写死其中，每当 skills 增减，整个 cache 就会失效。动态注入为 attachment 则彻底规避了这个问题。

---

## 四、SkillTool 展开特定 Skill（调用时）

当模型决定调用某个 skill 时（无论是用户显式输入 `/commit "msg"` 还是模型自主调用），SkillTool 开始工作。

### 4.1 SkillTool.call() 入口

```typescript
// src/tools/SkillTool/SkillTool.ts
async call({ skill, args }, context) {
  const commandName = skill.replace(/^\//, '')  // 去掉前缀 /，得到 "commit"

  // 检查是否 fork 模式（此 skill 无 context:fork，跳过）

  // 核心：展开 skill 内容
  const result = await processPromptSlashCommand(
    commandName,   // "commit"
    args ?? '',    // "feat: add feature"
    commands,
    context,
  )

  // 返回：展开后的消息 + 工具权限修改器
  return { newMessages: result.newMessages, contextModifier: ... }
}
```

### 4.2 `processPromptSlashCommand()` 调度

```typescript
// src/utils/processUserInput/processSlashCommand.tsx
export async function processPromptSlashCommand(
  commandName, args, commands, context
): Promise<SlashCommandResult> {
  
  // 在命令表中查找 "commit"
  const command = commands.find(cmd => cmd.name === commandName)
  
  // 委托展开
  return getMessagesForPromptSlashCommand(command, args, context)
}
```

### 4.3 `getPromptForCommand()` — 三步展开管线

这是整个 skill 处理的核心，三步**有序执行**（顺序不可颠倒）：

```
原始 Markdown 内容
        │
        ▼  Step 1: 参数替换
   substituteArguments()
        │   $message → "feat: add feature"
        ▼  Step 2: 环境变量替换
   ${CLAUDE_SKILL_DIR} → 实际路径
   ${CLAUDE_SESSION_ID} → 当前会话 ID
        │
        ▼  Step 3: Shell 命令执行
   executeShellCommandsInPrompt()
        │   !`git status --short` → 实际输出
        ▼
   最终展开后的文本
```

**Step 1：参数替换（`substituteArguments()`）**

```typescript
// src/utils/argumentSubstitution.ts
// argumentNames = ["message"]，对应位置参数
// args = "feat: add feature"

// 解析参数（支持 shell 引号规则）
const parsedArgs = tryParseShellCommand(args)
// → ["feat: add feature"]

// 替换 $message（后面不跟 [ 或 \w，避免误匹配 $message_id）
const regex = /\$message(?![\[\w])/g
content = content.replace(regex, "feat: add feature")
```

还支持以下几种占位符形式：

| 占位符 | 含义 | 示例 |
|--------|------|------|
| `$message` | 具名参数（来自 `argument-hint`） | `$message` → `"feat: add feature"` |
| `$ARGUMENTS` | 完整原始参数字符串 | `$ARGUMENTS` → `"feat: add feature"` |
| `$0`, `$1` | 按位置的参数 | `$0` → `"feat: add feature"` |
| `$ARGUMENTS[0]` | 按下标的参数 | `$ARGUMENTS[0]` → `"feat: add feature"` |

**Step 2（Step 1 结束后）：Shell 命令插值（`executeShellCommandsInPrompt()`）**

注意：Shell 执行发生在参数替换之后，因此 `!`...`` 内部可以使用已替换好的变量值。

```typescript
// src/utils/promptShellExecution.ts
// 支持两种语法：
// 内联：  !`command`
// 块级：  ```!\ncommand\n```

const INLINE_PATTERN = /(?<!\\)!`([^`]+)`/g

// 并行执行所有匹配到的 Shell 命令
await Promise.all(inlineMatches.map(async match => {
  const command = match[1].trim()  // "git status --short"

  // ① 权限门：检查 Bash 是否在 alwaysAllowRules 中
  //   因为 getPromptForCommand() 已提前注入 allowedTools，此处直接通过
  const permissionResult = await hasPermissionsToUseTool(bashTool, { command }, context)
  if (permissionResult.behavior !== 'allow') {
    throw new MalformedCommandError(...)  // 权限不足，整个 skill 加载失败
  }

  // ② 执行命令
  const { data } = await bashTool.call({ command }, context)

  // ③ 替换原文中的 !`git status --short` 为命令输出
  result = result.replace(match[0], () => data.stdout)
}))
```

**安全约束**：MCP 来源的 skills（`loadedFrom === 'mcp'`）**完全跳过** `executeShellCommandsInPrompt()` 调用，防止远程不可信内容触发任意 Shell 执行。

### 4.4 展开结果示例

假设执行 `/commit "feat: add feature"`，展开后文本为：

```markdown
# Git Commit

Create a git commit. Current status:
 M src/index.ts
?? newfile.txt

Commit message: feat: add feature

Run: `git add -A && git commit -m "feat: add feature"`
```

这段文本作为 **user 消息**注入对话历史，模型随后读取并执行其中的指令。

### 4.5 `contextModifier` — 工具权限注入

SkillTool.call() 除返回消息外，还返回一个 `contextModifier`，在后续模型调用中**修改工具权限上下文**：

```typescript
// src/tools/SkillTool/SkillTool.ts
contextModifier(ctx) {
  return {
    ...ctx,
    getAppState() {
      const appState = ctx.getAppState()
      return {
        ...appState,
        toolPermissionContext: {
          ...appState.toolPermissionContext,
          alwaysAllowRules: {
            ...appState.toolPermissionContext.alwaysAllowRules,
            command: [
              ...new Set([
                ...(appState.toolPermissionContext.alwaysAllowRules.command || []),
                ...allowedTools,   // ["Bash"]
              ]),
            ],
          },
        },
      }
    },
  }
}
```

效果：当模型随后决定运行 `git add -A && git commit -m "feat: add feature"` 时，**Bash 工具已在白名单中，不会弹出用户确认对话框**。

`allowed-tools` 的双重作用：

| 时机 | 用途 |
|------|------|
| `getPromptForCommand()` 执行期间 | 授权 `!`...`` 内的 Shell 命令执行 |
| SkillTool 返回后（通过 contextModifier） | 授权模型后续的工具调用 |

---

## 五、Fork 模式（`context: fork`）

如果 skill 的 frontmatter 中声明 `context: fork`，则不走上述内联路径，而是**派生一个子 agent** 来执行：

```
SkillTool.call()
    │  发现 context: fork
    ▼
executeForkedSkill()
    │
    ▼
prepareForkedCommandContext()       [src/utils/forkedAgent.ts]
    │  展开 skill 内容（同样走三步管线）
    ▼
派生子 agent（独立 token 预算，独立 context window）
    │  model / agent 类型来自 skill frontmatter
    ▼
收集子 agent 的输出文本
    │
    ▼
作为内联结果返回给父会话
```

Fork 模式适用于需要较长多步推理的复杂 skills，子 agent 的 context 消耗不影响父会话。

---

## 六、关键约束与设计边界汇总

### 6.1 内容约束

| 约束 | 参数 | 影响 |
|------|------|------|
| Skill 列表预算 | context window 的 **1%** | 控制 system-reminder 大小 |
| 单条描述上限 | **250 字符** | 过长描述被截断 |
| 执行计划分组 | 每个 plan 最多 **3-5 个 task** | 防止单次 skill 范围过大 |

### 6.2 安全约束

| 约束 | 位置 | 说明 |
|------|------|------|
| MCP skills 禁止 Shell 执行 | `loadSkillsDir.ts:374` | 防止远程内容注入任意命令 |
| `!`...`` 执行前强制权限检查 | `promptShellExecution.ts` | 无权限则抛 `MalformedCommandError`，skill 整体失败 |
| `allowed-tools` 白名单 | frontmatter | 未列出的工具，模型仍需走正常权限流程 |
| 不可见于用户 transcript | `isMeta: true` + `<system-reminder>` | skill 列表对用户界面隐藏 |

### 6.3 幂等性约束

| 约束 | 机制 | 说明 |
|------|------|------|
| Skill 列表只发送一次 | `sentSkillNames` Map | 每个 agent 各自维护，跨轮去重 |
| `--resume` 不重复注入 | `suppressNextSkillListing()` | 历史消息中已有列表，防止重复 |
| Skill 文件变更后重发 | `resetSentSkillNames()` | 插件重载 / 文件变更时触发 |

### 6.4 执行顺序约束

`getPromptForCommand()` 内部的三步操作**顺序固定，不可颠倒**：

```
参数替换 → 环境变量替换 → Shell 执行
```

原因：Shell 命令（`!`...``）执行时已能看到替换后的参数值，这是有意设计。如果顺序颠倒，Shell 命令中就无法引用 `$message` 等参数。

---

## 七、完整数据流总结

```
【加载阶段（启动时，一次）】

commit.md (磁盘)
    │ parseFrontmatter()
    ▼
{ frontmatter, content }
    │ createSkillCommand()
    ▼
Command 对象（含 getPromptForCommand 闭包）
    │ 注册到全局命令表
    ▼
commands[]


【注入阶段（会话首轮）】

用户输入（非 slash command）
    │ shouldExtractAttachments = true
    ▼
getAttachmentMessages()
    │ getSkillListingAttachments()
    │   → sentSkillNames 空 → 全量新 skill
    │   → formatCommandsWithinBudget() [1% 预算]
    ▼
{ type: 'skill_listing', content: "- commit: ..." }
    │ normalizeAttachmentForAPI()
    ▼
<system-reminder>
The following skills are available for use with the Skill tool:
- commit: Create a git commit... - When the user wants to commit...
</system-reminder>
    │ 随 Turn 0 消息发送给模型


【执行阶段（模型决定调用 /commit "feat: add feature"）】

SkillTool.call("commit", "feat: add feature")
    │ processPromptSlashCommand()
    │ getMessagesForPromptSlashCommand()
    ▼
command.getPromptForCommand("feat: add feature", context)
    │
    ├─ Step 1: substituteArguments()
    │    $message → "feat: add feature"
    │
    ├─ Step 2: 环境变量替换
    │    ${CLAUDE_SKILL_DIR} → 实际路径
    │
    └─ Step 3: executeShellCommandsInPrompt()
         !`git status --short` → " M src/index.ts\n?? newfile.txt"
    │
    ▼
展开后的 Markdown 文本（注入为 user 消息）
    +
contextModifier（Bash 加入 alwaysAllowRules）
    │
    ▼
模型读取展开内容，调用 Bash 执行 git 命令（无需用户确认）
```

---

## 八、关键文件索引

| 职责 | 文件 | 关键函数 |
|------|------|---------|
| Frontmatter 解析 | `src/utils/frontmatterParser.ts` | `parseFrontmatter()` |
| Skill 加载 & Command 创建 | `src/skills/loadSkillsDir.ts` | `getSkillDirCommands()`, `createSkillCommand()` |
| Command 聚合 & 过滤 | `src/commands.ts` | `getSkillToolCommands()` |
| Skill 列表格式化 | `src/tools/SkillTool/prompt.ts` | `formatCommandsWithinBudget()` |
| Attachment 生成 | `src/utils/attachments.ts` | `getSkillListingAttachments()` |
| Attachment → 消息 | `src/utils/messages.ts` | `normalizeAttachmentForAPI()` |
| Skill 展开入口 | `src/tools/SkillTool/SkillTool.ts` | `call()`, `contextModifier` |
| Slash command 调度 | `src/utils/processUserInput/processSlashCommand.tsx` | `processPromptSlashCommand()` |
| 参数替换 | `src/utils/argumentSubstitution.ts` | `substituteArguments()` |
| Shell 插值执行 | `src/utils/promptShellExecution.ts` | `executeShellCommandsInPrompt()` |
