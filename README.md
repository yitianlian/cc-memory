# Claude Code Memory 系统设计深度解析

> **Disclaimer / 免责声明**
>
> 本文基于 [instructkr/claude-code](https://github.com/instructkr/claude-code) 仓库的公开源码进行分析。该仓库为社区公开的 Claude Code 源码版本，本文仅用于技术学习与研究目的。文中所有分析均基于该仓库截至撰文时的代码状态，不代表 Anthropic 官方的最新实现。如有侵权请联系删除。

---

## 1. 背景

Claude Code 是 Anthropic 官方推出的 CLI 编程助手。与大多数 AI 编程工具不同，Claude Code 在其系统内部实现了一套完整的**持久化记忆系统**，使得 AI 助手能够跨会话地记住用户是谁、偏好什么工作方式、项目当前处于什么状态。

这套记忆系统的设计思路值得深入研究。它不是一个学术原型，而是一个已经在生产中运行的、经过 eval 验证的系统。它的核心代码集中在 `src/memdir/` 目录下，约 2000 行 TypeScript，涵盖存储、召回、时效性管理、安全防护和多人协作等完整生命周期。

---

## 2. 整体架构概览

Claude Code 的记忆系统是一个**纯文件系统的持久化方案**。没有数据库，没有向量检索，没有图结构——每条记忆就是一个 Markdown 文件，附带 YAML frontmatter 元数据。

整个系统由三层构成：

```
┌─────────────────────────────────────────────┐
│           System Prompt 注入层               │
│   loadMemoryPrompt() → 行为指令 + MEMORY.md  │
├─────────────────────────────────────────────┤
│           Sonnet 召回选择层                   │
│   findRelevantMemories() → 选 ≤5 条相关记忆   │
├─────────────────────────────────────────────┤
│           文件存储层                          │
│   ~/.claude/projects/<project>/memory/*.md   │
│   MEMORY.md (索引) + topic files (记忆正文)   │
└─────────────────────────────────────────────┘
```

### 2.1 MEMORY.md：索引，不是记忆

这是整个系统最关键的设计决策之一：**MEMORY.md 是一个索引文件，不存储记忆内容本身**。

```markdown
- [User Role](user_role.md) — data scientist focused on observability
- [Testing Policy](feedback_testing.md) — integration tests must hit real DB
- [Merge Freeze](project_freeze.md) — freeze begins 2026-03-05 for mobile cut
```

每条索引一行，不超过 150 字符。MEMORY.md 在每次会话启动时自动加载进上下文，模型扫一眼就知道有哪些记忆可用。

**截断机制**（`memdir.ts:57-103`）：

- 行数上限：200 行
- 字节上限：25,000 字节（~25KB）
- 先按行截断（自然断行），再按字节截断（在最近的换行符处切断）
- 触发截断时追加警告：`WARNING: MEMORY.md is 247 lines (limit: 200). Only part of it was loaded.`

这意味着整个记忆索引被控制在一个非常小的 token 预算内。真正的记忆正文存储在各个 topic 文件里，只在需要时按需加载。

### 2.2 单条记忆的文件格式

每条记忆是一个独立的 `.md` 文件，使用 YAML frontmatter：

```markdown
---
name: Testing Policy
description: integration tests must hit a real database — project-wide convention
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Last quarter mocked tests passed but the prod migration failed — mock/prod divergence masked a broken migration.

**How to apply:** When writing or reviewing test code that touches the database layer, always use a real test database. Do not introduce mocking for database operations.
```

frontmatter 三个字段各有明确职责：

| 字段 | 职责 |
|------|------|
| `name` | 记忆标识，用于索引显示 |
| `description` | 一行描述，**用于召回选择时的相关性判断** |
| `type` | 四种类型之一，决定存储范围和行为指导 |

---

## 3. 四类记忆类型

Claude Code 将记忆严格约束为四种类型。这不是一个随意的分类，而是经过 eval 验证的分类体系（源码注释中多次提到 eval case 编号和通过率）。

### 3.1 User Memory（用户画像）

**存什么：** 用户的角色、技能栈、工作偏好、知识背景。

**为什么需要：** 同样是"解释这段代码"，面对十年 Go 经验的后端工程师和第一次写代码的学生，回答方式应该完全不同。

**示例：**

```
user: I've been writing Go for ten years but this is my first time touching the React side of this repo
→ [saves user memory: deep Go expertise, new to React — frame frontend explanations in terms of backend analogues]
```

**关键约束：** 不记录可能被视为负面评价的内容，也不记录与工作无关的信息。

### 3.2 Feedback Memory（行为反馈）

**存什么：** 用户对 AI 工作方式的纠正和确认。

**这是最重要的记忆类型。** 源码注释明确说：

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.

也就是说，如果只记录"不要这样做"，模型会变得越来越保守。必须同时记录"这样做是对的"。

**纠正示例：**
```
user: stop summarizing what you just did at the end of every response, I can read the diff
→ [saves feedback memory: this user wants terse responses with no trailing summaries]
```

**确认示例：**
```
user: yeah the single bundled PR was the right call here, splitting this one would've just been churn
→ [saves feedback memory: for refactors in this area, user prefers one bundled PR over many small ones — a validated judgment call]
```

**结构化格式：** Feedback memory 要求按 `规则 → Why → How to apply` 的三段式记录，这样在边界情况下模型可以根据原因做判断，而不是盲目遵守规则。

### 3.3 Project Memory（项目上下文）

**存什么：** 不能从代码或 git 历史中推导出的项目状态——谁在做什么、为什么、截止日期、事故、决策背景。

**关键处理：** 要求将相对日期转换为绝对日期。

```
user: we're freezing all non-critical merges after Thursday
→ [saves project memory: merge freeze begins 2026-03-05]  // 不是 "Thursday"
```

这个要求很细致但非常实际——记忆跨会话使用，"Thursday" 在下周一看就毫无意义了。

### 3.4 Reference Memory（外部指针）

**存什么：** 外部系统中信息的位置——Linear 项目、Slack 频道、Grafana 看板。

```
user: the Grafana board at grafana.internal/d/api-latency is what oncall watches
→ [saves reference memory: grafana.internal/d/api-latency is the oncall latency dashboard — check it when editing request-path code]
```

Reference memory 的价值在于它不存储信息本身，只存储"去哪里找信息"。这避免了信息过期的问题——指针过期了最多找不到，但不会给出过期的错误信息。

---

## 4. 存储设计：目录结构与路径解析

### 4.1 默认路径

```
~/.claude/projects/<sanitized-git-root>/memory/
    ├── MEMORY.md          # 索引
    ├── user_role.md        # topic file
    ├── feedback_testing.md # topic file
    └── team/              # Team Memory（如启用）
        ├── MEMORY.md
        └── ...
```

几个关键设计点：

**Git root 共享：** 同一个 Git 仓库的所有 worktree 共享同一个记忆目录。通过 `findCanonicalGitRoot()` 找到规范的 Git root，而不是用当前工作目录。这意味着在不同 worktree 间切换不会丢失记忆。

**路径 sanitization：** Git root 路径会经过 `sanitizePath()` 处理后作为目录名，防止特殊字符导致问题。

### 4.2 路径解析链

路径解析有明确的优先级链（`paths.ts:223-235`）：

```
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 环境变量  → 完整路径覆盖（Cowork SDK 用）
2. autoMemoryDirectory in settings.json          → 用户显式配置（支持 ~/ 展开）
3. ~/.claude/projects/<sanitized-git-root>/memory/ → 默认
```

**安全约束：** `settings.json` 中的 `projectSettings`（项目仓库内的 `.claude/settings.json`）**被刻意排除在外**。源码注释解释了原因：

> a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories

这是一个很好的安全意识——恶意仓库不应该能通过配置文件重定向记忆写入位置到敏感目录。

### 4.3 路径校验

`validateMemoryPath()` 函数（`paths.ts:109-150`）会拒绝：

- 相对路径（`../foo`）
- 根路径或接近根的路径（长度 < 3）
- Windows 驱动器根（`C:\`）
- UNC 路径（`\\server\share`）
- 包含 null byte 的路径（可在系统调用中被截断）
- `~/` 展开后结果为 `$HOME` 或其父目录的路径

---

## 5. 写入流程

### 5.1 主动写入（两步法）

记忆写入是一个两步操作：

**Step 1：** 写 topic 文件。用 frontmatter 格式创建 `feedback_testing.md`。

**Step 2：** 更新 MEMORY.md 索引。添加一行指向该文件的条目。

这个两步设计的好处是清晰：索引和内容分离，索引始终在上下文中，内容按需加载。

### 5.2 "什么不该存"的约束

系统 prompt 中有一个明确的 `What NOT to save` 段落（`memoryTypes.ts:183-195`）：

- 代码模式、架构、文件路径、项目结构——**可以通过读当前代码推导**
- Git 历史、变更记录——**git log / git blame 是权威来源**
- 调试方案或修复配方——**fix 在代码里，context 在 commit message 里**
- CLAUDE.md 中已有的内容——**不重复**
- 临时任务状态——**用 Task 系统而非 Memory**

这些排除规则的核心逻辑是：**只存储不能从当前项目状态推导出来的信息**。代码会变，记忆里记下的代码结构会过期，但用户偏好、项目决策背景、外部系统指针不会因为代码变更而失效。

最有意思的一条：

> These exclusions apply even when the user explicitly asks you to save. If they ask you to save a PR list or activity summary, ask what was *surprising* or *non-obvious* about it — that is the part worth keeping.

即使用户说"帮我记住这个 PR 列表"，系统也会引导用户提炼出值得记录的非显而易见的信息，而不是机械地存储整个列表。

### 5.3 后台记忆提取（Background Extract）

除了主动写入，Claude Code 还有一个后台提取机制（feature-gated：`EXTRACT_MEMORIES` + `tengu_passport_quail`）：

- 在每轮对话结束后，fork 出一个后台 agent
- 该 agent 继承完整对话上下文（共享 prompt cache）
- 扫描对话内容，提取可能遗漏的记忆
- 如果主 agent 已经在某个区间内写了记忆，后台 agent 跳过该区间

这是一个很精巧的补充机制：主 agent 专注于回答问题和写代码，可能遗漏值得记录的信息。后台 agent 扫描整个对话，捕捉遗漏的记忆点。

---

## 6. 召回机制：Sonnet 驱动的相关性选择

这是整个系统中最有设计感的部分。

### 6.1 工作流程

```
用户发送 query
      ↓
scanMemoryFiles()
  - 递归读取 memory/ 目录下所有 .md 文件
  - 解析每个文件前 30 行的 frontmatter
  - 提取 filename, description, type, mtime
  - 按 mtime 降序排列，取前 200 个
      ↓
formatMemoryManifest()
  - 格式化为: "[type] filename (ISO timestamp): description"
      ↓
selectRelevantMemories()
  - 调用 Sonnet（不是 Opus）做 side query
  - System prompt: "Select up to 5 memories most useful for this query"
  - Input: query + manifest + recently-used tools
  - Output: JSON schema { selected_memories: string[] }
      ↓
返回 ≤5 个记忆的完整路径 + mtime
```

### 6.2 关键设计细节

**为什么用 Sonnet 而不是 embedding：** 这里没有用向量相似度检索，而是让一个轻量语言模型（Sonnet）根据 query 和记忆描述做判断。好处是能理解语义关系，而不仅仅是词汇重叠。坏处是多了一次 API 调用。

**recently-used tools 过滤（`findRelevantMemories.ts:88-95`）：**

```typescript
// When Claude Code is actively using a tool (e.g. mcp__X__spawn),
// surfacing that tool's reference docs is noise — the conversation
// already contains working usage.
```

如果模型正在使用某个工具，就不要再推送该工具的使用文档类记忆——这是噪音。但如果记忆包含该工具的 **warnings、gotchas、known issues**，仍然推送——正在使用时恰恰是这些警告最有价值的时候。

**alreadySurfaced 去重：** 已经在之前 turn 中展示过的记忆不会再次被选中，避免 5 个 slot 被重复消耗。

**选择器 prompt 的设计（`findRelevantMemories.ts:18-24`）：**

```
Return a list of filenames for the memories that will clearly be useful...
- If you are unsure if a memory will be useful, do not include it
- If there are no memories that would clearly be useful, return an empty list
```

注意措辞是 "clearly be useful" 和 "certain will be helpful"——宁可少召回，也不要召回不相关的。这是一个高精度、低召回率的设计取向。

### 6.3 为什么不用 embedding

值得单独讨论这个设计选择。传统的 RAG 系统几乎都用 embedding + 向量数据库做检索。Claude Code 没有这样做，而是：

1. **只读 frontmatter 的 description 字段**（不读全文）
2. **用 Sonnet 做语义选择**（不做向量相似度）

这个方案的优势：

- **零基础设施依赖**：不需要向量数据库、不需要 embedding 模型、不需要索引更新
- **语义理解更强**：Sonnet 可以理解 "fixing auth bug" 和 "session token storage policy" 的关联，embedding 可能做不到
- **上限很低的文件数**：最多 200 个 `.md` 文件的 description，拼成 manifest 也不过几 KB——完全在 Sonnet 的处理能力内

劣势是每次 query 多了一个 API 调用，但考虑到记忆召回的频率和 Sonnet 的速度，这个代价是可接受的。

---

## 7. 时效性追踪与过期防护

这是记忆系统容易被忽视但至关重要的一环。Claude Code 在这里做了非常细致的设计。

### 7.1 时间计算

```typescript
// memoryAge.ts
export function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}

