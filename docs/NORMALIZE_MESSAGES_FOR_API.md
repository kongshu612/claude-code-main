# `normalizeMessagesForAPI` 深度解析

> 文件位置：`src/utils/messages.ts`
> 函数签名：`normalizeMessagesForAPI(messages: Message[], tools: Tools = []): (UserMessage | AssistantMessage)[]`

---

## 这个函数在做什么？

Claude Code 内部维护着一条完整的"消息历史"，这条历史里不只有发给 API 的对话内容，还混合了很多"内务"数据：

- REPL 展示用的虚拟消息
- 工具执行时的进度汇报消息
- 记录"上次 API 报错是因为图片太大"的合成错误消息
- Tool Search beta 用的懒加载占位符块
- 因 Bedrock 多条 user 消息限制而需要合并的连续消息

这些数据对 UI 有用，对内部状态管理有用，但**全部发给 Anthropic API 就会出错**。

`normalizeMessagesForAPI` 的职责就是：**每次发 API 请求前，从内部消息历史里"蒸馏"出一份干净的、符合 API 规范的消息列表**，而不修改内部状态本身。这是一个纯变换函数，每次调用都从头重算。

---

## 函数结构总览

```
输入：Message[]（内部格式，含所有类型）
  │
  ├─ Step 1  reorderAttachmentsForAPI + 过滤虚拟消息
  ├─ Step 2  构建 stripTargets（附件错误恢复映射）
  ├─ Step 3  类型过滤（丢弃 progress、合成错误等）
  ├─ Step 4  逐条处理（类型转换 + 内容清洗 + 合并）
  └─ Step 5  后处理管道（多个单职责 pass）
  │
输出：(UserMessage | AssistantMessage)[]（API 合规格式）
```

---

## 各步骤目的与实现

### Step 1 — 附件重排 + 虚拟消息清除

```typescript
const reorderedMessages = reorderAttachmentsForAPI(messages).filter(
  m => !((m.type === 'user' || m.type === 'assistant') && m.isVirtual),
)
```

**目的：**

用户在 REPL 里拖入文件时，attachment 消息在内部历史里的位置是"拖入时刻"——可能出现在一段普通 user 消息之后，也可能出现在工具结果之后。但 API 要求附件必须紧跟在它所属轮次的 `assistant` 消息或 `tool_result` 消息**之后**（而不是飘在普通 user 消息之间）。

`reorderAttachmentsForAPI` 使用**从底部向上扫描**的算法：

1. 遇到 `attachment` → 暂存到 `pendingAttachments` 缓冲区
2. 遇到**停止点**（`assistant` 消息，或首块为 `tool_result` 的 `user` 消息）→ 把缓冲区中的 attachment 全部放到该停止点**之后**，清空缓冲区，继续向上扫
3. 遇到普通消息 → 直接写入结果（**不触发释放**，缓冲区继续积累）
4. 扫完若仍有剩余 → 全部移到历史顶部
5. 最后 `result.reverse()`（构建全程反向，最后翻转回正向）

> ⚠️ **常见误解**：attachment 是移到停止点**之后**（after），不是"之前"（before）。附件是某个 assistant 回复或工具调用的"尾随内容"，应紧随其后，而非跑到前面去。

`isVirtual` 消息是 REPL 为了展示子 agent 调用而插入的"幽灵消息"，它们从不应该出现在任何 API 请求里，在此一并过滤。

**重排场景示例：**

```
原始历史：
M1: AssistantMessage — "好的，我来看看"
M2: UserMessage      — "先说一下这张图的背景"   ← 用户先打字
M3: AttachmentMessage — image_block(cat.png)    ← 后来才拖文件

算法（从底部向上）：
  i=2: M3（attachment）→ pending: [M3]
  i=1: M2（普通 user）→ 非停止点，直接写入；pending 不清空
  i=0: M1（assistant，停止点）→ 先写 pending 中的 M3，再写 M1，清空 pending

result 反向 = [M2, M3, M1] → reverse → [M1, M3, M2]

重排后：
M1: AssistantMessage — "好的，我来看看"
M3: AttachmentMessage — image_block(cat.png)   ← 上浮到 M1（assistant）之后
M2: UserMessage      — "先说一下这张图的背景"
```

附件从"M2 之后"被上浮到了"M1（assistant）之后"，保证 API 看到的顺序合法。

