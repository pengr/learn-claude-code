# 08 — Memory 系统：CLAUDE.md、memdir 与 Session Memory

> 文件：`src/memdir/`，`src/services/SessionMemory/`，`src/context.ts`，`src/utils/claudemd.ts`

---

## 1. Memory 系统全景

Claude Code 的"记忆"分为四个层次，从持久到临时：

```
┌──────────────────────────────────────────────────────────┐
│  Layer 1: CLAUDE.md 文件                                  │
│  永久存储在磁盘，跨会话持久，手动维护                        │
│  位置：项目目录 / 用户主目录 / 系统目录                      │
├──────────────────────────────────────────────────────────┤
│  Layer 2: memdir（MEMORY.md）                             │
│  ~/.claude/projects/<hash>/memory/                        │
│  模型自动写入，跨会话持久，按类型分类                        │
├──────────────────────────────────────────────────────────┤
│  Layer 3: Session Memory                                  │
│  当前会话的自动摘要（token 触发生成）                        │
│  用于超长对话的上下文压缩辅助                               │
├──────────────────────────────────────────────────────────┤
│  Layer 4: 对话历史（messages[]）                           │
│  纯内存，会话结束即消失（由 compaction 管理）                │
└──────────────────────────────────────────────────────────┘
```

---

## 2. CLAUDE.md 发现算法（7 层优先级）

`src/utils/claudemd.ts` 实现了 CLAUDE.md 的发现逻辑，**优先级从低到高**：

```
优先级  来源                              路径示例
──────  ────────────────────────────────  ─────────────────────────────────
  1     Managed（企业管理）               /etc/claude-code/CLAUDE.md
  2     User（用户全局）                  ~/.claude/CLAUDE.md
  3     Project（项目根）                 /repo/CLAUDE.md
                                          /repo/.claude/CLAUDE.md
                                          /repo/.claude/rules/*.md
  4     Local（当前目录向上遍历）          ./CLAUDE.md（CWD）
                                          ../CLAUDE.md（父目录）
                                          ../../CLAUDE.md（祖父目录）
                                          ... 直到 git 根或文件系统根
  5     Additional Dirs（--add-dir 指定）  /extra/path/CLAUDE.md
  6     AutoMem（自动内存）               ~/.claude/projects/<hash>/memory/MEMORY.md
  7     TeamMem（团队内存）               ~/.claude/projects/<hash>/memory/team/MEMORY.md
```

**关键规则：**
- **Local 层从 CWD 向上遍历**，CWD 的 CLAUDE.md 优先级最高
- **Git Worktree 去重**：避免从 main repo 和 worktree 加载同一个文件两次
- **高优先级覆盖低优先级**（同 key 时）

### 2.1 查找路径

```typescript
// 每个目录下查找以下三种位置
const CLAUDE_MD_LOCATIONS = [
  'CLAUDE.md',              // 目录根
  '.claude/CLAUDE.md',      // .claude 子目录
  '.claude/rules/*.md',     // rules 目录下所有 .md 文件（glob）
]
```

---

## 3. CLAUDE.md 内容处理

### 3.1 大小限制

```typescript
const MAX_ENTRYPOINT_LINES = 200       // 单个文件最多 200 行
const MAX_ENTRYPOINT_BYTES = 25_000    // 单个文件最多 25KB

// 截断顺序：先按行截断，再按字节截断（在换行符处）
// 截断后追加提示："[CLAUDE.md truncated: exceeds size limit]"
```

### 3.2 @import 语法（跨文件引用）

CLAUDE.md 支持用 `@` 语法引用其他文件：

```markdown
<!-- CLAUDE.md -->
# 项目规范

@./docs/coding-style.md
@~/global-rules.md
@/absolute/path/to/rules.md
@path\ with\ spaces.md    <!-- 空格需要转义 -->
```

**处理规则：**
```
扫描规则：
  ✓ 在普通文本节点中查找 @path
  ✗ 不扫描代码块（```...```）
  ✗ 不扫描内联代码（`...`）
  → 防止代码示例中的 @reference 被误触发

安全限制：
  - MAX_INCLUDE_DEPTH = 5（最深 5 层嵌套）
  - processedPaths 集合防止循环引用
  - 不存在的文件静默忽略（不报错）
  - 需要用户批准外部 @include（安全策略）
```

### 3.3 注入到 System Prompt 的格式

```
系统提示词中的 Memory 部分：

<user-context>
  <claude-md source="user">
  [~/.claude/CLAUDE.md 的内容]
  </claude-md>

  <claude-md source="project" path="/repo/CLAUDE.md">
  [项目 CLAUDE.md 的内容]
  </claude-md>

  <claude-md source="local" path="/repo/src/CLAUDE.md">
  [局部 CLAUDE.md 的内容]
  </claude-md>
</user-context>
```

---

