# 08 — Memory System: CLAUDE.md, memdir & Session Memory

> Files: `src/memdir/`, `src/services/SessionMemory/`, `src/context.ts`, `src/utils/claudemd.ts`

---

## 1. Memory System Overview

Claude Code's "memory" is organized into four layers, from most persistent to most temporary:

```
┌──────────────────────────────────────────────────────────┐
│  Layer 1: CLAUDE.md files                                 │
│  Permanently stored on disk, persists across sessions,    │
│  manually maintained                                      │
│  Location: project dir / user home dir / system dir       │
├──────────────────────────────────────────────────────────┤
│  Layer 2: memdir (MEMORY.md)                              │
│  ~/.claude/projects/<hash>/memory/                        │
│  Automatically written by the model, persists across      │
│  sessions, categorized by type                            │
├──────────────────────────────────────────────────────────┤
│  Layer 3: Session Memory                                  │
│  Auto-summary for the current session (token-triggered)   │
│  Used to assist context compression in very long chats    │
├──────────────────────────────────────────────────────────┤
│  Layer 4: Conversation history (messages[])               │
│  In-memory only, gone when the session ends               │
│  (managed by compaction)                                  │
└──────────────────────────────────────────────────────────┘
```

---

## 2. CLAUDE.md Discovery Algorithm (7 Priority Levels)

`src/utils/claudemd.ts` implements the CLAUDE.md discovery logic, **from lowest to highest priority**:

```
Priority  Source                            Example path
────────  ────────────────────────────────  ─────────────────────────────────
  1       Managed (enterprise-managed)      /etc/claude-code/CLAUDE.md
  2       User (user global)                ~/.claude/CLAUDE.md
  3       Project (project root)            /repo/CLAUDE.md
                                            /repo/.claude/CLAUDE.md
                                            /repo/.claude/rules/*.md
  4       Local (walk up from CWD)          ./CLAUDE.md (CWD)
                                            ../CLAUDE.md (parent dir)
                                            ../../CLAUDE.md (grandparent dir)
                                            ... up to git root or filesystem root
  5       Additional Dirs (--add-dir)       /extra/path/CLAUDE.md
  6       AutoMem (auto memory)             ~/.claude/projects/<hash>/memory/MEMORY.md
  7       TeamMem (team memory)             ~/.claude/projects/<hash>/memory/team/MEMORY.md
```

**Key rules:**
- **The Local layer walks upward from CWD**; the CLAUDE.md in CWD has the highest priority
- **Git Worktree deduplication**: avoids loading the same file twice from the main repo and a worktree
- **Higher priority overrides lower priority** (for the same key)

### 2.1 Search Paths

```typescript
// Three locations are checked in each directory
const CLAUDE_MD_LOCATIONS = [
  'CLAUDE.md',              // directory root
  '.claude/CLAUDE.md',      // .claude subdirectory
  '.claude/rules/*.md',     // all .md files under the rules directory (glob)
]
```

---

## 3. CLAUDE.md Content Processing

### 3.1 Size Limits

```typescript
const MAX_ENTRYPOINT_LINES = 200       // max 200 lines per file
const MAX_ENTRYPOINT_BYTES = 25_000    // max 25KB per file

// Truncation order: lines first, then bytes (at a newline boundary)
// A notice is appended after truncation: "[CLAUDE.md truncated: exceeds size limit]"
```

### 3.2 @import Syntax (Cross-File References)

CLAUDE.md supports referencing other files with `@` syntax:

```markdown
<!-- CLAUDE.md -->
# Project Standards

@./docs/coding-style.md
@~/global-rules.md
@/absolute/path/to/rules.md
@path\ with\ spaces.md    <!-- spaces must be escaped -->
```

**Processing rules:**
```
Scanning rules:
  ✓ Find @path in regular text nodes
  ✗ Do not scan code blocks (```...```)
  ✗ Do not scan inline code (`...`)
  → Prevents @references in code examples from being mistakenly triggered

Safety limits:
  - MAX_INCLUDE_DEPTH = 5 (maximum 5 levels of nesting)
  - processedPaths set prevents circular references
  - Non-existent files are silently ignored (no error)
  - External @includes require user approval (security policy)
```

### 3.3 Format Injected into System Prompt

```
Memory section in the system prompt:

<user-context>
  <claude-md source="user">
  [Contents of ~/.claude/CLAUDE.md]
  </claude-md>

  <claude-md source="project" path="/repo/CLAUDE.md">
  [Contents of the project CLAUDE.md]
  </claude-md>

  <claude-md source="local" path="/repo/src/CLAUDE.md">
  [Contents of the local CLAUDE.md]
  </claude-md>
</user-context>
```

