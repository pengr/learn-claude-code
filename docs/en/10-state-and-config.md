# 10 — State Management, Settings Configuration, and Migrations

> Files: `src/state/`, `src/utils/settings/`, `src/migrations/`, `src/bootstrap/`

---

## 1. AppState: Global Application State

Claude Code uses a custom **immutable state container** (similar to Zustand, but lighter), centered on `src/state/AppStateStore.ts`.

### 1.1 Store Implementation

```typescript
// src/state/store.ts
type Store<T> = {
  getState(): T
  setState(updater: (prev: T) => T): void   // must return a new object (immutable)
  subscribe(listener: (state: T) => void): () => void  // returns unsubscribe function
}

function createStore<T>(initial: T, onChange?: (prev: T, next: T) => void): Store<T> {
  let state = initial
  const listeners = new Set<(state: T) => void>()

  return {
    getState: () => state,
    setState: (updater) => {
      const next = updater(state)
      if (Object.is(state, next)) return  // skip if reference-equal, avoids meaningless updates
      const prev = state
      state = next
      onChange?.(prev, next)             // trigger side effects
      for (const listener of listeners) listener(next)
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 1.2 AppState Main Fields

AppState contains **450+ fields**, grouped by module:

```typescript
type AppState = {
  // ── Basic Configuration ────────────────────────────
  settings: Settings               // merged configuration (see Section 2)
  model: string                    // currently active model
  verbose: boolean                 // whether verbose output is enabled
  fastMode: boolean                // whether fast mode is enabled (uses a smaller model)
  effortValue: EffortValue         // thinking depth (low/medium/high/max)
  advisorModel?: string            // advisor model for dual-model mode

  // ── Permissions ────────────────────────────────────
  toolPermissionContext: ToolPermissionContext  // permission rules + current mode

  // ── MCP ────────────────────────────────────────────
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]                  // all MCP tools (dynamically registered)
  }

  // ── Plugins ────────────────────────────────────────
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    errors: PluginError[]
  }

  // ── Task System ────────────────────────────────────
  tasks: Map<string, Task>
  agentNameRegistry: Map<string, string>  // agentId → display name

  // ── UI State ───────────────────────────────────────
  expandedView: boolean            // whether the detailed view is expanded
  footerSelection: string          // currently selected footer item

  // ── REPL Bridge (Remote Control) ───────────────────
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  replBridgeError?: string
  replBridgeSessionId?: string
  // ... ~10 bridge-related fields

  // ── Team & Agents ──────────────────────────────────
  teamContext?: TeamContext
  standaloneAgentContext?: AgentContext
}
```

### 1.3 Immutable Update Pattern

```typescript
// ✅ Correct: always return a new object
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    mode: 'plan',
  },
}))

// ❌ Wrong: direct mutation, will not trigger updates
setAppState(prev => {
  prev.model = 'claude-3-5-sonnet'  // has no effect!
  return prev
})
```

### 1.4 onChangeAppState Side-Effect Chain

When state changes, `src/state/onChangeAppState.ts` triggers a series of side effects:

```
State field change         Side effect triggered
─────────────────────    ─────────────────────────────────────────
model               → updateSettingsForSource('userSettings')  save to settings.json
toolPermissionContext.mode
                    → notifySessionMetadataChanged()            notify Bridge
