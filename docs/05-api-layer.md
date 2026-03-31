# 05 — API 层：鉴权、网络与重试

> 文件：`src/services/api/`，`src/services/oauth/`，`src/services/policyLimits/`

---

## 1. 四种后端支持

Claude Code 的 API 客户端工厂 `getAnthropicClient()` 根据环境变量自动选择后端：

```
环境变量检测顺序：

1. ANTHROPIC_BEDROCK_BASE_URL 或 AWS_BEDROCK_BASE_URL
   → AWS Bedrock 客户端

2. ANTHROPIC_FOUNDRY_RESOURCE 或 ANTHROPIC_FOUNDRY_BASE_URL
   → Azure AI Foundry 客户端

3. ANTHROPIC_VERTEX_PROJECT_ID 或 CLOUD_ML_PROJECT_ID
   → Google Vertex AI 客户端

4. 以上都没有
   → 直连 Anthropic API（ANTHROPIC_API_KEY 或 OAuth Token）
```

### 1.1 直连 API（最常用）

```typescript
// src/services/api/client.ts
const client = new Anthropic({
  apiKey: resolvedApiKey,               // ANTHROPIC_API_KEY 或 OAuth Bearer Token
  defaultHeaders: {
    'anthropic-version': '2023-06-01',
    'user-agent': `claude-code/${VERSION}`,
    ...betaHeaders,                      // 实验性功能的 beta 头
  },
  maxRetries: 0,                         // 重试由 withRetry.ts 自己管理
  fetch: fetchOverride ?? globalThis.fetch,
})
```

### 1.2 AWS Bedrock

```typescript
import AnthropicBedrock from '@anthropic-ai/bedrock-sdk'

const client = new AnthropicBedrock({
  awsRegion: process.env.AWS_REGION ?? 'us-east-1',
  // 凭证优先级：显式 key > 环境变量 > IAM Role（自动发现）
  awsAccessKey: process.env.AWS_ACCESS_KEY_ID,
  awsSecretKey: process.env.AWS_SECRET_ACCESS_KEY,
  awsSessionToken: process.env.AWS_SESSION_TOKEN,
})
// 模型名格式：anthropic.claude-opus-4-5-20251101-v1:0
```

### 1.3 Azure AI Foundry

```typescript
import AnthropicFoundry from '@anthropic-ai/foundry-sdk'

const client = new AnthropicFoundry({
  // 方式A：资源名（自动构建 baseURL）
  resource: process.env.ANTHROPIC_FOUNDRY_RESOURCE,
  // 方式B：直接 baseURL
  baseURL: process.env.ANTHROPIC_FOUNDRY_BASE_URL,
  apiKey: process.env.ANTHROPIC_FOUNDRY_API_KEY,   // Azure API Key
})
```

### 1.4 Google Vertex AI

```typescript
import AnthropicVertex from '@anthropic-ai/vertex-sdk'

const client = new AnthropicVertex({
  projectId: process.env.ANTHROPIC_VERTEX_PROJECT_ID ?? process.env.CLOUD_ML_PROJECT_ID,
  region: process.env.CLOUD_ML_REGION ?? 'us-east5',
  // 凭证：Application Default Credentials (gcloud auth / 服务账号)
  googleAuth: new GoogleAuth({
    scopes: ['https://www.googleapis.com/auth/cloud-platform'],
  }),
})
```

---

## 2. OAuth 2.0 PKCE 鉴权流程

Claude Code 登录使用标准 OAuth 2.0 + PKCE（Proof Key for Code Exchange），无需在本地存储密码。

```
完整登录流程（7步）：

  1. 生成 PKCE
     code_verifier = random(43 chars, base64url)
     code_challenge = base64url(SHA256(code_verifier))

  2. 启动本地 HTTP 服务器（监听随机端口，等待回调）

  3. 打开浏览器
     URL = https://claude.ai/oauth/authorize
           ?client_id=...
           &response_type=code
           &code_challenge=<hash>
           &code_challenge_method=S256
           &redirect_uri=http://localhost:<port>/callback
           &scope=openid profile email

  4. 用户在浏览器中授权

  5. 浏览器重定向回 localhost:<port>/callback?code=<auth_code>

  6. 本地服务器捕获 code，用 code + code_verifier 换取 token
     POST /oauth/token
     { grant_type: 'authorization_code', code, code_verifier }

  7. 收到 OAuthTokens { access_token, refresh_token, expires_at }
     保存到系统 keychain（macOS Keychain / Linux Secret Service / Windows WDPAPI）
```

