# 04 — Context Compaction System: Deep Dive into the 4-Layer Architecture

> Files: `src/services/compact/`, `src/services/contextCollapse/`

---

## 1. Why Context Compaction Is Needed

Claude's context window is limited (approximately 200k tokens for models like claude-sonnet-4). As a conversation progresses:
- File reads, Bash output, and search results accumulate in the message list
- Multi-turn tool call inputs + outputs consume a large amount of space
- Eventually the model's context window is exceeded

Claude Code implements a **4-layer progressive compaction strategy**, tried from lightest to heaviest, to reduce token usage while preserving as much information as possible.

---

## 2. Layer Overview

```
Executed before each API call (query.ts stages 2-5):

Layer 1: Tool Result Offloading (toolResultStorage)
   ↓ if still not enough
Layer 2: Snip (HISTORY_SNIP, trim old tool result content)
   ↓ if still not enough
Layer 3: Microcompact (inline-clear old tool results, preserve structure)
   ↓ if still not enough
Layer 4: Context Collapse (progressive folding, keep last N turns)
   ↓ if still above threshold
Layer 5: Autocompact (AI summarization, last resort)

Post-recovery (when API returns 413):
   Reactive Compact (triggered only after 413 is received)
```

---

## 3. Layer 0: Tool Result Offloading (toolResultStorage)

This is the lightest "compaction" — not true compaction, but rather **persisting oversized tool results to disk** so the tool only receives a preview.

```typescript
// src/utils/toolResultStorage.ts

// Each tool has a maxResultSizeChars limit
// When exceeded: saved to ~/.claude/sessions/<sessionId>/tool-results/<uuid>.txt
// Tool result is replaced with:
"[Result saved to file — use Read tool to access: /path/to/file]\nPreview (first 500 chars):\n..."

// Configuration examples:
BashTool.maxResultSizeChars = 100_000    // 100k characters
GrepTool.maxResultSizeChars  = 200_000
WebFetchTool.maxResultSizeChars = 50_000
FileReadTool.maxResultSizeChars = Infinity  // Never offloaded (avoids Read→file→Read loops)
```

---

## 4. Layer 1: Snip (History Trimming)

> Feature Flag: `HISTORY_SNIP`

Snip targets **the oldest tool results**, preserving message structure but clearing content:

```typescript
// src/services/compact/snipCompact.ts

// Trigger condition: estimated token count > snip threshold
// Action: starting from the oldest tool results, replace content with '[Content removed to save context]'
// Preserved: full content of the most recent N turns of tool results

const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
messagesForQuery = snipResult.messages
snipTokensFreed = snipResult.tokensFreed  // passed to autocompact to correct threshold calculation

if (snipResult.boundaryMessage) {
  yield snipResult.boundaryMessage  // system message notifying user that snip occurred
}
```

**Relationship with Microcompact:** The two are not mutually exclusive — they can run simultaneously.

---

## 5. Layer 2: Microcompact (Inline Tool Result Compaction)

Microcompact clears old results from **compactable tools**, but retains the message "skeleton" (tool call records remain).

### 5.1 Compactable Tools

```typescript
// src/services/compact/microCompact.ts

const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,    // Read
  ...SHELL_TOOL_NAMES,    // Bash, PowerShell, etc.
  GREP_TOOL_NAME,         // Grep
  GLOB_TOOL_NAME,         // Glob
  WEB_SEARCH_TOOL_NAME,   // WebSearch
  WEB_FETCH_TOOL_NAME,    // WebFetch
  FILE_EDIT_TOOL_NAME,    // Edit
  FILE_WRITE_TOOL_NAME,   // Write
])
// Not compacted: AgentTool, AskUserQuestion, TaskCreate, etc. (results are important)
```

### 5.2 Time-Based Microcompact

```typescript
// When the last assistant message exceeds a certain age (server-side cache has expired)
// Proactively clear old tool results regardless of token count

const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
if (timeBasedResult) return timeBasedResult

// Rationale: cache is cold, the cost of rewriting the entire prefix is fixed anyway
// Better to proactively trim content and reduce the amount of data rewritten
```

