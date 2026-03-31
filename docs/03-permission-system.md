# 03 — 权限系统深度解析：7 层流水线

> 文件：`src/utils/permissions/`，`src/hooks/toolPermission/`，`src/hooks/useCanUseTool.tsx`，`src/utils/sandbox/`

---

## 1. 6 种权限模式

```typescript
// src/types/permissions.ts

type PermissionMode =
  | 'default'           // 每次询问用户（标准模式）
  | 'plan'              // 计划模式：只读自动通过，写操作询问
  | 'acceptEdits'       // 自动接受文件编辑，其他操作询问
  | 'auto'              // ML 分类器自动决策（需要 opt-in）
  | 'bypassPermissions' // 跳过所有权限检查（--dangerously-skip-permissions）
  | 'dontAsk'           // 自动允许，仍记录（非交互/CI 模式）
```

**切换方式：**

```bash
# CLI flag
claude --dangerously-skip-permissions

# 运行时
/mode plan          # 切换到计划模式
/mode default       # 切换回默认

# Bridge 远程控制
{ "type": "control_request", "request": { "subtype": "set_permission_mode", "mode": "auto" } }
```

---

## 2. 权限决策优先级

```
任何 deny 规则匹配 → 立刻拒绝（不继续检查）
没有 deny 匹配 → 检查 ask 规则
没有 ask 匹配 → 检查 allow 规则
没有任何规则 → 使用默认行为（询问用户）
```

---

## 3. 权限规则 Glob 匹配

### 3.1 规则格式

```
ToolName(pattern)

示例：
  Bash(git *)              → 允许所有 git 命令
  Read(~/*)                → 允许读取 home 目录
  Write(/tmp/*)            → 允许写入 /tmp
  Bash(rm -rf *)           → 拒绝危险删除
  Edit(*.ts)               → 允许编辑 TypeScript 文件
  Bash(*)                  → 通配所有 Bash 命令
```

### 3.2 匹配算法

```typescript
// src/utils/permissions/（节选）
import ignore from 'ignore'  // gitignore 格式的 glob 匹配库

function matchesRule(toolName: string, input: unknown, rule: string): boolean {
  const [ruleToolName, rulePattern] = parseRule(rule)

  // 工具名匹配
  if (ruleToolName !== toolName && ruleToolName !== '*') return false

  // Glob 匹配（使用 gitignore 语义）
  const ig = ignore().add(rulePattern)
  const inputStr = normalizeForMatching(input)

  return ig.ignores(inputStr)
}

function normalizeForMatching(input: unknown): string {
  const str = typeof input === 'string' ? input : JSON.stringify(input)
  // Windows 路径：C:\Users\foo → /c/users/foo（跨平台统一）
  return str.replace(/\\/g, '/').toLowerCase()
}
```

### 3.3 符号链接安全

```typescript
// 同时检查原始路径和解析后的真实路径
// 防止通过符号链接绕过 deny 规则

async function checkPathPermission(path: string, rules: Rules): boolean {
  const realPath = await fs.realpath(path).catch(() => path)
  return matchesAnyRule(path, rules) || matchesAnyRule(realPath, rules)
}
```

### 3.4 配置格式

```jsonc
// settings.json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Read(~/projects/*)",
      "Edit(*.ts)",
      "Write(/tmp/*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Write(/etc/*)"
    ]
  }
}
```

---

## 4. 完整 7 层流水线

```
工具调用请求
    │
    ├─ Layer 1: tool.validateInput()
    │           输入格式和语义校验（Zod schema + 路径合法性）
    │           失败 → 返回错误消息
    │
    ├─ Layer 2: 危险命令检测（工具内部）
    │           黑名单模式匹配（rm -rf /、fork bomb 等）
    │           命中 → 立刻拒绝，不询问用户
    │
    ├─ Layer 3: alwaysDenyRules 检查
    │           来自 settings.json 的 deny 规则（精确 Glob 匹配）
    │           命中 → 立刻拒绝
    │
    ├─ Layer 4: alwaysAllowRules 检查
    │           来自 settings.json 的 allow 规则
    │           命中 → 立刻允许，跳过后续层
    │
    ├─ Layer 5: [Auto 模式] ML 分类器
    │           toAutoClassifierInput(input) → 分类器 API
    │           恩典期 2 秒：批准 → 自动允许（用户无感知）
    │           超时/拒绝 → 继续下一层
    │
    ├─ Layer 6: shouldAvoidPermissionPrompts 检查
    │           非交互模式（CI/CD）→ 根据 permissionMode 自动决策
    │           dontAsk → 允许；default → 拒绝
    │
    └─ Layer 7: 弹出交互式确认框
                等待用户选择：Allow / Deny / Always Allow / Always Deny
                Always Allow → 写入 settings.json（持久化）
```

---

## 5. 权限决策实现（useCanUseTool）