### Token 刷新机制

```typescript
// src/services/oauth/client.ts
async function refreshTokenIfNeeded(tokens: OAuthTokens): Promise<OAuthTokens> {
  // 提前 5 分钟刷新，避免使用中过期
  const expiresAt = new Date(tokens.expires_at)
  const shouldRefresh = expiresAt.getTime() - Date.now() < 5 * 60 * 1000

  if (!shouldRefresh) return tokens

  const response = await fetch('/oauth/token', {
    method: 'POST',
    body: JSON.stringify({
      grant_type: 'refresh_token',
      refresh_token: tokens.refresh_token,
    }),
  })
  return await response.json()
}
```

---

## 3. Prompt Caching

Claude Code 大量使用 Anthropic 的 Prompt Caching 功能来降低 API 费用（缓存命中时仅需 10% 费用）。

### 3.1 cache_control 的放置位置

```typescript
// 规则：在每个"稳定前缀"的最后一个内容块上加 cache_control
function buildMessagesWithCache(messages: Message[], querySource: string) {
  // system prompt 的最后一个块
  systemPrompt = [
    { type: 'text', text: staticSystemPrompt },
    {
      type: 'text',
      text: dynamicMemoryContent,
      cache_control: getCacheControl({ querySource }),  // ← 缓存点
    }
  ]

  // 对话历史中，较旧消息的最后一个块
  for (const msg of olderMessages) {
    lastBlock.cache_control = getCacheControl()  // ← 缓存点
  }
  // 最近 2 轮消息不加缓存（频繁变化）
}

function getCacheControl({ scope, querySource } = {}) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 某些场景用 1 小时 TTL
    ...(scope === 'global' && { scope: 'global' }),        // 跨用户共享缓存（Ant-only）
  }
}
```

### 3.2 不可缓存的块类型

以下类型的内容块不会加 `cache_control`：
- `thinking` / `redacted_thinking`（思考过程，每次不同）
- `connector_text`（连接词）

### 3.3 缓存破裂检测

`src/services/api/promptCacheBreakDetection.ts` 会追踪当前会话的 `cache_read_input_tokens`：
- 如果某轮请求缓存命中率骤降，说明 prompt 结构发生了意外变化
- 会记录日志并上报 analytics，帮助工程师发现缓存失效 bug

---

## 4. 错误重试（withRetry.ts）

`src/services/api/withRetry.ts`（823 行）实现了完整的指数退避重试逻辑，它本身也是一个 **AsyncGenerator**。

### 4.1 重试延迟算法

```typescript
function calculateRetryDelay(attempt: number, retryAfterHeader?: string): number {
  // 优先使用服务器返回的 Retry-After 头
  if (retryAfterHeader) {
    const retryAfterMs = parseRetryAfter(retryAfterHeader)
    if (retryAfterMs) return retryAfterMs
  }

  // 指数退避 + 随机抖动
  const baseDelay = 500  // ms
  const exponential = baseDelay * Math.pow(2, attempt - 1)
  const jitter = exponential * 0.25 * Math.random()  // ±25% 随机
  const delay = exponential + jitter

  return Math.min(delay, 32_000)  // 上限 32 秒
}
```

### 4.2 重试决策树

```
收到错误
  │
  ├─ 401 Unauthorized
  │    └─ 尝试刷新 OAuth token → 重试一次
  │
  ├─ 429 Rate Limited
  │    └─ 读取 anthropic-ratelimit-unified-reset 头（Unix 时间戳）
  │       等待到该时间点 → 重试
  │
  ├─ 529 Overloaded（Anthropic 服务过载）
  │    ├─ 交互模式：最多重试 3 次（MAX_529_RETRIES）
  │    └─ 后台/Agent 模式：立即失败（不阻塞用户）
  │
  ├─ 5xx Server Error
  │    └─ 指数退避重试
  │
  └─ 其他错误（4xx）
       └─ 直接失败，不重试
```