export function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

用 floor 而不是 round——如果一条记忆是 23 小时前写的，显示 "today" 而不是 "1 day ago"。

### 7.2 过期警告

记忆超过 1 天就会附加一个 staleness caveat：

```
This memory is 47 days old. Memories are point-in-time observations, not live state —
claims about code behavior or file:line citations may be outdated.
Verify against current code before asserting as fact.
```

这个警告直接用 `<system-reminder>` 标签包裹注入。源码注释解释了动机：

> Motivated by user reports of stale code-state memories (file:line citations to code that has since changed) being asserted as fact — the citation makes the stale claim sound more authoritative, not less.

也就是说，**带行号引用的过期记忆比没有引用的更危险**——用户看到行号会以为是准确的，但代码可能已经变了。

### 7.3 Prompt 层面的防护

除了运行时警告，system prompt 里还有一个独立的 section "Before recommending from memory"（`memoryTypes.ts:240-256`）：

```
A memory that names a specific function, file, or flag is a claim
that it existed *when the memory was written*. It may have been
renamed, removed, or never merged. Before recommending it:

- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation: verify first.

"The memory says X exists" is not the same as "X exists now."
```

这个 section 的标题本身就经过了 eval 验证——源码注释记录：

> Header wording matters: "Before recommending" (action cue at the decision point) tested better than "Trusting what you recall" (abstract). The appendSystemPrompt variant with this header went 3/3; the abstract header went 0/3 in-place.