---

### Step 2 — 构建 `stripTargets`（附件错误自愈）

```typescript
const errorToBlockTypes: Record<string, Set<string>> = {
  [getPdfTooLargeErrorMessage()]:        new Set(['document']),
  [getPdfPasswordProtectedErrorMessage()]: new Set(['document']),
  [getImageTooLargeErrorMessage()]:      new Set(['image']),
  [getRequestTooLargeErrorMessage()]:    new Set(['document', 'image']),
}

const stripTargets = new Map<string, Set<string>>()
// 向前倒查，把 isMeta user 消息的 uuid 记入 stripTargets
```

**目的：**

当用户上传了一张过大的图片，API 返回 `IMAGE_TOO_LARGE` 错误。Claude Code 把这个错误以"合成错误消息"（`isSyntheticApiError=true`）的形式存入对话历史，然后继续对话。

问题来了：如果不做处理，**下一次发 API 请求时仍然会带上那张大图**，导致每次都报同一个错误，对话永远卡死。

Step 2 解决这个问题：找到合成错误消息，向前倒查到发送了问题附件的那条 `isMeta` user 消息，把它的 uuid 和需要剥除的块类型记入 `stripTargets`。Step 4 处理该消息时会根据这个映射把对应块删掉，避免重复触发错误。

---

### Step 3 — 类型过滤

```typescript
.filter(_ => {
  if (_.type === 'progress') return false
  if (_.type === 'system' && !isSystemLocalCommandMessage(_)) return false
  if (isSyntheticApiErrorMessage(_)) return false
  return true
})
```

**目的：**

三类消息永远不应该发给 API：

| 类型 | 原因 |
|------|------|
| `progress` | 工具执行中的进度汇报，仅供 UI 展示 |
| `system`（非本地命令） | 来自远端的系统提示展示，API 层用 `system` 参数处理，不走消息列表 |
| `isSyntheticApiError` | 内部错误记录消息，Step 2 已经利用过它，现在清除 |

---

### Step 4 — 逐条处理（核心循环）

这一步对每类消息做不同处理：

#### `system`（本地命令输出）

```typescript
case 'system': {
  // 转换为 user 类型消息，让模型能在后续 turn 引用命令输出
  const userMsg = createUserMessage({ content: message.content, ... })
  if (lastMessage?.type === 'user') {
    result[result.length - 1] = mergeUserMessages(lastMessage, userMsg)
  } else {
    result.push(userMsg)
  }
}
```

`/clear`、`/bash` 等本地命令的输出存为 `system` 类型消息。API 没有 `system` 这个 role，需要转换成 `user`。转换后如果前一条也是 `user`，立即合并（见下文 Bedrock 合并逻辑）。

#### `user`（主要处理分支）

**① tool_reference 块清洗**

```typescript
if (!isToolSearchEnabledOptimistic()) {
  normalizedMessage = stripToolReferenceBlocksFromUserMessage(message)
} else {
  normalizedMessage = stripUnavailableToolReferencesFromUserMessage(
    message, availableToolNames
  )
}
```

`tool_reference` 块是 Tool Search beta 的懒加载占位符（仅发工具名，不发完整 schema）。如果没有启用 Tool Search，这种块对 API 是非法字段，必须剥除。如果启用了，也要剥除那些"引用的工具已经不存在了"的引用（例如 MCP server 已断开）。

**② 附件块剥除（使用 Step 2 的 stripTargets）**

```typescript
const typesToStrip = stripTargets.get(normalizedMessage.uuid)
if (typesToStrip && normalizedMessage.isMeta) {
  const filtered = content.filter(block => !typesToStrip.has(block.type))
  if (filtered.length === 0) return  // 全部剥除后跳过这条消息
  normalizedMessage = { ...normalizedMessage, message: { content: filtered } }
}
```

把曾经触发 PDF/图片过大错误的块从消息里删掉。如果删完后消息内容为空，直接跳过（不发给 API），比发一条空消息更安全。

**③ TURN_BOUNDARY 文本块注入**

```typescript
if (contentHasToolReference(contentAfterStrip) && !alreadyHasBoundary) {
  normalizedMessage = {
    ...normalizedMessage,
    message: {
      content: [...contentAfterStrip, { type: 'text', text: TOOL_REFERENCE_TURN_BOUNDARY }]
    }
  }
}
```

