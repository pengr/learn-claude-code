# 11 — Cost Tracking 与对话历史

> 文件：`src/cost-tracker.ts`，`src/costHook.ts`，`src/history.ts`，`src/utils/model/modelCost.ts`

---

## 1. 模型定价表

Claude Code 内置了所有模型的定价数据（每百万 token 的 USD 价格）：

```typescript
// src/utils/model/modelCost.ts（节选）
const MODEL_PRICES: Record<string, ModelPricing> = {
  // 格式：input / output / cache_read（10% 折扣）/ cache_write（125% 溢价）

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
    // 快速模式（fastMode）成本乘数：6x
    fastModeMultiplier: 6,
  },
}
```

### 成本计算公式

```typescript
function calculateCost(usage: TokenUsage, model: string): number {
  const prices = MODEL_PRICES[normalizeModelName(model)]
  if (!prices) return 0

  const inputCost  = (usage.input_tokens  / 1_000_000) * prices.input
  const outputCost = (usage.output_tokens / 1_000_000) * prices.output
  const cacheReadCost  = (usage.cache_read_input_tokens  ?? 0) / 1_000_000 * prices.cache_read
  const cacheWriteCost = (usage.cache_creation_input_tokens ?? 0) / 1_000_000 * prices.cache_write

  const baseCost = inputCost + outputCost + cacheReadCost + cacheWriteCost

  // 快速模式（Opus + fastMode）有额外乘数
  const multiplier = isFastMode ? (prices.fastModeMultiplier ?? 1) : 1

  return baseCost * multiplier
}
```

---

## 2. Cost Tracker

### 2.1 数据结构

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

### 2.2 持久化

成本数据保存在项目配置中，支持**会话恢复**（`--resume` 时恢复上次的成本统计）：

```typescript
// 保存路径：.claude/costs.json（项目级）
// 每次 API 调用后更新

async function saveCostEntry(entry: CostEntry) {
  const existingData = await readProjectConfig('costs') ?? { entries: [] }
  existingData.entries.push(entry)
  existingData.totalCostUSD = (existingData.totalCostUSD ?? 0) + entry.costUSD
  await writeProjectConfig('costs', existingData)
}
```

### 2.3 costHook：React 侧展示

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
    formattedCost: formatCost(totalCostUSD),  // "$0.0234" 格式
  }
}

// 在 REPL 底部状态栏显示：
// "claude-opus-4-6 · ↑1.2k ↓0.8k tokens · $0.04"
```

---

## 3. Token 计数与估算

```typescript
// src/utils/tokens.ts

// 方法一：精确计数（从 API 响应中读取）
function getExactTokenCount(response: APIResponse): TokenCounts {
  return {
    inputTokens:       response.usage.input_tokens,
    outputTokens:      response.usage.output_tokens,
    cacheReadTokens:   response.usage.cache_read_input_tokens ?? 0,
    cacheWriteTokens:  response.usage.cache_creation_input_tokens ?? 0,
  }
}

// 方法二：粗估（用于 compaction 阈值判断，无需 API 调用）
function roughTokenCountEstimation(text: string): number {
  // 规则：约每 4 个字符 = 1 个 token（英文）
  // 中文/日文等字符通常每个字符 ≈ 1~2 个 token
  return Math.ceil(text.length / 4)
}

// 对话历史的 token 估算（用于 autocompact 阈值）
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
  return Math.ceil(total * (4 / 3))  // 乘以 4/3 作为保守估算
}
```

---

## 4. 对话历史（history.ts）

### 4.1 存储格式

对话历史保存为 **JSONL 格式**（每行一个 JSON 对象），路径：
`~/.claude/history.jsonl`（全局共享，跨所有项目）

```typescript
// 每条历史记录的格式
type HistoryEntry = {
  sessionId: string           // 会话唯一 ID（区分不同会话）
  projectPath: string         // 项目路径（区分不同项目）
  timestamp: number           // Unix 毫秒时间戳
  prompt: string | null       // 用户输入内容（< 1024 字节）
  pasteRef?: string           // 大内容的外部引用（>= 1024 字节时）
}
```

### 4.2 大粘贴内容的外部存储

当用户粘贴大段内容（如整个文件）时，不直接存入 JSONL，而是存到独立文件：

```typescript
const PASTE_INLINE_THRESHOLD = 1024  // 字节