同样的内容，标题从抽象的 "Trusting what you recall" 改为具体的 "Before recommending from memory"，eval 通过率从 0/3 变成 3/3。这说明 prompt engineering 中标题的具体性和行动导向有多重要。

### 7.4 Memory Drift Caveat

整个 system prompt 中还嵌入了一段通用的记忆漂移警告：

```
Memory records can become stale over time. Use memory as context for
what was true at a given point in time. Before answering the user or
building assumptions based solely on information in memory records,
verify that the memory is still correct and up-to-date by reading the
current state of the files or resources. If a recalled memory conflicts
with current information, trust what you observe now — and update or
remove the stale memory rather than acting on it.
```

核心原则：**当记忆和当前事实冲突时，信任当前事实，更新或删除过期记忆。**

---

## 8. Team Memory：多人协作扩展

Claude Code 在个人记忆之上还设计了一层 Team Memory（feature-gated：`TEAMMEM` + `tengu_herring_clock`）。

### 8.1 目录结构

```
~/.claude/projects/<project>/memory/
    ├── MEMORY.md           # 个人索引
    ├── user_role.md         # 个人记忆
    └── team/
        ├── MEMORY.md       # 团队索引
        └── testing_policy.md # 团队记忆
```

Team memory 是 auto memory 的子目录。这意味着创建 team 目录时会自动创建父级 auto 目录（`mkdir -p` 效果）。