当 `tool_reference` 内容（工具 schema 的展开 XML）出现在 prompt 末尾时，capybara 系列模型有约 10% 的概率把它误判为终止符，直接采样出停止序列。注入一个文本块作为"分隔符"，强制在 tool schema 后创建一个干净的 `\n\nHuman:` 边界，消除歧义。

注意：这个文本块**不存入内部消息历史**（不修改 REPL 状态），只在每次 API 调用前临时注入。

**④ 连续 user 消息合并（Bedrock 兼容）**

```typescript
const lastMessage = last(result)
if (lastMessage?.type === 'user') {
  result[result.length - 1] = mergeUserMessages(lastMessage, normalizedMessage)
} else {
  result.push(normalizedMessage)
}
```

Bedrock（AWS 托管的 Claude）**不允许两条连续的 `user` 消息**，否则直接报错。原生 Anthropic API 对此宽容一些（服务端会自动合并），但统一合并更安全。`mergeUserMessages` 把两条消息的 `content` 数组拼接成一条。

#### `assistant`

```typescript
case 'assistant': {
  // 1. 标准化 tool_use 输入（去掉内部字段如 plan、caller）
  // 2. 把 tool_use 名称映射到规范名（处理 MCP 工具的 server::name 格式）
  // 3. 向前找同 message.id 的 assistant 消息，合并流式分片
}
```

assistant 消息主要做两件事：清洗不能发给 API 的内部字段（`ExitPlanMode` 的 `plan` 字段、Tool Search 的 `caller` 字段），以及合并流式响应产生的多个分片消息（同一个 `message.id` 的 assistant 消息可能分批到达，内部分别存储，发 API 前需合并成一条）。

---

### Step 5 — 后处理管道

Step 4 的 forEach 完成后，还有一系列单职责的清洗 pass：

| Pass | 目的 |
|------|------|
| `relocateToolReferenceSiblings` | 把 TURN_BOUNDARY 文本块从 `tool_reference` 消息挪到后面的非引用消息，避免产生"连续两条 human 消息"的模式（模型会学习在工具结果后发出停止序列） |
| `filterOrphanedThinkingOnlyMessages` | 删除只含 `thinking` 块的孤立 assistant 消息（compaction 切片后可能出现），否则 API 400 |
| `filterTrailingThinkingFromLastAssistant` | 删除最后一条 assistant 消息末尾的 `thinking` 块（流式响应被截断时残留），否则 API 400 |
| `filterWhitespaceOnlyAssistantMessages` | 删除内容全为空白的 assistant 消息 |
| `ensureNonEmptyAssistantContent` | 如果 assistant 消息经过上述过滤后 content 为空，注入 `{type:'text', text:''}` 防止 API 报"空 content" |
| `smooshSystemReminderSiblings` | 把 `<system-reminder>` 包裹的文本块融合进相邻 `tool_result` 的 content 字段，结构更紧凑 |
| `sanitizeErrorToolResultContent` | 如果 `is_error=true` 的 `tool_result` 里有 image 块，剥除（处理历史 transcript 兼容性） |
| `appendMessageTagToUserMessage` | 在每条 user 消息末尾追加 `\n[id:xxxx]`（SnipTool 功能，让模型能引用消息 ID 截断历史） |
| `validateImagesForAPI` | 最终校验图片尺寸，超限抛异常（fail fast，比等 API 报错更快定位） |

---

## 完整例子演示

### 初始内部消息历史

下面 7 条消息是 Claude Code 内部存储的原始状态：

```
M1: UserMessage(uuid='A', isVirtual=true)
    ← REPL 子 agent 调用的展示幽灵消息

M2: UserMessage(uuid='B', isMeta=true)
    content: [image_block(一张 50MB 的图), text('请分析这张图')]
    ← 用户上传的附件消息

M3: UserMessage(uuid='E', isSyntheticApiError=true)
    content: [text('<IMAGE_TOO_LARGE_ERROR>')]
    ← 发送 M2 时 API 返回图片过大错误，Claude Code 将其存为合成错误消息

M4: AssistantMessage(uuid='C')
    content: [tool_use(id='toolu_01', name='Bash', input={command:'ls -la'})]
    ← 模型决定调用 Bash 工具

M5: UserMessage(uuid='D', type='progress')
    content: [text('Bash 执行中... (PID 1234)')]
    ← 工具执行进度，仅 UI 展示用

M6: UserMessage(uuid='F')
    content: [tool_result(tool_use_id='toolu_01', content='file_a.txt\nfile_b.txt')]
    ← toolu_01 的执行结果

M7: UserMessage(uuid='G')
    content: [tool_result(tool_use_id='toolu_02', content='hello world')]
    ← toolu_02 的执行结果（并发工具调用的另一个结果）
```

