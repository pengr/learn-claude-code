# 11 — Cost Tracking and Conversation History

> Files: `src/cost-tracker.ts`, `src/costHook.ts`, `src/history.ts`, `src/utils/model/modelCost.ts`

---

## 1. Model Pricing Table

Claude Code bundles pricing data for all models (USD price per million tokens):

```typescript
// src/utils/model/modelCost.ts (excerpted)
const MODEL_PRICES: Record<string, ModelPricing> = {
  // Format: input / output / cache_read (10% discount) / cache_write (125% premium)

  'claude-haiku-3-5': {
    input:       0.80,   // $0.80 / MTok
    output:      4.00,
    cache_read:  0.08,   // 10% of input
    cache_write: 1.00,   // 125% of input
  },

  'claude-sonnet-4-5': {
    input:       3.00,
    output:     15.00,
    cache_read:  0.30,
    cache_write: 3.75,
  },

  'claude-sonnet-4-6': {
    input:       3.00,
    output:     15.00,
    cache_read:  0.30,
    cache_write: 3.75,
  },

  'claude-opus-4-5': {
    input:      15.00,
    output:     75.00,
    cache_read:  1.50,
    cache_write: 18.75,
  },

  'claude-opus-4-6': {
    input:      15.00,
    output:     75.00,
    cache_read:  1.50,
    cache_write: 18.75,
    // fast mode (fastMode) cost multiplier: 6x
    fastModeMultiplier: 6,
  },
}
```

### Cost Calculation Formula

```typescript
function calculateCost(usage: TokenUsage, model: string): number {
  const prices = MODEL_PRICES[normalizeModelName(model)]
  if (!prices) return 0

  const inputCost  = (usage.input_tokens  / 1_000_000) * prices.input
  const outputCost = (usage.output_tokens / 1_000_000) * prices.output
  const cacheReadCost  = (usage.cache_read_input_tokens  ?? 0) / 1_000_000 * prices.cache_read
  const cacheWriteCost = (usage.cache_creation_input_tokens ?? 0) / 1_000_000 * prices.cache_write

  const baseCost = inputCost + outputCost + cacheReadCost + cacheWriteCost

  // Fast mode (Opus + fastMode) applies an additional multiplier
  const multiplier = isFastMode ? (prices.fastModeMultiplier ?? 1) : 1

  return baseCost * multiplier
}
```

---

## 2. Cost Tracker

### 2.1 Data Structures

```typescript
// src/cost-tracker.ts
type CostEntry = {
  model: string
  inputTokens: number
  outputTokens: number
  cacheReadTokens: number
  cacheWriteTokens: number
  costUSD: number
  timestamp: number
}

type SessionCostState = {
  entries: CostEntry[]
  totalCostUSD: number
  totalInputTokens: number
  totalOutputTokens: number
  totalCacheReadTokens: number
  totalCacheWriteTokens: number
}
```

### 2.2 Persistence

Cost data is saved in the project config and supports **session resumption** (restoring previous cost stats on `--resume`):

```typescript
// Save path: .claude/costs.json (project-level)
// Updated after each API call

async function saveCostEntry(entry: CostEntry) {
  const existingData = await readProjectConfig('costs') ?? { entries: [] }
  existingData.entries.push(entry)
  existingData.totalCostUSD = (existingData.totalCostUSD ?? 0) + entry.costUSD
  await writeProjectConfig('costs', existingData)
}
```

### 2.3 costHook: React-Side Display

```typescript
// src/costHook.ts
export function useCost(): {
  totalCostUSD: number
  sessionTokens: TokenCounts
  formattedCost: string
} {
  const state = useAppState()
  const { totalCostUSD, ...tokens } = state.costState

  return {
    totalCostUSD,
    sessionTokens: tokens,
    formattedCost: formatCost(totalCostUSD),  // "$0.0234" format
  }
}

// Displayed in the REPL bottom status bar:
// "claude-opus-4-6 · ↑1.2k ↓0.8k tokens · $0.04"
```

---

## 3. Token Counting and Estimation

```typescript
// src/utils/tokens.ts

// Method 1: Exact count (read from API response)
function getExactTokenCount(response: APIResponse): TokenCounts {
  return {
    inputTokens:       response.usage.input_tokens,
    outputTokens:      response.usage.output_tokens,
    cacheReadTokens:   response.usage.cache_read_input_tokens ?? 0,
    cacheWriteTokens:  response.usage.cache_creation_input_tokens ?? 0,
  }
}

// Method 2: Rough estimation (for compaction threshold checks, no API call needed)
function roughTokenCountEstimation(text: string): number {
  // Rule: approximately 4 characters = 1 token (English)
  // Chinese/Japanese characters are typically ~1–2 tokens each
  return Math.ceil(text.length / 4)
}

// Token estimation for conversation history (used for autocompact threshold)
function tokenCountWithEstimation(messages: Message[]): number {
  let total = 0
  for (const msg of messages) {
    for (const block of msg.message.content ?? []) {
      if (block.type === 'text')         total += roughTokenCountEstimation(block.text)
      if (block.type === 'tool_result')  total += calculateToolResultTokens(block)
      if (block.type === 'image')        total += 2000  // IMAGE_MAX_TOKEN_SIZE
      if (block.type === 'thinking')     total += roughTokenCountEstimation(block.thinking)
      if (block.type === 'tool_use')     total += roughTokenCountEstimation(jsonStringify(block.input))
    }
  }
  return Math.ceil(total * (4 / 3))  // multiply by 4/3 as a conservative estimate
}
```