## 4. memdir：模型自动写入的记忆

### 4.1 四类记忆类型

```typescript
type MemoryType = 'user' | 'feedback' | 'project' | 'reference'

// user: 用户角色、偏好、技术背景（始终私密）
// feedback: 工作指导，"做什么/不做什么"（默认私密，可设为团队共享）
// project: 项目信息、截止日期、决策记录
// reference: 指向外部系统的指针（Linear、Jira、Slack 链接等）
```

### 4.2 存储路径结构

```
~/.claude/projects/<project-hash>/
├── memory/
│   ├── MEMORY.md                    # 主索引文件（模型生成）
│   ├── user_preferences.md          # 类型_名称.md 格式
│   ├── feedback_coding_style.md
│   ├── project_architecture.md
│   ├── reference_external_links.md
│   ├── team/
│   │   └── MEMORY.md                # 团队共享记忆
│   ├── logs/
│   │   └── 2025/01/15.md           # KAIROS 日志（按日期）
│   └── agent-memory/                # Agent 专属记忆
└── session-memory/
    └── memory.md                    # Session Memory 摘要
```

`<project-hash>` = SHA256(规范化的项目路径)

---

## 5. Session Memory：超长对话的自动摘要

### 5.1 触发阈值

```typescript
const DEFAULT_SESSION_MEMORY_CONFIG = {
  minimumMessageTokensToInit: 10_000,    // 首次生成：消息超过 1 万 token
  minimumTokensBetweenUpdate: 5_000,     // 增量更新：累积增加 5000 token
  toolCallsBetweenUpdates: 3,            // 或：新增了 3 次工具调用
}

// 触发条件（满足其一）：
// (token ≥ minimumTokensBetweenUpdate AND 工具调用 ≥ toolCallsBetweenUpdates)
// OR
// (token ≥ minimumTokensBetweenUpdate AND 没有尾部工具调用)
// 注意：token 阈值始终是必要条件
```

### 5.2 与 Autocompact 的配合

```
正常对话 ──→ Session Memory 后台更新（维护一份持续的摘要）
                    │
                    ▼
          Autocompact 触发时，优先尝试
          Session Memory Compaction：
          用已有的 session memory 文件替换对话历史
          （比重新请求 LLM 生成摘要更快、更便宜）
```

### 5.3 文件安全

```typescript
// session-memory/memory.md 的文件权限
fs.writeFileSync(memoryPath, content, { mode: 0o600 })
// 仅所有者可读写，其他用户无权访问
```

---

## 6. Team Memory Sync

团队成员可以共享项目级记忆，通过 `services/teamMemorySync/` 同步：

```
同步机制：
  - 使用 Git 仓库作为同步后端（`.claude-team/` 子目录）
  - Pull：会话开始时从团队仓库拉取最新 team/MEMORY.md
  - Push：模型写入团队记忆后，提交并推送到团队仓库
  - 冲突处理：Three-way merge，本地变更优先

适用场景：
  - 团队共享的编码规范（避免每个人重复告知）
  - 项目架构决策文档
  - 常用的外部链接和工具配置
```

---

## 7. Context 注入时序

```typescript
// src/context.ts
export async function getUserContext(
  toolUseContext: ToolUseContext,
): Promise<{ [key: string]: string }> {

  // 并行加载（不阻塞彼此）
  const [memoryFiles, gitStatus] = await Promise.all([
    getMemoryFiles(toolUseContext),          // 加载所有 CLAUDE.md
    getGitStatus(getCwd()),                  // git status（注入 context）
  ])

  return {
    workingDirectory: getCwd(),
    gitStatus: gitStatus ?? '',
    memoryContext: formatMemoryFiles(memoryFiles),
    // ...其他上下文
  }
}
```

`getUserContext()` 结果会被 `prependUserContext()` 注入到 API 请求的消息列表开头，作为第一条用户消息。

---

## 8. 缓存策略

```typescript
// CLAUDE.md 加载有会话级缓存
// 避免每次 API 请求都重新扫描文件系统

// 清除时机：
// - /compact 执行后（runPostCompactCleanup）
// - settings 变更后（clearCaches）
// - 用户显式 /reload（如果有）

getUserContext.cache.clear?.()  // 手动清除
```

---

## 关键文件速查

| 文件 | 功能 |
|------|------|
| `src/utils/claudemd.ts` | CLAUDE.md 发现算法（1000+ 行） |
| `src/memdir/memdir.ts` | MEMORY.md 读写逻辑 |
| `src/memdir/paths.ts` | 路径解析、安全校验 |
| `src/context.ts` | getUserContext，注入 git status + memory |
| `src/context/` | 通知系统、附件管理 |
| `src/services/SessionMemory/sessionMemory.ts` | Session Memory 提取与更新 |
| `src/services/teamMemorySync/` | 团队记忆同步 |