---

### Step 1 执行后

`M1`（`isVirtual=true`）被过滤：

```
reorderedMessages = [M2, M3, M4, M5, M6, M7]
```

---

### Step 2 执行后

扫描到 `M3`（`isSyntheticApiError=true`，内容匹配 `IMAGE_TOO_LARGE`）。

向前倒查：`M2`（`type=user`, `isMeta=true`）→ 命中。

```
stripTargets = {
  'B' (M2的uuid) → Set(['image'])
}
```

**含义**：下次处理 M2 时，把它的 `image` 块全部剥除。

---

### Step 3 执行后

丢弃：
- `M3`（`isSyntheticApiError`）
- `M5`（`type='progress'`）

```
待处理列表 = [M2, M4, M6, M7]
```

---

### Step 4 执行后（forEach 循环）

**处理 M2**：
- `stripTargets.get('B')` = `Set(['image'])` → 删除 image_block
- content 剩余：`[text('请分析这张图')]`
- result 为空 → push

```
result = [
  UserMessage(uuid='B', content=[text('请分析这张图')])
  ← 图片块已被剥除，不会再触发 IMAGE_TOO_LARGE
]
```

**处理 M4**（AssistantMessage）：
- 标准化 tool_use 字段
- 上一条是 user → 不合并，push

```
result = [
  UserMessage(uuid='B'),
  AssistantMessage(uuid='C', content=[tool_use(toulu_01, 'ls -la')])
]
```

**处理 M6**（tool_result for toolu_01）：
- 无 tool_reference，无 stripTargets 命中
- 上一条是 AssistantMessage → 不合并，push

```
result = [
  UserMessage(uuid='B'),
  AssistantMessage(uuid='C'),
  UserMessage(uuid='F', content=[tool_result(toolu_01)])
]
```

**处理 M7**（tool_result for toolu_02）：
- 无需清洗
- 上一条是 `UserMessage(F)` → **合并！**

```typescript
mergeUserMessages(M6, M7)
// 结果：{ content: [...M6.content, ...M7.content] }
```

```
result = [
  UserMessage(uuid='B', content=[text('请分析这张图')]),
  AssistantMessage(uuid='C', content=[tool_use(toulu_01, 'ls -la')]),
  UserMessage(uuid='F', content=[
    tool_result(toulu_01, 'file_a.txt\nfile_b.txt'),
    tool_result(toulu_02, 'hello world')
    ← 两条独立 UserMessage 合并为一条，满足 Bedrock 要求
  ])
]
```

---

### 最终 API 请求体（3 条消息）

```json
[
  {
    "role": "user",
    "content": [
      { "type": "text", "text": "请分析这张图" }
    ]
  },
  {
    "role": "assistant",
    "content": [
      {
        "type": "tool_use",
        "id": "toolu_01",
        "name": "Bash",
        "input": { "command": "ls -la" }
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_01",
        "content": "file_a.txt\nfile_b.txt"
      },
      {
        "type": "tool_result",
        "tool_use_id": "toolu_02",
        "content": "hello world"
      }
    ]
  }
]
```

**对比原始 7 条 → 最终 3 条：**

| 原始消息 | 结果 | 原因 |
|----------|------|------|
| M1 虚拟消息 | 删除 | `isVirtual=true` |
| M2 含图片 | 保留但剥除图片 | `stripTargets` 命中，图片曾触发错误 |
| M3 合成错误 | 删除 | `isSyntheticApiError=true` |
| M4 assistant | 保留 | 正常处理 |
| M5 进度消息 | 删除 | `type='progress'` |
| M6 tool_result | 保留，与 M7 合并 | Bedrock 连续 user 合并 |
| M7 tool_result | 合并入 M6 | Bedrock 连续 user 合并 |

---

## 为什么每次都重新计算？

`normalizeMessagesForAPI` 是一个**无副作用的纯变换函数**，每次 API 调用前都从内部消息历史完整重算，而不是增量更新。