### 8.2 Scope 决策

启用 Team Memory 后，每种记忆类型会附带 `<scope>` 标签指导存储位置：

| 类型 | Scope |
|------|-------|
| user | **always private** — 用户画像永远是私有的 |
| feedback | **default private** — 个人风格偏好是私有的；项目级约定（如测试策略）存 team |
| project | **bias toward team** — 项目上下文对所有人有用 |
| reference | **usually team** — 外部系统指针对所有人有用 |

### 8.3 安全防护

Team Memory 引入了额外的安全挑战——多人可写意味着路径遍历攻击面更大。`teamMemPaths.ts` 实现了严格的校验：

- **Null byte 检测**：可在系统调用中截断路径
- **URL 编码遍历**：`%2e%2e%2f` = `../`
- **Unicode 规范化攻击**：全角 `.` 和 `/`（fullwidth characters）
- **反斜杠注入**：Windows 路径分隔符
- **Symlink 逃逸检测**：对最深存在的祖先目录做 `realpath()` 解析，确保 symlink 不会指向 team 目录之外
- **前缀攻击防护**：确保 `/foo/team-evil` 不能匹配 `/foo/team`

---

## 9. KAIROS 模式：长时会话的记忆策略

对于长时运行的 assistant 会话（KAIROS 模式，feature-gated），记忆策略有所不同：

