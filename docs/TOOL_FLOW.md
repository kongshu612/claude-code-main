# Claude Code — Tool 执行主线

> 学习导航文档。后续所有对代码的深入学习都以本文为索引。

---

## 一、整体流程图

```
用户在终端输入消息
        │
        ▼
[1] CLI 入口层
    src/entrypoints/cli.tsx
        │  解析 CLI 参数，调用 cliMain()
        ▼
[2] 初始化层
    src/main.tsx
        │  初始化配置、鉴权、调用 launchRepl()
        ▼
[3] REPL 会话层
    src/screens/REPL.tsx  (L2793)
        │  for await (const event of query(...))
        │  onQueryEvent(event)
        ▼
[4] Query 引擎 — 核心驱动循环
    src/query.ts  (L219  queryLoop)
        │
        ├── 组装消息 + 工具 Schema
        │
        ▼
[5] API 调用层
    src/services/api/claude.ts  (L752  queryModelWithStreaming)
        │  向 Anthropic API 发流式请求
        │  解析 SSE 事件流
        │
        │  若响应含 tool_use block → needsFollowUp = true
        ▼
[6] 工具调度层
    src/services/tools/toolOrchestration.ts
        │  partitionToolCalls()  → 并发 / 串行分批
        │
        ▼
[7] 工具执行层
    src/services/tools/toolExecution.ts  (L337  runToolUse)
        │
        ├── [7a] 查找 tool（L345）
        ├── [7b] Zod schema 验证（L615）
        ├── [7c] Pre-tool hooks（L800）
        ├── [7d] 权限决策（L921）allow / deny / ask
        ├── [7e] tool.call() 实际执行（L1207）
        └── [7f] 结果格式化 → tool_result block（L1292）
        │
        ▼
[8] 结果回注 & 循环
    src/query.ts  (L1384)
        │  tool_result 追加到 messages
        │  回到 [4] 继续下一轮 API 调用
        │
        └── 直到 stop_reason = 'end_turn' 或达到 maxTurns
                │
                ▼
           结果渲染给用户 (REPL.tsx)
```

---

## 二、关键文件索引

| # | 文件 | 核心职责 | 关键位置 |
|---|------|---------|---------|
| 1 | `src/entrypoints/cli.tsx` | CLI 入口，参数解析 | 整文件 |
| 2 | `src/main.tsx` | 初始化 + 启动 REPL | `launchRepl()` |
| 3 | `src/screens/REPL.tsx` | 对话主循环，事件消费 | L2793 `for await` |
| 4 | `src/query.ts` | Tool 驱动引擎，状态管理 | L219 `query()`, L241 `queryLoop()` |
| 5 | `src/tools.ts` | 所有 Tool 注册表 | L193 `getAllBaseTools()` |
| 6 | `src/Tool.ts` | Tool 接口类型定义 | L158 `ToolUseContext` |
| 7 | `src/services/api/claude.ts` | API 调用 + 流式事件解析 | L752, L1940 |
| 8 | `src/services/tools/toolOrchestration.ts` | 并发/串行调度 | L91 `partitionToolCalls()` |
| 9 | `src/services/tools/toolExecution.ts` | 权限检查 + 实际执行 | L337 `runToolUse()` |

---

## 三、各层详解

### [1] CLI 入口 — `cli.tsx`

- 处理 `--version`、`--daemon-worker` 等快捷路径
- 最终调用 `src/main.tsx` 的 `cliMain()`

---

### [2] 初始化 — `main.tsx`

- 加载 `~/.claude/` 配置（settings、memory 等）
- 完成 OAuth 鉴权
- 调用 `launchRepl()` 进入交互模式

---

### [3] REPL 层 — `REPL.tsx`

- 维护 React 状态：消息列表、UI 渲染
- 核心：`for await (const event of query(...))` 消费每个事件
- 事件类型：stream delta、完整消息、tool 进度、MCP 资源等
- 每轮结束触发 `onTurnComplete()`

