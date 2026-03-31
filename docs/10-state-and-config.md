# 10 — State 管理、Settings 配置与 Migrations

> 文件：`src/state/`，`src/utils/settings/`，`src/migrations/`，`src/bootstrap/`

---

## 1. AppState：全局应用状态

Claude Code 使用自定义的 **不可变状态容器**（类似 Zustand，但更轻量），核心在 `src/state/AppStateStore.ts`。

### 1.1 Store 实现

```typescript
// src/state/store.ts
type Store<T> = {
  getState(): T
  setState(updater: (prev: T) => T): void   // 必须返回新对象（不可变）
  subscribe(listener: (state: T) => void): () => void  // 返回 unsubscribe 函数
}

function createStore<T>(initial: T, onChange?: (prev: T, next: T) => void): Store<T> {
  let state = initial
  const listeners = new Set<(state: T) => void>()

  return {
    getState: () => state,
    setState: (updater) => {
      const next = updater(state)
      if (Object.is(state, next)) return  // 引用相等则跳过，避免无意义更新
      const prev = state
      state = next
      onChange?.(prev, next)             // 触发副作用
      for (const listener of listeners) listener(next)
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 1.2 AppState 主要字段

AppState 包含 **450+ 个字段**，按模块分组：

```typescript
type AppState = {
  // ── 基础配置 ──────────────────────────────────────
  settings: Settings               // 合并后的配置（见第2节）
  model: string                    // 当前使用的模型
  verbose: boolean                 // 是否详细输出
  fastMode: boolean                // 是否启用快速模式（用更小的模型）
  effortValue: EffortValue         // 思考深度（low/medium/high/max）
  advisorModel?: string            // 双模型模式的顾问模型

  // ── 权限 ──────────────────────────────────────────
  toolPermissionContext: ToolPermissionContext  // 权限规则 + 当前模式

  // ── MCP ───────────────────────────────────────────
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]                  // 所有 MCP 工具（动态注册）
  }

  // ── 插件 ──────────────────────────────────────────
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    errors: PluginError[]
  }

  // ── 任务系统 ──────────────────────────────────────
  tasks: Map<string, Task>
  agentNameRegistry: Map<string, string>  // agentId → 显示名称

  // ── UI 状态 ───────────────────────────────────────
  expandedView: boolean            // 是否展开详细视图
  footerSelection: string          // 底部选中项

  // ── REPL Bridge（远程控制）────────────────────────
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  replBridgeError?: string
  replBridgeSessionId?: string
  // ... 约 10 个 bridge 相关字段

  // ── 团队 & 代理 ───────────────────────────────────
  teamContext?: TeamContext
  standaloneAgentContext?: AgentContext
}
```

### 1.3 不可变更新模式

```typescript
// ✅ 正确：每次返回新对象
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    mode: 'plan',
  },
}))

// ❌ 错误：直接修改，不会触发更新
setAppState(prev => {
  prev.model = 'claude-3-5-sonnet'  // 不会生效！
  return prev
})
```

### 1.4 onChangeAppState 副作用链

状态变化时，`src/state/onChangeAppState.ts` 会触发一系列副作用：

```
状态字段变化          触发的副作用
─────────────────    ─────────────────────────────────────────
model               → updateSettingsForSource('userSettings')  保存到 settings.json
toolPermissionContext.mode
                    → notifySessionMetadataChanged()            通知 Bridge