这个设计的核心好处：

1. **`stripTargets` 自动生效**：不需要"记住"哪条消息的图片曾经报错。只要合成错误消息还在历史里，每次重算都会正确识别并剥除。

2. **工具列表变化自动反映**：如果某个 MCP 工具断开了，`availableToolNames` 在下次调用时就不包含它，`tool_reference` 引用自动被剥除，不需要回头修改历史消息。

3. **兼容 compaction 和 session 恢复**：历史消息可能来自磁盘加载（旧格式），也可能来自 compaction 压缩。只要内部格式正确，归一化函数就能正确处理，不依赖"这次 session 做过什么操作"的运行时状态。

---

## 相关文件索引

| 文件 | 关联点 |
|------|--------|
| `src/utils/messages.ts` | 本函数实现，以及 `mergeUserMessages`、`hoistToolResults`、`smooshSystemReminderSiblings` 等辅助函数 |
| `src/query.ts` line 1396 | 调用点：每条工具结果更新时调用 |
| `src/services/api/claude.ts` lines 1100-1244 | 调用点：每次 API 请求前调用，传入完整历史 |
| `src/utils/telemetry/betaSessionTracing.ts` | `formatMessagesForContext` 对 `<system-reminder>` 的解析（用于 OTel tracing，不影响 API 消息内容） |
| `src/services/tools/StreamingToolExecutor.ts` | 工具结果生产者，产出的 `UserMessage` 最终经过本函数清洗 |

---

## 消息类型全览

> 类型定义来源：`src/types/message.ts`（构建时生成）；创建工厂函数集中在 `src/utils/messages.ts`。
>
> **图例**
> - ✅ 发给 API — 经过 `normalizeMessagesForAPI` 后会出现在 API 请求体里
> - ❌ 不发给 API — 在 `normalizeMessagesForAPI` 的某个 Step 中被丢弃或转换
> - 🔄 转换后发 API — 被转换成其他类型再发出

---

### 一、`UserMessage` — `type: 'user'`  ✅

最核心的消息类型，承载用户输入和工具结果。

```typescript
{
  type: 'user',
  uuid: UUID,
  timestamp: string,           // ISO 8601
  message: {
    role: 'user',
    content: string | ContentBlockParam[]
    // content 块类型：text / image / document / tool_result / tool_reference
  },

  // --- 内部标志位（不发给 API，被 normalizeMessagesForAPI 处理） ---
  isMeta?: true,               // 附件/工具调用输入，隐藏于 REPL UI 主视图
  isVirtual?: true,            // REPL 展示子 agent 的"幽灵消息"，Step 1 过滤
  isCompactSummary?: true,     // compact 后的摘要替代消息
  isVisibleInTranscriptOnly?: true, // 仅 transcript 可见，REPL 不展示

  // --- 工具结果专用 ---
  toolUseResult?: unknown,     // 原始工具返回值（供 SDK 消费者读取）
  mcpMeta?: { _meta?; structuredContent? }, // MCP 协议元数据，永不发给模型
  sourceToolAssistantUUID?: UUID, // 匹配的 tool_use 所在 assistant 消息的 UUID

  // --- 错误自愈 ---
  // （没有专属字段；合成错误消息是单独的 isSyntheticApiError 消息，见下）

  // --- 其他 ---
  imagePasteIds?: number[],    // 粘贴图片的 ID（UI 用）
  permissionMode?: PermissionMode, // 发送时的权限模式（rewind 恢复用）
  origin?: MessageOrigin,      // 来源：undefined=键盘输入, 'injected'=注入等
  summarizeMetadata?: {
    messagesSummarized: number
    userContext?: string
    direction?: PartialCompactDirection
  },
}
```

**常见实例：**

| 场景 | content 块类型 | 关键标志 |
|------|---------------|---------|
| 用户打字输入 | `[{type:'text'}]` | 无 |
| 上传图片/文件 | `[{type:'image'/'document'}]` | `isMeta:true` |
| 工具执行结果 | `[{type:'tool_result'}]` | `sourceToolAssistantUUID` |
| Skill listing | `[{type:'text'}]` + `<system-reminder>` | `isMeta:true` |
| REPL 子 agent 展示 | 任意 | `isVirtual:true` → Step 1 删除 |
| compact 摘要 | `[{type:'text'}]` | `isCompactSummary:true` |

