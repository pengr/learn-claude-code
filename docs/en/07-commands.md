# 07 — Commands System: Slash Command Registration & Dispatch

> Files: `src/commands.ts`, `src/commands/`, `src/skills/`

---

## 1. Command Type Definitions

Claude Code's command system supports three command types:

```typescript
// src/types/commands.ts (excerpt)

// Base attributes (shared by all commands)
type CommandBase = {
  name: string                         // Command name, e.g. "commit"
  description: string                  // User-visible description
  aliases?: string[]                   // Aliases, e.g. ["c"] → /c also triggers it
  isHidden?: boolean                   // Whether to hide from /help
  isEnabled: () => boolean             // Dynamic toggle (based on feature flag or condition)
  userInvocable?: boolean              // Can the user invoke it directly (vs. model-only)
  source: 'builtin' | 'bundled' | 'user' | 'plugin'  // Command origin
  loadedFrom?: string                  // File path (for user/plugin commands)
}

// Type 1: Prompt command (most common)
// Injects prompt text into the conversation when triggered
type PromptCommand = CommandBase & {
  type: 'prompt'
  getPromptForCommand: (args: string) => Promise<string>
  allowedTools?: string[]              // Restrict which tools this command can use
  model?: string                       // Override the model to use
  disableModelInvocation?: boolean     // true = don't invoke the model, execute directly
  progressMessage?: string             // Spinner text shown during execution
  hooks?: HooksSettings                // Command-level hooks
  context?: string                     // Inject additional context
  agent?: string                       // Specify the Agent type to use
}

// Type 2: Local command (pure client-side logic, no LLM call)
type LocalCommand = CommandBase & {
  type: 'local'
  call: (args: string, context: CommandContext) => Promise<void>
}

// Type 3: Local JSX command (renders an interactive React UI)
type LocalJSXCommand = CommandBase & {
  type: 'local-jsx'
  call: (
    args: string,
    onDone: (result?: string) => void,
    context: CommandContext,
  ) => Promise<React.ReactNode>
}

type Command = PromptCommand | LocalCommand | LocalJSXCommand
```

---

## 2. Command Registration Mechanism

All commands are loaded centrally in `src/commands.ts`, from **4 sources**:

```typescript
// src/commands.ts (simplified)
export async function getCommands(
  toolUseContext: ToolUseContext,
): Promise<Command[]> {
  const commands: Command[] = []

  // 1. Built-in commands (hardcoded, ~88 directories)
  commands.push(...getBuiltinCommands())

  // 2. Bundled skill commands (built-in skills from plugins)
  commands.push(...getBuiltinPluginSkillCommands())

  // 3. User-defined skills (~/.claude/commands/ and .claude/commands/)
  const userSkills = await loadUserSkillCommands()
  commands.push(...userSkills)

  // 4. Plugin commands (installed third-party plugins)
  const pluginCommands = await loadPluginCommands()
  commands.push(...pluginCommands)

  // Availability filter: remove commands unavailable in the current environment
  return commands.filter(cmd => cmd.isEnabled())
}
```

### 2.1 Built-in Command Directory (Full List)

There are 88 subdirectories under `src/commands/`, each corresponding to one command:

```
add-dir/           approve-run/        btw/
bug/               clear/              commit/
compact/           config/             context/
cost/              cr/                 diff-since/
doctor/            exit/               export/
find-files/        free-memory/        git/
help/              history/            ide/
init/              install-github-app/ issue/
jobs/              kv/                 launch-in-neovim/
list-mcp-servers/  list-sessions/      login/
logout/            logs/               memory/
mcp/               model/              new/
onboarding/        open-docs/          open-pr/
pause/             permissions/        plan/
pr-comments/       profile/            pull-request/
pwd/               reconnect-bridge/   release-notes/
remove-dir/        reply/              reset-auto-mode/
restart/           review/             run/
run-github-action/ search/             setup/
show-system-prompt/ skills/            sleep/
snapshot/          status/             stop/
subscribe/         switch-account/     sync-settings/
terminal/          think/              todos/
tour/              update/             usage/
verbose/           version/            vim-mode/
web/               worktree/           ...
```

---

## 3. Command Dispatch Mechanism

Full routing flow after the user types `/xxx args`:

```
User types "/commit --amend"
        │
        ▼
  1. Parse: isSlashCommand("/commit --amend")
     → name = "commit", args = "--amend"
        │
        ▼
  2. Look up: commands.find(c => c.name === 'commit' || c.aliases?.includes('commit'))
        │
        ▼
  3. Availability check: cmd.isEnabled()
     → false: show "command not available" message
     → true:  continue
        │
        ▼
  4. Type dispatch:
     │
     ├─ type === 'local'
     │    └─ await cmd.call(args, context)
     │         Execute directly, no LLM involved
     │
     ├─ type === 'local-jsx'
     │    └─ const jsx = await cmd.call(args, onDone, context)
     │       context.setToolJSX({ jsx, shouldHidePromptInput: true })
     │         Render interactive UI
     │
     └─ type === 'prompt'
          └─ const prompt = await cmd.getPromptForCommand(args)
             Inject prompt into conversation, go through normal LLM flow
```

