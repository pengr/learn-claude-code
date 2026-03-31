# 14 — Miscellaneous Features: Vim Mode, Keybindings, Voice, Teleport & VCR

> Files: `src/vim/`, `src/keybindings/`, `src/voice/`, `src/utils/teleport/`, `src/services/vcr.ts`

---

## 1. Vim Mode

### 1.1 State Machine Design

Vim mode implements a complete **modal editing state machine** that faithfully replicates Vim's behavior:

```typescript
// src/vim/types.ts

// Top-level state: INSERT or NORMAL
type VimState =
  | { mode: 'INSERT'; insertedText: string }   // insertedText is used for . repeat
  | { mode: 'NORMAL'; command: CommandState }

// Command state machine inside NORMAL mode (11 states)
type CommandState =
  | { type: 'idle' }                                          // waiting for input
  | { type: 'count'; digits: string }                        // entering numeric prefix (5dd)
  | { type: 'operator'; op: Operator; count: number }        // d/c/y entered, waiting for motion
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }  // df<char>
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }  // diw
  | { type: 'find'; find: FindType; count: number }          // f/F/t/T find
  | { type: 'g'; count: number }                             // g prefix (gg, gj, etc.)
  | { type: 'operatorG'; op: Operator; count: number }
  | { type: 'replace'; count: number }                       // r<char> replace single char
  | { type: 'indent'; dir: '>' | '<'; count: number }        // >> << indent
```

### 1.2 Persistent State (Cross-Command)

```typescript
type PersistentState = {
  lastChange: RecordedChange | null    // records last change, used for . repeat
  lastFind: { type: FindType; char: string } | null  // records last f/t, used for ; ,
  register: string                     // default register (copy/paste)
  registerIsLinewise: boolean          // whether register content is a full line
}
```

### 1.3 Supported Operations

**Operators:** `d` (delete), `c` (change), `y` (yank)

**Motions:** `h/j/k/l`, `w/W/b/B/e/E`, `0/^/$`, `gg/G`, `%`, `f/F/t/T`

**Text Objects:**

```
iw / aw    inner/outer word
iW / aW    inner/outer WORD
i" / a"    inner/outer double quotes
i' / a'    inner/outer single quotes
i` / a`    inner/outer backticks
i( / a( / ib    inner/outer parentheses (supports nesting)
i[ / a[         inner/outer square brackets
i{ / a{ / iB    inner/outer curly braces
i< / a<         inner/outer angle brackets
```

### 1.4 How to Enable

```jsonc
// settings.json
{
  "vim": true
}
// or toggle with the /vim-mode command
```

---

## 2. Keybindings System

### 2.1 Configuration File Format

```jsonc
// ~/.claude/keybindings.json
{
  "$schema": "https://claude.ai/keybindings-schema.json",
  "bindings": [
    {
      "context": "Global",
      "bindings": {
        "ctrl+c": "app:interrupt",
        "ctrl+d": "app:exit",
        "ctrl+r": "history:search",
        "ctrl+l": "app:redraw"
      }
    },
    {
      "context": "Chat",
      "bindings": {
        "enter": "chat:submit",
        "escape": "chat:cancel",
        "shift+tab": "chat:cycleMode",
        "ctrl+x ctrl+e": "chat:externalEditor",   // multi-key chord!
        "ctrl+x ctrl+k": "chat:killAgents",
        "space": null                              // null = unbind
      }
    }
  ]
}
```

### 2.2 19 Contexts

| Context | Description |
|---------|-------------|
| `Global` | Universal, applies in all situations |
| `Chat` | When the input box is active |
| `Autocomplete` | Autocomplete dropdown |
| `Confirmation` | Permission confirmation dialog |
| `Help` | /help screen |
| `Transcript` | Conversation history view mode |
| `HistorySearch` | Ctrl+R history search |
| `Task` | Task list |
| `ThemePicker` | Theme picker |
| `Settings` | Settings screen |
| `Tabs` | Tab bar |
| `Attachments` | Attachment manager |
| `Footer` | Bottom toolbar |
| `MessageSelector` | Message selector |
| `DiffDialog` | Diff viewer dialog |
| `ModelPicker` | Model picker |
| `Select` | Generic selection component |
| `Plugin` | Plugin UI |

### 2.3 Complete Action List (72+)