```typescript
// src/hooks/useCanUseTool.tsx（203 行）

export function useCanUseTool(): CanUseToolFn {
  return useCallback(async (tool, input, context) => {

    // Layer 1: 输入校验
    const validationResult = tool.validateInput?.(input)
    if (validationResult?.result === false) {
      return { behavior: 'deny', message: validationResult.message }
    }

    // Layer 2: 工具自定义权限检查（危险命令检测在这里）
    const toolPermResult = await tool.checkPermissions(input, context)
    if (toolPermResult.behavior === 'deny') return toolPermResult

    // Layer 3 & 4: 配置规则检查
    const configResult = checkConfigRules(tool.name, input, context)
    if (configResult.behavior !== 'ask') return configResult

    // Layer 5: Auto 模式 ML 分类器
    if (permissionMode === 'auto') {
      const classifierInput = tool.toAutoClassifierInput(input)
      if (classifierInput !== '') {
        // 恩典期：给分类器 2 秒时间
        const classifierResult = await Promise.race([
          runAutoClassifier(classifierInput),
          sleep(2000).then(() => null),  // 超时 → null
        ])

        if (classifierResult?.approved) {
          return { behavior: 'allow', updatedInput: input }
        }
        // 未批准（拒绝或超时）→ 继续往下走
      }
    }

    // Layer 6: 非交互模式
    if (context.options.isNonInteractiveSession) {
      return autoDecideForNonInteractive(tool.name, permissionMode)
    }

    // Layer 7: 交互式确认弹框
    return await showPermissionDialog(tool, input, context)

  }, [permissionMode, ...deps])
}
```

---

## 6. Auto 模式 ML 分类器

### 6.1 工具接入分类器

```typescript
// 只有实现了非空 toAutoClassifierInput 的工具才会被分类器处理
// 默认（buildTool 提供）：() => ''（跳过分类器）

// BashTool 示例：
toAutoClassifierInput: (input) => input.command

// FileWriteTool 示例：
toAutoClassifierInput: (input) =>
  `${input.file_path}\n${input.content?.slice(0, 500)}`

// FileReadTool：
toAutoClassifierInput: () => ''  // 只读操作，不需要分类器
```

### 6.2 恩典期机制

```typescript
const CLASSIFIER_GRACE_PERIOD_MS = 2000

// 如果分类器在 2 秒内完成且批准 → 用户无感知自动通过
// 如果超时 → 用户看到确认弹框，同时分类器仍在后台运行
//   → 如果分类器后来批准了，不影响（用户已看到弹框）

const result = await Promise.race([
  runAutoClassifier(classifierInput),
  sleep(CLASSIFIER_GRACE_PERIOD_MS).then(() => null),
])

if (result?.approved) {
  return { behavior: 'allow' }
}
// 走交互式确认
```

### 6.3 DenialTracking（防循环）

```typescript
// src/utils/permissions/denialTracking.ts

type DenialTrackingState = {
  consecutiveDenials: Map<string, number>  // toolName → 连续拒绝次数
  totalDenials: number
}

// 触发条件：
// 连续拒绝 ≥ 3 次 OR 总计拒绝 ≥ 20 次
// → 回退到用户交互式确认（不再依赖分类器）
// → 防止分类器连续拒绝导致死循环

function shouldFallbackToInteractive(state: DenialTrackingState, toolName: string): boolean {
  const consecutive = state.consecutiveDenials.get(toolName) ?? 0
  return consecutive >= 3 || state.totalDenials >= 20
}
```

---

## 7. 危险命令检测

### 7.1 BashTool 黑名单

```typescript
// src/tools/BashTool/（节选检测逻辑）

const DANGEROUS_PATTERNS = [
  // 递归删除根目录或所有文件
  /rm\s+(-[rfR]+\s+)?\/(!\w)/,        // rm -rf /
  /rm\s+(-[rfR]+\s+)?\*$/,            // rm -rf *

  // Fork Bomb
  /:\(\)\s*\{.*:.*\|.*:.*\}/,         // :() { :|:& };:

  // 覆盖重要系统文件
  />\s*\/etc\/passwd/,
  />\s*\/etc\/shadow/,

  // 危险管道执行（远程代码执行）
  /curl.+\|\s*(ba)?sh/,               // curl | bash
  /wget.+\|\s*(ba)?sh/,               // wget | bash
]

function detectBlockedCommand(command: string): string | null {
  for (const pattern of DANGEROUS_PATTERNS) {
    if (pattern.test(command)) {
      return `命令被阻止：匹配危险模式 ${pattern}`
    }
  }
  return null  // 安全
}
```

### 7.2 Sleep 检测（防工具卡住）

