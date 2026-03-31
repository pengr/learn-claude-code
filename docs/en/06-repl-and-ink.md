# 06 — REPL UI Layer: Custom Ink Engine & React Hooks

> Files: `src/screens/REPL.tsx`, `src/ink/`, `src/hooks/`, `src/ink.ts`

---

## 1. Technical Foundation: React + Ink

Claude Code's terminal UI is written in **React** and rendered to the terminal via **Ink**. Ink works by mapping React's virtual DOM to a terminal character grid:

```
React component tree
    │
    ▼
Ink Renderer (layout engine)
    │ Flexbox layout calculation
    ▼
Character grid (2D char[] + style[])
    │
    ▼
Terminal stdout (ANSI escape sequences)
```

---

## 2. Custom Ink Fork (src/ink/ink.tsx, 251KB)

Claude Code does **not use standard Ink**; instead it maintains a deeply customized fork with several key optimizations:

### 2.1 Microtask-Deferred Rendering

```typescript
// Standard Ink: setImmediate triggers rendering (1 event-loop delay)
// Custom fork:  queueMicrotask (faster, fires at end of current tick)

class InkRenderer {
  private onRender = () => {
    this.calculateLayout()
    this.output()
  }

  scheduleRender() {
    // queueMicrotask has lower latency than setImmediate
    // Native terminal cursor movement has no perceptible delay
    queueMicrotask(this.onRender)
  }
}
```

### 2.2 Explicit Double Buffering

```typescript
// Front buffer: content currently displayed in the terminal
private frontFrame: CharGrid

// Back buffer: newly computed content
private backFrame: CharGrid

// Each render cycle:
// 1. Compute new layout → write to backFrame
// 2. diff(frontFrame, backFrame) → send only changed characters
// 3. swap(frontFrame, backFrame)

// Diff output uses the log-update library to minimize stdout writes
```

### 2.3 Memory Pooling (Preventing Leaks in Long Sessions)

```typescript
// Reset the memory pool every 5 minutes
// Prevents unbounded growth of character/style objects in long sessions

const POOL_RESET_INTERVAL = 5 * 60 * 1000  // 5 minutes

class StylePool {
  private pool: Map<string, Style> = new Map()

  reset() {
    this.pool.clear()  // Periodic clear lets GC reclaim old objects
  }
}

const stylePool = new StylePool()
const charPool  = new CharPool()

setInterval(() => {
  stylePool.reset()
  charPool.reset()
}, POOL_RESET_INTERVAL)
```

### 2.4 Alt Screen Support

```typescript
// Enter Alt Screen (hides original terminal content, like vim/less fullscreen mode)
enterAltScreen()   // ESC[?1049h
exitAltScreen()    // ESC[?1049l

// Alt Screen supports:
// - Text selection tracking (mouse events)
// - Search highlighting
// - Mouse click tracking
// - BSU/ESU atomic cursor operations (prevents cursor position jumping)
```

### 2.5 Layout Damage Tracking

```typescript
// Detect whether a full repaint is needed (rather than just a partial update)
function didLayoutShift(prev: CharGrid, next: CharGrid): boolean {
  // If a structural layout change occurred (row count changed, column width changed)
  // mark the "damaged region" and fully repaint it on the next render
  return prev.rows !== next.rows || hasStructuralChange(prev, next)
}
```

---

## 3. ink.ts: Component Library Exports

`src/ink.ts` is the export entry point for the custom Ink fork, providing terminal UI components for the entire project:

```typescript
// Main components and APIs exported from src/ink.ts
export {
  // Layout
  Box,          // Similar to <div>, supports Flexbox (flexDirection, gap, padding, etc.)
  Text,         // Plain text, supports color, bold, italic, underline
  Newline,      // Line break
  Spacer,       // Flexible whitespace (flex: 1)
  Static,       // Static content excluded from re-renders

  // Interaction
  useInput,     // Keyboard input hook
  useFocus,     // Focus management
  useStdout,    // Direct stdout write
  useStderr,    // Direct stderr write

  // Render control
  render,       // Render root component
  measureElement,  // Measure component dimensions

  // Theming
  ThemeProvider,   // Provides Claude Code theme colors
}
```