### 3.1 Command Execution Context (CommandContext)

```typescript
type CommandContext = {
  toolUseContext: ToolUseContext    // Full tool-use context
  setToolJSX: SetToolJSXFn         // Inject React UI
  addNotification: (n) => void     // Show a notification
  messages: Message[]              // Current conversation
  appendSystemMessage: (msg) => void  // Append a system message
}
```

---

## 4. Availability Filtering

Commands use `isEnabled()` to determine availability. Common conditions:

```typescript
// Example: requires a Git repository
{
  isEnabled: () => isGitRepo(getCwd()),
}

// Example: controlled by a feature flag
{
  isEnabled: () => feature('SOME_FEATURE'),
}

// Example: Claude.ai users only
{
  isEnabled: () => isClaudeAiUser(),
}

// Example: requires network access
{
  isEnabled: () => !isOfflineMode(),
}

// Example: Ant internal users only
{
  isEnabled: () => process.env.USER_TYPE === 'ant',
}
```

---

## 5. User-Defined Skills (Skill Commands)

Users can create `.md` files in the following locations to add custom slash commands:

```
Priority (high → low):

  1. .claude/commands/              Local project-level (highest priority)
  2. ~/.claude/commands/            User-level global commands
  3. Commands installed by plugins
  4. Built-in bundled commands
  5. Built-in builtin commands      (lowest priority)
```

### 5.1 Skill File Format

```markdown
---
description: Commit and push current changes
argumentHint: "[commit message]"
allowedTools: ["Bash"]
---

Please help me commit and push the current changes.
Commit message: $ARGUMENTS

Steps:
1. git add -A
2. git commit -m "$ARGUMENTS"
3. git push
```

YAML frontmatter fields:

| Field | Description |
|-------|-------------|
| `description` | Description shown in `/help` |
| `argumentHint` | Argument hint, e.g. `"<branch-name>"` |
| `allowedTools` | Restrict available tools (e.g. `["Bash", "Read"]`) |
| `model` | Override the model to use |
| `disableModelInvocation` | `true` = inject directly as a system message, no LLM call |
| `whenToUse` | Tell the model when to invoke this command automatically |
| `agent` | Specify the Agent type |

### 5.2 Built-in Skill Directory (SkillTool Invocation)

In addition to direct `/xxx` invocation by users, skills can also be called by the model via the `SkillTool`:

```typescript
// SkillTool lets the model proactively trigger user-defined skills
// Tool name format seen by the model: skill__<command_name>
// Input: { name: "commit", args: "fix: typo in README" }
```

---

## 6. Typical Command Implementations

### 6.1 `/compact` — Manual Compaction (Local Command)

```typescript
// src/commands/compact/index.ts
{
  type: 'local',
  name: 'compact',
  description: 'Compact conversation history to free up context space',
  isEnabled: () => true,
  call: async (args, context) => {
    const { toolUseContext } = context
    // Trigger the manual compaction flow
    await triggerManualCompaction(toolUseContext)
    context.appendSystemMessage({
      type: 'system',
      content: 'Conversation history has been compacted.',
    })
  },
}
```

### 6.2 `/config` — Configuration Management (Local JSX Command)

```typescript
// src/commands/config/index.ts
{
  type: 'local-jsx',
  name: 'config',
  description: 'View and modify Claude Code configuration',
  isEnabled: () => true,
  call: async (args, onDone, context) => {
    // Return an interactive configuration UI
    return <ConfigEditor onDone={onDone} />
  },
}
```

### 6.3 `/permissions` — Permission Management (Local JSX Command)

```typescript
// src/commands/permissions/index.ts
{
  type: 'local-jsx',
  name: 'permissions',
  description: 'View and manage tool permission rules',
  call: async (args, onDone, context) => {
    return <PermissionsEditor
      permissionContext={context.toolUseContext.getAppState().toolPermissionContext}
      onDone={onDone}
    />
  },
}
```

### 6.4 `/review` — Code Review (Prompt Command)

```typescript
// src/commands/review/index.ts
{
  type: 'prompt',
  name: 'review',
  description: 'Review current changes',
  getPromptForCommand: async (args) => {
    const diff = await execGitDiff()
    return `Please review the following code changes:\n\n${diff}\n\n${args || 'Please focus on potential bugs, code style, and performance issues.'}`
  },
}
```

---

## 7. Relationship Between Commands and Agents

Prompt commands support specifying an `agent` field to run the command in a specific Agent environment:

```markdown
---
description: Execute a task in an isolated Git Worktree
agent: worktree-agent
---
```

This triggers AgentTool, which executes the command in an independent worktree without affecting the main workspace.

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/commands.ts` | Command registration hub, aggregates all sources |
| `src/commands/<name>/index.ts` | Individual command implementations |
| `src/skills/` | Skill file loading logic |
| `src/types/commands.ts` or `commands.ts` | Command type definitions |
| `src/plugins/builtinPlugins.ts` | Built-in plugin skill command registration |
