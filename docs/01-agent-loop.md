# 01 — Agent Loop 深度解析：query.ts 的 1729 行

> 文件：`src/query.ts`（1729 行），`src/query/`，`src/services/tools/`

---

## 1. 总体架构

Claude Code 的核心是一个**无限 while 循环**，每次迭代：

```
[准备消息] → [上下文压缩] → [调用模型 API] → [流式处理响应] → [执行工具] → [循环继续/终止]
```

整个循环是一个 **AsyncGenerator**，对外产出（yield）各种事件，调用方（REPL 或 SDK）消费这些事件来更新 UI 或返回结果。

---

## 2. 入口函数

```typescript
// src/query.ts

export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }  // API 任务预算（2026-03-13 beta）
  deps?: QueryDeps               // 依赖注入（测试用）
}

export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent         // API 流式数据（assistant 消息、工具调用）
  | RequestStartEvent  // 请求开始信号（触发 spinner）
  | Message            // 完整消息（user/assistant/system）
  | TombstoneMessage   // 消息删除信号（model 切换时）
  | ToolUseSummaryMessage,  // 工具调用摘要（Haiku 生成）
  Terminal             // 终止原因
>
```

---

## 3. 循环状态（State）

循环的所有可变状态都聚合在一个 `State` 对象中，在每个 `continue` 点整体替换（类函数式更新，避免遗漏字段）：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number      // max_output_tokens 恢复次数
  hasAttemptedReactiveCompact: boolean      // 是否尝试过反应式压缩
  maxOutputTokensOverride: number | undefined   // 覆盖输出 token 上限
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined       // 是否在 stop hook 循环中
  turnCount: number                         // 当前轮次数
  transition: Continue | undefined          // 前一次迭代的转换原因（用于断言）
}
```

**转换原因（Continue.reason）：**

| 原因 | 触发条件 |
|------|----------|
| `max_output_tokens_escalate` | 输出截断，将上限从 8k 升为 64k 重试 |
| `max_output_tokens_recovery` | 输出截断，注入恢复提示继续（最多 3 次）|
| `collapse_drain_retry` | Context Collapse 提交了阶段性折叠，重试 API |
| `reactive_compact_retry` | Reactive Compact 压缩成功，重试 API |
| `stop_hook_blocking` | Stop Hook 注入了阻断错误，需要继续 |
| `token_budget_continuation` | Token Budget 未耗尽，继续 |
| `model_fallback` | 模型切换（高峰期回退到备用模型）|

---

## 4. 每次迭代的 8 个阶段

### 阶段 1：初始化与前置检查

```typescript
yield { type: 'stream_request_start' }  // → REPL 显示 spinner

// 工具结果预算：大结果外置到磁盘（避免 context 爆炸）
messagesForQuery = await applyToolResultBudget(...)

// Skill 发现预取（并行，不阻塞）
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(...)
```

### 阶段 2：Snip（历史裁剪）

```typescript
// HISTORY_SNIP feature flag 控制
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed  // 传给 autocompact 修正阈值
  if (snipResult.boundaryMessage) yield snipResult.boundaryMessage
}
```

### 阶段 3：Microcompact（工具结果内联压缩）

```typescript
// 压缩旧的工具结果（保留最近 N 轮，清空更早的文件读取、bash 输出等）
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages
// 可压缩工具：Read、Bash、Grep、Glob、WebSearch、WebFetch、Edit、Write
```

### 阶段 4：Context Collapse（渐进式折叠）

```typescript
// CONTEXT_COLLAPSE feature flag
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
  messagesForQuery = collapseResult.messages
  // 折叠是"视图投影"——原始消息保留在 REPL 历史，
  // 折叠摘要存在 collapse store，每次循环 projectView() 重放
}
```

### 阶段 5：Autocompact（自动对话压缩）

```typescript
// 检查是否超过阈值，超过则用 Haiku 总结压缩
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  { ... },
  querySource,
  tracking,
  snipTokensFreed,
)

