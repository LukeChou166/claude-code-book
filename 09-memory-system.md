# 第9章 记忆系统：让 AI 跨会话学习

> **前置知识**：第2章启动流程、第3章 QueryEngine  
> **关键文件**：`src/memdir/memdir.ts`、`src/memdir/memoryTypes.ts`、`src/memdir/findRelevantMemories.ts`、`src/services/SessionMemory/sessionMemory.ts`

---

## 9.1 解决什么问题

大语言模型本身是无状态的——每次对话开始，模型对你一无所知：

- **"用 bun 不用 npm"**：你已经说过 100 次了，AI 还是会建议你 `npm install`
- **"不要在回复末尾总结"**：你已经纠正过它，下次会话全忘了
- **"auth 模块是遗留代码，别动"**：项目背景知识每次都要重新解释
- **"这个 Bug 和上周那个 API 超时问题有关"**：对话结束就丢失了这个关联

记忆系统解决的核心问题是：**如何让 AI 在跨会话的时间维度上积累和使用知识**，而不是每次会话都从零开始。

---

## 9.2 业界的做法

| 方案 | 代表 | 优点 | 缺点 |
|------|------|------|------|
| 无记忆 | 大多数早期助手 | 实现简单 | 每次重复解释 |
| 固定 System Prompt | 简单自定义助手 | 可控 | 内容静态，不随使用增长 |
| 对话摘要 | LangChain ConversationSummaryMemory | 自动化 | 摘要失真，无类型区分 |
| 向量数据库（RAG） | 许多 LLM 应用 | 容量无限 | 编程精确性差，召回不稳定 |
| 服务端托管记忆 | ChatGPT 记忆功能 | 自动同步 | 隐私问题，黑盒 |

Claude Code 选择了**本地文件系统 + AI 主动写入**的方案：记忆存储在用户机器的 Markdown 文件中，AI 自己决定写什么、读什么。

---

## 9.3 记忆的四种类型

`src/memdir/memoryTypes.ts` 定义了严格的四类记忆分类法，每类捕获不同维度的知识：

```typescript
// src/memdir/memoryTypes.ts 第 14-19 行
export const MEMORY_TYPES = [
  'user',       // 用户画像：技能、角色、偏好
  'feedback',   // 行为反馈：做什么/不做什么
  'project',    // 项目背景：决策、时间节点、进行中的工作
  'reference',  // 外部资源：链接、系统入口
] as const
```

**`user` 类型**：关于这个用户是谁的知识
```
"用户是数据科学家，正在研究项目的日志体系"
"深度 Go 经验，第一次接触这个项目的 React 前端 → 前端解释用后端类比"
```

**`feedback` 类型**：用户纠正过或确认过的行为准则（这是最重要的一类）
```
"不要在单元测试中 mock 数据库，上季度因此踩坑"
"回复末尾不要总结，用户能自己看 diff"
"对这个模块的重构，用户倾向于一个大 PR 而不是多个小 PR（已确认）"
```

**`project` 类型**：代码库里看不出来的项目背景
```
"合并冻结从 2026-03-05 开始，移动端在切发布分支"
"auth 中间件重写是因为法务合规要求，不是技术债"
```

**`reference` 类型**：外部系统的指针
```
"pipeline bug 在 Linear 项目 INGEST 中追踪"
"grafana.internal/d/api-latency 是 oncall 监控面板"
```

### 什么不应该存为记忆

```
// src/memdir/memoryTypes.ts 第 183-195 行
export const WHAT_NOT_TO_SAVE_SECTION = [
  '- Code patterns, conventions, architecture, file paths — derivable by reading the code',
  '- Git history, recent changes — git log/blame are authoritative',
  '- Debugging solutions or fix recipes — the fix is in the code; commit message has context',
  '- Anything already documented in CLAUDE.md files',
  '- Ephemeral task details: in-progress work, temporary state',
]
```

这个"什么不存"的列表同样重要：已经在代码里、git 里、CLAUDE.md 里的信息不应该冗余存为记忆，否则记忆会过时，和真实状态产生冲突。

---

## 9.4 存储结构：MEMORY.md 索引 + 独立文件

记忆存储在一个专用目录下：

