# 00 — 总览：Claude Code 完整系统地图

> 基于 Claude Code 源码（~1332 个 TypeScript 文件，~9.5MB 组件代码）的全面技术分析

---

## 1. 项目定位

Claude Code 是 Anthropic 开发的**终端 AI Coding Agent**。与 ChatGPT、Cursor 等工具不同，它是：
- **CLI 工具**：在终端运行，无需 IDE 插件
- **完全 Agentic**：自主调用 44+ 工具（读/写文件、执行命令、搜索 Web）
- **React 驱动的 UI**：使用 Ink（React for CLI）渲染终端界面
- **完整 TypeScript 实现**：~1332 个 .ts 文件，运行在 Bun 或 Node.js 上

---

## 2. 核心技术栈

```
TypeScript + React (Ink) + Bun/Node.js
    │
    ├─ 运行时：Bun（首选）或 Node.js
    ├─ UI 框架：Ink（自定义 Fork，终端 React）
    ├─ 状态管理：自制 AppState（450+ 字段的不可变状态树）
    ├─ Schema 验证：Zod v4
    ├─ 包管理：npm（发布），内部 Bun
    └─ 构建：bun:bundle（支持 feature flags 的死代码消除）
```

---

## 3. 系统架构图

```
┌─────────────────────────────────────────────────────────┐
│                      用户界面层                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │  REPL.tsx（React/Ink）                           │   │
│  │  • 消息列表 • 输入框 • 状态栏 • 工具 UI 注入     │   │
│  └─────────────────┬────────────────────────────────┘   │
│                    │                                      │
│  ┌─────────────────▼────────────────────────────────┐   │
│  │  Commands 系统（88+ 斜杠命令）                   │   │
│  │  /compact  /clear  /help  /vim-mode  ...         │   │
│  └─────────────────┬────────────────────────────────┘   │
└────────────────────┼────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                    Agent Loop 层                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  query.ts（1729 行 AsyncGenerator）              │   │
│  │  • Snip → Microcompact → Collapse → Autocompact  │   │
│  │  • callModel → 流式处理 → 工具执行 → 循环       │   │
│  └─────────────────┬────────────────────────────────┘   │
│                    │                                      │
│  ┌─────────────────▼────────────────────────────────┐   │
│  │  工具系统（44 个工具）                            │   │
│  │  Bash│Read│Edit│Write│Glob│Grep│WebFetch│Agent...│   │
│  └─────────────────┬────────────────────────────────┘   │
└────────────────────┼────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                    基础服务层                             │
│  ┌────────────┐ ┌──────────┐ ┌───────────┐             │
│  │ API 层     │ │ MCP 层   │ │ LSP 层    │             │
│  │ 4 种后端  │ │ 6 种传输 │ │ JSON-RPC  │             │
│  └────────────┘ └──────────┘ └───────────┘             │
│  ┌────────────┐ ┌──────────┐ ┌───────────┐             │
│  │ 权限系统   │ │ 内存系统 │ │ 配置系统  │             │
│  │ 7 层流水线 │ │ CLAUDE.md│ │ 5 层优先级│             │
│  └────────────┘ └──────────┘ └───────────┘             │
└─────────────────────────────────────────────────────────┘
```

---

## 4. 15 个核心系统速查

### 4.1 Agent Loop（`src/query.ts`）

Claude Code 的心脏。一个 `while(true)` 的 AsyncGenerator，每轮迭代：
1. 检查上下文大小 → 按需压缩（4 层）
2. 调用模型 API（流式）
3. 收集 tool_use blocks
4. 执行工具（并行或串行）
5. 把工具结果追加到消息，继续下一轮

**终止条件：** 模型不再调用工具、用户中断、达到 maxTurns、上下文超限、Stop Hook 阻止

---

### 4.2 工具系统（`src/tools/`，44 个工具）

所有工具通过 `buildTool(def)` 工厂函数创建，实现 `Tool<Input, Output>` 接口。

**工具分类：**
- 文件操作：Read, Write, Edit, Glob, Grep, NotebookEdit
- Shell：Bash, PowerShell, REPL
- Agent 协调：Agent, SendMessage, AskUserQuestion, TeamCreate
- 任务管理：TaskCreate/Update/List/Get/Output/Stop
- Web：WebFetch, WebSearch
- MCP：MCPTool, McpAuth, ListMcpResources, ReadMcpResource
- UI/交互：Config, EnterPlanMode, ExitPlanMode, EnterWorktree, Skill, ToolSearch
- 辅助：Sleep, Brief, LSPTool, ScheduleCron