```
app:interrupt          app:exit               app:toggleTodos
app:toggleTranscript   app:globalSearch       app:quickOpen
app:redraw             app:toggleVimMode

chat:submit            chat:cancel            chat:cycleMode
chat:externalEditor    chat:stash             chat:killAgents
chat:newLine           chat:toggleThinking    chat:toggleFastMode

history:previous       history:next           history:search

autocomplete:accept    autocomplete:dismiss   autocomplete:next
autocomplete:previous

confirm:yes            confirm:no

voice:pushToTalk

transcript:scrollUp    transcript:scrollDown  transcript:pageUp
transcript:pageDown    transcript:top         transcript:bottom
```

### 2.4 Multi-Key Chord Matching

```typescript
// Parsing phase
parseChord("ctrl+x ctrl+e")
  → [{ key: 'x', ctrl: true }, { key: 'e', ctrl: true }]

// Matching phase (two-phase parsing)
// Phase 1: receive first key (ctrl+x)
resolveKey(ctx, { key: 'x', ctrl: true }, activeContexts, bindings)
  → { type: 'chord_started', pending: [{ key: 'x', ctrl: true }] }

// Phase 2: receive second key (ctrl+e)
resolveKeyWithChordState(ctx, { key: 'e', ctrl: true }, ..., pending)
  → { type: 'match', action: 'chat:externalEditor' }
```

### 2.5 Platform-Specific Differences

```typescript
// Windows terminal VT mode detection (affects Shift+Tab support)
const SUPPORTS_VT_MODE =
  platform !== 'windows' ||
  (isBun
    ? satisfies(bunVersion, '>=1.2.23')
    : satisfies(nodeVersion, '>=22.17.0 <23.0.0 || >=24.2.0'))

// Mode cycle key:
// VT mode supported: Shift+Tab
// Not supported: Meta+M (fallback)
const MODE_CYCLE_KEY = SUPPORTS_VT_MODE ? 'shift+tab' : 'meta+m'

// Image paste key:
// Windows: Alt+V (avoids conflict with Ctrl+V system clipboard)
// Others: Ctrl+V
const IMAGE_PASTE_KEY = platform === 'windows' ? 'alt+v' : 'ctrl+v'
```

### 2.6 Reserved Keys That Cannot Be Rebound

The following key bindings are protected (require double-tap timing logic and cannot be overridden by normal keybindings):
- `Ctrl+C` (application interrupt)
- `Ctrl+D` (application exit)

---

## 3. Voice Mode

### 3.1 Three-Layer Feature Gate

```typescript
// src/voice/voiceModeEnabled.ts

// Layer 1: Feature Flag (controlled by GrowthBook, with emergency kill switch)
function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}

// Layer 2: Must be logged in via Anthropic OAuth (API Key / Bedrock / Vertex not supported)
function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) return false
  const tokens = getClaudeAIOAuthTokens()  // read from Keychain, cached (~20ms)
  return Boolean(tokens?.accessToken)
}

// Layer 3: Runtime master check
export function isVoiceModeEnabled(): boolean {
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}
```

### 3.2 How to Use

```
Default binding: hold Space in Chat context → voice input (push-to-talk)
Release Space → stop recording, speech-to-text, submit to input box

Custom binding (keybindings.json):
{ "context": "Chat", "bindings": { "space": null } }      // disable voice
{ "context": "Chat", "bindings": { "alt+v": "voice:pushToTalk" } }  // rebind
```

**Implementation:** Uses Claude.ai's `voice_stream` API endpoint (requires Anthropic OAuth)

---

## 4. Teleport: Cross-Machine Session Migration

### 4.1 Use Case

A Claude Code session started on Machine A can be migrated to Machine B to continue (with full conversation history and Git state preserved).

### 4.2 Migration Flow

```
Source machine
  │
  ├─ 1. Claude Haiku generates session title and Git branch name (JSON output)
  │       Title: ≤6 words, e.g. "Fix login button on mobile"
  │       Branch: e.g. "claude/fix-mobile-login-button"
  │
  ├─ 2. Verify Git working tree is clean (fails if there are uncommitted changes)
  │
  ├─ 3. Create Git Bundle (three-level fallback):
  │       Try --all (all branches + unpushed) → 100MB limit
  │       ↓ too large
  │       Try HEAD (current branch history)
  │       ↓ still too large
  │       squashed-root (single parentless snapshot)
  │
  ├─ 4. Upload Bundle to Anthropic Files API
  │
  └─ 5. Create Session record (containing bundle file_id)

Target machine
  │
  ├─ 1. OAuth fetch session history (paginated, 100 items/page)
  ├─ 2. Deserialize messages (restore conversation history)
  ├─ 3. Download Bundle, restore Git state
  └─ 4. Re-enter session
```

### 4.3 Title and Branch Name Generation (Using Haiku)

```typescript
const SESSION_TITLE_AND_BRANCH_PROMPT = `
You are generating a succinct title and git branch name for a coding session.