async function saveHistoryEntry(prompt: string, sessionId: string) {
  if (Buffer.byteLength(prompt, 'utf8') < PASTE_INLINE_THRESHOLD) {
    // 小内容：直接内联存储
    return { prompt, pasteRef: undefined }
  }

  // 大内容：外部存储
  const hash = crypto
    .createHash('sha256')
    .update(prompt)
    .digest('hex')
    .slice(0, 16)  // 取前 16 字符（碰撞概率 < 10^-10）

  const cacheDir = path.join(os.homedir(), '.claude', 'paste-cache')
  const filePath = path.join(cacheDir, `${hash}.txt`)

  await fs.writeFile(filePath, prompt, 'utf8')

  return { prompt: null, pasteRef: hash }
}

// 读取时：根据 pasteRef 从 paste-cache/ 还原内容
async function resolveHistoryEntry(entry: HistoryEntry): Promise<string> {
  if (entry.prompt !== null) return entry.prompt

  const filePath = path.join(os.homedir(), '.claude', 'paste-cache', `${entry.pasteRef}.txt`)
  return await fs.readFile(filePath, 'utf8')
}
```

### 4.3 并发保护

多个 Claude Code 实例可能同时读写 history.jsonl，使用文件锁保护：

```typescript
const LOCK_STALE_TIMEOUT = 10_000   // 10 秒后认为锁已过期（进程崩溃场景）
const LOCK_RETRY_COUNT   = 3        // 最多重试 3 次获取锁

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

### 4.4 容量管理

```typescript
const MAX_HISTORY_ENTRIES = 100  // 保留最近 100 条（可扩展）

// FIFO 淘汰：超出上限时删除最旧的条目
// 但 paste-cache/ 下的文件不自动清理（需要用户手动或定期清理）
```

---

## 5. Esc 键撤销机制

按 Esc 键可撤销刚刚添加的历史条目（适用于误提交的场景）：

```typescript
// src/history.ts

// 双路径设计：
// - 快速路径（< 5ms）：如果还在内存中，直接删除内存条目
// - 延迟路径（> 10ms）：从磁盘 JSONL 文件中按时间戳过滤删除

async function undoLastHistoryEntry(sessionId: string, timestamp: number) {
  // 快速路径：内存删除
  if (inMemoryEntries.has(timestamp)) {
    inMemoryEntries.delete(timestamp)
    return
  }

  // 延迟路径：磁盘标记过滤
  await withHistoryLock(async () => {
    const lines = await readHistoryLines()
    const filtered = lines.filter(line => {
      const entry = JSON.parse(line)
      // 通过 sessionId + 时间戳精确定位，防止误删其他会话的记录
      return !(entry.sessionId === sessionId && entry.timestamp === timestamp)
    })
    await writeHistoryLines(filtered)
  })
}
```

---

## 6. 会话恢复时的成本还原

`--resume <sessionId>` 时，cost tracker 会从 `.claude/costs.json` 恢复上次的成本数据：

```typescript
// 恢复流程
const previousCosts = await readProjectConfig('costs')
if (previousCosts) {
  setCostTrackerState({
    totalCostUSD: previousCosts.totalCostUSD,
    entries: previousCosts.entries,
  })
}
// 用户看到的是本次会话 + 上次会话的累计成本
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/cost-tracker.ts` | 成本累计、持久化（324 行） |
| `src/costHook.ts` | React Hook，供 UI 读取成本 |
| `src/history.ts` | 对话历史 JSONL 存储（465 行） |
| `src/utils/model/modelCost.ts` | 模型定价表 + 成本计算公式 |
| `src/utils/tokens.ts` | Token 精确计数和粗估算法 |
| `~/.claude/history.jsonl` | 对话历史文件（运行时生成） |
| `~/.claude/paste-cache/` | 大粘贴内容外部存储（运行时生成） |