---

### 4.3 上下文压缩（`src/services/compact/`）

4 层递进式压缩，越往后代价越高但压缩比越大：

| 层次 | 机制 | 触发 | 代价 |
|------|------|------|------|
| 工具结果外置 | 大结果写磁盘 | 单结果超限 | 最低 |
| Snip | 裁剪旧工具内容 | token 超阈值 | 低 |
| Microcompact | 清空旧工具结果 | token 超阈值 | 低 |
| Context Collapse | 渐进折叠摘要 | token 超阈值 | 中 |
| Autocompact | AI 摘要全对话 | 接近窗口上限 | 高（调用 Haiku）|

---

### 4.4 API 层（`src/services/api/`）

4 种后端，统一接口：
- **Direct**：直接调用 Anthropic API（`api.anthropic.com`）
- **Bedrock**：AWS 托管（需要 AWS 凭证）
- **Vertex**：Google Cloud 托管（GCP 服务账号）
- **Azure**：Azure AI（端点 URL + API Key）

OAuth PKCE 7 步流程用于 Anthropic 账号认证（Mac Keychain 存储）。

Prompt Cache：自动在 3 个位置插入 `cache_control`（工具列表末尾、CLAUDE.md 末尾、最新用户消息末尾）。

---

### 4.5 权限系统（`src/utils/permissions/`）

7 层流水线，决策优先级 Deny > Ask > Allow：

```
validateInput → 危险命令检测 → deny 规则 → allow 规则
  → ML 分类器 → 非交互模式自动决策 → 交互确认弹框
```

6 种权限模式：`default`、`plan`、`acceptEdits`、`auto`、`bypassPermissions`、`dontAsk`

---

### 4.6 REPL 界面（`src/screens/REPL.tsx`，`src/ink/`）

自定义 Ink Fork（比标准 Ink 快）：
- `queueMicrotask` 渲染（比 `setImmediate` 延迟更低）
- 显式双缓冲（diff 输出，最小化 stdout 写入）
- 5 分钟内存池重置（防长会话泄漏）
- Alt Screen 支持（全屏模式）

---

### 4.7 Commands 系统（`src/commands/`，88+ 命令）

3 种命令类型：
- `prompt`：替换输入框内容（`/help`、`/compact`）
- `local`：副作用，无 UI（`/clear`、`/login`）
- `local-jsx`：注入 React 组件到 REPL（`/config`、`/theme`）

---

### 4.8 内存系统（CLAUDE.md）

7 层优先级发现（从高到低）：`~/.claude/CLAUDE.md` → 每级父目录 → `工作目录/CLAUDE.md` → 子目录

支持 `@import path/to/file` 内联导入（3 级深度限制，防循环）。

---

### 4.9 MCP 集成（`src/services/mcp/`）

6 种传输协议：`stdio`、`sse`、`http-sse`、`streamable-http`、`http-streaming`、`in-process`

MCP 工具注册为：`mcp__<serverName>__<toolName>`（自动处理命名冲突）

---

### 4.10 状态管理（`src/state/AppState.ts`）

450+ 字段的不可变状态树。所有 UI 状态、权限上下文、MCP 连接等都在这里。

更新模式：`setAppState(prev => ({ ...prev, field: newValue }))`

5 层配置优先级：`user settings` > `project settings` > `local settings` > `CLI flags` > `policy settings`

---

### 4.11 Cost Tracking（`src/cost-tracker.ts`）

内置所有模型的定价表。每次 API 调用后更新，保存到 `.claude/costs.json`，支持 `--resume` 时恢复。

对话历史存储在 `~/.claude/history.jsonl`（JSONL 格式，最多 100 条，大内容外置到 `paste-cache/`）。

---

### 4.12 多 Agent（`src/tools/AgentTool/`）

Fork 子 Agent 的关键优化：**所有 Fork 共享同一 Prompt Cache 前缀**（用相同占位符文本），只有末尾的 directive 不同。

```
[...完全相同的消息 + 相同占位符]  ← 所有 Fork 共享，Cache 命中
[唯一的 directive]               ← 每个 Fork 不同
```