---

### [4] Query 引擎 — `query.ts`

这是整个 tool 执行的**驱动核心**，包含一个无限循环：

```
while (true) {
  1. 组装消息（含压缩/折叠）
  2. 调 queryModelWithStreaming()  →  API
  3. 若 needsFollowUp（有 tool_use）→ 执行工具
  4. 把 tool_result 追加到 messages
  5. 检查终止条件（end_turn / maxTurns / token 预算）
}
```

状态对象携带：`messages`、`toolUseContext`、`turnCount`、`pendingToolUseSummary`

---

### [5] API 层 — `claude.ts`

**请求阶段**（L752）：
- 将 tool schema（JSON Schema 格式）附在请求体中
- 使用 `anthropic.beta.messages.create({ stream: true })` 发起流

**流解析阶段**（L1940）：

| 事件 | 处理 |
|------|------|
| `message_start` | 初始化消息，记录 usage |
| `content_block_start` (tool_use) | 建立 tool_use block，记录 id/name |
| `content_block_delta` | 累积 tool input JSON 字符串 |
| `content_block_stop` | block 完成 |
| `message_stop` | 流结束，检查是否含 tool_use → 设 `needsFollowUp` |

---

### [6] 工具调度 — `toolOrchestration.ts`

`partitionToolCalls()` 将 tool call 列表分批：
- **并发安全**（`isConcurrencySafe = true`）：同批并发执行（如 Glob、Grep、Read）
- **非并发安全**：单独串行执行（如 Bash、Edit）

---

### [7] 工具执行 — `toolExecution.ts`

`runToolUse()` 的完整流程：

```
查找 tool（name → Tool 对象）
    ↓
Zod schema 验证 input
    ↓
自定义 validateInput()
    ↓
Pre-tool hooks（可修改 input / 拦截执行）
    ↓
权限决策  allow | deny | ask
    │
    ├── deny → 返回错误 tool_result
    ├── ask  → 弹出用户确认 UI
    └── allow ↓
              tool.call(input, ctx, progress)
                ↓
              mapToolResultToToolResultBlockParam()
                ↓
              返回 tool_result block
```

---

### [8] 结果回注 — `query.ts:1384`

- 收集本轮所有 tool_result，以 `user` 角色消息追加到 `messages`
- **立即返回 `query.ts` 主循环的下一轮**，携带更新后的 messages 再次调用 API
- API 收到 tool_result 后生成下一个 assistant 回复（可能再次含 tool_use）

---

## 四、Tool 接口定义（学习切入点）

每个 tool 实现 `Tool` 接口（`src/Tool.ts`），核心字段：

```typescript
interface Tool {
  name: string                    // 工具名，API 中唯一标识
  inputSchema: ZodSchema          // 输入验证 schema
  isConcurrencySafe: boolean      // 是否可并发
  isEnabled(): boolean            // 特性开关
  validateInput(input): ...       // 自定义校验
  call(input, ctx, canUseTool, msg, progress): Promise<{data}>  // 实际执行
  mapToolResultToToolResultBlockParam(result): ...  // 格式化给 API
}
```

内置 Tool 示例：`BashTool`、`FileReadTool`、`FileEditTool`、`GlobTool`、`GrepTool`

---

## 五、学习路线建议

1. **先读接口** — `src/Tool.ts`，理解 Tool 契约
2. **读一个简单 Tool** — 如 `GlobTool`，看 `call()` 实现
3. **跟权限系统** — `toolExecution.ts` L800–L978，理解 hooks + canUseTool
4. **跟 API 流** — `claude.ts` L1940，理解 tool_use block 如何从流中组装
5. **读 queryLoop** — `query.ts` L241，理解多轮 tool 调用如何驱动
6. **读 REPL 事件消费** — `REPL.tsx` L2793，理解 UI 层如何响应

---

*最后更新：2026-04-08*