---

### 二、`AssistantMessage` — `type: 'assistant'`  ✅

模型的回复消息，一对一对应 Anthropic API 的 `MessageResponse`。

```typescript
{
  type: 'assistant',
  uuid: UUID,
  timestamp: string,
  message: {
    id: string,           // API 返回的消息 ID（流式分片靠它合并）
    role: 'assistant',
    model: string,        // 实际推理模型名
    stop_reason: string,  // 'end_turn' | 'tool_use' | 'stop_sequence' | ...
    stop_sequence: string | null,
    type: 'message',
    usage: Usage,         // token 计数（含 cache_creation/cache_read）
    content: ContentBlock[], // text / thinking / tool_use / ...
    container: null,
    context_management: null,
  },
  requestId?: string,     // API 请求 ID（用于错误追踪）

  // --- 错误状态 ---
  isApiErrorMessage?: boolean,    // 是否是 API 错误占位消息
  apiError?: { type; status; message; ... }, // API 级错误
  error?: SDKAssistantMessageError,          // SDK 级错误
  errorDetails?: string,

  isVirtual?: true,       // 子 agent 展示用幽灵消息，Step 1 过滤
}
```

**注意：** 流式响应会产生多个同 `message.id` 的 AssistantMessage（分片），Step 4 的 `assistant` 分支会将它们合并成一条。

`content` 中的块类型：

| 块类型 | 含义 | normalizeMessagesForAPI 处理 |
|--------|------|--------------------------|
| `text` | 纯文本回复 | 直接保留 |
| `thinking` | 扩展思考内容（Claude 3.7+） | 孤立 thinking 消息、末尾残留 thinking 被后处理 Pass 删除 |
| `tool_use` | 工具调用请求 | `input` 中的内部字段（`plan`、`caller`）被清除；工具名被映射到规范名 |
| `redacted_thinking` | 被服务端加密的思考块 | 直接保留 |

---

### 三、`ProgressMessage` — `type: 'progress'`  ❌

工具执行期间的中间状态汇报，**仅供 REPL UI 展示进度，永不发给 API**。

```typescript
{
  type: 'progress',
  uuid: UUID,
  timestamp: string,
  toolUseID: string,       // 对应的 tool_use 块 ID
  parentToolUseID: string, // 父工具（Agent 调用链中的上层工具）
  data: Progress,          // 泛型，实际类型由具体工具决定
                           // 例如 BashProgress: { pid; command; output; ... }
}
```

**生命周期：** 工具开始执行 → 产出若干 ProgressMessage → 工具完成 → 产出 UserMessage(tool_result)。ProgressMessage 在 Step 3 中被整批过滤，不进入 API 请求。

---

### 四、`AttachmentMessage` — `type: 'attachment'`  🔄

用户拖入或粘贴的附件（图片、文件、hook 输出等）。**不直接发给 API**，而是在 Step 1 的 `reorderAttachmentsForAPI` 中被重新插入到正确位置，最终作为 `UserMessage` 的 content 块的一部分发出。

```typescript
{
  type: 'attachment',
  uuid: UUID,
  timestamp: string,
  attachment: Attachment,  // 泛型，实际类型如下：
}
```

`Attachment` 的主要子类型：

| 子类型 | 描述 |
|--------|------|
| `HookAttachment` | pre/post hook 产出的结果块 |
| `HookPermissionDecisionAttachment` | hook 权限决策记录 |
| `skill_listing` | skill 列表注入（由 `getSkillListingAttachments()` 产出） |
| `image`/`document` | 用户粘贴的图片或文档文件 |

---

### 五、`SystemMessage` — `type: 'system'`  ❌ / 🔄

内部状态机消息，通过 `subtype` 区分具体用途。**绝大多数不发给 API**，只有 `subtype: 'local_command'` 会在 Step 4 被转换成 `UserMessage` 后发出。

以下是所有 subtype 一览：

#### 5a. `informational` — 普通通知  ❌

```typescript
{ type: 'system', subtype: 'informational',
  content: string,      // 显示内容
  level: 'info' | 'warn' | 'error',
  toolUseID?: string,   // 关联工具
  preventContinuation?: boolean,
  isMeta: false }
```

**用途：** 展示给用户看的一般性信息（工具权限提示、断线通知等）。不参与 LLM 推理。