```typescript
// 防止模型用超长 sleep 阻塞工具执行
function detectBlockedSleepPattern(command: string): boolean {
  const sleepMatch = command.match(/^\s*sleep\s+(\d+)\s*$/)
  if (sleepMatch) {
    const seconds = parseInt(sleepMatch[1])
    if (seconds > 300) return true  // 超过 5 分钟，阻止
  }
  return false
}
```

### 7.3 Windows 路径安全（7 种绕过攻击检测）

```typescript
function detectWindowsPathBypass(path: string): boolean {
  // 1. NTFS ADS（交替数据流）：file.txt:secret
  if (/:\w+$/.test(path)) return true

  // 2. 8.3 短文件名：PROGRA~1 （绕过 deny 规则）
  if (/~\d/.test(path)) return true

  // 3. 长路径前缀：\\?\C:\... （跳过 MAX_PATH 检查）
  if (path.startsWith('\\\\?\\')) return true

  // 4. 尾部点（Windows 忽略）：file.txt.
  if (/\.$/.test(path)) return true

  // 5. DOS 设备名：CON, PRN, AUX, NUL, COM1-9, LPT1-9
  if (/^(CON|PRN|AUX|NUL|COM\d|LPT\d)$/i.test(path)) return true

  // 6. 三连点：C:\...\file （跨多级目录）
  if (/\.\.\./.test(path)) return true

  // 7. UNC 路径：\\server\share （网络路径）
  if (path.startsWith('\\\\')) return true

  return false
}
```

---

## 8. Sandbox：工作目录限制

Claude Code 的沙箱不是容器级别的隔离，而是**工作目录限制**：

```typescript
// src/utils/sandbox/

type PermissionContext = {
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  // 默认：只有 getCwd()（当前工作目录）
  // 用户可通过 /add-dir 命令添加更多允许目录
}

// 文件操作前的路径检查
function isPathAllowed(path: string, context: ToolPermissionContext): boolean {
  const allowedDirs = [
    getCwd(),
    ...context.additionalWorkingDirectories.keys(),
  ]

  const resolvedPath = fs.realpathSync(path)  // 解析符号链接

  return allowedDirs.some(dir =>
    resolvedPath.startsWith(resolveWorkingDir(dir))
  )
}
```

### 8.1 路径安全校验（4 种非法路径）

```typescript
function validatePath(inputPath: string): ValidationResult {
  // 1. 拒绝相对路径（必须是绝对路径）
  if (!path.isAbsolute(inputPath)) {
    return { result: false, message: '路径必须是绝对路径', errorCode: 1 }
  }

  // 2. 拒绝根目录（/）
  if (inputPath === '/') {
    return { result: false, message: '不允许操作根目录', errorCode: 2 }
  }

  // 3. 拒绝空字节注入（null byte attack：/etc/pass\x00wd）
  if (inputPath.includes('\0')) {
    return { result: false, message: '路径包含非法字符', errorCode: 3 }
  }

  // 4. Windows：拒绝 UNC 路径
  if (process.platform === 'win32' && inputPath.startsWith('\\\\')) {
    return { result: false, message: 'UNC 路径不被允许', errorCode: 4 }
  }

  return { result: true }
}
```

---

## 9. 权限缓存（会话内记忆）

为了避免重复询问同一个操作，权限决策在会话内缓存：

```typescript
type ToolDecision = {
  source: string           // 决策来源：'user', 'classifier', 'rule'
  decision: 'accept' | 'reject'
  timestamp: number
}

// 存储在 ToolUseContext 中
toolDecisions: Map<string, ToolDecision>

// 缓存 key = hash(toolName + JSON.stringify(input))

// 用户选择 "Always Allow" → 将规则持久化到 settings.json
//   → 下次启动时从配置读取，永久生效
```

---

## 10. Plan 模式的特殊逻辑

```typescript
// plan 模式：只读操作自动通过，写操作需要询问
if (permissionMode === 'plan') {
  if (tool.isReadOnly(input)) {
    return { behavior: 'allow' }
  }
  // 写操作 → 弹框询问，但带"计划模式"提示
}

// 同时，plan 模式下模型使用更大上下文的变体：
// 如果最近 assistant 消息超过 200k tokens，切换到支持 200k 的模型
const currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens: permissionMode === 'plan' && doesMostRecentAssistantMessageExceed200k(messages),
})
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/hooks/useCanUseTool.tsx` | 权限决策 Hook，完整 7 层（203 行）|
| `src/utils/permissions/` | 权限规则解析、Glob 匹配（60+ 文件）|
| `src/utils/permissions/denialTracking.ts` | DenialTracking 防循环 |
| `src/hooks/toolPermission/` | 权限决策流水线实现 |
| `src/tools/BashTool/` | 危险命令检测、sleep 检测 |
| `src/utils/sandbox/` | 工作目录隔离、路径校验 |
| `src/types/permissions.ts` | PermissionMode、PermissionResult 类型定义 |
