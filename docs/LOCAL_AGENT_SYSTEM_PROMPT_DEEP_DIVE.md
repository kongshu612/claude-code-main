# Local Agent 最终 System Prompt 的构成

> 深入剖析：一个 **local agent**（从 `.claude/agents/*.md` 文件加载的子代理）最终发送给模型 API 的 system prompt 究竟由哪些部分拼装而成、每一部分来自哪里、长什么样。
>
> 本文所有结论均基于 `claude-code-main` 参考实现，并已对照源码逐条核实（含 `file:line` 引用）。

---

## TL;DR

对于一个 local agent，发送给 API 的 system prompt **不会**被包进 `DEFAULT_AGENT_PROMPT`，也没有任何内置前言/后缀。它本质上是：

```
[ <.md 文件正文 body，trim 后>  (+ 可选的 memory 块) ]
+ [ "Notes:" 块（绝对路径 / 禁用 emoji / tool 调用前不加冒号 等规则） ]
+ [ 可选的 DiscoverSkills 引导（external 构建中通常被 DCE 掉，不出现） ]
+ [ "<env>" 块（cwd、是否 git repo、平台、shell、OS、模型名、knowledge cutoff） ]
+ [ "gitStatus: ..." 块（在更靠后的 query 层追加） ]
```

agent 自己的文字 = **就是 `.md` 的正文 body**。其余全是 `enhanceSystemPromptWithEnvDetails` 和 query 层附加的机械内容。

**关键点：** `CLAUDE.md` 不在 system prompt 里 —— 它作为一条单独 prepend 的 **user message**（包在 `<system-reminder>` 里）送达模型。

---

## 1. "local" agent 与其他 agent 的区别

判别字段是 agent 定义对象上的 `source`。

| Agent 种类 | `source` 值 | 发现位置 | `getSystemPrompt` 返回的 body |
|---|---|---|---|
| **Local（user）** | `'userSettings'` | `~/.claude/agents/*.md` | `.md` 正文（`content.trim()`） |
| **Local（project）** | `'projectSettings'` | `<repo>/.claude/agents/*.md`（向上走到 git root） | `.md` 正文 |
| Managed/policy | `'policySettings'` | 受管的 `.claude/agents` | `.md` 正文 |
| Built-in | `'built-in'` | `src/tools/AgentTool/built-in/*.ts` | 硬编码的 TS 函数 |
| Plugin | `'plugin'` | `loadPluginAgents()` | 插件内的 `.md` 正文 |