**例子：** `"Allowed tools: Bash, FileRead"` / `"Network error, retrying..."`

---

#### 5b. `local_command` — 本地命令输出  🔄

```typescript
{ type: 'system', subtype: 'local_command',
  content: string,      // /bash 等本地命令的执行输出
  level: 'info',
  isMeta: false }
```

**用途：** `/bash`、`/clear` 等斜杠命令的输出结果。**Step 4 中唯一被转换成 `UserMessage` 发给 API 的 system 消息**。转换后如果前一条也是 user，会被合并（Bedrock 兼容）。

---

#### 5c. `api_error` — API 错误重试通知  ❌

```typescript
{ type: 'system', subtype: 'api_error',
  level: 'error',
  error: APIError,
  retryInMs: number,
  retryAttempt: number,
  maxRetries: number }
```

**用途：** API 调用失败时在 UI 展示的错误信息（含重试倒计时）。多条连续的 api_error 消息只保留最后一条（避免刷屏）。

---

#### 5d. `compact_boundary` — compact 边界标记  ❌

```typescript
{ type: 'system', subtype: 'compact_boundary',
  content: 'Conversation compacted',
  level: 'info',
  compactMetadata: {
    trigger: 'manual' | 'auto',
    preTokens: number,
    userContext?: string,
    messagesSummarized?: number,
  },
  logicalParentUuid?: UUID, // compact 前最后一条消息的 UUID
  isMeta: false }
```

**用途：** 标记 compact 压缩的分界点。用于 UI 展示"已压缩"分割线，以及 session 恢复时定位压缩位置。

---

#### 5e. `microcompact_boundary` — 微压缩边界标记  ❌

```typescript
{ type: 'system', subtype: 'microcompact_boundary',
  content: 'Context microcompacted',
  level: 'info',
  microcompactMetadata: {
    trigger: 'auto',
    preTokens: number,
    tokensSaved: number,
    compactedToolIds: string[],       // 被压缩的 tool_use ID 列表
    clearedAttachmentUUIDs: string[], // 被清除的附件 UUID
  },
  isMeta: false }
```

**用途：** 比 compact 更轻量的局部压缩，只清除工具结果和附件，不重写对话历史。

---

#### 5f. `permission_retry` — 权限重试  ❌

```typescript
{ type: 'system', subtype: 'permission_retry',
  content: string,      // "Allowed bash, vim"
  commands: string[],   // 被批准的命令列表
  level: 'info',
  isMeta: false }
```

**用途：** 用户在权限提示中批准工具后，在 UI 显示"已批准 X, Y"的确认消息。

---

#### 5g. `turn_duration` — 耗时统计  ❌

```typescript
{ type: 'system', subtype: 'turn_duration',
  durationMs: number,
  budgetTokens?: number,
  budgetLimit?: number,
  budgetNudges?: number,
  messageCount?: number,
  isMeta: false }
```

**用途：** 在每个 turn 结束后显示耗时与 token 预算使用情况。纯 UI 统计，不参与推理。

---

#### 5h. `api_metrics` — API 性能指标  ❌

```typescript
{ type: 'system', subtype: 'api_metrics',
  ttftMs: number,          // Time to first token（毫秒）
  otps: number,            // Output tokens per second
  isP50?: boolean,
  hookDurationMs?: number,
  turnDurationMs?: number,
  toolDurationMs?: number,
  classifierDurationMs?: number,
  toolCount?: number,
  hookCount?: number,
  classifierCount?: number,
  configWriteCount?: number,
  isMeta: false }
```

**用途：** 在 REPL 底部显示的性能指标（TTFT、TPS 等）。

---

#### 5i. `stop_hook_summary` — stop hook 执行摘要  ❌

```typescript
{ type: 'system', subtype: 'stop_hook_summary',
  hookCount: number,
  hookInfos: StopHookInfo[],
  hookErrors: string[],
  preventedContinuation: boolean,  // hook 是否阻止了 LLM 继续
  stopReason?: string,
  hasOutput: boolean,
  level: SystemMessageLevel,
  toolUseID?: string,
  hookLabel?: string,
  totalDurationMs?: number }
```

**用途：** 每次 LLM 停止时 stop hook 执行结果的摘要。如果 `preventedContinuation=true`，会触发重新推理。

---

