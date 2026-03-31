# 05 — API Layer: Authentication, Networking & Retries

> Files: `src/services/api/`, `src/services/oauth/`, `src/services/policyLimits/`

---

## 1. Four Supported Backends

Claude Code's API client factory `getAnthropicClient()` automatically selects a backend based on environment variables:

```
Environment variable detection order:

1. ANTHROPIC_BEDROCK_BASE_URL or AWS_BEDROCK_BASE_URL
   → AWS Bedrock client

2. ANTHROPIC_FOUNDRY_RESOURCE or ANTHROPIC_FOUNDRY_BASE_URL
   → Azure AI Foundry client

3. ANTHROPIC_VERTEX_PROJECT_ID or CLOUD_ML_PROJECT_ID
   → Google Vertex AI client

4. None of the above
   → Direct Anthropic API (ANTHROPIC_API_KEY or OAuth Token)
```

### 1.1 Direct API (Most Common)

```typescript
// src/services/api/client.ts
const client = new Anthropic({
  apiKey: resolvedApiKey,               // ANTHROPIC_API_KEY or OAuth Bearer Token
  defaultHeaders: {
    'anthropic-version': '2023-06-01',
    'user-agent': `claude-code/${VERSION}`,
    ...betaHeaders,                      // Beta headers for experimental features
  },
  maxRetries: 0,                         // Retries are managed by withRetry.ts
  fetch: fetchOverride ?? globalThis.fetch,
})
```

### 1.2 AWS Bedrock

```typescript
import AnthropicBedrock from '@anthropic-ai/bedrock-sdk'

const client = new AnthropicBedrock({
  awsRegion: process.env.AWS_REGION ?? 'us-east-1',
  // Credential priority: explicit key > env vars > IAM Role (auto-discovered)
  awsAccessKey: process.env.AWS_ACCESS_KEY_ID,
  awsSecretKey: process.env.AWS_SECRET_ACCESS_KEY,
  awsSessionToken: process.env.AWS_SESSION_TOKEN,
})
// Model name format: anthropic.claude-opus-4-5-20251101-v1:0
```

### 1.3 Azure AI Foundry

```typescript
import AnthropicFoundry from '@anthropic-ai/foundry-sdk'

const client = new AnthropicFoundry({
  // Option A: resource name (baseURL is constructed automatically)
  resource: process.env.ANTHROPIC_FOUNDRY_RESOURCE,
  // Option B: direct baseURL
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
  // Credentials: Application Default Credentials (gcloud auth / service account)
  googleAuth: new GoogleAuth({
    scopes: ['https://www.googleapis.com/auth/cloud-platform'],
  }),
})
```

---

## 2. OAuth 2.0 PKCE Authentication Flow

Claude Code login uses standard OAuth 2.0 + PKCE (Proof Key for Code Exchange), eliminating the need to store passwords locally.

```
Full login flow (7 steps):

  1. Generate PKCE
     code_verifier = random(43 chars, base64url)
     code_challenge = base64url(SHA256(code_verifier))

  2. Start local HTTP server (listening on a random port, waiting for callback)

  3. Open browser
     URL = https://claude.ai/oauth/authorize
           ?client_id=...
           &response_type=code
           &code_challenge=<hash>
           &code_challenge_method=S256
           &redirect_uri=http://localhost:<port>/callback
           &scope=openid profile email

  4. User authorizes in the browser

  5. Browser redirects back to localhost:<port>/callback?code=<auth_code>

  6. Local server captures code, exchanges code + code_verifier for a token
     POST /oauth/token
     { grant_type: 'authorization_code', code, code_verifier }

  7. Receive OAuthTokens { access_token, refresh_token, expires_at }
     Save to system keychain (macOS Keychain / Linux Secret Service / Windows WDPAPI)
```

### Token Refresh Mechanism

```typescript
// src/services/oauth/client.ts
async function refreshTokenIfNeeded(tokens: OAuthTokens): Promise<OAuthTokens> {
  // Refresh 5 minutes early to avoid expiry during active use
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

Claude Code makes heavy use of Anthropic's Prompt Caching feature to reduce API costs (only 10% of the cost on cache hits).

### 3.1 Placement of cache_control

```typescript
// Rule: add cache_control to the last content block of each "stable prefix"
function buildMessagesWithCache(messages: Message[], querySource: string) {
  // Last block of the system prompt
  systemPrompt = [
    { type: 'text', text: staticSystemPrompt },
    {
      type: 'text',
      text: dynamicMemoryContent,
      cache_control: getCacheControl({ querySource }),  // ← cache point
    }
  ]

  // Last block of older messages in conversation history
  for (const msg of olderMessages) {
    lastBlock.cache_control = getCacheControl()  // ← cache point
  }
  // Most recent 2 turns are not cached (change too frequently)
}