if (compactionResult) {
  // 压缩成功：yield 压缩后的消息，继续 loop
  const postCompactMessages = buildPostCompactMessages(compactionResult)
  for (const message of postCompactMessages) yield message
  messagesForQuery = postCompactMessages
}
```

**Autocompact 阈值计算：**

```typescript
// 有效上下文窗口 = 模型最大窗口 - 预留输出空间（20,000 tokens）
effectiveContextWindow = contextWindow - min(maxOutputTokens, 20_000)

// 触发阈值 = 有效窗口 - 13,000 token 缓冲
autocompactThreshold = effectiveContextWindow - 13_000

// 警告阈值 = 触发阈值 - 20,000（用于 UI 提示）
warningThreshold = autocompactThreshold - 20_000

// 阻断阈值（autocompact 关闭时）= 100% 有效窗口
blockingLimit = effectiveContextWindow
```

### 阶段 6：调用模型 API（流式）

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fallbackModel,
    querySource,
    // ... 其他参数
  },
})) {
  // 处理每个流式事件
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    // 收集 tool_use blocks
    const toolUseBlocks = message.message.content.filter(b => b.type === 'tool_use')
    if (toolUseBlocks.length > 0) needsFollowUp = true
  }

  // 过滤可恢复的错误（先不 yield，等待恢复逻辑）
  if (!withheld) yield message
}
```

**可恢复的错误（先扣留不 yield）：**
- `prompt_too_long`（413）→ 尝试 Context Collapse 或 Reactive Compact
- `max_output_tokens`（截断）→ 尝试升级上限或注入恢复提示
- 媒体大小错误（图片/PDF 过大）→ Reactive Compact 删除媒体

**模型回退（FallbackTriggeredError）：**

```typescript
catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true  // 重试整个 API 调用

    // 为孤立的已 yield 消息发送 tombstone（让 UI 删除它们）
    for (const msg of assistantMessages) {
      yield { type: 'tombstone', message: msg }
    }
    // 清空并重试
    assistantMessages.length = 0
    continue
  }
}
```

### 阶段 7：执行工具

```typescript
// 两种执行路径：
// 1. StreamingToolExecutor（并行流式执行，边流式接收边执行）
// 2. runTools（串行，等 stream 结束后统一执行）
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) yield update.message
  if (update.newContext) updatedToolUseContext = update.newContext
}

// 工具执行后，并行生成工具摘要（用 Haiku，不阻塞下一轮）
nextPendingToolUseSummary = generateToolUseSummary({ tools: toolInfoForSummary })
```

### 阶段 8：决定是否继续循环

```typescript
if (!needsFollowUp) {
  // 没有工具调用 → 检查 stop hooks、token budget → 终止

  const stopHookResult = yield* handleStopHooks(...)
  if (stopHookResult.preventContinuation) return { reason: 'stop_hook_prevented' }
  if (stopHookResult.blockingErrors.length > 0) {
    // Stop hook 注入了错误 → 继续循环，让模型处理
    state = { ..., transition: { reason: 'stop_hook_blocking' } }
    continue
  }

  return { reason: 'completed' }  // 正常完成
}

// 有工具调用 → 组装下一轮消息，继续循环
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({ type: 'max_turns_reached', ... })
  return { reason: 'max_turns' }
}

state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults, ...attachments],
  turnCount: turnCount + 1,
  // ...
}
// while(true) 继续下一轮
```

---

## 5. 终止原因（Terminal.reason）

```typescript
type Terminal = { reason:
  | 'completed'               // 正常完成（无工具调用，stop hooks 通过）
  | 'aborted_streaming'       // 用户在 streaming 时中断（Ctrl+C）
  | 'aborted_tools'           // 用户在工具执行时中断
  | 'max_turns'               // 达到最大轮次限制
  | 'blocking_limit'          // 上下文超出阻断阈值（autocompact 关闭时）
  | 'prompt_too_long'         // 413 且无法恢复
  | 'image_error'             // 图片/媒体错误
  | 'model_error'             // 模型调用内部错误
  | 'stop_hook_prevented'     // Stop hook 阻止了继续
  | 'hook_stopped'            // Hook 发送 STOP 信号
}
```