#### 5j. `away_summary` — 离开摘要  ❌

```typescript
{ type: 'system', subtype: 'away_summary',
  content: string,   // 离开期间发生的事情摘要
  isMeta: false }
```

**用途：** 用户离开一段时间后后台 agent 继续工作，回来后展示的进展摘要。

---

#### 5k. `memory_saved` — 记忆文件写入通知  ❌

```typescript
{ type: 'system', subtype: 'memory_saved',
  writtenPaths: string[], // 写入的记忆文件路径列表
  isMeta: false }
```

**用途：** 自动记忆功能写入 `~/.claude/memory/` 后的通知消息。

---

#### 5l. `agents_killed` — 子 agent 被终止  ❌

```typescript
{ type: 'system', subtype: 'agents_killed',
  isMeta: false }
```

**用途：** 用户中断（Ctrl+C）导致正在运行的子 agent 被杀死时的通知。

---

#### 5m. `bridge_status` — 远程控制连接状态  ❌

```typescript
{ type: 'system', subtype: 'bridge_status',
  content: string,       // "/remote-control is active at http://..."
  url: string,
  upgradeNudge?: string,
  isMeta: false }
```

**用途：** Claude Code 被 IDE 扩展（VSCode/JetBrains）通过 `/remote-control` 接管时的状态提示。

---

#### 5n. `scheduled_task_fire` — 定时任务触发  ❌

```typescript
{ type: 'system', subtype: 'scheduled_task_fire',
  content: string,   // "Scheduled task 'xxx' triggered"
  isMeta: false }
```

**用途：** AGENT_TRIGGERS 功能的定时任务（Cron）触发时的通知。

---

### 六、`TombstoneMessage` — `type: 'tombstone'`  ❌

一种特殊的"反消息"——它不是添加消息，而是**删除**一条已有的消息。

```typescript
{
  type: 'tombstone',
  message: Message,   // 要被删除的消息
}
```

**用途：** 主要用于 Agent 工具的孤儿消息清理。当 agent 中途退出时，已经渲染到 UI 的流式中间消息需要从历史里抹去，就通过 yield 一个 TombstoneMessage 来触发删除。

`normalizeMessagesForAPI` 中直接跳过（`tombstone` 不是内部历史里的持久消息）。

---

### 七、`ToolUseSummaryMessage` — `type: 'tool_use_summary'`  ❌

工具执行的汇总消息，仅 SDK 模式使用（不出现在 REPL 历史中）。

```typescript
{
  type: 'tool_use_summary',
  // ...内容结构为 SDK 内部格式
}
```

`normalizeMessagesForAPI` 中直接跳过。

---

### 快速查阅索引

| `type` | `subtype` | 发给 API？ | 主要用途 |
|--------|-----------|-----------|---------|
| `user` | — | ✅ | 用户输入、工具结果、skill 注入 |
| `assistant` | — | ✅ | LLM 回复 |
| `progress` | — | ❌ | 工具执行中间进度（UI 专用） |
| `attachment` | — | 🔄 reorder 后并入 user | 图片/文件/hook 输出 |
| `system` | `informational` | ❌ | 普通通知展示 |
| `system` | `local_command` | 🔄 转 UserMessage | `/bash` 等本地命令输出 |
| `system` | `api_error` | ❌ | API 错误重试通知 |
| `system` | `compact_boundary` | ❌ | compact 分界线标记 |
| `system` | `microcompact_boundary` | ❌ | 微压缩分界线标记 |
| `system` | `permission_retry` | ❌ | 权限批准确认 |
| `system` | `turn_duration` | ❌ | 耗时/Token 统计 |
| `system` | `api_metrics` | ❌ | TTFT/TPS 等性能指标 |
| `system` | `stop_hook_summary` | ❌ | Stop hook 执行摘要 |
| `system` | `away_summary` | ❌ | 离开期间进展摘要 |
| `system` | `memory_saved` | ❌ | 记忆文件写入通知 |
| `system` | `agents_killed` | ❌ | 子 agent 终止通知 |
| `system` | `bridge_status` | ❌ | 远程控制连接状态 |
| `system` | `scheduled_task_fire` | ❌ | 定时任务触发通知 |
| `tombstone` | — | ❌ | 删除某条已有消息（反消息） |
| `tool_use_summary` | — | ❌ | SDK 模式工具汇总（非 REPL）|