```
~/.claude/projects/<项目路径哈希>/memory/
├── MEMORY.md                   ← 索引文件（总是被加载）
├── user_role.md                ← 单条记忆文件
├── feedback_no_mocking.md      ← 单条记忆文件
├── project_auth_rewrite.md     ← 单条记忆文件
└── team/                       ← 团队共享记忆（TEAMMEM feature）
    ├── MEMORY.md
    └── ...
```

每个记忆文件用 frontmatter 格式：

```markdown
---
name: 不要在测试中 mock 数据库
description: 集成测试必须连接真实数据库，上季度踩坑
type: feedback
---

不要在单元测试中 mock 数据库连接。

**Why:** 上季度发生了一次生产事故：mock 通过了测试但真实迁移失败，
原因是 mock 没有复现数据库的外键约束行为。

**How to apply:** 任何涉及数据库的测试，使用 testcontainers 启动真实 
PostgreSQL 实例，不使用 jest.mock 或 sinon.stub 替换数据库层。
```

**`MEMORY.md` 只是索引，不存内容**：

```markdown
# MEMORY.md（索引文件示例）
- [不要 mock 数据库](feedback_no_mocking.md) — 上季度踩坑，测试通过但生产失败
- [用户是数据科学家](user_role.md) — 关注日志和可观测性
- [auth 中间件重写原因](project_auth_rewrite.md) — 法务合规，非技术债
- [Linear INGEST 项目](reference_linear.md) — pipeline bug 追踪入口
```

### 为什么索引和内容分离？

`MEMORY.md` 有严格的大小限制：

```typescript
// src/memdir/memdir.ts 第 35-38 行
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000  // 约 25KB
```

**`MEMORY.md` 在每次会话时自动注入 system prompt**，如果它无限增长，会大量消耗 token 预算。通过只在索引里存"一行摘要 + 文件指针"，MEMORY.md 保持精简，总能被完整加载。具体的记忆内容按需检索。

---

## 9.5 两级加载：索引总是加载，详情按需检索

### 第一级：索引自动注入（每次会话）

```typescript
// src/memdir/memdir.ts 第 419-507 行
export async function loadMemoryPrompt(): Promise<string | null> {
  if (!isAutoMemoryEnabled()) return null
  
  const autoDir = getAutoMemPath()
  await ensureMemoryDirExists(autoDir)
  
  // 返回记忆指导 + MEMORY.md 内容
  // 这段文字被注入到 system prompt
  return buildMemoryLines('auto memory', autoDir).join('\n')
}
```

每次启动会话，AI 看到的 system prompt 包含：
1. 记忆系统的操作指南（怎么读、怎么写、什么时候用）
2. `MEMORY.md` 的当前内容（索引摘要）

### 第二级：AI 驱动的语义召回

当 `MEMORY.md` 的索引暗示某条记忆可能相关时，AI 主动用 FileReadTool 去读那个具体的记忆文件。

更高级的做法是 **`findRelevantMemories`**：

```typescript
// src/memdir/findRelevantMemories.ts 第 39-75 行
export async function findRelevantMemories(
  query: string,         // 用户的当前问题
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],   // 最近用过的工具（避免冗余）
  alreadySurfaced: ReadonlySet<string> = new Set(),  // 已展示过的记忆
): Promise<RelevantMemory[]> {
  
  // 1. 扫描所有记忆文件，读取 frontmatter（name + description）
  const memories = await scanMemoryFiles(memoryDir, signal)
  
  // 2. 用 Sonnet 小模型判断哪些记忆和当前查询相关
  const selectedFilenames = await selectRelevantMemories(query, memories, signal)
  
  // 3. 返回相关记忆的路径（最多 5 个）
  return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }))
}
```

**关键设计：用 AI 做语义相关性判断**

```typescript
// src/memdir/findRelevantMemories.ts 第 18-24 行
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code.
Return a list of filenames for memories that will CLEARLY be useful (up to 5).
Only include memories you are CERTAIN will be helpful based on their name and description.

- If a list of recently-used tools is provided, do NOT select reference/API docs 
  for those tools (already in context), but DO select warnings and gotchas about them.`
```

Sonnet（小模型，便宜快）负责从描述中判断哪 5 条记忆最相关——这比向量检索精确（理解语义），又比让 opus 全量读所有记忆文件便宜。

---

## 9.6 记忆的自动提取（Background Extract）

Claude Code 还有一个后台记忆提取机制（`EXTRACT_MEMORIES` feature flag），会话结束后自动运行：

```typescript
// src/services/SessionMemory/sessionMemory.ts 第 1-5 行
/**
 * Session Memory automatically maintains a markdown file with notes 
 * about the current conversation. It runs periodically in the background 
 * using a forked subagent to extract key information without interrupting 
 * the main conversation flow.
 */