- `source` 在 [markdownConfigLoader.ts:343/353/366](../src/utils/markdownConfigLoader.ts#L343) 根据文件来自哪个 `.claude` 目录被打上。
- 类型守卫见 [loadAgentsDir.ts:168-184](../src/tools/AgentTool/loadAgentsDir.ts#L168-L184)：
  - `isBuiltInAgent` = `source === 'built-in'`
  - `isCustomAgent` = `source !== 'built-in' && source !== 'plugin'` —— 这就是 "local" 这一桶
  - `isPluginAgent` = `source === 'plugin'`
- **同名优先级**（后者覆盖前者）：built-in < plugin < user < project < flag < managed，见 `getActiveAgentsFromList` [loadAgentsDir.ts:203-220](../src/tools/AgentTool/loadAgentsDir.ts#L203-L220)。
- 从 local `.md` 解析的 frontmatter 字段：`name`→`agentType`，`description`→`whenToUse`（必填），以及可选的 `tools`、`disallowedTools`、`model`、`effort`、`permissionMode`、`mcpServers`、`hooks`、`maxTurns`、`skills`、`initialPrompt`、`memory`、`background`、`isolation`、`color`，见 [loadAgentsDir.ts:548-747](../src/tools/AgentTool/loadAgentsDir.ts#L548-L747)。**frontmatter 之后的正文 body** 成为 `systemPrompt = content.trim()`（[loadAgentsDir.ts:713](../src/tools/AgentTool/loadAgentsDir.ts#L713)）。
- 只有 `name` 和 `description` 是必填的；两者都缺的文件会被当作"同目录文档"静默跳过（[loadAgentsDir.ts:320-325, 554-562](../src/tools/AgentTool/loadAgentsDir.ts#L554-L562)）。

---

## 2. 构造链（端到端）

```
.claude/agents/code-reviewer.md  (frontmatter + body)
        │
        │  loadMarkdownFilesForSubdir('agents', cwd)            markdownConfigLoader.ts:298+
        │     → source: 'projectSettings' | 'userSettings'      （这就是它"local"的来源）
        ▼
parseAgentFromMarkdown(...)                                     loadAgentsDir.ts:541
        │     systemPrompt = content.trim()                     loadAgentsDir.ts:713
        │     getSystemPrompt = () => systemPrompt [+ memory]   loadAgentsDir.ts:726-732
        ▼
CustomAgentDefinition  (source = 'projectSettings'/'userSettings', 非 'built-in')
        │
        ▼
AgentTool.tsx  call.exec  （正常的、非 fork 路径）
        │  agentPrompt = selectedAgent.getSystemPrompt({toolUseContext})   AgentTool.tsx:518
        │  enhancedSystemPrompt =
        │     enhanceSystemPromptWithEnvDetails([agentPrompt], model, dirs) AgentTool.tsx:534
        ▼
enhanceSystemPromptWithEnvDetails(...)                          prompts.ts:760-791
        │  返回: [ agentPrompt, notes, (discoverSkills?), envInfo ]
        ▼
runAgent({ override: { systemPrompt: asSystemPrompt(enhancedSystemPrompt) } })  AgentTool.tsx:622-626
        ▼
query({ systemPrompt, userContext, systemContext, ... })       runAgent.ts:748-753
        │  fullSystemPrompt = appendSystemContext(systemPrompt, systemContext)  query.ts:449-451
        │     → 末尾追加一个 "gitStatus: ..." 块                  api.ts:437-447
        ▼
callModel({ systemPrompt: fullSystemPrompt,
            messages: prependUserContext(messages, userContext) })  query.ts:659-661
            （userContext/claudeMd 变成单独的一条 USER message，不是 system）  api.ts:449-474
```

### 两个等价的构造点（结果相同）

正常路径在 [AgentTool.tsx:534](../src/tools/AgentTool/AgentTool.tsx#L534) 构造 prompt，并作为 `override.systemPrompt` 传下去。
如果存在 cwd/worktree override（即 `enhancedSystemPrompt && !worktreeInfo && !cwd` 在 [AgentTool.tsx:624](../src/tools/AgentTool/AgentTool.tsx#L624) 为 false），则 `override.systemPrompt` 为 `undefined`，[runAgent.ts:508](../src/tools/AgentTool/runAgent.ts#L508) 转而调用 `getAgentSystemPrompt`（[runAgent.ts:906-932](../src/tools/AgentTool/runAgent.ts#L906-L932)），它执行**完全相同**的 `getSystemPrompt(...)` → `enhanceSystemPromptWithEnvDetails(...)` 序列。唯一区别：它把 `enabledToolNames` 作为第 4 个参数传入，使 DiscoverSkills 那行受"该工具是否真的在工具池中"门控。

---

## 3. `selectedAgent.getSystemPrompt({toolUseContext})` 对 local agent 返回什么

对 local（custom）agent，它就是 [loadAgentsDir.ts:726-732](../src/tools/AgentTool/loadAgentsDir.ts#L726-L732) 构造的闭包：

```js
getSystemPrompt: () => {
  if (isAutoMemoryEnabled() && memory) {
    const memoryPrompt = loadAgentMemoryPrompt(agentType, memory)
    return systemPrompt + '\n\n' + memoryPrompt
  }
  return systemPrompt        // systemPrompt === content.trim()  （.md 的原始正文）
}
```

要点：

- **它逐字返回 markdown 正文 body。** 没有 `DEFAULT_AGENT_PROMPT`、没有 `SHARED_PREFIX`、没有前言/后缀、不插值 tools/cwd/agent 列表。该闭包**完全忽略**它的 `{toolUseContext}` 参数（接收它只是为了与 built-in agent 的调用形态一致 —— [AgentTool.tsx:517](../src/tools/AgentTool/AgentTool.tsx#L517) 注释 "All agents have getSystemPrompt"）。

- **`DEFAULT_AGENT_PROMPT`**（[prompts.ts:758](../src/constants/prompts.ts#L758)）内容是：

  > "You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Complete the task fully—don't gold-plate, but don't leave it half-done. When you complete the task, respond with a concise report covering what was done and any key findings — the caller will relay this to the user, so it only needs the essentials."

  它**仅**在 `getAgentSystemPrompt` 的 `catch` 块中作为兜底（[runAgent.ts:925-930](../src/tools/AgentTool/runAgent.ts#L925-L930)）—— 当 `getSystemPrompt()` 抛异常时才用。正常的 local agent 永远见不到它。（一个几乎相同的 `SHARED_PREFIX` 被烤进了 built-in agent，如 [generalPurposeAgent.ts:3,20](../src/tools/AgentTool/built-in/generalPurposeAgent.ts#L3)。）

- **与 built-in agent 的对比：** built-in（如 `GENERAL_PURPOSE_AGENT`，[generalPurposeAgent.ts:25-34](../src/tools/AgentTool/built-in/generalPurposeAgent.ts#L25-L34)）把 `getSystemPrompt` 定义为一个硬编码 TS 函数，返回 `SHARED_PREFIX + … + SHARED_GUIDELINES`。所以 built-in 自带一大段 Anthropic 撰写的 prompt；而 local 只带用户写在 `.md` 正文里的内容。

---

## 4. `enhanceSystemPromptWithEnvDetails(prompts, model, dirs)`

实现见 [prompts.ts:760-791](../src/constants/prompts.ts#L760-L791)：

```js
return [
  ...existingSystemPrompt,                 // = [agentPrompt]
  notes,                                    // 下方的 "Notes:" 块
  ...(discoverSkillsGuidance !== null ? [discoverSkillsGuidance] : []),
  envInfo,                                  // = await computeEnvInfo(model, dirs)
]
```

**顺序：** agent body 在前，然后 `notes`，然后（可选）DiscoverSkills，最后 `envInfo`。返回一个字符串**数组**（每个元素是一个 system-prompt 块）；这里**不做 join**。

### 4a. `notes` 块（[prompts.ts:766-770](../src/constants/prompts.ts#L766-L770)，逐字）

```
Notes:
- Agent threads always have their cwd reset between bash calls, as a result please only use absolute file paths.
- In your final response, share file paths (always absolute, never relative) that are relevant to the task. Include code snippets only when the exact text is load-bearing (e.g., a bug you found, a function signature the caller asked for) — do not recap code you merely read.
- For clear communication with the user the assistant MUST avoid using emojis.
- Do not use a colon before tool calls. Text like "Let me read the file:" followed by a read tool call should just be "Let me read the file." with a period.
```

### 4b. DiscoverSkills 引导（[prompts.ts:777-783](../src/constants/prompts.ts#L777-L783)，条件性）

仅当 `feature('EXPERIMENTAL_SKILL_SEARCH')` **且** skill search 已启用 **且** `DISCOVER_SKILLS_TOOL_NAME !== null` **且** `(enabledToolNames?.has(DISCOVER_SKILLS_TOOL_NAME) ?? true)` 时才包含。在 external 构建中该 feature 被 DCE 折叠为 `false`，所以这一块**通常不出现**。

注意：AgentTool.tsx 这个构造点（:534）**不传** `enabledToolNames`，所以依赖 `?? true`；而 runAgent 构造点会传入它。

### 4c. `envInfo` —— 来自 `computeEnvInfo(modelId, dirs)`（[prompts.ts:606-649](../src/constants/prompts.ts#L606-L649)）

```
Here is useful information about the environment you are running in:
<env>
Working directory: <getCwd()>
Is directory a git repo: <Yes|No>          ← await getIsGit()
<Additional working directories: a, b\n>     ← 仅当 dirs.length>0  (prompts.ts:630-633)
Platform: <env.platform>                     ← 如 linux/darwin/win32
Shell: <zsh|bash|...>                         ← getShellInfoLine(); win32 会追加一条 Unix 语法警告 (prompts.ts:732-743)
OS Version: <uname -sr | Windows 版本>        ← getUnameSR() (prompts.ts:745-756)
</env>
<modelDescription><knowledgeCutoffMessage>
```

其中：

- `modelDescription` = `"You are powered by the model named <marketingName>. The exact model ID is <modelId>."`（若无 marketing name 则为 `"...by the model <modelId>."`）。在 ant-undercover 构建中被整体抑制（[prompts.ts:620-628](../src/constants/prompts.ts#L620-L628)）。
- `knowledgeCutoffMessage` = `"\n\nAssistant knowledge cutoff is <date>."`，按模型家族取值（`getKnowledgeCutoff`，[prompts.ts:713-730](../src/constants/prompts.ts#L713-L730)）；未知则为空。

> **注意：** 这里的 `computeEnvInfo` 是**子代理**版本的 env 块 —— 简洁的 `<env>` 形式。更丰富的带 bullet 的 `computeSimpleEnvInfo`（[prompts.ts:651-710](../src/constants/prompts.ts#L651-L710)）是给**主会话**的 `getSystemPrompt` 用的，不是给子代理用的。

---

## 5. 最终拼装 + query 层的追加

`enhanceSystemPromptWithEnvDetails` 之后，数组被传给 `runAgent` → `query`。在 query 边界还会发生两件事（[query.ts:449-451, 659-661](../src/query.ts#L449-L451)）：

1. **`appendSystemContext(systemPrompt, systemContext)`**（[api.ts:437-447](../src/utils/api.ts#L437-L447)）在 system-prompt 数组**末尾再追加一个块**：对每个 `systemContext` 条目追加 `"<key>: <value>"`。对 local agent，唯一的 key 是 `gitStatus`（来自 `getSystemContext`，[context.ts:116-149](../src/context.ts#L116-L149)），所以最后一个 system 块字面上就是 `"gitStatus: " + <git status 文本>`。（built-in 的 `Explore`/`Plan` 会剥掉它；local agent **不会** —— [runAgent.ts:404-410](../src/tools/AgentTool/runAgent.ts#L404-L410)。）

2. **`prependUserContext(messages, userContext)`**（[api.ts:449-474](../src/utils/api.ts#L449-L474))—— `userContext`（`claudeMd` + `currentDate`，[context.ts:155-189](../src/context.ts#L155-L189)）**不会**进 system prompt。它变成一条单独 prepend 的 **user** message，包在 `<system-reminder>…# claudeMd…# currentDate…</system-reminder>` 里（`isMeta: true`）。所以 CLAUDE.md 会到达模型，但是作为 user message，不是 system。

### 最终 system-prompt 数组逐块表

| # | 块 | 来源（file:line） | 贡献内容 |
|---|---|---|---|
| 0 | Agent body（`content.trim()`）[+ `\n\n` + memory] | [loadAgentsDir.ts:713,726-732](../src/tools/AgentTool/loadAgentsDir.ts#L713) | 用户撰写的 agent 指令 —— 唯一的 agent 专属文本 |
| 1 | `Notes:` 块 | [prompts.ts:766-770](../src/constants/prompts.ts#L766-L770) | 子代理的绝对路径 / 禁 emoji / tool 调用前不加冒号 等规则 |
| 2 | DiscoverSkills 引导 *（通常不出现）* | [prompts.ts:777-783](../src/constants/prompts.ts#L777-L783) | skill-search 框架；受门控，external 构建被 DCE 掉 |
| 3 | `<env>` 块 + 模型 + cutoff | `computeEnvInfo`，[prompts.ts:606-649](../src/constants/prompts.ts#L606-L649) | cwd、是否 git repo、附加目录、平台、shell、OS 版本、模型名/ID、knowledge cutoff |
| 4 | `gitStatus: …` 块 | `appendSystemContext` [api.ts:437-447](../src/utils/api.ts#L437-L447) ← `getGitStatus` [context.ts:60-103](../src/context.ts#L60-L103) | branch、main branch、git user、`git status --short`、最近 5 个 commit |
| — | claudeMd / currentDate | `prependUserContext` [api.ts:449-474](../src/utils/api.ts#L449-L474) | **单独的 user message**，不属于 system prompt |

发往 API 时，数组会被拼接（default/3P 模式用 `\n\n` 连接非 prefix 的块，见 `splitSysPromptPrefix`，[api.ts:321-434](../src/utils/api.ts#L321-L434)）；主会话用的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记在子代理数组里**不存在**。

---

## 6. 具体示例

> 仅作示意 —— 把可信的值填入真实模板。假设 `<repo>/.claude/agents/code-reviewer.md`：

```markdown
---
name: code-reviewer
description: Reviews code changes for bugs, security, and style issues
tools: Read, Grep, Glob
model: sonnet
---

You are a meticulous code reviewer. When given a diff or set of files:
- Check for bugs, edge cases, and security issues (injection, auth, unsafe IO).
- Flag style inconsistencies with the project's conventions.
- Be concise: report only actionable findings, most severe first.
```

最终 **system prompt**（块数组；这里展示为 API 看到的拼接形态）：

```
You are a meticulous code reviewer. When given a diff or set of files:
- Check for bugs, edge cases, and security issues (injection, auth, unsafe IO).
- Flag style inconsistencies with the project's conventions.
- Be concise: report only actionable findings, most severe first.

Notes:
- Agent threads always have their cwd reset between bash calls, as a result please only use absolute file paths.
- In your final response, share file paths (always absolute, never relative) that are relevant to the task. Include code snippets only when the exact text is load-bearing (e.g., a bug you found, a function signature the caller asked for) — do not recap code you merely read.
- For clear communication with the user the assistant MUST avoid using emojis.
- Do not use a colon before tool calls. Text like "Let me read the file:" followed by a read tool call should just be "Let me read the file." with a period.

Here is useful information about the environment you are running in:
<env>
Working directory: /home/jane/myproject
Is directory a git repo: Yes
Platform: linux
Shell: bash
OS Version: Linux 6.6.4
</env>
You are powered by the model named Claude Sonnet 4.6. The exact model ID is claude-sonnet-4-6.

Assistant knowledge cutoff is August 2025.

gitStatus: This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.

Current branch: feature/login

Main branch (you will usually use this for PRs): main

Git user: Jane Dev

Status:
 M src/auth/login.ts

Recent commits:
abc1234 Add login form
def5678 Wire up auth service
```

并且 —— 单独地，作为第一条 **user** message（不是 system）—— CLAUDE.md + 日期：

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
<项目 CLAUDE.md 的内容>
# currentDate
Today's date is 2026-06-09.

      IMPORTANT: this context may or may not be relevant to your tasks. You should not respond to this context unless it is highly relevant to your task.
</system-reminder>
```

> 若有 `--add-dir` 激活的附加目录，`<env>` 块会在 git-repo 行与 `Platform:` 行之间多出一行 `Additional working directories: …`；在 Windows 上 `Shell:` 行会追加 Unix 语法警告。

---

## 7. 边界情况

- **空的 agent body：** `parseAgentFromMarkdown` 只要求 `name` + `description`，不要求 body 有内容（[loadAgentsDir.ts:554-562](../src/tools/AgentTool/loadAgentsDir.ts#L554-L562)）。空 body 得到 `content.trim() === ""`，所以块 0 是空字符串。无害：`splitSysPromptPrefix` 会跳过 falsy 的块（[api.ts:337](../src/utils/api.ts#L337)，`if (!prompt) continue`），实际 prompt 直接从 `Notes:` 块开始。（相比之下 JSON-agent 路径通过 `z.string().min(1)` 要求非空 `prompt`，[loadAgentsDir.ts:78](../src/tools/AgentTool/loadAgentsDir.ts#L78) —— markdown 路径不要求。）

- **`tools` / `model` frontmatter：** 二者都**不改变** agent body 文本。
  - `model` 会**间接**改变 system prompt：它设置 `resolvedAgentModel`，进而喂给 `computeEnvInfo` 的 "You are powered by the model …" 行与 knowledge-cutoff 行（块 3）。它当然也选定实际使用的模型。
  - `tools` 只塑造 worker 工具池（`assembleToolPool`，[AgentTool.tsx:577](../src/tools/AgentTool/AgentTool.tsx#L577)）与 `enabledToolNames`（用于门控可选的 DiscoverSkills 行）。它**不贡献**其他 system-prompt 文本 —— local agent 的 prompt 里**没有**插值进去的"可用工具列表"。

- **Memory 注入（`memory` frontmatter）：** 若 `isAutoMemoryEnabled()` **且** `memory` 被设置，`getSystemPrompt` 会在**块 0 内部**追加 `'\n\n' + loadAgentMemoryPrompt(agentType, scope)`（[loadAgentsDir.ts:727-729](../src/tools/AgentTool/loadAgentsDir.ts#L727-L729)）。该 prompt（由 `buildMemoryPrompt` 构造，带一段 scope 专属说明，[agentMemory.ts:138-177](../src/tools/AgentTool/agentMemory.ts#L138-L177)）内嵌 agent 的 `MEMORY.md` 内容与 memory 目录路径。所以 memory 落在 **system prompt** 里（追加到 body 之后），不是 user message。当 auto-memory 关闭或 `memory` 未设置时，块 0 就只是 body。（`memory` 还会把 Read/Write/Edit 注入工具池 —— [loadAgentsDir.ts:662-674](../src/tools/AgentTool/loadAgentsDir.ts#L662-L674) —— 但那是工具，不是 prompt 文本。）

---

## 关键源码引用

- Local agent 解析 + `getSystemPrompt` 闭包：[loadAgentsDir.ts:541, 713, 726-732](../src/tools/AgentTool/loadAgentsDir.ts#L541)
- `source` 打标（local vs 其他）：[markdownConfigLoader.ts:343, 353, 366](../src/utils/markdownConfigLoader.ts#L343)；类型守卫 [loadAgentsDir.ts:168-184](../src/tools/AgentTool/loadAgentsDir.ts#L168-L184)
- 正常路径构造点：[AgentTool.tsx:518, 534, 622-626](../src/tools/AgentTool/AgentTool.tsx#L518)
- 备用构造点（cwd/worktree override）：[runAgent.ts:508-516, 906-932](../src/tools/AgentTool/runAgent.ts#L508-L516)
- `enhanceSystemPromptWithEnvDetails` + `notes` + `DEFAULT_AGENT_PROMPT`：[prompts.ts:758, 760-791](../src/constants/prompts.ts#L758)
- `computeEnvInfo` 的 `<env>` 块：[prompts.ts:606-649](../src/constants/prompts.ts#L606-L649)；shell/OS 辅助 [prompts.ts:732-756](../src/constants/prompts.ts#L732-L756)；knowledge cutoff [prompts.ts:713-730](../src/constants/prompts.ts#L713-L730)
- query 层 append/prepend：[query.ts:449-451, 659-661](../src/query.ts#L449-L451)；[api.ts:437-474](../src/utils/api.ts#L437-L474)
- systemContext（gitStatus）/ userContext（claudeMd）：[context.ts:60-103, 116-149, 155-189](../src/context.ts#L60-L103)
- built-in 对比：[generalPurposeAgent.ts:3, 20, 25-34](../src/tools/AgentTool/built-in/generalPurposeAgent.ts#L3)
- Memory prompt：[agentMemory.ts:138-177](../src/tools/AgentTool/agentMemory.ts#L138-L177)

---

*文档基于 `claude-code-main` 参考实现，核心论断已对照源码核实。*
