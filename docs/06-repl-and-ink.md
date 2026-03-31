# 06 — REPL 界面层：自定义 Ink 引擎与 React Hooks

> 文件：`src/screens/REPL.tsx`，`src/ink/`，`src/hooks/`，`src/ink.ts`

---

## 1. 技术基础：React + Ink

Claude Code 的终端 UI 用 **React** 写，通过 **Ink** 渲染到终端。Ink 的本质是把 React 的虚拟 DOM 映射到终端字符网格：

```
React 组件树
    │
    ▼
Ink Renderer（布局引擎）
    │ Flexbox 布局计算
    ▼
字符网格（二维 char[] + style[]）
    │
    ▼
终端 stdout（ANSI 转义序列）
```

---

## 2. 自定义 Ink Fork（src/ink/ink.tsx，251KB）

Claude Code **没有使用标准 Ink**，而是维护了一个深度定制的 Fork，包含多项关键优化：

### 2.1 微任务延迟渲染

```typescript
// 标准 Ink：setImmediate 触发渲染（有 1 个事件循环延迟）
// 自定义 Fork：queueMicrotask（更快，在当前 tick 末尾触发）

class InkRenderer {
  private onRender = () => {
    this.calculateLayout()
    this.output()
  }

  scheduleRender() {
    // queueMicrotask 比 setImmediate 延迟更低
    // 原生终端光标移动没有可感知的延迟
    queueMicrotask(this.onRender)
  }
}
```

### 2.2 显式双缓冲

```typescript
// 前缓冲：当前显示在终端的内容
private frontFrame: CharGrid

// 后缓冲：新计算出的内容
private backFrame: CharGrid

// 每次渲染：
// 1. 计算新布局 → 写入 backFrame
// 2. diff(frontFrame, backFrame) → 只发送变化的字符
// 3. swap(frontFrame, backFrame)

// 差异输出使用 log-update 库，最小化 stdout 写入量
```

### 2.3 内存池化（防长会话泄漏）

```typescript
// 每 5 分钟重置一次内存池
// 防止长时间会话中的字符/样式对象无界增长

const POOL_RESET_INTERVAL = 5 * 60 * 1000  // 5 分钟

class StylePool {
  private pool: Map<string, Style> = new Map()

  reset() {
    this.pool.clear()  // 定期清空，GC 回收旧对象
  }
}

const stylePool = new StylePool()
const charPool  = new CharPool()

setInterval(() => {
  stylePool.reset()
  charPool.reset()
}, POOL_RESET_INTERVAL)
```

### 2.4 Alt Screen 支持

```typescript
// 进入 Alt Screen（隐藏原终端内容，类似 vim/less 的全屏模式）
enterAltScreen()   // ESC[?1049h
exitAltScreen()    // ESC[?1049l

// Alt Screen 中支持：
// - 文本选中追踪（鼠标事件）
// - 搜索高亮
// - 鼠标点击追踪
// - BSU/ESU 原子化光标操作（防止光标位置跳动）
```

### 2.5 布局损伤追踪

```typescript
// 检测是否需要完整重绘（而不仅仅是局部更新）
function didLayoutShift(prev: CharGrid, next: CharGrid): boolean {
  // 如果布局发生了结构性变化（行数变化、列宽变化）
  // 标记"损伤区域"，下次渲染时完整重绘该区域
  return prev.rows !== next.rows || hasStructuralChange(prev, next)
}
```

---

## 3. ink.ts：组件库导出

`src/ink.ts` 是自定义 Ink Fork 的导出入口，为整个项目提供终端 UI 组件：

