# 03 — Permission System Deep Dive: The 7-Layer Pipeline

> Files: `src/utils/permissions/`, `src/hooks/toolPermission/`, `src/hooks/useCanUseTool.tsx`, `src/utils/sandbox/`

---

## 1. 6 Permission Modes

```typescript
// src/types/permissions.ts

type PermissionMode =
  | 'default'           // prompt the user each time (standard mode)
  | 'plan'              // plan mode: read-only ops auto-approved, write ops prompt the user
  | 'acceptEdits'       // auto-accept file edits, prompt for other operations
  | 'auto'              // ML classifier makes decisions automatically (requires opt-in)
  | 'bypassPermissions' // skip all permission checks (--dangerously-skip-permissions)
  | 'dontAsk'           // auto-allow, still logged (non-interactive / CI mode)
```

**How to switch:**

```bash
# CLI flag
claude --dangerously-skip-permissions

# At runtime
/mode plan          # switch to plan mode
/mode default       # switch back to default

# Bridge remote control
{ "type": "control_request", "request": { "subtype": "set_permission_mode", "mode": "auto" } }
```

---

## 2. Permission Decision Priority

```
Any deny rule matches → reject immediately (no further checks)
No deny match → check ask rules
No ask match → check allow rules
No rules at all → use default behavior (prompt the user)
```

---

## 3. Permission Rule Glob Matching

### 3.1 Rule Format

```
ToolName(pattern)

Examples:
  Bash(git *)              → allow all git commands
  Read(~/*)                → allow reading the home directory
  Write(/tmp/*)            → allow writing to /tmp
  Bash(rm -rf *)           → deny dangerous deletion
  Edit(*.ts)               → allow editing TypeScript files
  Bash(*)                  → wildcard for all Bash commands
```

### 3.2 Matching Algorithm

```typescript
// src/utils/permissions/ (excerpted)
import ignore from 'ignore'  // glob matching library with gitignore semantics

function matchesRule(toolName: string, input: unknown, rule: string): boolean {
  const [ruleToolName, rulePattern] = parseRule(rule)

  // Tool name match
  if (ruleToolName !== toolName && ruleToolName !== '*') return false

  // Glob match (using gitignore semantics)
  const ig = ignore().add(rulePattern)
  const inputStr = normalizeForMatching(input)

  return ig.ignores(inputStr)
}

function normalizeForMatching(input: unknown): string {
  const str = typeof input === 'string' ? input : JSON.stringify(input)
  // Windows paths: C:\Users\foo → /c/users/foo (cross-platform normalization)
  return str.replace(/\\/g, '/').toLowerCase()
}
```

### 3.3 Symlink Safety

```typescript
// Check both the original path and the resolved real path
// Prevents bypassing deny rules via symlinks

async function checkPathPermission(path: string, rules: Rules): boolean {
  const realPath = await fs.realpath(path).catch(() => path)
  return matchesAnyRule(path, rules) || matchesAnyRule(realPath, rules)
}
```

### 3.4 Configuration Format

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

## 4. Complete 7-Layer Pipeline

```
Tool call request
    │
    ├─ Layer 1: tool.validateInput()
    │           Input format and semantic validation (Zod schema + path validity)
    │           Failure → return error message
    │
    ├─ Layer 2: Dangerous command detection (inside the tool)
    │           Blocklist pattern matching (rm -rf /, fork bombs, etc.)
    │           Match → reject immediately, do not prompt the user
    │
    ├─ Layer 3: alwaysDenyRules check
    │           Deny rules from settings.json (exact Glob matching)
    │           Match → reject immediately
    │
    ├─ Layer 4: alwaysAllowRules check
    │           Allow rules from settings.json
    │           Match → allow immediately, skip subsequent layers
    │
    ├─ Layer 5: [Auto mode] ML classifier
    │           toAutoClassifierInput(input) → classifier API
    │           Grace period 2s: approved → auto-allow (user unaware)
    │           Timeout/rejected → continue to next layer
    │
    ├─ Layer 6: shouldAvoidPermissionPrompts check
    │           Non-interactive mode (CI/CD) → auto-decide based on permissionMode
    │           dontAsk → allow;  default → reject
    │
    └─ Layer 7: Show interactive confirmation dialog
                Wait for user choice: Allow / Deny / Always Allow / Always Deny
                Always Allow → write to settings.json (persisted)
```

---

## 5. Permission Decision Implementation (useCanUseTool)