function getCacheControl({ scope, querySource } = {}) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 1-hour TTL for some scenarios
    ...(scope === 'global' && { scope: 'global' }),        // Cross-user shared cache (Ant-only)
  }
}
```

### 3.2 Non-cacheable Block Types

The following content block types will not have `cache_control` applied:
- `thinking` / `redacted_thinking` (reasoning process, differs each time)
- `connector_text` (connector words)

### 3.3 Cache Break Detection

`src/services/api/promptCacheBreakDetection.ts` tracks `cache_read_input_tokens` for the current session:
- If the cache hit rate drops sharply in a request, it means the prompt structure has unexpectedly changed
- This is logged and reported to analytics, helping engineers detect cache invalidation bugs

---

## 4. Error Retries (withRetry.ts)

`src/services/api/withRetry.ts` (823 lines) implements full exponential backoff retry logic, and is itself an **AsyncGenerator**.

### 4.1 Retry Delay Algorithm

```typescript
function calculateRetryDelay(attempt: number, retryAfterHeader?: string): number {
  // Prefer the server-returned Retry-After header
  if (retryAfterHeader) {
    const retryAfterMs = parseRetryAfter(retryAfterHeader)
    if (retryAfterMs) return retryAfterMs
  }

  // Exponential backoff + random jitter
  const baseDelay = 500  // ms
  const exponential = baseDelay * Math.pow(2, attempt - 1)
  const jitter = exponential * 0.25 * Math.random()  // ±25% random
  const delay = exponential + jitter

  return Math.min(delay, 32_000)  // cap at 32 seconds
}
```

### 4.2 Retry Decision Tree

```
Receive error
  │
  ├─ 401 Unauthorized
  │    └─ Attempt to refresh OAuth token → retry once
  │
  ├─ 429 Rate Limited
  │    └─ Read anthropic-ratelimit-unified-reset header (Unix timestamp)
  │       Wait until that point in time → retry
  │
  ├─ 529 Overloaded (Anthropic service overloaded)
  │    ├─ Interactive mode: retry up to 3 times (MAX_529_RETRIES)
  │    └─ Background/Agent mode: fail immediately (don't block the user)
  │
  ├─ 5xx Server Error
  │    └─ Exponential backoff retry
  │
  └─ Other errors (4xx)
       └─ Fail immediately, no retry
```

### 4.3 Persistent Retry Mode (Unattended)

When the user is detected to be absent (non-interactive session), enter persistent mode:

```typescript
const PERSISTENT_MAX_RETRY_DURATION = 6 * 60 * 60 * 1000  // 6-hour total cap
const PERSISTENT_MAX_BACKOFF        = 5 * 60 * 1000        // max 5-minute wait per attempt

// Applies to: --print mode, Agent subtasks, CI/CD environments
```

### 4.4 Model Fallback

```typescript
// When a model's consecutive 529 count exceeds the threshold, automatically fall back
if (consecutive529Count >= MODEL_FALLBACK_THRESHOLD) {
  // Opus → Sonnet (if user permits)
  // Sonnet → Haiku (if user permits)
  onStreamingFallback?.()  // notify query.ts to trigger Tombstone mechanism
}
```

---

## 5. Rate Limiting & Policy Limits

### 5.1 Response Header Parsing

```typescript
// Read rate limit info from response headers
const rateLimitReset = response.headers.get('anthropic-ratelimit-unified-reset')
// Value format: Unix second timestamp, e.g. "1735000000"

const retryAfter = response.headers.get('retry-after')
// Value format: seconds ("30") or HTTP date

const shouldRetry = response.headers.get('x-should-retry')
// Value: 'true' | 'false' (server recommendation on whether to retry)
```

### 5.2 Policy Limits Service

`src/services/policyLimits/index.ts` manages quota limits for enterprise/team accounts:

```typescript
// How it works
// 1. Fetch policy limits from backend at startup (with ETag caching)
// 2. Poll once per hour in the background
// 3. Use If-None-Match to avoid redundant transfers

type PolicyLimits = {
  maxTokensPerDay?: number
  maxRequestsPerDay?: number
  allowedModels?: string[]
  enforceRateLimit?: boolean
}

// Applies to:
// - Enterprise OAuth accounts
// - Team accounts
// - Console API Key users (subject to org policies)
```

---

## 6. Request Construction Details

### 6.1 Beta Headers

Claude Code dynamically adds beta flags based on the features in use:

```typescript
const betaHeaders: string[] = []

if (useThinking) betaHeaders.push('interleaved-thinking-2025-05-14')
if (useFiles)    betaHeaders.push('files-api-2025-04-14')
if (usePDFs)     betaHeaders.push('pdfs-2024-09-25')
if (useMCP)      betaHeaders.push('mcp-client-2025-04-04')
// ...approximately 10 beta features total
```

### 6.2 CLAUDE_CODE_EXTRA_BODY

Advanced users can inject extra request body fields via an environment variable:

```bash
export CLAUDE_CODE_EXTRA_BODY='{"some_experimental_field": true}'
```

### 6.3 Streaming Request Format

```typescript
// All requests use streaming (stream: true)
const stream = await client.messages.stream({
  model: 'claude-opus-4-5-20251101',
  max_tokens: 8192,
  system: [{ type: 'text', text: '...', cache_control: { type: 'ephemeral' } }],
  messages: normalizedMessages,
  tools: toolDefinitions,
  stream: true,
  // Pass extra parameters via extra_body
  ...getExtraBodyParams(betaHeaders),
})

// Consume stream asynchronously
for await (const chunk of stream) {
  // chunk.type: 'content_block_delta' | 'message_delta' | ...
}
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `services/api/client.ts` | `getAnthropicClient()` factory, 4 backend implementations |
| `services/api/withRetry.ts` | Error retry + exponential backoff (823 lines) |
| `services/api/claude.ts` | Main API call logic, cache control generation |
| `services/api/errors.ts` | Error type definitions and recognition |
| `services/api/promptCacheBreakDetection.ts` | Cache break detection |
| `services/oauth/index.ts` | OAuth PKCE flow entry point |
| `services/oauth/client.ts` | Token exchange and refresh (567 lines) |
| `services/oauth/crypto.ts` | PKCE cryptography (`generateCodeVerifier` / `generateCodeChallenge`) |
| `services/policyLimits/index.ts` | Enterprise quota service (664 lines) |