```typescript
// src/ink.ts 导出的主要组件和 API
export {
  // 布局
  Box,          // 类似 <div>，支持 Flexbox（flexDirection, gap, padding 等）
  Text,         // 纯文本，支持 color, bold, italic, underline
  Newline,      // 换行符
  Spacer,       // 弹性空白（flex: 1）
  Static,       // 不参与重渲染的静态内容

  // 交互
  useInput,     // 键盘输入 Hook
  useFocus,     // 焦点管理
  useStdout,    // 直接写 stdout
  useStderr,    // 直接写 stderr

  // 渲染控制
  render,       // 渲染根组件
  measureElement,  // 测量组件尺寸

  // 主题
  ThemeProvider,   // 提供 Claude Code 主题颜色
}
```

---

## 4. REPL.tsx：核心交互界面

`src/screens/REPL.tsx`（~900 行）是整个 UI 的入口组件。

### 4.1 核心 State

```typescript
function REPL() {
  // 对话消息列表
  const [messages, setMessages] = useState<Message[]>([])

  // 输入框内容
  const [inputValue, setInputValue] = useState('')

  // 当前工具注入的自定义 UI
  const [toolJSX, setToolJSXState] = useState<ToolJSXState | null>(null)

  // 是否正在等待 LLM 响应
  const [isQuerying, setIsQuerying] = useState(false)

  // 流式输出模式（spinner 样式）
  const [streamMode, setStreamMode] = useState<SpinnerMode>('thinking')

  // 正在执行的工具数量
  const [inProgressToolCount, setInProgressToolCount] = useState(0)
}
```

### 4.2 流式渲染：消费 query() 的 AsyncGenerator

REPL 通过 `for await` 消费 `query()` 产生的事件流，实时更新 UI：

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
      setStreamMode('thinking')          // 显示"思考中..."spinner
      break

    case 'assistant':
      // 新的助手消息（可能是流式追加的文本）
      setMessages(prev => appendOrUpdate(prev, event))
      break

    case 'user':
      // 工具结果消息
      setMessages(prev => [...prev, event])
      break

    case 'tombstone':
      // 抹掉失败的消息（模型切换时）
      setMessages(prev => prev.filter(m => m.uuid !== event.message.uuid))
      break

    case 'system':
      // 系统提示（压缩通知等）
      setMessages(prev => [...prev, event])
      break
  }
}
```

### 4.3 setToolJSX 注入机制

工具可以向 REPL 注入自定义 React 组件，用于交互式 UI：

```typescript
// 工具执行时调用
context.setToolJSX?.({
  jsx: <InteractivePermissionDialog tool={tool} onAllow={...} onDeny={...} />,
  shouldHidePromptInput: true,   // 隐藏输入框，等用户操作完
  showSpinner: false,
})

// 工具完成后清除
context.setToolJSX?.(null)
```

**状态保护逻辑：**

```typescript
// 本地 JSX 命令（/btw, /config 等）激活后有保护
// 只有显式 clearLocalJSX: true 才能清除
// 防止后台工具意外覆盖前台交互 UI

