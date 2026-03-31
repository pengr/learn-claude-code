# 07 — Commands 系统：斜杠命令的注册与分发

> 文件：`src/commands.ts`，`src/commands/`，`src/skills/`

---

## 1. Command 类型定义

Claude Code 的命令系统支持三种命令类型：

```typescript
// src/types/commands.ts（节选）

// 基础属性（所有命令共享）
type CommandBase = {
  name: string                         // 命令名，如 "commit"
  description: string                  // 用户可见描述
  aliases?: string[]                   // 别名，如 ["c"] → /c 也触发
  isHidden?: boolean                   // 是否在 /help 中隐藏
  isEnabled: () => boolean             // 动态开关（基于 feature flag 或条件）
  userInvocable?: boolean              // 用户能否直接调用（vs 仅模型调用）
  source: 'builtin' | 'bundled' | 'user' | 'plugin'  // 命令来源
  loadedFrom?: string                  // 文件路径（user/plugin 命令用）
}

// 类型一：Prompt 命令（最常见）
// 触发后将 prompt 文本注入对话
type PromptCommand = CommandBase & {
  type: 'prompt'
  getPromptForCommand: (args: string) => Promise<string>
  allowedTools?: string[]              // 限制该命令可用的工具集
  model?: string                       // 覆盖使用的模型
  disableModelInvocation?: boolean     // true = 不调用模型，直接执行
  progressMessage?: string             // 执行时的 spinner 文字
  hooks?: HooksSettings                // 命令级 hooks
  context?: string                     // 注入额外上下文
  agent?: string                       // 指定要用的 Agent 类型
}

// 类型二：Local 命令（纯客户端逻辑，不调用 LLM）
type LocalCommand = CommandBase & {
  type: 'local'
  call: (args: string, context: CommandContext) => Promise<void>
}

// 类型三：Local JSX 命令（渲染交互式 React UI）
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

## 2. 命令注册机制

所有命令在 `src/commands.ts` 中集中加载，共有 **4 个来源**：

```typescript
// src/commands.ts（简化）
export async function getCommands(
  toolUseContext: ToolUseContext,
): Promise<Command[]> {
  const commands: Command[] = []

  // 1. 内置命令（硬编码，约 88 个目录）
  commands.push(...getBuiltinCommands())

  // 2. Bundled Skill 命令（来自插件的内置技能）
  commands.push(...getBuiltinPluginSkillCommands())

  // 3. 用户自定义技能（~/.claude/commands/ 和 .claude/commands/）
  const userSkills = await loadUserSkillCommands()
  commands.push(...userSkills)

  // 4. 插件命令（已安装的第三方插件）
  const pluginCommands = await loadPluginCommands()
  commands.push(...pluginCommands)

  // 可用性过滤：根据当前环境过滤掉不可用的命令
  return commands.filter(cmd => cmd.isEnabled())
}
```

### 2.1 内置命令目录（完整列表）

`src/commands/` 下共 88 个子目录，每个目录对应一个命令：

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

## 3. 命令分发机制

用户输入 `/xxx args` 后的完整路由流程：

```
用户输入 "/commit --amend"
        │
        ▼
  1. 解析：isSlashCommand("/commit --amend")
     → name = "commit", args = "--amend"
        │
        ▼
  2. 查找：commands.find(c => c.name === 'commit' || c.aliases?.includes('commit'))
        │
        ▼
  3. 可用性检查：cmd.isEnabled()
     → false：显示 "命令不可用" 提示
     → true：继续
        │
        ▼
  4. 类型分发：
     │
     ├─ type === 'local'
     │    └─ await cmd.call(args, context)
     │         直接执行，不走 LLM
     │
     ├─ type === 'local-jsx'
     │    └─ const jsx = await cmd.call(args, onDone, context)
     │       context.setToolJSX({ jsx, shouldHidePromptInput: true })
     │         渲染交互式 UI
     │
     └─ type === 'prompt'
          └─ const prompt = await cmd.getPromptForCommand(args)
             将 prompt 注入对话，走正常 LLM 流程
```

### 3.1 命令执行上下文（CommandContext）

```typescript
type CommandContext = {
  toolUseContext: ToolUseContext    // 完整工具上下文
  setToolJSX: SetToolJSXFn         // 注入 React UI
  addNotification: (n) => void     // 显示通知
  messages: Message[]              // 当前对话
  appendSystemMessage: (msg) => void  // 追加系统消息
}
```

---

## 4. 可用性过滤

命令通过 `isEnabled()` 决定是否可用，常见条件：

```typescript
// 示例：需要 Git 仓库
{
  isEnabled: () => isGitRepo(getCwd()),
}