```typescript
// src/hooks/useCanUseTool.tsx (203 lines)

export function useCanUseTool(): CanUseToolFn {
  return useCallback(async (tool, input, context) => {

    // Layer 1: input validation
    const validationResult = tool.validateInput?.(input)
    if (validationResult?.result === false) {
      return { behavior: 'deny', message: validationResult.message }
    }

    // Layer 2: tool-specific permission check (dangerous command detection happens here)
    const toolPermResult = await tool.checkPermissions(input, context)
    if (toolPermResult.behavior === 'deny') return toolPermResult

    // Layer 3 & 4: config rule check
    const configResult = checkConfigRules(tool.name, input, context)
    if (configResult.behavior !== 'ask') return configResult

    // Layer 5: Auto mode ML classifier
    if (permissionMode === 'auto') {
      const classifierInput = tool.toAutoClassifierInput(input)
      if (classifierInput !== '') {
        // Grace period: give the classifier 2 seconds
        const classifierResult = await Promise.race([
          runAutoClassifier(classifierInput),
          sleep(2000).then(() => null),  // timeout → null
        ])

        if (classifierResult?.approved) {
          return { behavior: 'allow', updatedInput: input }
        }
        // Not approved (rejected or timed out) → continue down the pipeline
      }
    }

    // Layer 6: non-interactive mode
    if (context.options.isNonInteractiveSession) {
      return autoDecideForNonInteractive(tool.name, permissionMode)
    }

    // Layer 7: interactive confirmation dialog
    return await showPermissionDialog(tool, input, context)

  }, [permissionMode, ...deps])
}
```

---

## 6. Auto Mode ML Classifier

### 6.1 Tool Integration with Classifier

```typescript
// Only tools that implement a non-empty toAutoClassifierInput are processed by the classifier
// Default (provided by buildTool): () => ''  (skip the classifier)

// BashTool example:
toAutoClassifierInput: (input) => input.command

// FileWriteTool example:
toAutoClassifierInput: (input) =>
  `${input.file_path}\n${input.content?.slice(0, 500)}`

// FileReadTool:
toAutoClassifierInput: () => ''  // read-only operation, no classifier needed
```

### 6.2 Grace Period Mechanism

```typescript
const CLASSIFIER_GRACE_PERIOD_MS = 2000

// If the classifier finishes within 2 seconds and approves → auto-allow, user unaware
// If timeout → user sees the confirmation dialog; classifier still runs in background
//   → if the classifier later approves, it has no effect (user already saw the dialog)

const result = await Promise.race([
  runAutoClassifier(classifierInput),
  sleep(CLASSIFIER_GRACE_PERIOD_MS).then(() => null),
])

if (result?.approved) {
  return { behavior: 'allow' }
}
// Fall through to interactive confirmation
```

### 6.3 DenialTracking (Preventing Loops)

```typescript
// src/utils/permissions/denialTracking.ts

type DenialTrackingState = {
  consecutiveDenials: Map<string, number>  // toolName → consecutive denial count
  totalDenials: number
}

// Trigger conditions:
// consecutive denials ≥ 3  OR  total denials ≥ 20
// → fall back to interactive user confirmation (no longer rely on classifier)
// → prevents infinite loops caused by the classifier repeatedly rejecting

function shouldFallbackToInteractive(state: DenialTrackingState, toolName: string): boolean {
  const consecutive = state.consecutiveDenials.get(toolName) ?? 0
  return consecutive >= 3 || state.totalDenials >= 20
}
```

---

## 7. Dangerous Command Detection

### 7.1 BashTool Blocklist

```typescript
// src/tools/BashTool/ (detection logic, excerpted)

const DANGEROUS_PATTERNS = [
  // Recursive deletion of root or all files
  /rm\s+(-[rfR]+\s+)?\/(!\w)/,        // rm -rf /
  /rm\s+(-[rfR]+\s+)?\*$/,            // rm -rf *

  // Fork Bomb
  /:\(\)\s*\{.*:.*\|.*:.*\}/,         // :() { :|:& };:

  // Overwrite critical system files
  />\s*\/etc\/passwd/,
  />\s*\/etc\/shadow/,

  // Dangerous pipe execution (remote code execution)
  /curl.+\|\s*(ba)?sh/,               // curl | bash
  /wget.+\|\s*(ba)?sh/,               // wget | bash
]

function detectBlockedCommand(command: string): string | null {
  for (const pattern of DANGEROUS_PATTERNS) {
    if (pattern.test(command)) {
      return `Command blocked: matches dangerous pattern ${pattern}`
    }
  }
  return null  // safe
}
```

### 7.2 Sleep Detection (Preventing Tool Stalls)

```typescript
// Prevents the model from blocking tool execution with an extremely long sleep
function detectBlockedSleepPattern(command: string): boolean {
  const sleepMatch = command.match(/^\s*sleep\s+(\d+)\s*$/)
  if (sleepMatch) {
    const seconds = parseInt(sleepMatch[1])
    if (seconds > 300) return true  // over 5 minutes → block
  }
  return false
}
```