---

### 4.13 安全机制（`src/utils/permissions/`，`src/utils/sandbox/`）

- **Glob 匹配**：使用 `ignore` 库（gitignore 格式）
- **符号链接检查**：同时检查原始路径和 `realpath`
- **Windows 路径绕过**：7 种 NTFS 特殊路径攻击检测
- **危险命令黑名单**：`rm -rf /`、fork bomb、`curl | bash` 等
- **ML 分类器**：2 秒恩典期，超时则人工确认
- **DenialTracking**：连续拒绝 3 次或总计拒绝 20 次，回退到人工确认

---

### 4.14 Vim/Keybindings（`src/vim/`，`src/keybindings/`）

- Vim 模式：11 种 CommandState 的完整状态机，支持文本对象、`.` 重复、寄存器
- Keybindings：19 种上下文，72+ 个 action，支持多键弦（`ctrl+x ctrl+e`）
- 平台差异：Windows VT 模式检测，Shift+Tab 支持判断

---

### 4.15 VCR 测试框架（`src/services/vcr.ts`）

录制真实 API 响应为 cassette 文件（JSON），回放时不调用真实 API。

脱水/再水化：动态值（路径、数字、时间戳）替换为占位符，确保跨机器可用。

---

## 5. 关键数字速查

| 指标 | 数值 |
|------|------|
| TypeScript 文件数 | ~1332 |
| 组件代码 | ~9.5 MB |
| 工具数量 | 44 |
| Slash 命令数 | 88+ |
| AppState 字段数 | 450+ |
| Keybinding 上下文数 | 19 |
| Keybinding Action 数 | 72+ |
| Vim CommandState 数 | 11 |
| 配置优先级层数 | 5 |
| 权限决策流水线层数 | 7 |
| 上下文压缩层数 | 4 |
| API 后端数 | 4 |
| MCP 传输协议数 | 6 |
| 数据库迁移版本数 | 11 |
| query.ts 行数 | 1729 |
| 自定义 Ink Fork | 251 KB |
| Prompt Cache 插入点 | 3 |
| Autocompact 缓冲区 | 13,000 tokens |
| 对话历史最大条数 | 100 |
| 历史内联阈值 | 1024 字节 |

---

## 6. 文件系统结构

```
~/.claude/
  ├─ CLAUDE.md              # 全局内存（最高优先级）
  ├─ settings.json          # 用户配置（第1层优先级）
  ├─ keybindings.json       # 自定义快捷键
  ├─ history.jsonl          # 对话历史（FIFO 100 条）
  ├─ paste-cache/           # 大粘贴内容外置存储
  └─ projects/
       └─ <project-hash>/
            └─ ...           # 项目级配置

<project>/
  ├─ CLAUDE.md              # 项目内存
  ├─ .claude/
  │    ├─ settings.json     # 项目配置（第2层优先级）
  │    ├─ settings.local.json # 本地覆盖（第3层，不提交 git）
  │    ├─ costs.json        # 成本追踪
  │    └─ commands/         # 自定义命令目录
  └─ ...
```

---

## 7. 技术决策解析

### 为什么用 React/Ink 写 CLI？

Ink 将 React 虚拟 DOM 映射到终端字符网格，使 Claude Code 能用声明式的方式构建复杂的终端 UI（多面板、富文本、动态更新）而无需手动管理 ANSI 转义序列。

### 为什么 Prompt Cache 如此重要？

对于长对话，Prompt Cache 可以把输入成本从 $15/MTok 降低到 $1.5/MTok（90% 折扣）。Claude Code 的所有设计（消息不可变、Fork 子 Agent 共享前缀、Microcompact 不破坏 cache）都在最大化 cache 命中率。

### 为什么用 Zod 而不是 TypeScript 类型？

工具的 `inputSchema` 是 Zod schema，可以在运行时验证 LLM 的工具参数，并自动生成 JSON Schema 传给 Anthropic API。这是类型安全 + 运行时验证的最佳实践。

### 为什么 Bun 而不是 Node.js？

Bun 提供更快的启动速度（重要，因为用户在命令行中频繁调用）。同时，Bun 的 `bun:bundle` 支持基于 `feature()` 函数的条件编译，实现内部功能的树摇（如 Ant 内部特性不出现在外部版本中）。
