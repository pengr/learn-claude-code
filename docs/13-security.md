# 13 — 安全机制：权限规则、危险命令检测与 Sandbox

> 文件：`src/utils/permissions/`，`src/hooks/toolPermission/`，`src/utils/sandbox/`

---

## 1. 权限模式（6 种）

```typescript
type PermissionMode =
  | 'default'            // 每次询问用户（标准模式）
  | 'plan'               // 计划模式：只读自动通过，写操作询问
  | 'acceptEdits'        // 自动接受文件编辑，其他操作询问
  | 'auto'               // ML 分类器自动决策（需要 opt-in）
  | 'bypassPermissions'  // 跳过所有权限检查（--dangerously-skip-permissions）
  | 'dontAsk'            // 自动允许，但仍记录（非交互模式）
```

**决策优先级（Deny > Ask > Allow）：**

```
任何 deny 规则匹配 → 立刻拒绝（不继续检查）
没有 deny 匹配 → 检查 ask 规则
没有 ask 匹配 → 检查 allow 规则
没有任何规则 → 使用默认行为（询问用户）
```

---

## 2. 权限规则 Glob 匹配

### 2.1 规则格式

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

### 2.2 匹配算法

```typescript
// 使用 ignore 库（gitignore 格式的 glob 匹配）
import ignore from 'ignore'

function matchesRule(toolName: string, input: unknown, rule: string): boolean {
  // 1. 解析规则：ToolName(pattern)
  const [ruleToolName, rulePattern] = parseRule(rule)

  // 2. 工具名匹配
  if (ruleToolName !== toolName && ruleToolName !== '*') return false

  // 3. 模式匹配
  const ig = ignore().add(rulePattern)

  // 关键：统一转换为 POSIX 路径（跨平台）
  const inputStr = normalizeForMatching(input)

  return ig.ignores(inputStr)
}

function normalizeForMatching(input: unknown): string {
  const str = typeof input === 'string' ? input : JSON.stringify(input)
  // Windows 路径：C:\Users\foo → /c/users/foo
  return str.replace(/\\/g, '/').toLowerCase()
}
```

### 2.3 符号链接安全

```typescript
// 检查原始路径 AND 解析后的路径
// 防止通过符号链接绕过 deny 规则

async function checkPathPermission(path: string, rules: Rules): boolean {
  const realPath = await fs.realpath(path).catch(() => path)

  // 两个路径都要检查
  return matchesAnyRule(path, rules) || matchesAnyRule(realPath, rules)
}
```

---

## 3. 危险命令检测

### 3.1 Bash 命令黑名单

```typescript
// src/tools/BashTool/（节选真实检测逻辑）

const DANGEROUS_PATTERNS = [
  // 递归删除
  /rm\s+(-[rfR]+\s+)?\/(?!\w)/,        // rm -rf /
  /rm\s+(-[rfR]+\s+)?\*$/,             // rm -rf *

  // Fork Bomb
  /:\(\)\s*\{.*:.*\|.*:.*\}/,          // :() { :|:& };:

  // 覆盖重要文件
  />\s*\/etc\/passwd/,
  />\s*\/etc\/shadow/,

  // 危险的管道执行
  /curl.+\|\s*(ba)?sh/,               // curl | bash
  /wget.+\|\s*(ba)?sh/,               // wget | bash

  // 长时间 sleep（防止工具卡住）
  // 独立的 sleep > 300 秒会被检测并阻止
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

### 3.2 特殊规则：Sleep 检测

```typescript
// 防止模型用 sleep 阻塞工具执行
function detectBlockedSleepPattern(command: string): boolean {
  // 检测独立的 sleep 命令（不是管道或条件的一部分）
  const sleepMatch = command.match(/^\s*sleep\s+(\d+)\s*$/)
  if (sleepMatch) {
    const seconds = parseInt(sleepMatch[1])
    if (seconds > 300) {  // 超过 5 分钟
      return true  // 阻止
    }
  }
  return false
}
```

### 3.3 Windows 路径安全（7 种绕过检测）

```typescript
// 检测 Windows NTFS 特有的路径绕过技巧
function detectWindowsPathBypass(path: string): boolean {
  // 1. NTFS ADS（Alternative Data Streams）: file.txt:secret
  if (/:\w+$/.test(path)) return true

  // 2. 8.3 短文件名：PROGRA~1
  if (/~\d/.test(path)) return true

  // 3. 长路径前缀：\\?\C:\...
  if (path.startsWith('\\\\?\\')) return true

  // 4. 尾部点（Windows 会忽略）：file.txt.
  if (/\.$/.test(path)) return true

  // 5. DOS 设备名：CON, PRN, AUX, NUL, COM1-9, LPT1-9
  if (/^(CON|PRN|AUX|NUL|COM\d|LPT\d)$/i.test(path)) return true

  // 6. 三连点：C:\...\file（跨越多级目录）
  if (/\.\.\./.test(path)) return true

  // 7. UNC 路径：\\server\share
  if (path.startsWith('\\\\')) return true

  return false
}
```

---

## 4. Auto 模式 ML 分类器

### 4.1 接口设计

```typescript
// 工具定义自己的分类器输入
// 安全相关的工具必须实现此方法（否则默认跳过分类器）