```
~/.claude/memory/logs/YYYY/MM/YYYY-MM-DD.md   # 每日追加日志
~/.claude/memory/MEMORY.md                     # 蒸馏后的索引（只读）
```

- 不再维护 MEMORY.md 作为实时索引
- 改为 append-only 的每日日志文件
- 每晚由 `/dream` skill 将日志蒸馏为 topic 文件 + MEMORY.md
- MEMORY.md 仍然加载进上下文，但标记为只读

这个设计适应了 assistant 模式的特点：会话可能持续一整天甚至更长，频繁更新 MEMORY.md 索引不现实，追加日志更自然。

---

## 10. 与其他 Memory 系统的对比

### 10.1 vs A-Mem（Agentic Memory）

A-Mem 把每条记忆表示为 `MemoryNote`，强调记忆之间的主动组织关系——新记忆进来时会自动找相近旧记忆，决定是否演化（strengthen / update neighbor）。

**对比：**

| 维度 | Claude Code Memory | A-Mem |
|------|-------------------|-------|
| 存储 | 纯文件系统 (.md) | 内存字典 + embedding 索引 |
| 检索 | Sonnet 语义选择 | embedding cosine similarity |
| 记忆间关系 | 无（平铺的 topic 文件） | links 构成轻量图 |
| 记忆演化 | 手动更新 + 后台提取 | LLM 驱动的自动演化 |
| 定位 | 生产系统 | 研究原型 |