---

## 4. Conversation History (history.ts)

### 4.1 Storage Format

Conversation history is saved in **JSONL format** (one JSON object per line), at:
`~/.claude/history.jsonl` (globally shared, across all projects)

```typescript
// Format for each history entry
type HistoryEntry = {
  sessionId: string           // unique session ID (distinguishes different sessions)
  projectPath: string         // project path (distinguishes different projects)
  timestamp: number           // Unix millisecond timestamp
  prompt: string | null       // user input content (< 1024 bytes)
  pasteRef?: string           // external reference for large content (>= 1024 bytes)
}
```

### 4.2 External Storage for Large Paste Content

When a user pastes a large block of content (such as an entire file), it is not stored directly in the JSONL — instead it is written to a separate file:

```typescript
const PASTE_INLINE_THRESHOLD = 1024  // bytes

async function saveHistoryEntry(prompt: string, sessionId: string) {
  if (Buffer.byteLength(prompt, 'utf8') < PASTE_INLINE_THRESHOLD) {
    // Small content: store inline
    return { prompt, pasteRef: undefined }
  }

  // Large content: external storage
  const hash = crypto
    .createHash('sha256')
    .update(prompt)
    .digest('hex')
    .slice(0, 16)  // take first 16 characters (collision probability < 10^-10)

  const cacheDir = path.join(os.homedir(), '.claude', 'paste-cache')
  const filePath = path.join(cacheDir, `${hash}.txt`)

  await fs.writeFile(filePath, prompt, 'utf8')

  return { prompt: null, pasteRef: hash }
}

// On read: restore content from paste-cache/ using pasteRef
async function resolveHistoryEntry(entry: HistoryEntry): Promise<string> {
  if (entry.prompt !== null) return entry.prompt

  const filePath = path.join(os.homedir(), '.claude', 'paste-cache', `${entry.pasteRef}.txt`)
  return await fs.readFile(filePath, 'utf8')
}
```

### 4.3 Concurrency Protection

Multiple Claude Code instances may read and write history.jsonl simultaneously; file locking is used for protection:

```typescript
const LOCK_STALE_TIMEOUT = 10_000   // consider lock stale after 10 seconds (process crash scenario)
const LOCK_RETRY_COUNT   = 3        // retry acquiring lock up to 3 times

async function withHistoryLock<T>(fn: () => Promise<T>): Promise<T> {
  const lockPath = historyPath + '.lock'
  await lockFile.lock(lockPath, {
    stale: LOCK_STALE_TIMEOUT,
    retries: LOCK_RETRY_COUNT,
  })
  try {
    return await fn()
  } finally {
    await lockFile.unlock(lockPath)
  }
}
```

### 4.4 Capacity Management

```typescript
const MAX_HISTORY_ENTRIES = 100  // keep the most recent 100 entries (extensible)

// FIFO eviction: delete oldest entries when the limit is exceeded
// Note: files under paste-cache/ are NOT automatically cleaned up (requires manual or periodic cleanup)
```

---

## 5. Esc Key Undo Mechanism

Pressing Esc undoes the most recently added history entry (useful for accidentally submitted entries):

```typescript
// src/history.ts

// Dual-path design:
// - Fast path (< 5ms): if still in memory, delete the in-memory entry directly
// - Slow path (> 10ms): filter and delete from the on-disk JSONL file by timestamp

async function undoLastHistoryEntry(sessionId: string, timestamp: number) {
  // Fast path: in-memory deletion
  if (inMemoryEntries.has(timestamp)) {
    inMemoryEntries.delete(timestamp)
    return
  }

  // Slow path: on-disk filter
  await withHistoryLock(async () => {
    const lines = await readHistoryLines()
    const filtered = lines.filter(line => {
      const entry = JSON.parse(line)
      // Precisely locate by sessionId + timestamp to avoid deleting records from other sessions
      return !(entry.sessionId === sessionId && entry.timestamp === timestamp)
    })
    await writeHistoryLines(filtered)
  })
}
```

---

## 6. Cost Restoration on Session Resume

When using `--resume <sessionId>`, the cost tracker restores previous cost data from `.claude/costs.json`:

```typescript
// Restore flow
const previousCosts = await readProjectConfig('costs')
if (previousCosts) {
  setCostTrackerState({
    totalCostUSD: previousCosts.totalCostUSD,
    entries: previousCosts.entries,
  })
}
// The user sees the cumulative cost of this session + the previous session
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/cost-tracker.ts` | Cost accumulation and persistence (324 lines) |
| `src/costHook.ts` | React Hook for UI cost display |
| `src/history.ts` | Conversation history JSONL storage (465 lines) |
| `src/utils/model/modelCost.ts` | Model pricing table + cost calculation formula |
| `src/utils/tokens.ts` | Exact token counting and rough estimation algorithms |
| `~/.claude/history.jsonl` | Conversation history file (generated at runtime) |
| `~/.claude/paste-cache/` | External storage for large paste content (generated at runtime) |