### 7.3 Windows Path Safety (7 Bypass Attack Detections)

```typescript
function detectWindowsPathBypass(path: string): boolean {
  // 1. NTFS ADS (Alternate Data Streams): file.txt:secret
  if (/:\w+$/.test(path)) return true

  // 2. 8.3 short filenames: PROGRA~1 (bypasses deny rules)
  if (/~\d/.test(path)) return true

  // 3. Long path prefix: \\?\C:\... (skips MAX_PATH check)
  if (path.startsWith('\\\\?\\')) return true

  // 4. Trailing dot (Windows ignores): file.txt.
  if (/\.$/.test(path)) return true

  // 5. DOS device names: CON, PRN, AUX, NUL, COM1-9, LPT1-9
  if (/^(CON|PRN|AUX|NUL|COM\d|LPT\d)$/i.test(path)) return true

  // 6. Triple dot: C:\...\file (traverses multiple directory levels)
  if (/\.\.\./.test(path)) return true

  // 7. UNC path: \\server\share (network path)
  if (path.startsWith('\\\\')) return true

  return false
}
```

---

## 8. Sandbox: Working Directory Restriction

Claude Code's sandbox is not container-level isolation — it is a **working directory restriction**:

```typescript
// src/utils/sandbox/

type PermissionContext = {
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  // Default: only getCwd() (current working directory)
  // Users can add more allowed directories via the /add-dir command
}

// Path check before file operations
function isPathAllowed(path: string, context: ToolPermissionContext): boolean {
  const allowedDirs = [
    getCwd(),
    ...context.additionalWorkingDirectories.keys(),
  ]

  const resolvedPath = fs.realpathSync(path)  // resolve symlinks

  return allowedDirs.some(dir =>
    resolvedPath.startsWith(resolveWorkingDir(dir))
  )
}
```

### 8.1 Path Safety Validation (4 Illegal Path Types)

```typescript
function validatePath(inputPath: string): ValidationResult {
  // 1. Reject relative paths (must be absolute)
  if (!path.isAbsolute(inputPath)) {
    return { result: false, message: 'Path must be absolute', errorCode: 1 }
  }

  // 2. Reject root directory (/)
  if (inputPath === '/') {
    return { result: false, message: 'Operating on the root directory is not allowed', errorCode: 2 }
  }

  // 3. Reject null byte injection (null byte attack: /etc/pass\x00wd)
  if (inputPath.includes('\0')) {
    return { result: false, message: 'Path contains illegal characters', errorCode: 3 }
  }

  // 4. Windows: reject UNC paths
  if (process.platform === 'win32' && inputPath.startsWith('\\\\')) {
    return { result: false, message: 'UNC paths are not allowed', errorCode: 4 }
  }

  return { result: true }
}
```

---

## 9. Permission Caching (In-Session Memory)

To avoid prompting repeatedly for the same operation, permission decisions are cached within the session:

```typescript
type ToolDecision = {
  source: string           // decision source: 'user', 'classifier', 'rule'
  decision: 'accept' | 'reject'
  timestamp: number
}

// Stored in ToolUseContext
toolDecisions: Map<string, ToolDecision>

// Cache key = hash(toolName + JSON.stringify(input))

// User selects "Always Allow" → rule is persisted to settings.json
//   → loaded from config on next startup, effective permanently
```

---

## 10. Special Logic in Plan Mode

```typescript
// plan mode: read-only operations are auto-approved; write operations prompt the user
if (permissionMode === 'plan') {
  if (tool.isReadOnly(input)) {
    return { behavior: 'allow' }
  }
  // Write operation → show dialog, but with a "plan mode" indication
}

// Also, in plan mode the model uses a larger-context variant:
// If the most recent assistant message exceeds 200k tokens, switch to a model that supports 200k
const currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens: permissionMode === 'plan' && doesMostRecentAssistantMessageExceed200k(messages),
})
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/hooks/useCanUseTool.tsx` | Permission decision hook; full 7-layer pipeline (203 lines) |
| `src/utils/permissions/` | Permission rule parsing, Glob matching (60+ files) |
| `src/utils/permissions/denialTracking.ts` | DenialTracking loop prevention |
| `src/hooks/toolPermission/` | Permission decision pipeline implementation |
| `src/tools/BashTool/` | Dangerous command detection, sleep detection |
| `src/utils/sandbox/` | Working directory isolation, path validation |
| `src/types/permissions.ts` | PermissionMode, PermissionResult type definitions |