type Tool = {
  // 返回分类器使用的字符串表示
  // 空字符串 = 跳过分类器（buildTool 的默认值）
  toAutoClassifierInput(input: z.infer<Input>): unknown
}

// BashTool 示例
toAutoClassifierInput: (input) => input.command

// FileWriteTool 示例
toAutoClassifierInput: (input) => `${input.file_path}\n${input.content?.slice(0, 500)}`
```

### 4.2 恩典期机制

```typescript
// 恩典期：给分类器 2 秒完成的机会
// 如果分类器批准 → 无感知自动通过
// 如果超时 → 等待用户手动确认

const CLASSIFIER_GRACE_PERIOD_MS = 2000

const result = await Promise.race([
  runAutoClassifier(classifierInput),       // 分类器请求
  sleep(CLASSIFIER_GRACE_PERIOD_MS).then(() => null),  // 超时
])

if (result?.approved) {
  return { behavior: 'allow' }  // 快速路径
}
// 超时或拒绝 → 走交互式确认流程
```

### 4.3 DenialTracking（防止 Auto 模式循环）

```typescript
// 追踪 Auto 模式的拒绝历史
// 防止分类器连续拒绝导致无限询问

type DenialTrackingState = {
  consecutiveDenials: Map<string, number>  // toolName → 连续拒绝次数
  totalDenials: number
}

// 触发条件：
// 连续拒绝 ≥ 3 次 OR 总计拒绝 ≥ 20 次
// → 回退到用户交互式确认（不再依赖分类器）

function shouldFallbackToInteractive(state: DenialTrackingState, toolName: string): boolean {
  const consecutive = state.consecutiveDenials.get(toolName) ?? 0
  return consecutive >= 3 || state.totalDenials >= 20
}
```

---

## 5. 权限决策完整流水线（7 层）

```
工具调用请求
    │
    ├─ Layer 1: tool.validateInput()
    │           输入格式和语义校验（如路径合法性）
    │
    ├─ Layer 2: 危险命令检测（工具内部）
    │           黑名单模式匹配（rm -rf /、fork bomb 等）
    │
    ├─ Layer 3: alwaysDenyRules 检查
    │           来自 settings.json 的 deny 规则
    │           匹配 → 立刻拒绝
    │
    ├─ Layer 4: alwaysAllowRules 检查
    │           来自 settings.json 的 allow 规则
    │           匹配 → 立刻允许
    │
    ├─ Layer 5: [Auto 模式] ML 分类器
    │           toAutoClassifierInput → 分类器 API
    │           恩典期 2 秒：批准 → 自动允许
    │
    ├─ Layer 6: shouldAvoidPermissionPrompts 检查
    │           非交互模式（CI/CD）→ 根据 permissionMode 自动决策
    │
    └─ Layer 7: 弹出交互式确认框
                等待用户 Allow / Deny / Always Allow / Always Deny
```

---

## 6. Sandbox 执行隔离

Claude Code 的沙箱不是容器级别的隔离，而是**工作目录限制 + 路径校验**：

```typescript
// 核心约束：工具只能操作"允许的工作目录"内的文件

type PermissionContext = {
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  // 默认：只有 getCwd()（当前工作目录）
  // 用户可通过 /add-dir 命令添加更多允许的目录
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

### 6.1 路径安全校验

```typescript
// 防止路径遍历攻击（../../../etc/passwd 等）
function validatePath(inputPath: string): ValidationResult {
  // 1. 拒绝相对路径（必须是绝对路径）
  if (!path.isAbsolute(inputPath)) {
    return { result: false, message: '路径必须是绝对路径', errorCode: 1 }
  }

  // 2. 拒绝根目录
  if (inputPath === '/') {
    return { result: false, message: '不允许操作根目录', errorCode: 2 }
  }

  // 3. 拒绝空字节（null byte 注入攻击）
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

## 7. 权限缓存

为了避免重复询问同一个操作，权限决策会在会话内缓存：

```typescript
type ToolDecision = {
  source: string           // 决策来源（'user', 'classifier', 'rule'）
  decision: 'accept' | 'reject'
  timestamp: number
}

// 存储在 ToolUseContext 中
toolDecisions: Map<string, ToolDecision>

// 缓存 key = hash(toolName + JSON.stringify(input))
// 用户选择 "Always Allow" 时，将规则持久化到 settings.json
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/utils/permissions/` | 权限规则解析、Glob 匹配（60+ 文件） |
| `src/utils/permissions/denialTracking.ts` | DenialTracking 防循环 |
| `src/hooks/toolPermission/` | 权限决策流水线实现 |
| `src/hooks/useCanUseTool.tsx` | React 侧权限 Hook |
| `src/tools/BashTool/` | 危险命令检测、sleep 检测 |
| `src/utils/sandbox/` | 工作目录隔离、路径校验 |
| `src/types/permissions.ts` | PermissionMode、PermissionResult 类型定义 |