### 5.3 Cached Microcompact (Advanced Mode)

```typescript
// Feature Flag: CACHED_MICROCOMPACT (Ant internal)
// Rationale: uses the API's cache_edits parameter to delete old tool results on the server side
//            does not modify local messages, does not break the Prompt Cache prefix

// Comparison:
// Regular Microcompact: modifies message content locally → breaks cache (bytes change)
// Cached Microcompact: API parameter → no local change → cache remains valid

const deletedContent = TIME_BASED_MC_CLEARED_MESSAGE  // '[Old tool result content cleared]'
```

---

## 6. Layer 3: Context Collapse (Progressive Folding)

> Feature Flag: `CONTEXT_COLLAPSE`

Context Collapse is an **incremental compaction** strategy: it folds older tool call + result pairs into concise single-line summaries.

```typescript
// src/services/contextCollapse/index.ts

// Fold operation:
// BEFORE (3 messages):
//   assistant: [Read tool_use, file_path="/src/main.ts"]
//   user: [tool_result: 500 lines of code...]
//   assistant: [text: "I've read the file, now let me..."]
//
// AFTER (1 message):
//   system: [Read /src/main.ts → 500 lines (collapsed)]

// Folding is a "view projection" (original messages are not modified):
// - Original messages are preserved in REPL history (disk + memory)
// - Fold summaries live in the collapse store
// - On each entry into queryLoop, projectView() replays the fold log to produce the current view

// This allows folds to accumulate across turns:
// Turn 5: fold turns 1-3
// Turn 8: fold turns 4-6 + existing folds
```

### 6.1 Recovery from API 413

```typescript
// Context Collapse as the first line of defense against 413:
if (isWithheld413) {
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    // Commit all pending folds at once
    const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
    if (drained.committed > 0) {
      // Folds were committed → retry API, should succeed this time
      state = { ...state, messages: drained.messages,
                transition: { reason: 'collapse_drain_retry' } }
      continue
    }
  }
}
```

---

## 7. Layer 4: Autocompact (AI Summarization)

This is the last resort: using Claude Haiku to generate a summary of the entire conversation, replacing old messages.

### 7.1 Trigger Threshold

```typescript
// src/services/compact/autoCompact.ts

// Reserve output space (based on p99.99 summary length of 17,387 tokens, rounded up to 20k)
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000

// Effective context window
effectiveContextWindow = contextWindow - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)

// Trigger threshold
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

// Warning UI threshold (shows "X% context remaining" warning)
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
warningThreshold = autocompactThreshold - WARNING_THRESHOLD_BUFFER_TOKENS

// Environment variable overrides (for testing)
process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW  // limit context window size
process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE  // override threshold by percentage
```

### 7.2 Compaction Flow

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

// Post-compaction message structure:
// buildPostCompactMessages(result) returns:
[
  boundaryMarker,     // compaction boundary marker (system message recording compaction metadata)
  ...summaryMessages, // Haiku-generated summary (user messages)
  ...messagesToKeep,  // preserved most recent N messages (suffix-preserving)
  ...attachments,     // file attachments (re-injected file content after compaction)
  ...hookResults,     // PostCompact hook results
]
```

### 7.3 Post-Compaction File Re-injection

```typescript
// src/services/compact/compact.ts

// After compaction, file content the model previously read must be re-provided
// Re-inject at most 5 files (to avoid immediately exhausting the context again)
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000        // total token budget for files
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000  // per-file limit

// Re-injected files are created as AttachmentMessages via generateFileAttachment()
// Sort order: by most recent edit/read time (most relevant first)
```

### 7.4 Compaction Circuit Breaker

```typescript
// Stop retrying after 3 consecutive failures (prevents API backlog)
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// Rationale: data from 2026-03-10 showed
// 1,279 sessions had 50+ consecutive failures (maximum 3,272)
// Globally wasting approximately 250,000 API calls per day
```

### 7.5 Session Memory Compact (Parallel Path)

```typescript
// src/services/compact/sessionMemoryCompact.ts