---

## 4. REPL.tsx: Core Interaction Interface

`src/screens/REPL.tsx` (~900 lines) is the entry component for the entire UI.

### 4.1 Core State

```typescript
function REPL() {
  // Conversation message list
  const [messages, setMessages] = useState<Message[]>([])

  // Input field content
  const [inputValue, setInputValue] = useState('')

  // Custom UI injected by the current tool
  const [toolJSX, setToolJSXState] = useState<ToolJSXState | null>(null)

  // Whether we are waiting for an LLM response
  const [isQuerying, setIsQuerying] = useState(false)

  // Streaming output mode (spinner style)
  const [streamMode, setStreamMode] = useState<SpinnerMode>('thinking')

  // Number of tools currently executing
  const [inProgressToolCount, setInProgressToolCount] = useState(0)
}
```

### 4.2 Streaming Rendering: Consuming the AsyncGenerator from query()

REPL uses `for await` to consume the event stream produced by `query()`, updating the UI in real time:

```typescript
async function onQuery(userMessage: string) {
  setIsQuerying(true)

  for await (const event of query(queryParams)) {
    onQueryEvent(event)
  }

  setIsQuerying(false)
}

function onQueryEvent(event: StreamEvent | Message | TombstoneMessage) {
  switch (event.type) {
    case 'stream_request_start':
      setStreamMode('thinking')          // Show "thinking..." spinner
      break

    case 'assistant':
      // New assistant message (may be streaming-appended text)
      setMessages(prev => appendOrUpdate(prev, event))
      break

    case 'user':
      // Tool result message
      setMessages(prev => [...prev, event])
      break

    case 'tombstone':
      // Remove a failed message (on model switch)
      setMessages(prev => prev.filter(m => m.uuid !== event.message.uuid))
      break

    case 'system':
      // System prompt (compaction notification, etc.)
      setMessages(prev => [...prev, event])
      break
  }
}
```

### 4.3 setToolJSX Injection Mechanism

Tools can inject custom React components into the REPL for interactive UI:

```typescript
// Called when a tool executes
context.setToolJSX?.({
  jsx: <InteractivePermissionDialog tool={tool} onAllow={...} onDeny={...} />,
  shouldHidePromptInput: true,   // Hide input field while waiting for user action
  showSpinner: false,
})

// Clear after tool completes
context.setToolJSX?.(null)
```

**State protection logic:**

```typescript
// Local JSX commands (/btw, /config, etc.) are protected when active
// Only an explicit clearLocalJSX: true can dismiss them
// Prevents background tools from accidentally overwriting foreground interactive UI

function setToolJSX(args: SetToolJSXArgs | null) {
  if (args === null && localJSXCommandRef.current && !args?.clearLocalJSX) {
    return  // During a local command, ignore clear requests from tools
  }
  setToolJSXState(args)
}
```

---

## 5. React Hooks (src/hooks/)

### 5.1 useCanUseTool: Permission Decision Hook

This is the most critical hook, implementing a complete permission decision pipeline (203 lines):

```typescript
// src/hooks/useCanUseTool.tsx
export function useCanUseTool(): CanUseToolFn {
  return useCallback(async (tool, input, context) => {

    // 1. Check config rules (alwaysAllow / alwaysDeny)
    const configResult = checkConfigRules(tool, input, context)
    if (configResult.behavior !== 'ask') return configResult

    // 2. Auto mode: ML classifier
    if (permissionMode === 'auto') {
      const classifierInput = tool.toAutoClassifierInput(input)

      // Grace Period: give the classifier 2 seconds
      // If the classifier finishes within 2 seconds and approves → auto-allow, no user interruption
      const classifierResult = await Promise.race([
        runClassifier(classifierInput),
        sleep(2000).then(() => null),  // timeout → return null
      ])

      if (classifierResult?.approved) {
        return { behavior: 'allow' }  // Fast path, invisible to user
      }
    }

    // 3. Non-interactive mode (CI/CD): auto-decide, no dialog
    if (context.isNonInteractiveSession) {
      return autoDecideForNonInteractive(tool, input, permissionMode)
    }

    // 4. Show interactive confirmation dialog (wait for user action)
    return await showPermissionDialog(tool, input, context)

  }, [permissionMode, ...deps])
}
```

