# 14 — 杂项特性：Vim 模式、Keybindings、Voice、Teleport 与 VCR

> 文件：`src/vim/`，`src/keybindings/`，`src/voice/`，`src/utils/teleport/`，`src/services/vcr.ts`

---

## 1. Vim 模式

### 1.1 状态机设计

Vim 模式实现了一个完整的**模态编辑状态机**，精确复现 Vim 的行为：

```typescript
// src/vim/types.ts

// 顶层状态：INSERT 或 NORMAL
type VimState =
  | { mode: 'INSERT'; insertedText: string }   // insertedText 用于 . 重复
  | { mode: 'NORMAL'; command: CommandState }

// NORMAL 模式内部的命令状态机（11 种状态）
type CommandState =
  | { type: 'idle' }                                          // 等待输入
  | { type: 'count'; digits: string }                        // 输入数字前缀（5dd）
  | { type: 'operator'; op: Operator; count: number }        // 输入了 d/c/y，等待 motion
  | { type: 'operatorCount'; op: Operator; count: number; digits: string }
  | { type: 'operatorFind'; op: Operator; count: number; find: FindType }  // df<char>
  | { type: 'operatorTextObj'; op: Operator; count: number; scope: TextObjScope }  // diw
  | { type: 'find'; find: FindType; count: number }          // f/F/t/T 查找
  | { type: 'g'; count: number }                             // g 前缀（gg、gj 等）
  | { type: 'operatorG'; op: Operator; count: number }
  | { type: 'replace'; count: number }                       // r<char> 替换单字符
  | { type: 'indent'; dir: '>' | '<'; count: number }        // >> << 缩进
```

### 1.2 持久状态（跨命令）

```typescript
type PersistentState = {
  lastChange: RecordedChange | null    // 记录上次变更，用于 . 重复
  lastFind: { type: FindType; char: string } | null  // 记录上次 f/t，用于 ; ,
  register: string                     // 默认寄存器（复制粘贴）
  registerIsLinewise: boolean          // 寄存器内容是否为整行
}
```

### 1.3 支持的操作

**Operator（操作符）：** `d`（删除）、`c`（修改）、`y`（复制）

**Motion（移动）：** `h/j/k/l`、`w/W/b/B/e/E`、`0/^/$`、`gg/G`、`%`、`f/F/t/T`

**文本对象（Text Objects）：**

```
iw / aw    内/外 词（word）
iW / aW    内/外 大词（WORD）
i" / a"    内/外 双引号
i' / a'    内/外 单引号
i` / a`    内/外 反引号
i( / a( / ib    内/外 圆括号（支持嵌套）
i[ / a[         内/外 方括号
i{ / a{ / iB    内/外 花括号
i< / a<         内/外 角括号
```

### 1.4 启用方式

```jsonc
// settings.json
{
  "vim": true
}
// 或使用 /vim-mode 命令切换
```

---

## 2. Keybindings 系统

### 2.1 配置文件格式

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
        "ctrl+x ctrl+e": "chat:externalEditor",   // 多键弦！
        "ctrl+x ctrl+k": "chat:killAgents",
        "space": null                              // null = 解除绑定
      }
    }
  ]
}
```

### 2.2 19 种上下文

| 上下文 | 说明 |
|--------|------|
| `Global` | 所有场景通用 |
| `Chat` | 输入框激活时 |
| `Autocomplete` | 自动补全下拉框 |
| `Confirmation` | 权限确认弹框 |
| `Help` | /help 界面 |
| `Transcript` | 对话历史查看模式 |
| `HistorySearch` | Ctrl+R 历史搜索 |
| `Task` | 任务列表 |
| `ThemePicker` | 主题选择器 |
| `Settings` | 设置界面 |
| `Tabs` | 标签栏 |
| `Attachments` | 附件管理 |
| `Footer` | 底部工具栏 |
| `MessageSelector` | 消息选择器 |
| `DiffDialog` | Diff 查看对话框 |
| `ModelPicker` | 模型选择器 |
| `Select` | 通用选择组件 |
| `Plugin` | 插件 UI |

### 2.3 完整动作列表（72+ 个）

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

### 2.4 多键弦匹配（Chord）

```typescript
// 解析阶段
parseChord("ctrl+x ctrl+e")
  → [{ key: 'x', ctrl: true }, { key: 'e', ctrl: true }]