// Runs in parallel with the main compact: also compacts session memory (cross-session session_memory)
// Does not depend on the main compact result, runs independently

await trySessionMemoryCompaction(messages, toolUseContext, cacheSafeParams)
```

---

## 8. Reactive Compact

> Feature Flag: `REACTIVE_COMPACT`

Reactive Compact does not run proactively — it is triggered **in response to a 413 error from the API**:

```typescript
// Trigger scenarios:
// 1. Received prompt_too_long (413) → attempt compaction then retry
// 2. Received media size error (image/PDF too large) → remove media then retry

// Difference from proactive autocompact:
// - proactive: preventive compaction before the API call
// - reactive: emergency compaction after the API reports an error

const compacted = await reactiveCompact.tryReactiveCompact({
  hasAttempted: hasAttemptedReactiveCompact,  // only try once
  querySource,
  aborted: toolUseContext.abortController.signal.aborted,
  messages: messagesForQuery,
  cacheSafeParams: { systemPrompt, userContext, systemContext, toolUseContext },
})

if (compacted) {
  // Compaction succeeded → yield compaction message → retry
  state = { ...state, hasAttemptedReactiveCompact: true,
            transition: { reason: 'reactive_compact_retry' } }
  continue
}
// Failed → yield 413 error to user
```

---

## 9. Token Warning State Machine

```
token usage increases
    │
    ├─ < warningThreshold    → green (normal)
    │
    ├─ ≥ warningThreshold    → yellow warning ("Context N% full, consider /compact")
    │
    ├─ ≥ autoCompactThreshold → trigger Autocompact (if enabled)
    │                         → or red error (if disabled)
    │
    └─ ≥ effectiveContextWindow → blocked (blockingLimit)
                                  yield ERROR, return { reason: 'blocking_limit' }
```

```typescript
// UI display logic
export function calculateTokenWarningState(tokenUsage, model): {
  percentLeft: number
  isAboveWarningThreshold: boolean
  isAboveErrorThreshold: boolean
  isAboveAutoCompactThreshold: boolean
  isAtBlockingLimit: boolean
}

// Bottom status bar shows:
// "claude-opus-4-6 · ↑1.2k ↓0.8k tokens · $0.04 · 23% context remaining"
```

---

## 10. Compaction Metadata (CompactionResult)

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage       // compaction boundary (with timestamp, token stats)
  summaryMessages: UserMessage[]      // Haiku-generated summaries
  attachments: AttachmentMessage[]    // re-injected file content
  hookResults: HookResultMessage[]    // PostCompact hook results
  messagesToKeep?: Message[]          // preserved recent messages
  userDisplayMessage?: string         // notification shown to user ("Auto-compacted...")
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number  // actual count including messagesToKeep
  compactionUsage?: TokenUsage        // tokens consumed by compaction itself (Haiku call)
}
```

---

## 11. Manual Compaction (/compact Command)

```
/compact [instructions]

Triggers compactConversation() → same compaction logic as autocompact
Differences:
- isAutoCompact = false (bypasses threshold check)
- customInstructions can be set by the user ("focus on the database changes")
- suppressFollowUpQuestions = false (allows questions to be preserved in summary)
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/services/compact/autoCompact.ts` | Threshold calculation, trigger logic, circuit breaker |
| `src/services/compact/compact.ts` | compactConversation main function (1500+ lines) |
| `src/services/compact/microCompact.ts` | Microcompact inline clearing |
| `src/services/compact/sessionMemoryCompact.ts` | Session memory independent compaction |
| `src/services/compact/reactiveCompact.ts` | 413 reactive compaction |
| `src/services/compact/snipCompact.ts` | Snip history trimming |
| `src/services/contextCollapse/` | Context Collapse progressive folding |
| `src/utils/toolResultStorage.ts` | Tool result disk offloading |
| `src/utils/tokens.ts` | Token estimation algorithm |
| `src/utils/context.ts` | Model context window size table |