// 示例：Feature Flag 控制
{
  isEnabled: () => feature('SOME_FEATURE'),
}

// 示例：仅 Claude.ai 用户
{
  isEnabled: () => isClaudeAiUser(),
}

// 示例：需要联网
{
  isEnabled: () => !isOfflineMode(),
}

// 示例：仅 Ant 内部用户
{
  isEnabled: () => process.env.USER_TYPE === 'ant',
}
```

---

## 5. 用户自定义技能（Skill Commands）

用户可以在以下位置创建 `.md` 文件来添加自定义斜杠命令：

```
优先级（高→低）：

  1. .claude/commands/              本地项目级（优先级最高）
  2. ~/.claude/commands/            用户级全局命令
  3. 插件安装的命令
  4. 内置 bundled 命令
  5. 内置 builtin 命令             （最低优先级）
```

### 5.1 Skill 文件格式

```markdown
---
description: 提交并推送当前改动
argumentHint: "[commit message]"
allowedTools: ["Bash"]
---

请帮我提交并推送当前的改动。
提交信息：$ARGUMENTS

步骤：
1. git add -A
2. git commit -m "$ARGUMENTS"
3. git push
```

YAML frontmatter 字段：

| 字段 | 说明 |
|------|------|
| `description` | 在 `/help` 中显示的描述 |
| `argumentHint` | 参数提示，如 `"<branch-name>"` |
| `allowedTools` | 限制可用工具（`["Bash", "Read"]` 等） |
| `model` | 覆盖使用的模型 |
| `disableModelInvocation` | `true` = 直接作为系统消息注入，不调用 LLM |
| `whenToUse` | 告诉模型什么时候自动调用此命令 |
| `agent` | 指定 Agent 类型 |

### 5.2 内置技能目录（SkillTool 调用）

除了用户直接 `/xxx` 调用外，技能还可以被模型通过 `SkillTool` 工具调用：

```typescript
// SkillTool 让模型能主动触发用户定义的 skill
// 模型看到的工具名格式：skill__<command_name>
// 输入：{ name: "commit", args: "fix: typo in README" }
```

---

## 6. 典型命令实现

### 6.1 `/compact` — 手动压缩（Local 命令）

```typescript
// src/commands/compact/index.ts
{
  type: 'local',
  name: 'compact',
  description: '压缩对话历史以释放上下文空间',
  isEnabled: () => true,
  call: async (args, context) => {
    const { toolUseContext } = context
    // 触发手动 compaction 流程
    await triggerManualCompaction(toolUseContext)
    context.appendSystemMessage({
      type: 'system',
      content: '对话历史已压缩。',
    })
  },
}
```

### 6.2 `/config` — 配置管理（Local JSX 命令）

```typescript
// src/commands/config/index.ts
{
  type: 'local-jsx',
  name: 'config',
  description: '查看和修改 Claude Code 配置',
  isEnabled: () => true,
  call: async (args, onDone, context) => {
    // 返回一个交互式配置 UI
    return <ConfigEditor onDone={onDone} />
  },
}
```

### 6.3 `/permissions` — 权限管理（Local JSX 命令）

```typescript
// src/commands/permissions/index.ts
{
  type: 'local-jsx',
  name: 'permissions',
  description: '查看和管理工具权限规则',
  call: async (args, onDone, context) => {
    return <PermissionsEditor
      permissionContext={context.toolUseContext.getAppState().toolPermissionContext}
      onDone={onDone}
    />
  },
}
```

### 6.4 `/review` — 代码审查（Prompt 命令）

```typescript
// src/commands/review/index.ts
{
  type: 'prompt',
  name: 'review',
  description: '审查当前改动',
  getPromptForCommand: async (args) => {
    const diff = await execGitDiff()
    return `请审查以下代码改动：\n\n${diff}\n\n${args || '请关注潜在的 bug、代码风格和性能问题。'}`
  },
}
```

---

## 7. 命令与 Agent 的关系

Prompt 命令支持指定 `agent` 字段，让命令在特定 Agent 环境中运行：

```markdown
---
description: 在隔离的 Git Worktree 中执行任务
agent: worktree-agent
---
```

这会触发 AgentTool，在独立的 worktree 中执行命令，不影响主工作区。

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/commands.ts` | 命令注册中心，汇总所有来源 |
| `src/commands/<name>/index.ts` | 各命令实现 |
| `src/skills/` | Skill 文件加载逻辑 |
| `src/types/commands.ts` 或 `commands.ts` | Command 类型定义 |
| `src/plugins/builtinPlugins.ts` | 内置插件的 Skill 命令注册 |
