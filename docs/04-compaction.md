# 04 — 上下文压缩系统：4 层架构深度解析

> 文件：`src/services/compact/`，`src/services/contextCollapse/`

---

## 1. 为什么需要上下文压缩

Claude 的上下文窗口有限（claude-sonnet-4 等模型约 200k tokens）。随着对话进行：
- 文件读取、Bash 输出、搜索结果在消息列表中积累
- 多轮工具调用的 input+output 占据大量空间
- 最终超出模型上下文窗口

Claude Code 实现了**4 层递进式压缩策略**，由轻到重依次尝试，在保留最多信息的前提下减少 token 用量。

---

## 2. 层次结构总览

```
每次 API 调用前执行（query.ts 阶段 2-5）：

Layer 1: 工具结果外置（toolResultStorage）
   ↓ 如果还不够
Layer 2: Snip（HISTORY_SNIP，裁剪旧的工具结果内容）
   ↓ 如果还不够
Layer 3: Microcompact（内联清空旧工具结果，保留结构）
   ↓ 如果还不够
Layer 4: Context Collapse（渐进折叠，保留最近 N 轮）
   ↓ 如果还超过阈值
Layer 5: Autocompact（AI 摘要压缩，最后手段）

事后恢复（API 返回 413 时）：
   Reactive Compact（反应式压缩，已触发 413 才运行）
```

---

## 3. Layer 0：工具结果外置（toolResultStorage）

这是最轻量的"压缩"——不是真正压缩，而是**把超大工具结果持久化到磁盘**，工具只收到预览。

```typescript
// src/utils/toolResultStorage.ts

// 每个工具有 maxResultSizeChars 限制
// 超出时：保存到 ~/.claude/sessions/<sessionId>/tool-results/<uuid>.txt
// 工具结果替换为：
"[Result saved to file — use Read tool to access: /path/to/file]\nPreview (first 500 chars):\n..."

// 可配置示例：
BashTool.maxResultSizeChars = 100_000    // 10万字符
GrepTool.maxResultSizeChars  = 200_000
WebFetchTool.maxResultSizeChars = 50_000
FileReadTool.maxResultSizeChars = Infinity  // 永不外置（避免 Read→file→Read 循环）
```

---

## 4. Layer 1：Snip（历史裁剪）

> Feature Flag：`HISTORY_SNIP`

Snip 针对**最旧的工具结果**进行裁剪，保留消息结构但清空内容：

```typescript
// src/services/compact/snipCompact.ts

// 触发条件：估算 token 数 > snip 阈值
// 操作：从最老的工具结果开始，把内容替换为 '[Content removed to save context]'
// 保留：最近 N 轮的工具结果完整内容

const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
messagesForQuery = snipResult.messages
snipTokensFreed = snipResult.tokensFreed  // 传给 autocompact，修正阈值计算

if (snipResult.boundaryMessage) {
  yield snipResult.boundaryMessage  // 系统消息通知用户发生了 snip
}
```

**与 Microcompact 的关系：** 两者不互斥，可以同时运行。

---

## 5. Layer 2：Microcompact（内联工具结果压缩）

Microcompact 清空**可压缩工具**的旧结果，但保留消息的"骨架"（工具调用记录仍在）。

### 5.1 可压缩的工具

```typescript
// src/services/compact/microCompact.ts

const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,    // Read
  ...SHELL_TOOL_NAMES,    // Bash、PowerShell 等
  GREP_TOOL_NAME,         // Grep
  GLOB_TOOL_NAME,         // Glob
  WEB_SEARCH_TOOL_NAME,   // WebSearch
  WEB_FETCH_TOOL_NAME,    // WebFetch
  FILE_EDIT_TOOL_NAME,    // Edit
  FILE_WRITE_TOOL_NAME,   // Write
])
// 不压缩：AgentTool、AskUserQuestion、TaskCreate 等（结果重要）
```

### 5.2 时间触发的 Microcompact

```typescript
// 当上次 assistant 消息超过一定时间（服务端缓存已过期时）
// 无论 token 数多少，都主动清空旧工具结果

const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
if (timeBasedResult) return timeBasedResult

// 原理：缓存已冷，重写整个前缀的代价固定
// 不如主动删减内容，减少重写的数据量
```

### 5.3 Cached Microcompact（高级模式）