// 匹配阶段（两阶段解析）
// 阶段1：收到第一个键（ctrl+x）
resolveKey(ctx, { key: 'x', ctrl: true }, activeContexts, bindings)
  → { type: 'chord_started', pending: [{ key: 'x', ctrl: true }] }

// 阶段2：收到第二个键（ctrl+e）
resolveKeyWithChordState(ctx, { key: 'e', ctrl: true }, ..., pending)
  → { type: 'match', action: 'chat:externalEditor' }
```

### 2.5 平台特定差异

```typescript
// Windows 终端 VT 模式检测（影响 Shift+Tab 支持）
const SUPPORTS_VT_MODE =
  platform !== 'windows' ||
  (isBun
    ? satisfies(bunVersion, '>=1.2.23')
    : satisfies(nodeVersion, '>=22.17.0 <23.0.0 || >=24.2.0'))

// 模式切换键：
// VT 模式支持：Shift+Tab
// 不支持：Meta+M（备用方案）
const MODE_CYCLE_KEY = SUPPORTS_VT_MODE ? 'shift+tab' : 'meta+m'

// 图片粘贴键：
// Windows：Alt+V（避免与 Ctrl+V 系统剪贴板冲突）
// 其他：Ctrl+V
const IMAGE_PASTE_KEY = platform === 'windows' ? 'alt+v' : 'ctrl+v'
```

### 2.6 不可重绑定的保留键

以下键绑定受到保护（需要双击计时逻辑，不能被普通 keybinding 覆盖）：
- `Ctrl+C`（应用中断）
- `Ctrl+D`（应用退出）

---

## 3. Voice 模式

### 3.1 三层功能门控

```typescript
// src/voice/voiceModeEnabled.ts

// 层 1：Feature Flag（GrowthBook 控制，有紧急禁用开关）
function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}

// 层 2：必须是 Anthropic OAuth 登录（不支持 API Key / Bedrock / Vertex）
function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) return false
  const tokens = getClaudeAIOAuthTokens()  // 从 Keychain 读取，有缓存（~20ms）
  return Boolean(tokens?.accessToken)
}

// 层 3：运行时总检查
export function isVoiceModeEnabled(): boolean {
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}
```

### 3.2 使用方式

```
默认绑定：Chat 上下文中按住 Space → 语音输入（push-to-talk）
松开 Space → 停止录音，语音转文字，提交到输入框

自定义绑定（keybindings.json）：
{ "context": "Chat", "bindings": { "space": null } }      // 禁用语音
{ "context": "Chat", "bindings": { "alt+v": "voice:pushToTalk" } }  // 重绑定
```

**实现方式：** 使用 Claude.ai 的 `voice_stream` API 端点（需要 Anthropic OAuth）

---

## 4. Teleport：跨机器 Session 迁移

### 4.1 使用场景

在机器 A 上开始的 Claude Code 会话，可以迁移到机器 B 继续（保留完整对话历史和 Git 状态）。

### 4.2 迁移流程

```
源机器
  │
  ├─ 1. Claude Haiku 生成会话标题和 Git 分支名（JSON 输出）
  │       标题：≤6 词，如 "Fix login button on mobile"
  │       分支：如 "claude/fix-mobile-login-button"
  │
  ├─ 2. 验证 Git 工作区干净（有未提交改动则失败）
  │
  ├─ 3. 创建 Git Bundle（三级回退）：
  │       尝试 --all（所有分支 + 未推送）→ 100MB 限制
  │       ↓ 太大
  │       尝试 HEAD（当前分支历史）
  │       ↓ 仍太大
  │       squashed-root（单个无父快照）
  │
  ├─ 4. 上传 Bundle 到 Anthropic Files API
  │
  └─ 5. 创建 Session 记录（含 bundle file_id）

目标机器
  │
  ├─ 1. OAuth 获取 Session 历史（分页，100 条/页）
  ├─ 2. 反序列化消息（恢复对话历史）
  ├─ 3. 下载 Bundle，恢复 Git 状态
  └─ 4. 重新进入会话