```

这个后台 Agent 会：
1. 分析本次会话中发生了什么
2. 识别值得保存的新知识（用户偏好、项目决策、重要背景）
3. 写入对应类型的记忆文件
4. 更新 `MEMORY.md` 索引

```typescript
// src/memdir/paths.ts 第 69-77 行
export function isExtractModeActive(): boolean {
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_passport_quail', false)) {
    return false  // 由 GrowthBook feature flag 控制
  }
  return (
    !getIsNonInteractiveSession() ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_slate_thimble', false)
  )
}
```

### Session Memory 的更新节奏

```typescript
// src/services/SessionMemory/sessionMemoryUtils.ts（简化）
const DEFAULT_SESSION_MEMORY_CONFIG = {
  toolCallsBetweenUpdates: 10,  // 每 10 次工具调用，提取一次记忆
  initializationThreshold: 5,  // 至少 5 次工具调用后才开始
}
```

不是每次工具调用都提取——每 10 次工具调用触发一次后台提取，避免频繁提取消耗大量 token。

---

## 9.7 记忆的验证：防止过时记忆误导

过时的记忆是危险的，比没有记忆更坏：

```typescript
// src/memdir/memoryTypes.ts 第 240-256 行
export const TRUSTING_RECALL_SECTION = [
  '## Before recommending from memory',
  
  'A memory that names a specific function, file, or flag is a claim that it 
   existed WHEN THE MEMORY WAS WRITTEN. It may have been renamed, removed, 
   or never merged. Before recommending it:',
  
  '- If the memory names a file path: check the file exists.',
  '- If the memory names a function or flag: grep for it.',
  '- If the user is about to act on your recommendation, verify first.',
  
  '"The memory says X exists" is not the same as "X exists now."',
]
```

这段文字被注入到 system prompt 里，告诉 AI：**记忆是历史快照，使用前必须验证**。

例如：
- 记忆说"用 `getAuthToken()` 函数" → 先 grep 确认这个函数还存在
- 记忆说"合并冻结从周四开始" → 这是相对时间，先问用户现在是几号

`MEMORY_DRIFT_CAVEAT` 是系统提示中专门用于提醒 AI 处理记忆漂移的段落：

```typescript
export const MEMORY_DRIFT_CAVEAT =
  '- Memory records can become stale over time. If a recalled memory 
    conflicts with current information, trust what you observe now — 
    and update or remove the stale memory rather than acting on it.'
```

---

## 9.8 团队记忆（TEAMMEM）

当多人在同一代码库工作时，某些知识应该共享（测试规范、合并冻结）：

```typescript
// feature('TEAMMEM') 开启时：
// ~/.claude/projects/<hash>/memory/
//   ├── MEMORY.md         ← 个人记忆索引
//   ├── user_role.md      ← 私有记忆（仅自己可见）
//   └── team/
//       ├── MEMORY.md     ← 团队记忆索引（所有人共享）
//       └── feedback_no_mocking.md  ← 团队级记忆
```

记忆的 scope 字段决定存哪里：
- `user` 类型 → 永远私有（个人偏好不应该影响同事）
- `feedback` 类型 → 默认私有，明显是项目约定才存为 team
- `project` 类型 → 强烈偏向 team（项目背景人人共享）
- `reference` 类型 → 通常 team（外部链接对整个团队有价值）

---

## 9.9 助手模式的日志式记忆（KAIROS）

长时间运行的助手模式（`feature('KAIROS')`）有不同的记忆策略：

```typescript
// src/memdir/memdir.ts 第 327-370 行
function buildAssistantDailyLogPrompt(): string {
  // 不维护实时更新的 MEMORY.md 索引
  // 而是按日期追加写日志：
  // memory/logs/2026/05/2026-05-06.md
  
  return `This session is long-lived. As you work, record anything worth 
  remembering by APPENDING to today's daily log file.
  
  Write each entry as a short timestamped bullet.
  Do not rewrite or reorganize the log — it is append-only.
  A separate nightly process distills these logs into MEMORY.md and topic files.`
}
```

日志示例：
```markdown
# 2026-05-06