```typescript
// Feature Flag: CACHED_MICROCOMPACT（Ant 内部）
// 原理：通过 API 的 cache_edits 参数，在服务端删除旧工具结果
//       不改变本地消息，不破坏 Prompt Cache 前缀

// 对比：
// 普通 Microcompact：本地修改消息内容 → 破坏 cache（字节变化）
// Cached Microcompact：API 参数 → 不改本地 → cache 依然有效

const deletedContent = TIME_BASED_MC_CLEARED_MESSAGE  // '[Old tool result content cleared]'
```

---

## 6. Layer 3：Context Collapse（渐进式折叠）

> Feature Flag：`CONTEXT_COLLAPSE`

Context Collapse 是一种**增量式压缩**策略：把较旧的工具调用+结果对，折叠成简洁的单行摘要。

```typescript
// src/services/contextCollapse/index.ts

// 折叠操作：
// BEFORE（3条消息）：
//   assistant: [Read tool_use, file_path="/src/main.ts"]
//   user: [tool_result: 500行代码...]
//   assistant: [text: "I've read the file, now let me..."]
//
// AFTER（1条消息）：
//   system: [Read /src/main.ts → 500 lines (collapsed)]

// 折叠是"视图投影"（不修改原始消息）：
// - 原始消息保留在 REPL 历史（disk + memory）
// - 折叠摘要存在 collapse store
// - 每次进入 queryLoop，projectView() 重放折叠日志，生成当前视图

// 这使得折叠可以跨轮次累积：
// 轮次 5：折叠轮次 1-3
// 轮次 8：折叠轮次 4-6 + 已有折叠
```

### 6.1 从 API 413 恢复

```typescript
// Context Collapse 作为 413 的第一道防线：
if (isWithheld413) {
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    // 把所有已暂存的折叠全部提交
    const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
    if (drained.committed > 0) {
      // 有折叠被提交了 → 重试 API，这次应该能通过
      state = { ...state, messages: drained.messages,
                transition: { reason: 'collapse_drain_retry' } }
      continue
    }
  }
}
```

---

## 7. Layer 4：Autocompact（AI 摘要压缩）

这是最后手段：用 Claude Haiku 对整个对话生成摘要，替换旧消息。

### 7.1 触发阈值

```typescript
// src/services/compact/autoCompact.ts

// 预留输出空间（基于 p99.99 摘要长度 17,387 tokens，取整到 20k）
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000

// 有效上下文窗口
effectiveContextWindow = contextWindow - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)

// 触发阈值
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

// 警告 UI 阈值（显示"context 剩余 X%"警告）
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
warningThreshold = autocompactThreshold - WARNING_THRESHOLD_BUFFER_TOKENS

// 环境变量覆盖（测试用）
process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW  // 限制上下文窗口大小
process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE  // 按百分比覆盖阈值
```

### 7.2 压缩流程

```typescript
// src/services/compact/compact.ts

export async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
  recompactionInfo?: RecompactionInfo,
): Promise<CompactionResult>

// 压缩后的消息结构：
// buildPostCompactMessages(result) 返回：
[
  boundaryMarker,     // 压缩边界标记（system 消息，记录压缩元数据）
  ...summaryMessages, // Haiku 生成的摘要（user 消息）
  ...messagesToKeep,  // 保留的最近 N 条消息（suffix-preserving）
  ...attachments,     // 文件附件（压缩后重新注入读取过的文件内容）
  ...hookResults,     // PostCompact hook 的结果
]
```

### 7.3 压缩后的文件重注入

```typescript
// src/services/compact/compact.ts

// 压缩后，需要重新提供模型曾经读取过的文件内容
// 最多重新注入 5 个文件（避免 context 立刻又爆）
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000        // 文件总 token 预算
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000  // 单文件上限

// 重注入的文件通过 generateFileAttachment() 创建 AttachmentMessage
// 排序规则：按最近编辑/读取时间排序（最相关的优先）
```

### 7.4 压缩熔断器

```typescript
// 连续失败 3 次后停止重试（防止 API 积压）
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// 原因：2026-03-10 数据显示，
// 1,279 个会话有 50+ 次连续失败（最高 3,272 次）
// 全球每天浪费约 25 万次 API 调用
```

### 7.5 Session Memory Compact（并行路径）