---

## 6. 关键设计模式

### 6.1 依赖注入（QueryDeps）

```typescript
type QueryDeps = {
  callModel: typeof callModel
  microcompact: typeof microcompact
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

// 生产环境
const deps = params.deps ?? productionDeps()

// 测试环境可以替换为 mock
const testDeps = {
  callModel: mockCallModel,
  microcompact: noopMicrocompact,
  autocompact: noopAutocompact,
  uuid: deterministicUuid,
}
```

### 6.2 Prompt Cache 保护

```typescript
// 工具输入的"可观测拷贝"与"API 发送内容"分离
// backfillObservableInput 只改拷贝，不改原件
// 原件保持字节不变 → 最大化 Prompt Cache 命中率

let yieldMessage: typeof message = message
if (message.type === 'assistant') {
  // 克隆并填充 legacy/derived 字段，只供 SDK/transcript 看
  // API 重发时用原始 message（字节相同）
}
```

### 6.3 Task Budget（2026-03-13 新特性）

```typescript
// API 级别的任务预算（非 token 数，而是 context window 占用）
// 每次 compact 时记录"已用掉的 context"
taskBudgetRemaining = Math.max(
  0,
  (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
)
// 传给下一次 API 调用，让服务端知道剩余预算
```

---

## 7. 流程全图

```
query()
  │
  └─ queryLoop()
       │
       while(true)
         │
         ├─ yield stream_request_start
         │
         ├─ 1. applyToolResultBudget（外置大工具结果）
         │
         ├─ 2. snipCompactIfNeeded（HISTORY_SNIP）
         │
         ├─ 3. microcompact（工具结果内联压缩）
         │
         ├─ 4. contextCollapse.applyCollapsesIfNeeded（渐进折叠）
         │
         ├─ 5. autoCompactIfNeeded
         │      ├─ 未超阈值 → 跳过
         │      └─ 超阈值 → 用 Haiku 总结 → yield 压缩消息
         │
         ├─ 6. callModel（流式）
         │      ├─ yield assistant messages
         │      ├─ yield tool_use blocks（needsFollowUp = true）
         │      ├─ 扣留可恢复错误（PTL、max_output_tokens、media）
         │      └─ 模型回退 → yield tombstones → 切换模型重试
         │
         ├─ 7. executePostSamplingHooks（异步，不阻塞）
         │
         ├─ 8. 处理可恢复错误
         │      ├─ PTL → collapse drain → reactive compact → yield error
         │      ├─ max_output_tokens → 升级上限 → 注入恢复提示
         │      └─ 媒体错误 → reactive compact
         │
         ├─ 9. 无 needsFollowUp → stop hooks → token budget → 终止
         │
         ├─ 10. runTools / StreamingToolExecutor
         │       └─ yield tool results（包括权限确认弹框）
         │
         ├─ 11. 获取附件（队列命令、内存附件）
         │
         └─ 12. maxTurns 检查 → 组装下一轮 state → continue
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/query.ts` | Agent Loop 主体（1729 行）|
| `src/query/config.ts` | 从环境/statsig 快照配置（QueryConfig）|
| `src/query/deps.ts` | 依赖注入接口（productionDeps）|
| `src/query/transitions.ts` | Terminal / Continue 类型定义 |
| `src/query/stopHooks.ts` | Stop Hook 处理逻辑 |
| `src/query/tokenBudget.ts` | Token Budget 追踪 |
| `src/services/tools/toolOrchestration.ts` | runTools 串行执行 |
| `src/services/tools/StreamingToolExecutor.ts` | 并行流式工具执行器 |
| `src/services/compact/autoCompact.ts` | Autocompact 阈值计算与触发 |
| `src/services/compact/microCompact.ts` | Microcompact 内联压缩 |