expandedView        → saveGlobalConfig()                       持久化 UI 偏好
settings.env        → applyConfigEnvironmentVariables()        更新进程环境变量
settings.*          → clearAuthCaches()                        清除鉴权缓存
```

---

## 2. Settings：多层配置系统

### 2.1 五层优先级（后面覆盖前面）

```
优先级  来源              路径
──────  ────────────────  ──────────────────────────────────────────
  1     userSettings      ~/.claude/settings.json
  2     projectSettings   .claude/settings.json
  3     localSettings     .claude/settings.local.json
  4     flagSettings      --settings <path>（CLI 参数指定）
  5     policySettings    /etc/claude-code/managed-settings.json
                          /etc/claude-code/settings.d/*.json（drop-in 目录）
```

### 2.2 Settings Schema 主要字段

```typescript
// settings.json 完整可配置项（节选）
type Settings = {
  // 模型与性能
  model?: string                     // 默认模型
  effortLevel?: EffortValue          // 思考深度
  smallFastModel?: string            // 快速模式模型

  // 权限
  permissions?: {
    allow?: string[]                 // 永远允许的规则 ["Bash(git *)"]
    deny?: string[]                  // 永远拒绝的规则 ["Bash(rm -rf *)"]
    ask?: string[]                   // 每次询问的规则
    defaultMode?: PermissionMode     // 默认权限模式
  }

  // Hooks（钩子）
  hooks?: {
    PreToolUse?: HookConfig[]        // 工具执行前
    PostToolUse?: HookConfig[]       // 工具执行后
    Notification?: HookConfig[]      // 通知时
    Stop?: HookConfig[]              // 停止时
    SessionStart?: HookConfig[]      // 会话开始时
  }

  // 环境变量
  env?: Record<string, string>       // 注入到子进程的环境变量

  // MCP
  mcpServers?: Record<string, McpServerConfig>

  // 插件
  enabledPlugins?: Record<string, boolean>

  // 自动更新
  autoUpdates?: boolean

  // 自定义提示词
  customSystemPrompt?: string        // 替换默认 system prompt
  appendSystemPrompt?: string        // 追加到默认 system prompt

  // 开发者选项
  includeCoAuthoredBy?: boolean      // 提交时是否加 Co-authored-by
  cleanupPeriodDays?: number         // 历史清理周期
}
```

### 2.3 配置加载流程

```typescript
function loadSettings(): Settings {
  // 1. 从各来源加载（有 ETag 缓存，避免重复读磁盘）
  const sources = getEnabledSettingSources()
  const allSettings = sources.map(source => getSettingsForSource(source))

  // 2. 深度合并（数组字段拼接，标量字段覆盖）
  const merged = mergeWith({}, ...allSettings, settingsMergeCustomizer)

  // 3. Zod schema 验证（无效字段会产生警告，不中断启动）
  const result = SettingsSchema.safeParse(merged)
  if (!result.success) {
    logSettingsWarnings(result.error)
    return merged as Settings  // 容错：继续使用未验证的值
  }

  return result.data
}
```

### 2.4 热加载

配置文件修改后**自动生效**，无需重启：

```
文件系统 watch（Chokidar）
    │
    ▼
检测到 settings.json 变更
    │
    ▼
resetSettingsCache()    清除内存缓存
    │
    ▼
重新合并所有来源
    │
    ▼
setAppState(prev => ({ ...prev, settings: newSettings }))
    │
    ▼
onChangeAppState() 触发相关副作用
（如：重新应用环境变量、清除鉴权缓存）
```

---

## 3. Migrations：版本迁移

### 3.1 当前版本号

```typescript
// src/main.tsx
const CURRENT_MIGRATION_VERSION = 11
```

### 3.2 完整迁移列表

| 版本 | 迁移函数 | 说明 |
|------|----------|------|
| v1 | `migrateAutoUpdatesToSettings()` | 自动更新设置迁移 |
| v2 | `migrateBypassPermissionsAcceptedToSettings()` | 权限绕过设置迁移 |
| v3 | `migrateEnableAllProjectMcpServersToSettings()` | MCP 服务器启用设置迁移 |
| v4 | `resetProToOpusDefault()` | Pro 账户默认模型重置 |
| v5 | `migrateSonnet1mToSonnet45()` | 模型名更新 |
| v6 | `migrateLegacyOpusToCurrent()` | Opus 旧版本迁移 |
| v7 | `migrateSonnet45ToSonnet46()` | Sonnet 4.5 → 4.6 |
| v8 | `migrateOpusToOpus1m()` | Opus → Opus 1M 上下文版本 |
| v9 | `migrateReplBridgeEnabledToRemoteControlAtStartup()` | Bridge 设置字段重命名 |
| v10 | `resetAutoModeOptInForDefaultOffer()` | Auto 模式 opt-in 重置（条件：TRANSCRIPT_CLASSIFIER feature） |
| v11 | `migrateFennecToOpus()` | Fennec 内部代号迁移（仅 Ant 内部） |

### 3.3 执行机制

```typescript
// 执行时机：main() 之前的 preAction 中间件（CLI 启动早期）
// 保证：在任何用户交互前完成所有迁移

async function runMigrations() {
  const currentVersion = await getGlobalConfigValue('migrationVersion') ?? 0

  for (let v = currentVersion + 1; v <= CURRENT_MIGRATION_VERSION; v++) {
    try {
      await runMigration(v)               // 执行对应迁移
      await setGlobalConfigValue('migrationVersion', v)  // 记录版本
      logEvent('tengu_migration_ran', { version: v })
    } catch (error) {
      logError(error)                     // 捕获错误，不中断启动
      // 失败的迁移不阻止 Claude Code 启动
    }
  }
}
```

**设计原则：**
- ✅ **幂等**：重复运行无害（可以安全地重跑整个迁移序列）
- ✅ **顺序执行**：v1 → v2 → ... → v11，保证依赖顺序
- ✅ **失败容错**：单个迁移失败不阻断启动，只记录日志
- ✅ **条件迁移**：部分迁移有 feature flag 守卫

---

## 4. Bootstrap 初始化顺序

`src/bootstrap/` 管理应用启动时的全局状态初始化：

```
CLI 进程启动
    │
    ▼
1. 并行预加载（不阻塞主流程）
   - startMdmRawRead()          MDM 企业策略
   - startKeychainPrefetch()    OAuth token + API key
    │
    ▼
2. 运行 Migrations（preAction）
    │
    ▼
3. 初始化全局状态（bootstrap/）
   - setOriginalCwd()           记录原始工作目录
   - setSessionId()             生成本次会话 ID
   - initCostTracker()          初始化成本追踪
   - initTokenCounters()        初始化 token 计数
    │
    ▼
4. 加载配置（读取所有 settings 来源）
    │
    ▼
5. 鉴权（OAuth / API key / Bedrock / Vertex）
    │
    ▼
6. 加载工具 + 命令 + MCP servers
    │
    ▼
7. 启动 REPL
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/state/AppStateStore.ts` | AppState 类型定义（450+ 字段） |
| `src/state/store.ts` | Store<T> 不可变状态容器实现 |
| `src/state/onChangeAppState.ts` | 状态变化副作用链 |
| `src/utils/settings/` | Settings 加载、合并、验证 |
| `src/utils/settings/applySettingsChange.ts` | 热加载处理 |
| `src/migrations/` | 11 个迁移函数 |
| `src/bootstrap/` | 全局状态初始化（56 个文件） |
| `src/main.tsx` | `CURRENT_MIGRATION_VERSION` 定义和调用入口 |