Claude Code 的方案更简洁、更实用——200 个 `.md` 文件 + 1 个 Sonnet 调用就够了。A-Mem 的方案更学术、更有野心——试图让记忆自组织，但对 LLM 结构化输出稳定性要求很高。

### 10.2 vs 传统 RAG

传统 RAG 用 embedding 做检索，Claude Code 用 Sonnet 做选择。

关键差异：
- RAG 需要 embedding 模型 + 向量数据库，Claude Code 零基础设施
- RAG 按向量相似度排序，Claude Code 按语义相关性判断（更准但更慢）
- RAG 通常检索 chunk（文档片段），Claude Code 检索完整的 topic 文件
- Claude Code 的上限是 200 个文件——如果记忆量增长到数千条，这个方案会需要改进

### 10.3 vs Profile-based Memory

像 ChatGPT 的 Memory 功能，本质上是一个扁平的 key-value 画像。Claude Code 的四类记忆类型比 profile 更结构化：

- Profile 不区分"用户偏好"和"项目策略"
- Profile 没有时效性追踪
- Profile 没有团队共享维度
- Profile 不关心"什么不该存"

---

## 11. 设计哲学总结

从 Claude Code Memory 的实现中，可以提炼出几个核心设计哲学：

### 11.1 只存不能推导的

> Code patterns, conventions, architecture, file paths, or project structure — these can be derived by reading the current project state.

这是最核心的原则。代码是活的，随时在变。把代码结构存进记忆，等于给自己埋了一颗定时炸弹。只存那些**不在代码里的东西**：用户偏好、决策背景、外部指针。

### 11.2 高精度优于高召回率

Sonnet 选择器的 prompt 反复强调 "clearly useful"、"certain will be helpful"——宁可少推一条有用的记忆，也不推一条无关的。这和传统 IR（信息检索）系统的 "recall-oriented" 思路完全相反。

原因很直觉：在 LLM 的 context window 里，每一条无关信息都是噪音，可能干扰模型的注意力。空间是有限的，精度比召回率更重要。

### 11.3 时效性是一等公民

几乎每一层都有时效性处理：
- frontmatter 里没有 timestamp 字段，直接用文件 mtime
- 召回结果按 mtime 排序
- 超过 1 天的记忆自动附加 staleness warning
- System prompt 里有独立的 section 教模型验证记忆是否过期
- 明确规定"当记忆和现实冲突时，信任现实"

### 11.4 Prompt engineering 是记忆系统的一部分

Claude Code Memory 系统有一半的代码量在构建 system prompt。哪些行为指令放在哪个 section、用什么标题、措辞是抽象的还是具体的——这些都经过了 eval 验证。

源码注释中多次出现这样的记录：

```
// Eval-validated (memory-prompt-iteration case 3, 0/2 → 3/3)
// Header wording matters: "Before recommending" went 3/3;
// the abstract header went 0/3 in-place
```

这意味着记忆系统的"提示词工程"不是凭感觉写的，而是用评测数据驱动的。

### 11.5 安全约束不打折

- `projectSettings` 不能覆盖记忆路径（防止恶意仓库重定向写入 `~/.ssh`）
- 路径遍历防护覆盖了 null byte、URL 编码、Unicode 规范化、symlink 逃逸
- User memory 永远 private，不会泄露到 team scope
- 即使用户明确要求存不该存的内容，系统也会引导用户提炼

---

## 12. 一句话总结

Claude Code 的记忆系统不追求复杂的图结构或向量检索，而是用**纯文件系统 + 四类记忆分类 + Sonnet 语义选择 + 严格的时效性追踪**，实现了一套实用、安全、可维护的跨会话记忆方案。

它最值得学习的不是技术方案本身，而是它背后的克制：**只存不能推导的，宁可少召回也不推噪音，时效性是一等公民，安全约束不打折。**