```

### 4.3 标题和分支名生成（使用 Haiku）

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

// 失败时回退：截断用户第一条消息 + "claude/task"
```

### 4.4 重试策略

```typescript
// 4 次指数退避重试：2s、4s、8s、16s
const RETRY_DELAYS = [2000, 4000, 8000, 16000]

// 只重试瞬时错误（网络错误、5xx）
// 4xx 错误（权限、资源不存在）立即失败
```

---

## 5. VCR 录制回放（测试基础设施）

VCR 是 Claude Code 的**测试固定装置系统**，用于录制真实 API 响应并回放，使测试确定性且不依赖网络。

### 5.1 核心函数

```typescript
// src/services/vcr.ts

// 用于普通 Claude API 调用
async function withVCR(
  messages: Message[],
  f: () => Promise<APIResult[]>
): Promise<APIResult[]>

// 用于流式 API 调用
async function* withStreamingVCR(
  messages: Message[],
  f: () => AsyncGenerator<StreamEvent>
): AsyncGenerator<StreamEvent>

// 用于 token 计数
async function withTokenCountVCR(messages, f)
```

### 5.2 Cassette 文件格式

```jsonc
// fixtures/<name>-<hash12>.json
{
  "input": {
    "dehydrated": "..."    // 脱水后的消息（路径替换为占位符）
  },
  "output": [
    {
      "type": "assistant",
      "uuid": "UUID-0",                // 确定性 UUID（用索引生成）
      "requestId": "REQUEST_ID",       // 固定值
      "message": {
        "model": "claude-3-5-sonnet-20241022",
        "usage": { "input_tokens": 100, "output_tokens": 50 },
        "content": [
          {
            "type": "text",
            "text": "分析 [CWD]/src/main.ts..."  // 路径被替换为占位符
          }
        ]
      }
    }
  ]
}
```

### 5.3 脱水/再水化（确定性哈希）

为了使 Cassette 文件在不同机器上可用，动态值会被替换：

```typescript
// 脱水（录制时）：将动态值替换为占位符
function dehydrateValue(s: unknown): unknown {
  return replace(s, [
    [/\d+/g, '[NUM]'],                    // 数字
    [/\d+ms/g, '[DURATION]'],             // 毫秒时间
    [/\$[\d.]+/g, '[COST]'],              // 成本
    [getCwd(), '[CWD]'],                  // 工作目录
    [getClaudeConfigHomeDir(), '[CONFIG_HOME]'],  // 配置目录
    ['Available commands: ...', 'Available commands: [COMMANDS]'],
  ])
}

// 再水化（回放时）：还原占位符
function hydrateValue(s: unknown): unknown {
  return replace(s, [
    ['[NUM]', '1'],
    ['[DURATION]', '100'],
    ['[CWD]', getCwd()],
    ['[CONFIG_HOME]', getClaudeConfigHomeDir()],
  ])
}
```

### 5.4 启用条件

```bash
# 测试环境自动启用
NODE_ENV=test

# Ant 内部强制启用
FORCE_VCR=1

# 录制新的 Cassette（而不是报错说缺失）
VCR_RECORD=1

# 自定义 Cassette 目录
CLAUDE_CODE_TEST_FIXTURES_ROOT=/path/to/fixtures
```

**CI 行为：** 如果 Cassette 文件缺失且不在录制模式，测试直接报错（确保测试不意外调用真实 API）

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/vim/types.ts` | Vim 状态机类型定义 |
| `src/vim/operators.ts` | d/c/y 等操作符实现 |
| `src/vim/textObjects.ts` | iw/a"/i( 等文本对象 |
| `src/keybindings/parser.ts` | 按键序列解析器 |
| `src/keybindings/resolver.ts` | 多键弦匹配算法 |
| `src/keybindings/defaultBindings.ts` | 平台默认绑定 |
| `src/keybindings/match.ts` | 修饰符匹配逻辑 |
| `src/voice/voiceModeEnabled.ts` | Voice 三层门控 |
| `src/utils/teleport/teleport.tsx` | Teleport 主流程 |
| `src/utils/teleport/api.ts` | Session API 类型和请求 |
| `src/utils/teleport/gitBundle.ts` | Git Bundle 创建和上传 |
| `src/services/vcr.ts` | VCR 核心实现 |