---

## 4. memdir: Memory Automatically Written by the Model

### 4.1 Four Memory Types

```typescript
type MemoryType = 'user' | 'feedback' | 'project' | 'reference'

// user:      user role, preferences, technical background (always private)
// feedback:  working guidance, "do / don't do" (private by default, can be set to team-shared)
// project:   project info, deadlines, decision records
// reference: pointers to external systems (Linear, Jira, Slack links, etc.)
```

### 4.2 Storage Path Structure

```
~/.claude/projects/<project-hash>/
├── memory/
│   ├── MEMORY.md                    # Main index file (model-generated)
│   ├── user_preferences.md          # type_name.md format
│   ├── feedback_coding_style.md
│   ├── project_architecture.md
│   ├── reference_external_links.md
│   ├── team/
│   │   └── MEMORY.md                # Team-shared memory
│   ├── logs/
│   │   └── 2025/01/15.md           # KAIROS logs (by date)
│   └── agent-memory/                # Agent-specific memory
└── session-memory/
    └── memory.md                    # Session Memory summary
```

`<project-hash>` = SHA256(normalized project path)

---

## 5. Session Memory: Auto-Summarization for Very Long Conversations

### 5.1 Trigger Thresholds

```typescript
const DEFAULT_SESSION_MEMORY_CONFIG = {
  minimumMessageTokensToInit: 10_000,    // Initial generation: messages exceed 10k tokens
  minimumTokensBetweenUpdate: 5_000,     // Incremental update: accumulated growth of 5000 tokens
  toolCallsBetweenUpdates: 3,            // Or: 3 new tool calls added
}

// Trigger condition (either):
// (tokens ≥ minimumTokensBetweenUpdate AND tool calls ≥ toolCallsBetweenUpdates)
// OR
// (tokens ≥ minimumTokensBetweenUpdate AND no trailing tool calls)
// Note: the token threshold is always a necessary condition
```

### 5.2 Coordination with Autocompact

```
Normal conversation ──→ Session Memory updated in background
                               (maintains a running summary)
                    │
                    ▼
          When Autocompact triggers, it first tries
          Session Memory Compaction:
          replace conversation history with the existing
          session memory file
          (faster and cheaper than requesting LLM to regenerate a summary)
```

### 5.3 File Security

```typescript
// File permissions for session-memory/memory.md
fs.writeFileSync(memoryPath, content, { mode: 0o600 })
// Owner read/write only; no access for other users
```

---

## 6. Team Memory Sync

Team members can share project-level memory, synchronized via `services/teamMemorySync/`:

```
Sync mechanism:
  - Uses a Git repository as the sync backend (.claude-team/ subdirectory)
  - Pull: fetch the latest team/MEMORY.md from the team repo at session start
  - Push: commit and push to the team repo after the model writes team memory
  - Conflict resolution: three-way merge, local changes take precedence

Use cases:
  - Team-shared coding standards (avoids repeating them to every team member)
  - Project architecture decision documents
  - Common external links and tool configurations
```

---

## 7. Context Injection Sequence

```typescript
// src/context.ts
export async function getUserContext(
  toolUseContext: ToolUseContext,
): Promise<{ [key: string]: string }> {

  // Load in parallel (non-blocking)
  const [memoryFiles, gitStatus] = await Promise.all([
    getMemoryFiles(toolUseContext),          // Load all CLAUDE.md files
    getGitStatus(getCwd()),                  // git status (injected as context)
  ])

  return {
    workingDirectory: getCwd(),
    gitStatus: gitStatus ?? '',
    memoryContext: formatMemoryFiles(memoryFiles),
    // ...other context
  }
}
```

The result of `getUserContext()` is injected by `prependUserContext()` at the beginning of the API request's message list, as the first user message.

---

## 8. Caching Strategy

```typescript
// CLAUDE.md loading has a session-level cache
// Avoids re-scanning the filesystem on every API request

// When the cache is cleared:
// - After /compact executes (runPostCompactCleanup)
// - After settings change (clearCaches)
// - On explicit user /reload (if available)

getUserContext.cache.clear?.()  // Manual clear
```

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/utils/claudemd.ts` | CLAUDE.md discovery algorithm (1000+ lines) |
| `src/memdir/memdir.ts` | MEMORY.md read/write logic |
| `src/memdir/paths.ts` | Path resolution, security validation |
| `src/context.ts` | getUserContext, injects git status + memory |
| `src/context/` | Notification system, attachment management |
| `src/services/SessionMemory/sessionMemory.ts` | Session Memory extraction and updates |
| `src/services/teamMemorySync/` | Team memory synchronization |