```typescript
// src/services/compact/sessionMemoryCompact.ts

// 与主 compact 并行：同时压缩会话记忆（跨会话的 session_memory）
// 不依赖主 compact 结果，独立运行

await trySessionMemoryCompaction(messages, toolUseContext, cacheSafeParams)
```

---

## 8. Reactive Compact（反应式压缩）

> Feature Flag：`REACTIVE_COMPACT`

Reactive Compact 不是提前运行，而是**响应 API 返回的 413 错误**触发：

```typescript
// 触发场景：
// 1. 收到 prompt_too_long (413) → 尝试压缩后重试
// 2. 收到媒体大小错误（图片/PDF 太大）→ 删除媒体后重试

// 与 proactive autocompact 的区别：
// - proactive：在 API 调用前预防性压缩
// - reactive：在 API 报错后紧急压缩

const compacted = await reactiveCompact.tryReactiveCompact({
  hasAttempted: hasAttemptedReactiveCompact,  // 只尝试一次
  querySource,
  aborted: toolUseContext.abortController.signal.aborted,
  messages: messagesForQuery,
  cacheSafeParams: { systemPrompt, userContext, systemContext, toolUseContext },
})

if (compacted) {
  // 压缩成功 → yield 压缩消息 → 重试
  state = { ...state, hasAttemptedReactiveCompact: true,
            transition: { reason: 'reactive_compact_retry' } }
  continue
}
// 失败 → 将 413 错误 yield 给用户
```

---

## 9. Token Warning 状态机

```
token 用量增长
    │
    ├─ < warningThreshold    → 绿色（正常）
    │
    ├─ ≥ warningThreshold    → 黄色警告（"Context N% full，建议 /compact"）
    │
    ├─ ≥ autoCompactThreshold → 触发 Autocompact（如果开启）
    │                         → 或红色错误（如果关闭）
    │
    └─ ≥ effectiveContextWindow → 阻断（blockingLimit）
                                  yield ERROR，return { reason: 'blocking_limit' }
```

```typescript
// UI 展示逻辑
export function calculateTokenWarningState(tokenUsage, model): {
  percentLeft: number
  isAboveWarningThreshold: boolean
  isAboveErrorThreshold: boolean
  isAboveAutoCompactThreshold: boolean
  isAtBlockingLimit: boolean
}

// 底部状态栏显示：
// "claude-opus-4-6 · ↑1.2k ↓0.8k tokens · $0.04 · 23% context remaining"
```

---

## 10. 压缩元数据（CompactionResult）

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage       // 压缩边界（含时间戳、tokens 统计）
  summaryMessages: UserMessage[]      // Haiku 生成的摘要
  attachments: AttachmentMessage[]    // 重注入的文件内容
  hookResults: HookResultMessage[]    // PostCompact hook 结果
  messagesToKeep?: Message[]          // 保留的最近消息
  userDisplayMessage?: string         // 给用户看的通知（"已自动压缩..."）
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number  // 含 messagesToKeep 的真实数量
  compactionUsage?: TokenUsage        // 压缩本身消耗的 tokens（Haiku 调用）
}
```

---

## 11. 手动压缩（/compact 命令）

```
/compact [instructions]

触发 compactConversation() → 与 autocompact 相同的压缩逻辑
区别：
- isAutoCompact = false（不经过阈值检查）
- customInstructions 可由用户自定义（"focus on the database changes"）
- suppressFollowUpQuestions = false（允许摘要中保留问题）
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/services/compact/autoCompact.ts` | 阈值计算、触发判断、熔断器 |
| `src/services/compact/compact.ts` | compactConversation 主函数（1500+ 行）|
| `src/services/compact/microCompact.ts` | Microcompact 内联清空 |
| `src/services/compact/sessionMemoryCompact.ts` | 会话记忆独立压缩 |
| `src/services/compact/reactiveCompact.ts` | 413 响应式压缩 |
| `src/services/compact/snipCompact.ts` | Snip 历史裁剪 |
| `src/services/contextCollapse/` | Context Collapse 渐进折叠 |
| `src/utils/toolResultStorage.ts` | 工具结果磁盘外置 |
| `src/utils/tokens.ts` | token 估算算法 |
| `src/utils/context.ts` | 模型上下文窗口大小表 |