- 10:23 用户确认了 auth 模块重构方案，目标是移除 JWT 中间件
- 11:45 发现 utils/crypto.ts 有一个历史遗留的 MD5 哈希，用户要求保留（三方依赖）
- 14:30 用户偏好：不用 any 类型，宁可加多余的 as SomeType
```

每晚有一个单独的 `/dream` 任务（`DreamTask`）把日志蒸馏成结构化记忆文件，更新 `MEMORY.md` 索引。

---

## 9.10 记忆系统的完整生命周期

```
第一次会话
    │
    ├── AI 发现"用 npm" 的错误倾向
    ├── 用户纠正："我们用 bun"
    └── AI 写入记忆：feedback_use_bun.md
         └── MEMORY.md 更新索引

第二次会话（不同的一天）
    │
    ├── loadMemoryPrompt() → 读取 MEMORY.md 注入 system prompt
    ├── AI 看到："[使用 bun](feedback_use_bun.md) — 用户项目使用 bun"
    ├── 需要安装包时，AI 主动 FileRead("feedback_use_bun.md")
    └── AI 建议 bun install，而不是 npm install

六个月后，项目切换到 pnpm
    │
    ├── 用户："我们现在用 pnpm 了"
    └── AI 更新 feedback_use_bun.md，将内容改为使用 pnpm
         └── 验证：先检查 package.json 确认确实在用 pnpm
```

---

## 9.11 与其他持久化机制的区别

```typescript
// src/memdir/memdir.ts 第 254-258 行
'## Memory and other forms of persistence',
'- Use a Plan (not memory) for current task implementation approach',
'- Use Tasks (not memory) for in-progress steps and progress tracking',
'- Memory = information useful in FUTURE conversations',
'- Plan/Tasks = information useful in the CURRENT conversation',
```

| 机制 | 存储什么 | 生命周期 | 文件位置 |
|------|----------|----------|----------|
| **记忆（Memory）** | 跨会话有价值的知识 | 永久（显式删除才消失） | `~/.claude/projects/*/memory/` |
| **计划（Plan）** | 当前任务的实施方案 | 当前任务完成后可删除 | 项目目录下的 `PLAN.md` |
| **任务（Tasks）** | 当前进行中的步骤清单 | 会话结束后无意义 | 内存中（不持久化） |
| **会话日志（JSONL）** | 完整的对话记录 | 永久（用于恢复和审计） | `~/.claude/projects/*/sessions/` |

---

## 9.12 优势与不足

### 优势

**四类记忆分类法非常精准**：`user`、`feedback`、`project`、`reference` 四类覆盖了 Agent 需要跨会话保留的核心知识，同时"什么不该存"的规则防止了信息膨胀。

**MEMORY.md 大小限制保护 token 预算**：索引最多 200 行/25KB，始终被完整加载，不会因记忆增长而悄悄消耗大量 context。

**用 AI 做语义相关性判断（而非向量搜索）**：`findRelevantMemories` 用 Sonnet 根据描述和查询判断相关性，对编程领域的精确知识（函数名、配置项）比向量检索更可靠。

**记忆验证机制**：system prompt 中的 `TRUSTING_RECALL_SECTION` 明确要求 AI 在使用记忆前验证其时效性，防止了过时记忆误导 AI。

**团队记忆支持协同**：同一代码库的团队成员可以共享项目级记忆，避免每个人都要向 AI 解释相同的背景知识。

### 不足

**记忆更新延迟**：后台提取是异步的，当前会话刚纠正 AI 的行为，下一条消息里 AI 可能仍然用旧行为——因为提取任务还没完成，新记忆还没写入。

**记忆冲突无自动解决**：如果个人记忆和团队记忆对同一行为有相反的规定（团队说"要写注释"，个人记忆说"这个用户不需要注释"），AI 需要自己判断优先级，没有系统层面的冲突解决规则。

**索引大小上限会被触发**：重度用户的 `MEMORY.md` 可能超过 200 行限制，超出的记忆条目不会被加载，相当于从索引里"消失"了。需要用户手动维护索引的精简性，这对普通用户要求过高。

**`EXTRACT_MEMORIES` 目前是 feature flag 控制的实验功能**：自动后台提取对于"记忆积累的闭环"至关重要，但它是内部实验特性，外部用户只能依赖 AI 主动调用 `/memory` 命令来写记忆，自动化程度不足。