### 4.3 持久重试模式（无人值守）

当检测到用户不在场时（非交互式会话），进入持久模式：

```typescript
const PERSISTENT_MAX_RETRY_DURATION = 6 * 60 * 60 * 1000  // 6 小时总上限
const PERSISTENT_MAX_BACKOFF        = 5 * 60 * 1000        // 单次最长等待 5 分钟

// 适用于：--print 模式、Agent 子任务、CI/CD 环境
```

### 4.4 模型降级（Fallback）

```typescript
// 当某模型连续 529 次数超过阈值时，自动降级
if (consecutive529Count >= MODEL_FALLBACK_THRESHOLD) {
  // Opus → Sonnet（如果用户允许）
  // Sonnet → Haiku（如果用户允许）
  onStreamingFallback?.()  // 通知 query.ts 触发 Tombstone 机制
}
```

---

## 5. Rate Limiting & Policy Limits

### 5.1 请求头解析

```typescript
// 从响应头读取限流信息
const rateLimitReset = response.headers.get('anthropic-ratelimit-unified-reset')
// 值格式：Unix 秒时间戳，例如 "1735000000"

const retryAfter = response.headers.get('retry-after')
// 值格式：秒数（"30"）或 HTTP 日期

const shouldRetry = response.headers.get('x-should-retry')
// 值：'true' | 'false'（服务端建议是否重试）
```

### 5.2 Policy Limits 服务

`src/services/policyLimits/index.ts` 管理企业/团队账户的配额限制：

```typescript
// 运行机制
// 1. 启动时从后端拉取 policy limits（含 ETag 缓存）
// 2. 后台每小时轮询一次
// 3. 使用 If-None-Match 避免重复传输

type PolicyLimits = {
  maxTokensPerDay?: number
  maxRequestsPerDay?: number
  allowedModels?: string[]
  enforceRateLimit?: boolean
}

// 适用对象：
// - 企业版 OAuth 账户
// - 团队账户
// - Console API Key 用户（受组织策略约束）
```

---

## 6. 请求构造详情

### 6.1 Beta Headers

Claude Code 会根据使用的功能动态添加 beta 标志：

```typescript
const betaHeaders: string[] = []

if (useThinking) betaHeaders.push('interleaved-thinking-2025-05-14')
if (useFiles)    betaHeaders.push('files-api-2025-04-14')
if (usePDFs)     betaHeaders.push('pdfs-2024-09-25')
if (useMCP)      betaHeaders.push('mcp-client-2025-04-04')
// ...等约 10 个 beta 功能
```

### 6.2 CLAUDE_CODE_EXTRA_BODY

高级用户可通过环境变量注入额外请求体字段：

```bash
export CLAUDE_CODE_EXTRA_BODY='{"some_experimental_field": true}'
```

### 6.3 流式请求格式

```typescript
// 所有请求都使用流式（stream: true）
const stream = await client.messages.stream({
  model: 'claude-opus-4-5-20251101',
  max_tokens: 8192,
  system: [{ type: 'text', text: '...', cache_control: { type: 'ephemeral' } }],
  messages: normalizedMessages,
  tools: toolDefinitions,
  stream: true,
  // 通过 extra_body 传递额外参数
  ...getExtraBodyParams(betaHeaders),
})

// 异步消费 stream
for await (const chunk of stream) {
  // chunk.type: 'content_block_delta' | 'message_delta' | ...
}
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `services/api/client.ts` | `getAnthropicClient()` 工厂，4 种后端实现 |
| `services/api/withRetry.ts` | 错误重试 + 指数退避（823 行） |
| `services/api/claude.ts` | 主要 API 调用逻辑，缓存控制生成 |
| `services/api/errors.ts` | 错误类型定义和识别 |
| `services/api/promptCacheBreakDetection.ts` | 缓存破裂检测 |
| `services/oauth/index.ts` | OAuth PKCE 流程入口 |
| `services/oauth/client.ts` | Token 交换和刷新（567 行） |
| `services/oauth/crypto.ts` | PKCE 加密（`generateCodeVerifier` / `generateCodeChallenge`） |
| `services/policyLimits/index.ts` | 企业配额服务（664 行） |
