# 13 — Security: Permission Rules, Dangerous Command Detection & Sandbox

> Files: `src/utils/permissions/`, `src/hooks/toolPermission/`, `src/utils/sandbox/`

---

## 1. Permission Modes (6 Types)

```typescript
type PermissionMode =
  | 'default'            // Ask the user each time (standard mode)
  | 'plan'               // Plan mode: read-only auto-approved, write ops ask
  | 'acceptEdits'        // Auto-accept file edits, ask for other operations
  | 'auto'               // ML classifier auto-decision (requires opt-in)
  | 'bypassPermissions'  // Skip all permission checks (--dangerously-skip-permissions)
  | 'dontAsk'            // Auto-allow but still log (non-interactive mode)
```

**Decision priority (Deny > Ask > Allow):**

```
Any deny rule matches → reject immediately (no further checks)
No deny match → check ask rules
No ask match → check allow rules
No rules match → use default behavior (ask the user)
```

---

## 2. Permission Rule Glob Matching

### 2.1 Rule Format

```
ToolName(pattern)

Examples:
  Bash(git *)              → allow all git commands
  Read(~/*)                → allow reading home directory
  Write(/tmp/*)            → allow writing to /tmp
  Bash(rm -rf *)           → deny dangerous deletion
  Edit(*.ts)               → allow editing TypeScript files
  Bash(*)                  → wildcard all Bash commands
```

### 2.2 Matching Algorithm

```typescript
// Uses the ignore library (gitignore-format glob matching)
import ignore from 'ignore'

function matchesRule(toolName: string, input: unknown, rule: string): boolean {
  // 1. Parse rule: ToolName(pattern)
  const [ruleToolName, rulePattern] = parseRule(rule)

  // 2. Tool name match
  if (ruleToolName !== toolName && ruleToolName !== '*') return false

  // 3. Pattern match
  const ig = ignore().add(rulePattern)

  // Key: normalize to POSIX paths uniformly (cross-platform)
  const inputStr = normalizeForMatching(input)

  return ig.ignores(inputStr)
}

function normalizeForMatching(input: unknown): string {
  const str = typeof input === 'string' ? input : JSON.stringify(input)
  // Windows paths: C:\Users\foo → /c/users/foo
  return str.replace(/\\/g, '/').toLowerCase()
}
```

### 2.3 Symlink Safety

```typescript
// Check both the original path AND the resolved path
// Prevents bypassing deny rules via symlinks

async function checkPathPermission(path: string, rules: Rules): boolean {
  const realPath = await fs.realpath(path).catch(() => path)

  // Both paths must be checked
  return matchesAnyRule(path, rules) || matchesAnyRule(realPath, rules)
}
```

---

## 3. Dangerous Command Detection

### 3.1 Bash Command Blocklist

```typescript
// src/tools/BashTool/ (excerpt of real detection logic)

const DANGEROUS_PATTERNS = [
  // Recursive deletion
  /rm\s+(-[rfR]+\s+)?\/(?!\w)/,        // rm -rf /
  /rm\s+(-[rfR]+\s+)?\*$/,             // rm -rf *

  // Fork Bomb
  /:\(\)\s*\{.*:.*\|.*:.*\}/,          // :() { :|:& };:

  // Overwrite critical files
  />\s*\/etc\/passwd/,
  />\s*\/etc\/shadow/,

  // Dangerous pipe execution
  /curl.+\|\s*(ba)?sh/,               // curl | bash
  /wget.+\|\s*(ba)?sh/,               // wget | bash

  // Long sleep (prevent tool from hanging)
  // Standalone sleep > 300 seconds will be detected and blocked
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

### 3.2 Special Rule: Sleep Detection

```typescript
// Prevent the model from blocking tool execution with sleep
function detectBlockedSleepPattern(command: string): boolean {
  // Detect standalone sleep commands (not part of a pipe or conditional)
  const sleepMatch = command.match(/^\s*sleep\s+(\d+)\s*$/)
  if (sleepMatch) {
    const seconds = parseInt(sleepMatch[1])
    if (seconds > 300) {  // more than 5 minutes
      return true  // block
    }
  }
  return false
}
```

### 3.3 Windows Path Security (7 Bypass Detection Types)

```typescript
// Detect Windows NTFS-specific path bypass tricks
function detectWindowsPathBypass(path: string): boolean {
  // 1. NTFS ADS (Alternative Data Streams): file.txt:secret
  if (/:\w+$/.test(path)) return true

  // 2. 8.3 short filenames: PROGRA~1
  if (/~\d/.test(path)) return true

  // 3. Long path prefix: \\?\C:\...
  if (path.startsWith('\\\\?\\')) return true

  // 4. Trailing dot (Windows ignores it): file.txt.
  if (/\.$/.test(path)) return true

  // 5. DOS device names: CON, PRN, AUX, NUL, COM1-9, LPT1-9
  if (/^(CON|PRN|AUX|NUL|COM\d|LPT\d)$/i.test(path)) return true

  // 6. Triple dot: C:\...\file (spans multiple directory levels)
  if (/\.\.\./.test(path)) return true

  // 7. UNC paths: \\server\share
  if (path.startsWith('\\\\')) return true

  return false
}
```

---

## 4. Auto Mode ML Classifier

### 4.1 Interface Design

```typescript
// Tools define their own classifier input
// Security-relevant tools must implement this method (otherwise the classifier is skipped by default)