### 5.2 useInput: Keyboard Input Handling

```typescript
// From Ink, used to capture terminal keystrokes
useInput((input: string, key: Key) => {
  // input: string (regular characters)
  // key: { upArrow, downArrow, leftArrow, rightArrow, escape, return, ctrl, meta, shift, tab, backspace, delete }

  if (key.escape) {
    handleCancel()
    return
  }

  if (key.return) {
    handleSubmit()
    return
  }

  // Ctrl+C special handling (requires double-tap timing)
  if (key.ctrl && input === 'c') {
    handleInterrupt()
    return
  }
})
```

### 5.3 Other Important Hooks

```typescript
// useCancelRequest: handle interrupts (Ctrl+C / Esc)
export function useCancelRequest(abortController: AbortController) {
  // Send abort signal, stop current API request and tool execution
}

// useKeyBindings: apply user-configured keyboard shortcuts
export function useKeyBindings(): ResolvedBindings {
  // Load ~/.claude/keybindings.json
  // Merge with platform defaults
  // Return action → chord mapping
}

// useTerminalSize: respond to terminal size changes
export function useTerminalSize(): { columns: number; rows: number } {
  // Listen for SIGWINCH signal
  // Trigger re-layout
}
```

---

## 6. REPL Overall Layout Structure

```tsx
function REPL() {
  return (
    <ThemeProvider theme={currentTheme}>
      <Box flexDirection="column" height="100%">

        {/* Message list area (scrollable) */}
        <MessageList
          messages={messages}
          tools={tools}
          verbose={verbose}
        />

        {/* Custom UI injected by tools (e.g. permission dialog) */}
        {toolJSX && (
          <Box>
            {toolJSX.jsx}
          </Box>
        )}

        {/* Bottom input area */}
        {!toolJSX?.shouldHidePromptInput && (
          <InputArea
            value={inputValue}
            onChange={setInputValue}
            onSubmit={handleSubmit}
            isQuerying={isQuerying}
            streamMode={streamMode}
          />
        )}

        {/* Status bar (model, token usage, cost) */}
        <StatusBar
          model={model}
          cost={totalCostUSD}
          tokens={sessionTokens}
        />

      </Box>
    </ThemeProvider>
  )
}
```

---

## 7. Short-Lived Message Optimization

For frequently updated tool output (e.g. Bash progress, Sleep countdown), a **replace strategy** is used instead of appending:

```typescript
// Regular messages: append to the end of the list
setMessages(prev => [...prev, newMessage])

// Short-lived messages (BashTool progress, Sleep tool): replace the last one
setMessages(prev => {
  const lastMsg = prev[prev.length - 1]
  if (lastMsg?.isShortLived && lastMsg.toolUseId === newMessage.toolUseId) {
    // Replace, not append
    return [...prev.slice(0, -1), newMessage]
  }
  return [...prev, newMessage]
})
```

This prevents the Bash tool's progress output from causing the message list to explode (from a handful of messages to hundreds).

---

## Key File Reference

| File | Purpose |
|------|---------|
| `src/ink/ink.tsx` | Custom Ink fork, terminal rendering core (251KB) |
| `src/ink.ts` | Ink component library export entry point |
| `src/screens/REPL.tsx` | REPL main interface (~900 lines) |
| `src/hooks/useCanUseTool.tsx` | Permission decision hook (203 lines) |
| `src/hooks/` | All React hooks (104 files) |
| `src/components/` | UI component library (113 files, 9.5MB) |