expandedView        → saveGlobalConfig()                       persist UI preference
settings.env        → applyConfigEnvironmentVariables()        update process environment variables
settings.*          → clearAuthCaches()                        clear auth caches
```

---

## 2. Settings: Multi-Layer Configuration System

### 2.1 Five Priority Layers (later layers override earlier ones)

```
Priority  Source              Path
──────    ────────────────    ──────────────────────────────────────────
  1       userSettings        ~/.claude/settings.json
  2       projectSettings     .claude/settings.json
  3       localSettings       .claude/settings.local.json
  4       flagSettings        --settings <path>  (CLI argument)
  5       policySettings      /etc/claude-code/managed-settings.json
                              /etc/claude-code/settings.d/*.json  (drop-in directory)
```

### 2.2 Settings Schema Main Fields

```typescript
// Full configurable fields in settings.json (excerpted)
type Settings = {
  // Model & Performance
  model?: string                     // default model
  effortLevel?: EffortValue          // thinking depth
  smallFastModel?: string            // fast mode model

  // Permissions
  permissions?: {
    allow?: string[]                 // always-allow rules ["Bash(git *)"]
    deny?: string[]                  // always-deny rules ["Bash(rm -rf *)"]
    ask?: string[]                   // always-ask rules
    defaultMode?: PermissionMode     // default permission mode
  }

  // Hooks
  hooks?: {
    PreToolUse?: HookConfig[]        // before tool execution
    PostToolUse?: HookConfig[]       // after tool execution
    Notification?: HookConfig[]      // on notification
    Stop?: HookConfig[]              // on stop
    SessionStart?: HookConfig[]      // on session start
  }

  // Environment Variables
  env?: Record<string, string>       // environment variables injected into subprocesses

  // MCP
  mcpServers?: Record<string, McpServerConfig>

  // Plugins
  enabledPlugins?: Record<string, boolean>

  // Auto Updates
  autoUpdates?: boolean

  // Custom Prompts
  customSystemPrompt?: string        // replaces the default system prompt
  appendSystemPrompt?: string        // appended to the default system prompt

  // Developer Options
  includeCoAuthoredBy?: boolean      // whether to add Co-authored-by on commits
  cleanupPeriodDays?: number         // history cleanup interval
}
```

### 2.3 Configuration Loading Flow

```typescript
function loadSettings(): Settings {
  // 1. Load from each source (with ETag caching to avoid redundant disk reads)
  const sources = getEnabledSettingSources()
  const allSettings = sources.map(source => getSettingsForSource(source))

  // 2. Deep merge (array fields concatenated, scalar fields overridden)
  const merged = mergeWith({}, ...allSettings, settingsMergeCustomizer)

  // 3. Zod schema validation (invalid fields produce warnings, do not interrupt startup)
  const result = SettingsSchema.safeParse(merged)
  if (!result.success) {
    logSettingsWarnings(result.error)
    return merged as Settings  // fault-tolerant: continue with unvalidated values
  }

  return result.data
}
```

### 2.4 Hot Reload

Configuration file changes **take effect automatically** — no restart required:

```
Filesystem watch (Chokidar)
    │
    ▼
settings.json change detected
    │
    ▼
resetSettingsCache()    clear in-memory cache
    │
    ▼
Re-merge all sources
    │
    ▼
setAppState(prev => ({ ...prev, settings: newSettings }))
    │
    ▼
onChangeAppState() triggers relevant side effects
(e.g., re-apply environment variables, clear auth caches)
```

---

## 3. Migrations: Version Upgrades

### 3.1 Current Version Number

```typescript
// src/main.tsx
const CURRENT_MIGRATION_VERSION = 11
```

### 3.2 Full Migration List

| Version | Migration Function | Description |
|---------|-------------------|-------------|
| v1 | `migrateAutoUpdatesToSettings()` | Migrate auto-update settings |
| v2 | `migrateBypassPermissionsAcceptedToSettings()` | Migrate permission bypass settings |
| v3 | `migrateEnableAllProjectMcpServersToSettings()` | Migrate MCP server enable settings |
| v4 | `resetProToOpusDefault()` | Reset default model for Pro accounts |
| v5 | `migrateSonnet1mToSonnet45()` | Update model name |
| v6 | `migrateLegacyOpusToCurrent()` | Migrate legacy Opus versions |
| v7 | `migrateSonnet45ToSonnet46()` | Sonnet 4.5 → 4.6 |
| v8 | `migrateOpusToOpus1m()` | Opus → Opus 1M context version |
| v9 | `migrateReplBridgeEnabledToRemoteControlAtStartup()` | Rename Bridge settings field |
| v10 | `resetAutoModeOptInForDefaultOffer()` | Reset Auto mode opt-in (condition: TRANSCRIPT_CLASSIFIER feature) |
| v11 | `migrateFennecToOpus()` | Migrate Fennec internal codename (Ant internal only) |

### 3.3 Execution Mechanism

```typescript
// Execution timing: preAction middleware before main() (early in CLI startup)
// Guarantee: all migrations complete before any user interaction

async function runMigrations() {
  const currentVersion = await getGlobalConfigValue('migrationVersion') ?? 0

  for (let v = currentVersion + 1; v <= CURRENT_MIGRATION_VERSION; v++) {
    try {
      await runMigration(v)               // execute corresponding migration
      await setGlobalConfigValue('migrationVersion', v)  // record version
      logEvent('tengu_migration_ran', { version: v })
    } catch (error) {
      logError(error)                     // capture error, do not interrupt startup
      // A failed migration does not prevent Claude Code from starting
    }
  }
}
```

**Design principles:**
- ✅ **Idempotent**: safe to run multiple times (the entire migration sequence can be safely re-run)
- ✅ **Sequential**: v1 → v2 → ... → v11, guaranteeing dependency order
- ✅ **Failure-tolerant**: a single migration failure does not block startup, only logs are recorded
- ✅ **Conditional migrations**: some migrations are guarded by feature flags

---

## 4. Bootstrap Initialization Order

`src/bootstrap/` manages the initialization of global state at application startup:

```
CLI process starts
    │
    ▼
1. Parallel preloads (non-blocking)
   - startMdmRawRead()          MDM enterprise policy
   - startKeychainPrefetch()    OAuth token + API key
    │
    ▼
2. Run Migrations (preAction)
    │
    ▼
3. Initialize global state (bootstrap/)
   - setOriginalCwd()           record original working directory
   - setSessionId()             generate session ID for this run
   - initCostTracker()          initialize cost tracker
   - initTokenCounters()        initialize token counters
    │
    ▼
4. Load configuration (read all settings sources)
    │
    ▼
5. Authentication (OAuth / API key / Bedrock / Vertex)
    │
    ▼
6. Load tools + commands + MCP servers
    │
    ▼
7. Start REPL
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/state/AppStateStore.ts` | AppState type definition (450+ fields) |
| `src/state/store.ts` | Store<T> immutable state container implementation |
| `src/state/onChangeAppState.ts` | State change side-effect chain |
| `src/utils/settings/` | Settings loading, merging, and validation |
| `src/utils/settings/applySettingsChange.ts` | Hot reload handling |
| `src/migrations/` | 11 migration functions |
| `src/bootstrap/` | Global state initialization (56 files) |
| `src/main.tsx` | `CURRENT_MIGRATION_VERSION` definition and call entry point |