function setToolJSX(args: SetToolJSXArgs | null) {
  if (args === null && localJSXCommandRef.current && !args?.clearLocalJSX) {
    return  // 本地命令期间，忽略工具的清除请求
  }
  setToolJSXState(args)
}
```

---

## 5. React Hooks（src/hooks/）

### 5.1 useCanUseTool：权限决策 Hook

这是最关键的 Hook，实现了完整的权限决策流水线（203 行）：

```typescript
// src/hooks/useCanUseTool.tsx
export function useCanUseTool(): CanUseToolFn {
  return useCallback(async (tool, input, context) => {

    // 1. 检查配置规则（alwaysAllow / alwaysDeny）
    const configResult = checkConfigRules(tool, input, context)
    if (configResult.behavior !== 'ask') return configResult

    // 2. Auto 模式：ML 分类器
    if (permissionMode === 'auto') {
      const classifierInput = tool.toAutoClassifierInput(input)

      // 恩典期（Grace Period）：给分类器 2 秒时间
      // 如果分类器在 2 秒内完成且批准 → 自动通过，无需打扰用户
      const classifierResult = await Promise.race([
        runClassifier(classifierInput),
        sleep(2000).then(() => null),  // 超时 → 返回 null
      ])

      if (classifierResult?.approved) {
        return { behavior: 'allow' }  // 快速路径，用户无感知
      }
    }

    // 3. 非交互模式（CI/CD）：自动决策，不弹框
    if (context.isNonInteractiveSession) {
      return autoDecideForNonInteractive(tool, input, permissionMode)
    }

    // 4. 弹出交互式确认框（等待用户操作）
    return await showPermissionDialog(tool, input, context)

  }, [permissionMode, ...deps])
}
```

### 5.2 useInput：键盘输入处理

```typescript
// 来自 Ink，用于捕获终端按键
useInput((input: string, key: Key) => {
  // input: 字符串（普通字符）
  // key: { upArrow, downArrow, leftArrow, rightArrow, escape, return, ctrl, meta, shift, tab, backspace, delete }

  if (key.escape) {
    handleCancel()
    return
  }

  if (key.return) {
    handleSubmit()
    return
  }

  // Ctrl+C 特殊处理（需要双击计时）
  if (key.ctrl && input === 'c') {
    handleInterrupt()
    return
  }
})
```

### 5.3 其他重要 Hooks

```typescript
// useCancelRequest：处理中断（Ctrl+C / Esc）
export function useCancelRequest(abortController: AbortController) {
  // 发送 abort 信号，停止当前的 API 请求和工具执行
}

// useKeyBindings：应用用户配置的快捷键
export function useKeyBindings(): ResolvedBindings {
  // 加载 ~/.claude/keybindings.json
  // 与平台默认值合并
  // 返回 action → chord 的映射
}

// useTerminalSize：响应终端尺寸变化
export function useTerminalSize(): { columns: number; rows: number } {
  // 监听 SIGWINCH 信号
  // 触发重新布局
}
```

---

## 6. REPL 整体布局结构

```tsx
function REPL() {
  return (
    <ThemeProvider theme={currentTheme}>
      <Box flexDirection="column" height="100%">

        {/* 消息列表区域（可滚动） */}
        <MessageList
          messages={messages}
          tools={tools}
          verbose={verbose}
        />

        {/* 工具注入的自定义 UI（如权限对话框） */}
        {toolJSX && (
          <Box>
            {toolJSX.jsx}
          </Box>
        )}

        {/* 底部输入区域 */}
        {!toolJSX?.shouldHidePromptInput && (
          <InputArea
            value={inputValue}
            onChange={setInputValue}
            onSubmit={handleSubmit}
            isQuerying={isQuerying}
            streamMode={streamMode}
          />
        )}

        {/* 状态栏（模型、token 用量、成本） */}
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

## 7. 短期消息优化

对于频繁更新的工具输出（如 Bash 进度、Sleep 倒计时），使用**替换策略**而非追加：

```typescript
// 普通消息：追加到列表末尾
setMessages(prev => [...prev, newMessage])

// 短期消息（BashTool 进度、Sleep 工具）：替换上一条
setMessages(prev => {
  const lastMsg = prev[prev.length - 1]
  if (lastMsg?.isShortLived && lastMsg.toolUseId === newMessage.toolUseId) {
    // 替换，而不是追加
    return [...prev.slice(0, -1), newMessage]
  }
  return [...prev, newMessage]
})
```

这样可以防止 Bash 工具的进度输出导致消息列表爆炸（从几条变成几百条）。

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/ink/ink.tsx` | 自定义 Ink Fork，终端渲染核心（251KB） |
| `src/ink.ts` | Ink 组件库导出入口 |
| `src/screens/REPL.tsx` | REPL 主界面（~900 行） |
| `src/hooks/useCanUseTool.tsx` | 权限决策 Hook（203 行） |
| `src/hooks/` | 所有 React Hooks（104 个文件） |
| `src/components/` | UI 组件库（113 个文件，9.5MB） |
