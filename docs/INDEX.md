# Learn Claude Code — 完整文档索引

> 基于 Claude Code 源码的完整技术分析文档集
>
> 目标读者：想理解 Claude Code 内部原理的开发者、想构建类似工具的工程师

---

## 快速导航

### 🗺️ 总览与架构

| 文档 | 内容简介 |
|------|----------|
| **[00 — 总览](./00-overview.md)** | 系统架构图、15 个核心系统速查、关键数字、技术决策解析 |
| **[README（原始版）](./README.md)** | Agent Loop、工具系统、权限、压缩、API 的入门概览（749 行）|

### 🔧 核心 Agent 机制

| 文档 | 内容简介 |
|------|----------|
| **[01 — Agent Loop](./01-agent-loop.md)** | query.ts 1729 行深度解析：8 个阶段、State 状态机、转换原因、终止条件 |
| **[02 — 工具系统](./02-tool-system.md)** | 44 个工具完整清单、buildTool() 工厂、ToolUseContext 字段、工具生命周期 |
| **[03 — 权限系统](./03-permission-system.md)** | 7 层流水线、6 种模式、Glob 匹配、危险命令检测、Windows 绕过防御 |
| **[04 — 上下文压缩](./04-compaction.md)** | 4 层压缩架构：Snip → Microcompact → Collapse → Autocompact，阈值计算 |

### 🌐 基础设施层

| 文档 | 内容简介 |
|------|----------|
| **[05 — API 层](./05-api-layer.md)** | 4 种后端、OAuth PKCE 7 步流程、Prompt Cache 插入规则、withRetry 算法 |
| **[06 — REPL 界面](./06-repl-and-ink.md)** | 自定义 Ink Fork、双缓冲渲染、REPL 状态机、setToolJSX 注入机制 |
| **[07 — Commands 系统](./07-commands.md)** | 88+ 斜杠命令、3 种类型、调度流程、Skill 格式 |
| **[08 — 内存系统](./08-memory-system.md)** | CLAUDE.md 7 层优先级、@import 语法、memdir、Session Memory 阈值 |
| **[09 — MCP 集成](./09-mcp.md)** | 6 种传输协议、MCPServerConnection 状态机、工具注册、配置格式 |

### ⚙️ 配置、状态与数据

| 文档 | 内容简介 |
|------|----------|
| **[10 — 状态与配置](./10-state-and-config.md)** | AppState 450+ 字段、不可变更新、5 层配置优先级、11 次数据库迁移 |
| **[11 — Cost & History](./11-cost-and-history.md)** | 模型定价表、成本公式、history.jsonl、paste-cache、Esc 撤销机制 |

### 🚀 高级特性

| 文档 | 内容简介 |
|------|----------|
| **[12 — 高级特性](./12-advanced-features.md)** | 多 Agent Fork Prompt Cache 优化、Bridge 远程控制、LSP 集成、插件系统 |
| **[13 — 安全机制](./13-security.md)** | 权限规则、危险命令检测、Auto 模式 ML 分类器、DenialTracking 防循环 |
| **[14 — 杂项特性](./14-misc-features.md)** | Vim 11 状态机、Keybindings 19 上下文、Voice 3 层门控、Teleport、VCR |

---

## 学习路径建议

### 路径 A：快速了解（30 分钟）

1. [00 — 总览](./00-overview.md)：建立整体概念
2. [README](./README.md)：核心数据流

### 路径 B：技术深度（2-3 小时）

1. [00 — 总览](./00-overview.md)
2. [01 — Agent Loop](./01-agent-loop.md)：理解核心循环
3. [02 — 工具系统](./02-tool-system.md)：理解工具机制
4. [04 — 上下文压缩](./04-compaction.md)：理解最复杂的优化
5. [05 — API 层](./05-api-layer.md)：理解 Prompt Cache

### 路径 C：构建类似系统（全部文档）

按上方列表顺序阅读全部 15 份文档，每份约 20-40 分钟。

---

## 各文档主题词速查

| 主题词 | 相关文档 |
|--------|----------|
| AsyncGenerator、yield、流式 | 01, 05, 06 |
| 工具调用、tool_use、tool_result | 01, 02, 03 |
| Prompt Cache、缓存命中 | 01, 04, 05, 12 |
| Zod schema、输入校验 | 02, 03 |
| 权限弹框、ML 分类器 | 03, 06 |
| 上下文压缩、context window | 01, 04 |
| Anthropic API、Bedrock、Vertex | 05 |
| OAuth、PKCE、Keychain | 05 |
| React、Ink、终端 UI | 06 |
| 斜杠命令、/compact、/help | 07 |
| CLAUDE.md、项目内存 | 08 |
| MCP、Model Context Protocol | 09 |
| settings.json、配置优先级 | 10 |
| 成本追踪、token 定价 | 11 |
| 子 Agent、Fork、Coordinator | 12 |
| Bridge、远程控制 | 12 |
| LSP、语言服务器 | 12 |
| 插件、Plugin | 12 |
| 危险命令、安全 | 03, 13 |
| Vim 模式、状态机 | 14 |
| Keybindings、快捷键 | 14 |
| Voice、语音输入 | 14 |
| Teleport、会话迁移 | 14 |
| VCR、测试 fixtures | 14 |

---

## 源码文件快速索引

| 源文件 | 所在文档 |
|--------|----------|
| `src/query.ts` | 01 |
| `src/Tool.ts` | 02 |
| `src/tools/` | 02, 13 |
| `src/utils/permissions/` | 03, 13 |
| `src/utils/sandbox/` | 03, 13 |
| `src/services/compact/` | 04 |
| `src/services/contextCollapse/` | 04 |
| `src/services/api/` | 05 |
| `src/screens/REPL.tsx` | 06 |
| `src/ink/ink.tsx` | 06 |
| `src/hooks/` | 06, 03 |
| `src/commands/` | 07 |
| `src/utils/memory/` | 08 |
| `src/services/mcp/` | 09 |
| `src/state/AppState.ts` | 10 |
| `src/utils/config.ts` | 10 |
| `src/cost-tracker.ts` | 11 |
| `src/history.ts` | 11 |
| `src/tools/AgentTool/` | 12 |
| `src/bridge/` | 12 |
| `src/services/lsp/` | 12 |
| `src/plugins/` | 12 |
| `src/vim/` | 14 |
| `src/keybindings/` | 14 |
| `src/voice/` | 14 |
| `src/utils/teleport/` | 14 |
| `src/services/vcr.ts` | 14 |

---

*文档集基于 Claude Code 源码分析编写，覆盖约 95% 的核心技术系统。*
