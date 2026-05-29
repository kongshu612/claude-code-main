# Claude Code Tool 系统深度解析

本文以 `FileReadTool` 读取本地 `hello world` 文本文件为贯穿全文的具体例子，从**工具注册**和 **StreamingToolExecutor 执行**两个维度，系统讲解一个 Tool 必须具备哪些字段、这些字段如何被模型使用，以及工具结果如何反馈给模型。

---

## 目录

1. [Tool 类型概览](#1-tool-类型概览)
2. [必要字段逐一解析](#2-必要字段逐一解析)
3. [FileReadTool 注册示例](#3-filereadtool-注册示例)
4. [buildTool() 工厂与默认值](#4-buildtool-工厂与默认值)
5. [StreamingToolExecutor 执行流程](#5-streamingtoolexecutor-执行流程)
6. [mapToolResultToToolResultBlockParam — 结果反馈给模型](#6-maptoolresulttotoolresultblockparam--结果反馈给模型)
7. [完整端到端示例：读取 hello world.txt](#7-完整端到端示例读取-hello-worldtxt)

---

## 1. Tool 类型概览

所有工具都必须实现 `src/Tool.ts` 中定义的 `Tool<Input, Output, P>` 泛型接口：

```typescript
// src/Tool.ts（简化）
type Tool<Input = unknown, Output = unknown, P = unknown> = {
  // ── 静态元数据 ──────────────────────────────────────────
  name: string                    // 工具唯一标识，发送给模型
  inputSchema: ZodSchema<Input>   // 输入校验 schema，同时生成 JSON Schema
  maxResultSizeChars: number      // 输出截断阈值

  // ── 模型感知方法 ─────────────────────────────────────────
  description(): string           // 工具简介（一句话）
  prompt(): string                // 完整工具说明，注入系统 prompt
  toAutoClassifierInput(): string // 自动权限分类器使用的描述
  userFacingName(): string        // UI 显示名称（非模型可见）

  // ── 执行与权限 ───────────────────────────────────────────
  call(input: Input, ctx: ToolUseContext): AsyncGenerator<Output>
  checkPermissions(input: Input, ctx: ToolUseContext): Promise<PermissionCheckResult>
  isEnabled(ctx: ToolUseContext): boolean
  isConcurrencySafe(): boolean    // 是否可与其他工具并行执行
  isReadOnly(): boolean           // 是否只读（影响权限策略）
  isDestructive(): boolean        // 是否有破坏性（影响 UI 警告）

  // ── 结果序列化 ───────────────────────────────────────────
  mapToolResultToToolResultBlockParam(
    output: Output,
    ctx: ToolUseContext
  ): ToolResultBlockParam         // 转换为 Anthropic SDK 格式

  // ── UI 渲染 ──────────────────────────────────────────────
  renderToolUseMessage(input: Input, ctx: ToolUseContext): ReactNode
  renderToolResultMessage(output: Output, ctx: ToolUseContext): ReactNode
}
```

---

## 2. 必要字段逐一解析

### 2.1 `name` — 工具身份标识

```typescript
name: string  // 例：'Read'、'Bash'、'mcp__filesystem__read_file'
```

**用途**：
- 模型调用工具时在 `tool_use` 块中使用此名称：`{ "type": "tool_use", "name": "Read", "id": "...", "input": {...} }`
- `filterToolsByDenyRules()` 和 `toolMatchesRule()` 用此字段匹配拒绝规则
- MCP 工具名称格式为 `mcp__{serverName}__{toolName}`

### 2.2 `inputSchema` — 输入类型定义

```typescript
inputSchema: ZodSchema<Input>
```

**用途**：
- 通过 `zodToJsonSchema(tool.inputSchema)` 生成 JSON Schema，随工具列表发送给模型
- 模型据此生成合法的 `input` 对象，填入 `tool_use` 块
- 执行前用 `inputSchema.parse(rawInput)` 校验，避免注入攻击

**示例（FileReadTool）**：
```typescript
inputSchema: z.strictObject({
  file_path: z.string().describe('文件的绝对路径'),
  offset: z.number().int().optional().describe('从第几行开始读取（1-based）'),
  limit: z.number().int().optional().describe('最多读取多少行'),
  pages: z.string().optional().describe('PDF 页码范围，如 "1-5"'),
})
```

发送给模型的 JSON Schema 等效为：
```json
{
  "type": "object",
  "properties": {
    "file_path": { "type": "string" },
    "offset":    { "type": "integer" },
    "limit":     { "type": "integer" },
    "pages":     { "type": "string" }
  },
  "required": ["file_path"],
  "additionalProperties": false
}
```

### 2.3 `maxResultSizeChars` — 输出截断阈值

```typescript
maxResultSizeChars: number  // FileReadTool 设为 Infinity，Bash 设为 30_000
```

**用途**：结果超出此阈值时，调用链会截断并追加提示，防止单个工具结果撑满 context window。FileReadTool 因需要完整文件内容而设为 `Infinity`。

### 2.4 `description()` — 工具简介

```typescript
description(): string
```

**用途**：出现在工具列表的 `description` 字段，是模型决定是否调用该工具的第一依据。须简洁（一句话）、精准。

**FileReadTool 示例**：
```
"Reads a file from the local filesystem. ..."
```

### 2.5 `prompt()` — 详细使用说明

```typescript
prompt(): string
```

**用途**：注入系统 prompt（`<tools>` 段落），为模型提供：
- 详细功能描述
- 参数使用规范
- 最佳实践和注意事项
- 何时应/不应使用此工具

**FileReadTool prompt 片段**：
```
You can use this tool to read a file. For large files use offset and limit.
For images the tool returns the image content as base64 encoded data url.
For Jupyter notebooks (.ipynb) it returns all cells...
```

### 2.6 `call()` — 核心执行逻辑

```typescript
call(input: Input, ctx: ToolUseContext): AsyncGenerator<Output>
```

**用途**：工具的实际执行入口，返回**异步生成器**，可以 `yield` 多个值：
- 中间进度（类型通常为 `{ type: 'progress', ... }`）
- 最终结果（`yield` 最后一个值）

`ToolUseContext` 包含：
```typescript
type ToolUseContext = {
  abortController: AbortController   // 用于中断执行
  getAppState(): AppState            // 读取全局状态
  setAppState(state: AppState): void // 写入全局状态
  messages: Message[]                // 当前对话历史
  // ... 其他 UI/权限相关字段
}
```

### 2.7 `checkPermissions()` — 权限校验

```typescript
checkPermissions(input: Input, ctx: ToolUseContext): Promise<PermissionCheckResult>
// PermissionCheckResult = { behavior: 'allow'|'deny'|'ask', updatedInput?: Input, ... }
```

**用途**：内容级权限检查。当 Deny 规则包含 `ruleContent`（如 `"Bash(rm -rf)"`）时，此方法负责判断具体输入是否命中规则。返回 `'ask'` 时会弹出用户确认对话框。

### 2.8 `isConcurrencySafe()` — 并发安全标志

```typescript
isConcurrencySafe(): boolean
```

**用途**：由 `StreamingToolExecutor` 使用，决定工具是否可与其他工具并行执行：
- `true`（如 FileReadTool）：只读操作，可与任意只读工具并发
- `false`（如 Bash）：有副作用，必须串行执行

### 2.9 `mapToolResultToToolResultBlockParam()` — 结果序列化

```typescript
mapToolResultToToolResultBlockParam(
  output: Output,
  ctx: ToolUseContext
): ToolResultBlockParam
```

**用途**：将工具输出转换为 Anthropic API 的 `tool_result` 格式，作为下一轮对话的 `user` 消息发送给模型。详见第 6 节。

---

## 3. FileReadTool 注册示例

`src/tools/FileReadTool/FileReadTool.ts` 是 codebase 中最复杂的工具之一，完整展示了所有字段的最佳实践：

```typescript
// 节选关键字段
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,  // = 'Read'

  inputSchema: z.strictObject({
    file_path: z.string(),
    offset: z.number().int().optional(),
    limit: z.number().int().optional(),
    pages: z.string().optional(),
  }),

  maxResultSizeChars: Infinity,  // 文件内容不截断

  isConcurrencySafe: () => true,   // 只读，可并行
  isReadOnly: () => true,          // 影响权限策略
  isDestructive: () => false,      // 无破坏性

  description: () => 'Reads a file from the local filesystem...',

  prompt: () => `...详细使用说明...`,

  async *call(input, ctx) {
    // 1. 路径展开（处理 ~ 等）
    // 2. 类型检测（文本 / 图片 / PDF / Notebook）
    // 3. 去重检查（mtime 相同则返回 file_unchanged）
    // 4. yield 最终结果
    yield result
  },

  mapToolResultToToolResultBlockParam(output, ctx) {
    // 文本 → 带行号的字符串 + 安全提示
    // 图片 → base64 image block
    // file_unchanged → 占位存根
  },

  // 省略 validateInput、renderToolUseMessage 等
})
```

工具通过 `getAllBaseTools()` 注册到全局列表：

```typescript
// src/tools.ts
function getAllBaseTools(): BuiltTool[] {
  return [
    BashTool,
    FileReadTool,      // ← 在这里
    FileEditTool,
    // ...
  ]
}
```

---

## 4. buildTool() 工厂与默认值

并非所有字段都需要手动实现。`buildTool()` 为 7 个可选字段提供安全默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled:          () => true,         // 默认启用
  isConcurrencySafe:  () => false,        // 默认串行（保守）
  isReadOnly:         () => false,        // 默认非只读
  isDestructive:      () => false,        // 默认非破坏性
  checkPermissions:   (input, _ctx) =>    // 默认放行
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',        // 默认不参与自动分类
  userFacingName:     () => '',           // 默认用 name 字段
}

function buildTool<D extends ToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, ...def }
}
```

**FileReadTool 覆盖的默认值**：

| 字段 | 默认值 | FileReadTool 覆盖值 | 原因 |
|------|--------|---------------------|------|
| `isConcurrencySafe` | `false` | `true` | 只读，安全并发 |
| `isReadOnly` | `false` | `true` | 标记只读权限 |
| `checkPermissions` | 放行 | 自定义（Deny 规则检查） | 支持内容级规则 |

---

## 5. StreamingToolExecutor 执行流程

`src/services/tools/StreamingToolExecutor.ts` 管理工具调用的并发执行，是模型输出 → 工具结果的中间层。

### 5.1 内部状态机

每个工具调用由一个 `TrackedTool` 对象跟踪：

```typescript
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string                    // tool_use_id
  block: ToolUseBlock           // 模型返回的原始 tool_use 块
  assistantMessage: Message     // 所属 assistant 消息
  status: ToolStatus            // 当前状态
  isConcurrencySafe: boolean    // 缓存的并发标志
  promise?: Promise<void>       // 执行 Promise（executing 阶段）
  results?: ToolOutput[]        // 执行结果（completed 阶段）
  pendingProgress: ToolOutput[] // 待 yield 的进度消息
  contextModifiers?: ContextModifier[]  // 需要修改上下文的操作
}
```

### 5.2 工具进入执行器：addTool()

```typescript
executor.addTool(block, assistantMessage)
```

1. 创建 `TrackedTool`，状态为 `queued`
2. 调用 `processQueue()` 尝试立即执行

### 5.3 并发调度：processQueue()

```typescript
private processQueue(): void {
  for (const tool of this.queue) {
    if (tool.status !== 'queued') continue
    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      this.executeTool(tool)
    } else {
      // 非并发安全工具阻塞了队列 → 停止
      if (!tool.isConcurrencySafe) break
    }
  }
}
```

**并发规则** `canExecuteTool(isConcurrencySafe)`：
```typescript
// 可以执行当且仅当：
// (1) 当前无正在执行的工具，或
// (2) 新工具和所有正在执行的工具都是并发安全的
executingTools.length === 0 ||
(isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
```

**以 FileReadTool 为例**：
- FileReadTool A 正在执行（`isConcurrencySafe=true`）
- FileReadTool B 到来：`canExecuteTool(true)` → `true` ✅ 立即并发
- Bash 到来：`canExecuteTool(false)` → `false` ❌ 进队列等待

### 5.4 执行工具：executeTool()

```typescript
private async executeTool(tracked: TrackedTool): Promise<void> {
  tracked.status = 'executing'

  // 三层 AbortController 层级：
  // parent → siblingAbortController → toolAbortController
  const toolAbortController = new AbortController()

  tracked.promise = runToolUse(tracked.block, {
    abortController: toolAbortController,
    // ...其他 ctx 字段
  }).then(results => {
    tracked.results = results
    tracked.status = 'completed'
    if (wasBashError(results)) {
      // Bash 失败 → 中止所有兄弟工具
      this.siblingAbortController.abort()
    }
    this.processQueue()  // 唤醒队列
  })
}
```

**AbortController 三层层级**：

```
parent AbortController        ← 用户按 Ctrl+C 或全局中断
  └── siblingAbortController  ← Bash 工具报错时中断所有兄弟工具
        └── toolAbortController  ← 单个工具自身的中断控制
```

### 5.5 收集结果：getCompletedResults() + getRemainingResults()

调用方（REPL 主循环）通过两步收集结果：

#### 第一步：同步收集已完成结果

```typescript
for (const result of executor.getCompletedResults()) {
  // 处理 progress 消息和已完成工具的结果
}
```

`getCompletedResults()` 是**同步生成器**：
- 按队列顺序遍历所有工具
- 立即 yield `pendingProgress`（进度消息不等顺序）
- 对于 `completed` 工具，按到达顺序 yield 结果，然后设为 `yielded`
- 遇到 `executing` 状态的非并发安全工具 → `break`（后续串行工具必须等待）

#### 第二步：异步等待剩余结果

```typescript
for await (const result of executor.getRemainingResults()) {
  // 处理剩余的 progress 和结果
}
```

`getRemainingResults()` 是**异步生成器**，核心是：

```typescript
while (this.hasRemainingTools()) {
  await Promise.race([
    ...executingPromises,   // 等待任意工具完成
    progressPromise,        // 或等待任意进度消息
  ])
  yield* this.getCompletedResults()  // 收获本轮结果
}
```

`Promise.race` 确保有进度消息时立即响应，不必等待工具完全完成。

### 5.6 discard() — 流式降级保护

```typescript
executor.discard()
```

当上层决定放弃本次流式响应（如网络错误需要重试）时调用。设置 `this.discarded = true`，防止已执行工具的 `tool_use_id` 泄漏到下一次请求，导致 API 报"孤立的 tool_use_id"错误。

---

## 6. mapToolResultToToolResultBlockParam — 结果反馈给模型

工具执行完成后，结果需要序列化为 Anthropic API 格式，作为 `user` 消息中的 `tool_result` 块发送给模型。

```typescript
// Anthropic SDK 格式
type ToolResultBlockParam = {
  type: 'tool_result'
  tool_use_id: string
  content: string | ContentBlock[]
  is_error?: boolean
}
```

**FileReadTool 的实现**：

```typescript
mapToolResultToToolResultBlockParam(output, ctx) {
  if (output.type === 'text') {
    // 文本文件：返回带行号的内容 + 安全提示
    const lineNumbered = addLineNumbers(output.content, output.offset)
    return {
      type: 'tool_result',
      tool_use_id: ctx.toolUseId,
      content: lineNumbered + CYBER_RISK_MITIGATION_REMINDER,
    }
  }

  if (output.type === 'image') {
    // 图片：返回 base64 image block
    return {
      type: 'tool_result',
      tool_use_id: ctx.toolUseId,
      content: [{
        type: 'image',
        source: { type: 'base64', media_type: output.mediaType, data: output.data }
      }]
    }
  }

  if (output.type === 'file_unchanged') {
    // 文件未变化：返回轻量存根
    return {
      type: 'tool_result',
      tool_use_id: ctx.toolUseId,
      content: FILE_UNCHANGED_STUB,
    }
  }

  // PDF、notebook 等类似处理...
}
```

**`CYBER_RISK_MITIGATION_REMINDER`** 是附加在每次文本读取结果末尾的安全提示：

```
<system-reminder>
When providing file paths to other tools, use the path from this tool's output...
</system-reminder>
```

这个设计防止路径遍历攻击：模型看到的路径是工具实际访问的路径，而非用户可能伪造的路径。

---

## 7. 完整端到端示例：读取 hello world.txt

假设有文件 `/tmp/hello-world.txt`，内容为：

```
Hello, World!
This is a test file.
```

### 步骤 1：工具注册 — 模型感知工具能力

系统启动时，`assembleToolPool()` 将 `FileReadTool` 纳入工具池，并构建发送给模型的工具定义：

```json
{
  "name": "Read",
  "description": "Reads a file from the local filesystem...",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": { "type": "string" },
      "offset":    { "type": "integer" },
      "limit":     { "type": "integer" },
      "pages":     { "type": "string" }
    },
    "required": ["file_path"],
    "additionalProperties": false
  }
}
```

同时，`FileReadTool.prompt()` 的内容被注入系统 prompt，告知模型何时及如何使用此工具。

### 步骤 2：模型决策 — 生成 tool_use 块

模型收到用户指令"读取 /tmp/hello-world.txt"后，在 `assistant` 消息中返回：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "我来读取这个文件。"
    },
    {
      "type": "tool_use",
      "id": "toolu_01XYZ",
      "name": "Read",
      "input": {
        "file_path": "/tmp/hello-world.txt"
      }
    }
  ]
}
```

### 步骤 3：进入 StreamingToolExecutor

REPL 主循环解析 `assistant` 消息，将 `tool_use` 块交给执行器：

```typescript
// 伪代码
const executor = new StreamingToolExecutor(tools, ctx)
for (const block of assistantMessage.content) {
  if (block.type === 'tool_use') {
    executor.addTool(block, assistantMessage)
    // addTool 内部：
    //   1. 创建 TrackedTool { id: 'toolu_01XYZ', status: 'queued', isConcurrencySafe: true }
    //   2. 调用 processQueue()
  }
}
```

### 步骤 4：并发调度与执行

`processQueue()` 检查：
- 当前无正在执行的工具（`executingTools.length === 0`）
- → `canExecuteTool(true)` 返回 `true`
- → 调用 `executeTool(tracked)`

`executeTool()` 内部：
```
TrackedTool 状态：queued → executing
创建 toolAbortController
调用 FileReadTool.call({ file_path: '/tmp/hello-world.txt' }, ctx)
```

`FileReadTool.call()` 执行：
1. 展开路径（已是绝对路径，无需处理）
2. 检测文件类型 → 文本文件
3. 检查 mtime 去重 → 首次读取，继续
4. 读取文件内容 → `"Hello, World!\nThis is a test file.\n"`
5. `yield { type: 'text', content: '...', offset: 0 }`

执行完成：
```
TrackedTool 状态：executing → completed
tracked.results = [{ type: 'text', content: 'Hello, World!\nThis is a test file.\n', offset: 0 }]
```

### 步骤 5：收集结果

```typescript
// 同步收集（getCompletedResults）
for (const result of executor.getCompletedResults()) {
  // result = { trackedTool: ..., output: { type: 'text', ... } }
  // TrackedTool 状态：completed → yielded
}

// 异步等待（getRemainingResults）— 本例无剩余工具，立即结束
for await (const result of executor.getRemainingResults()) {
  // 不会进入此循环
}
```

### 步骤 6：序列化为 tool_result — 结果反馈给模型

调用 `FileReadTool.mapToolResultToToolResultBlockParam(output, ctx)`：

```typescript
// 输入
output = { type: 'text', content: 'Hello, World!\nThis is a test file.\n', offset: 0 }

// 输出
{
  type: 'tool_result',
  tool_use_id: 'toulu_01XYZ',
  content:
    '     1\tHello, World!\n' +
    '     2\tThis is a test file.\n' +
    '\n<system-reminder>\n' +
    'When providing file paths to other tools, use the path...\n' +
    '</system-reminder>'
}
```

### 步骤 7：发送 tool_result 给模型

REPL 构建下一轮 `user` 消息：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01XYZ",
      "content": "     1\tHello, World!\n     2\tThis is a test file.\n\n<system-reminder>..."
    }
  ]
}
```

模型收到此消息后，即可基于文件内容继续生成回应：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "文件内容如下：\n第1行：Hello, World!\n第2行：This is a test file.\n"
    }
  ]
}
```

---

## 总结：字段用途速查表

| 字段 | 谁使用 | 用途 |
|------|--------|------|
| `name` | 模型 / 权限系统 | 工具调用标识；Deny 规则匹配 |
| `inputSchema` | 模型 / 运行时 | 生成 JSON Schema；校验模型输入 |
| `maxResultSizeChars` | 执行层 | 防止单工具输出撑满 context |
| `description()` | 模型 | 工具选择依据 |
| `prompt()` | 模型（系统 prompt）| 详细使用规范 |
| `call()` | StreamingToolExecutor | 实际执行逻辑 |
| `checkPermissions()` | 权限管道（步骤 4）| 内容级权限校验 |
| `isConcurrencySafe()` | StreamingToolExecutor | 并发调度决策 |
| `isReadOnly()` | 权限系统 | 区分读写权限策略 |
| `mapToolResultToToolResultBlockParam()` | REPL 主循环 | 结果序列化，反馈给模型 |
| `isEnabled()` | 工具加载阶段 | 特性开关，决定工具是否出现在工具池 |
| `renderToolUseMessage()` | React UI | 终端/UI 显示，模型不可见 |