type Tool = {
  // Returns the string representation used by the classifier
  // Empty string = skip classifier (default for buildTool)
  toAutoClassifierInput(input: z.infer<Input>): unknown
}

// BashTool example
toAutoClassifierInput: (input) => input.command

// FileWriteTool example
toAutoClassifierInput: (input) => `${input.file_path}\n${input.content?.slice(0, 500)}`
```

### 4.2 Grace Period Mechanism

```typescript
// Grace period: give the classifier 2 seconds to finish
// If classifier approves → silently auto-approved
// If timeout → wait for manual user confirmation

const CLASSIFIER_GRACE_PERIOD_MS = 2000

const result = await Promise.race([
  runAutoClassifier(classifierInput),       // classifier request
  sleep(CLASSIFIER_GRACE_PERIOD_MS).then(() => null),  // timeout
])

if (result?.approved) {
  return { behavior: 'allow' }  // fast path
}
// Timeout or denied → go through interactive confirmation flow
```

### 4.3 DenialTracking (Prevent Auto Mode Loops)

```typescript
// Track Auto mode denial history
// Prevent infinite asking caused by consecutive classifier denials

type DenialTrackingState = {
  consecutiveDenials: Map<string, number>  // toolName → consecutive denial count
  totalDenials: number
}

// Trigger conditions:
// Consecutive denials ≥ 3 OR total denials ≥ 20
// → Fall back to interactive user confirmation (no longer relying on classifier)

function shouldFallbackToInteractive(state: DenialTrackingState, toolName: string): boolean {
  const consecutive = state.consecutiveDenials.get(toolName) ?? 0
  return consecutive >= 3 || state.totalDenials >= 20
}
```

---

## 5. Complete Permission Decision Pipeline (7 Layers)

```
Tool call request
    │
    ├─ Layer 1: tool.validateInput()
    │           Input format and semantic validation (e.g. path validity)
    │
    ├─ Layer 2: Dangerous command detection (inside the tool)
    │           Blocklist pattern matching (rm -rf /, fork bomb, etc.)
    │
    ├─ Layer 3: alwaysDenyRules check
    │           Deny rules from settings.json
    │           Match → reject immediately
    │
    ├─ Layer 4: alwaysAllowRules check
    │           Allow rules from settings.json
    │           Match → allow immediately
    │
    ├─ Layer 5: [Auto mode] ML classifier
    │           toAutoClassifierInput → classifier API
    │           Grace period 2s: approved → auto-allow
    │
    ├─ Layer 6: shouldAvoidPermissionPrompts check
    │           Non-interactive mode (CI/CD) → auto-decide based on permissionMode
    │
    └─ Layer 7: Show interactive confirmation dialog
                Wait for user: Allow / Deny / Always Allow / Always Deny
```

---

## 6. Sandbox Execution Isolation

Claude Code's sandbox is not container-level isolation; it is **working directory restriction + path validation**:

```typescript
// Core constraint: tools can only operate on files within "allowed working directories"

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

### 6.1 Path Security Validation

```typescript
// Prevent path traversal attacks (../../../etc/passwd, etc.)
function validatePath(inputPath: string): ValidationResult {
  // 1. Reject relative paths (must be absolute)
  if (!path.isAbsolute(inputPath)) {
    return { result: false, message: 'Path must be absolute', errorCode: 1 }
  }

  // 2. Reject root directory
  if (inputPath === '/') {
    return { result: false, message: 'Operating on the root directory is not allowed', errorCode: 2 }
  }

  // 3. Reject null bytes (null byte injection attack)
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

## 7. Permission Caching

To avoid repeatedly prompting for the same operation, permission decisions are cached within a session:

```typescript
type ToolDecision = {
  source: string           // Decision source ('user', 'classifier', 'rule')
  decision: 'accept' | 'reject'
  timestamp: number
}

// Stored in ToolUseContext
toolDecisions: Map<string, ToolDecision>

// Cache key = hash(toolName + JSON.stringify(input))
// When the user selects "Always Allow", the rule is persisted to settings.json
```

---

## Key File Reference

| File | Function |
|------|----------|
| `src/utils/permissions/` | Permission rule parsing, Glob matching (60+ files) |
| `src/utils/permissions/denialTracking.ts` | DenialTracking loop prevention |
| `src/hooks/toolPermission/` | Permission decision pipeline implementation |
| `src/hooks/useCanUseTool.tsx` | React-side permission hook |
| `src/tools/BashTool/` | Dangerous command detection, sleep detection |
| `src/utils/sandbox/` | Working directory isolation, path validation |
| `src/types/permissions.ts` | PermissionMode, PermissionResult type definitions |