Title: clear, concise, max 6 words, sentence case
Branch: starts with "claude/", lowercase, max 4 words, dash-separated

Return JSON: {"title": "...", "branch": "claude/..."}

Examples:
- {"title": "Fix login button on mobile", "branch": "claude/fix-mobile-login-button"}
- {"title": "Update README", "branch": "claude/update-readme"}
`

// Fallback on failure: truncate user's first message + "claude/task"
```

### 4.4 Retry Strategy

```typescript
// 4 exponential backoff retries: 2s, 4s, 8s, 16s
const RETRY_DELAYS = [2000, 4000, 8000, 16000]

// Only retry transient errors (network errors, 5xx)
// 4xx errors (permission, resource not found) fail immediately
```

---

## 5. VCR Record & Replay (Test Infrastructure)

VCR is Claude Code's **test fixture system**, used to record real API responses and replay them, making tests deterministic and network-independent.

### 5.1 Core Functions

```typescript
// src/services/vcr.ts

// For regular Claude API calls
async function withVCR(
  messages: Message[],
  f: () => Promise<APIResult[]>
): Promise<APIResult[]>

// For streaming API calls
async function* withStreamingVCR(
  messages: Message[],
  f: () => AsyncGenerator<StreamEvent>
): AsyncGenerator<StreamEvent>

// For token counting
async function withTokenCountVCR(messages, f)
```

### 5.2 Cassette File Format

```jsonc
// fixtures/<name>-<hash12>.json
{
  "input": {
    "dehydrated": "..."    // dehydrated messages (paths replaced with placeholders)
  },
  "output": [
    {
      "type": "assistant",
      "uuid": "UUID-0",                // deterministic UUID (generated from index)
      "requestId": "REQUEST_ID",       // fixed value
      "message": {
        "model": "claude-3-5-sonnet-20241022",
        "usage": { "input_tokens": 100, "output_tokens": 50 },
        "content": [
          {
            "type": "text",
            "text": "Analyzing [CWD]/src/main.ts..."  // path replaced with placeholder
          }
        ]
      }
    }
  ]
}
```

### 5.3 Dehydration / Rehydration (Deterministic Hashing)

To make Cassette files usable across different machines, dynamic values are replaced:

```typescript
// Dehydration (during recording): replace dynamic values with placeholders
function dehydrateValue(s: unknown): unknown {
  return replace(s, [
    [/\d+/g, '[NUM]'],                    // numbers
    [/\d+ms/g, '[DURATION]'],             // millisecond durations
    [/\$[\d.]+/g, '[COST]'],              // costs
    [getCwd(), '[CWD]'],                  // working directory
    [getClaudeConfigHomeDir(), '[CONFIG_HOME]'],  // config directory
    ['Available commands: ...', 'Available commands: [COMMANDS]'],
  ])
}

// Rehydration (during playback): restore placeholders
function hydrateValue(s: unknown): unknown {
  return replace(s, [
    ['[NUM]', '1'],
    ['[DURATION]', '100'],
    ['[CWD]', getCwd()],
    ['[CONFIG_HOME]', getClaudeConfigHomeDir()],
  ])
}
```

### 5.4 Activation Conditions

```bash
# Auto-enabled in test environment
NODE_ENV=test

# Force-enable internally at Ant
FORCE_VCR=1

# Record new Cassettes (instead of erroring on missing ones)
VCR_RECORD=1

# Custom Cassette directory
CLAUDE_CODE_TEST_FIXTURES_ROOT=/path/to/fixtures
```

**CI behavior:** If a Cassette file is missing and not in record mode, the test fails immediately (ensures tests don't accidentally call the real API)

---

## Key File Reference

| File | Function |
|------|----------|
| `src/vim/types.ts` | Vim state machine type definitions |
| `src/vim/operators.ts` | d/c/y and other operator implementations |
| `src/vim/textObjects.ts` | iw/a"/i( and other text object implementations |
| `src/keybindings/parser.ts` | Key sequence parser |
| `src/keybindings/resolver.ts` | Multi-key chord matching algorithm |
| `src/keybindings/defaultBindings.ts` | Platform default bindings |
| `src/keybindings/match.ts` | Modifier key matching logic |
| `src/voice/voiceModeEnabled.ts` | Voice three-layer gate |
| `src/utils/teleport/teleport.tsx` | Teleport main flow |
| `src/utils/teleport/api.ts` | Session API types and requests |
| `src/utils/teleport/gitBundle.ts` | Git Bundle creation and upload |
| `src/services/vcr.ts` | VCR core implementation |
