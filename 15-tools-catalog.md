# 第15章 工具目录详解——TodoWrite、ToolSearch、Skill 与 Task/Team 系列

> **前置知识**：第 5 章（工具系统基础）、第 7 章（任务编排）  
> **本章目标**：逐一剖析 10 个"元级"工具的定位、触发时机、内部实现和设计哲学

---

## 本章导读

在 Claude Code 的工具库中，有一类工具不操作文件、不执行命令，而是管理**任务状态**、**发现工具**、**调用 Skill**、**编排多 Agent 团队**——这类工具我们称之为"元级工具"（meta-tools）。

本章覆盖 10 个：

| 分组 | 工具名 | 一句话定位 |
|------|--------|-----------|
| 任务（V1） | `TodoWriteTool` | 会话内 todo list，读写合一的旧实现 |
| 搜索 | `ToolSearchTool` | 延迟工具的 schema 加载器 |
| Skill | `SkillTool` | 调用 `/slash-command` 技能 |
| 任务（V2） | `TaskCreateTool` | 创建持久化任务 |
| 任务（V2） | `TaskGetTool` | 按 ID 获取任务详情 |
| 任务（V2） | `TaskListTool` | 列出全部任务 |
| 任务（V2） | `TaskUpdateTool` | 更新任务状态/属性 |
| 任务（V2） | `TaskStopTool` | 停止后台运行的任务进程 |
| Team | `TeamCreateTool` | 创建多 Agent 协作团队 |
| Team | `TeamDeleteTool` | 解散团队并清理资源 |

> **V1 vs V2**：`TodoWriteTool` 是旧版实现（`isTodoV2Enabled()=false` 时启用），新版改为 Task 系列（5 个工具）。二者不共存。

---

## 15.1 TodoWriteTool — 会话任务清单（V1）

### 15.1.1 解决什么问题

AI 执行多步骤任务时，**用户看不到当前进度**，AI 自己也容易丢失上下文（遗漏步骤）。`TodoWriteTool` 给会话附加一个"白板"——AI 把计划写上去，用户随时看到进度条。

**C++ 类比**：想象一个 `std::vector<Task>` 对象，存在全局 `AppState` 里。AI 每次调用这个工具，就把整个 vector 全量覆写一次（不是追加，是替换），类似：
```cpp
app_state.todos[session_id] = new_todos;  // 全量替换
```

### 15.1.2 何时触发

提示词（`prompt.ts`）规定了以下场景**必须**使用：

- 任务需要 3 步以上（复杂多步骤任务）
- 用户明确要求 todo list
- 用户提供了多个任务（逗号分隔或编号列表）
- 接到新指令后，立即捕获为 todo
- 开始一个任务前，先标记 `in_progress`
- 完成一个任务后，立即标记 `completed`

**不应使用**：单步骤简单任务、纯会话/信息查询。

### 15.1.3 核心实现

```
src/tools/TodoWriteTool/TodoWriteTool.ts
```

**输入**：
```typescript
{
  todos: Array<{
    id: string,
    content: string,      // 祈使句，如 "Fix authentication bug"
    status: 'pending' | 'in_progress' | 'completed',
    activeForm?: string,  // 进行时，如 "Fixing authentication bug"
  }>
}
```

**输出**：
```typescript
{
  oldTodos: TodoList,  // 更新前的状态（用于 diff）
  newTodos: TodoList,  // 更新后的状态
  verificationNudgeNeeded?: boolean  // 是否需要跑验证 Agent
}
```

**关键逻辑**（`call()` 函数）：

```typescript
// 1. 拿到当前 todos（按 session_id 或 agent_id 隔离）
const todoKey = context.agentId ?? getSessionId()
const oldTodos = appState.todos[todoKey] ?? []

// 2. 如果全部完成，自动清空列表
const allDone = todos.every(_ => _.status === 'completed')
const newTodos = allDone ? [] : todos  // 全完成时变空数组

// 3. 写回 AppState（会触发 UI 重新渲染）
context.setAppState(prev => ({
  ...prev,
  todos: { ...prev.todos, [todoKey]: newTodos },
}))
```

**验证 Nudge**（彩蛋功能）：当 `VERIFICATION_AGENT` feature flag 开启、且 AI 关闭了 3+ 个任务但没有任何验证步骤时，工具返回消息末尾会追加：
```
NOTE: You just closed out 3+ tasks and none of them was a verification step. 
Before writing your final summary, spawn the verification agent...
```

这是一个**提示注入机制**——通过工具返回结果内嵌指令，引导 AI 补做验证，而不是依赖系统 prompt。

### 15.1.4 启用条件

```typescript
isEnabled() {
  return !isTodoV2Enabled()  // 只在 V2 关闭时才启用
}
```

### 15.1.5 C++ 语法对照

| TypeScript 写法 | C++ 等价 |
|----------------|---------|
| `todos.every(...)` | `std::all_of(todos.begin(), todos.end(), ...)` |
| `AppState.todos[todoKey] ?? []` | `map.count(key) ? map[key] : std::vector<>{}` |
| `feature('VERIFICATION_AGENT')` | `#ifdef VERIFICATION_AGENT` 条件编译 |

---

## 15.2 ToolSearchTool — 延迟工具的 Schema 加载器

### 15.2.1 解决什么问题

Claude Code 有几十个工具，如果一次性全部注入 System Prompt，会消耗大量 token（每个工具的参数 schema 少则几百字，多则几千字）。

解决方案：**延迟加载（Deferred Tools）**。大部分工具仅在 System Prompt 中以名字列出，完整 schema 在需要时才加载。`ToolSearchTool` 就是这个"按需加载"的机制核心。

**C++ 类比**：就像动态链接库（.so/.dll）的延迟绑定。函数地址先登记一个占位符，只有第一次被调用时才真正链接。

```
System Prompt 第一页（轻量）：
  [可用工具名列表]  ← 只有名字，约等于 extern 声明

ToolSearchTool 返回：
  [完整 JSONSchema]  ← 等于把 .so 中的符号表加载进来
```

### 15.2.2 何时触发

AI 想调用某个工具，但只知道它的名字（不知道参数格式）时，必须先调用 `ToolSearchTool` 获取 schema，然后才能构造调用。

两种查询模式：
1. **精确选择**：`select:Read,Edit,Grep`——一次加载多个指定工具
2. **关键词搜索**：`notebook jupyter`——按功能描述模糊匹配

### 15.2.3 核心实现

```
src/tools/ToolSearchTool/ToolSearchTool.ts
```

**输入**：
```typescript
{
  query: string,      // "select:FileRead" 或 "read file content"
  max_results?: number  // 默认 5
}
```

**输出**：
```typescript
{
  matches: string[],          // 匹配到的工具名
  query: string,
  total_deferred_tools: number,
  pending_mcp_servers?: string[]  // 还在连接中的 MCP 服务器
}
```

**关键实现：延迟工具判定**

```typescript
// prompt.ts 中的 isDeferredTool()
export function isDeferredTool(tool: Tool): boolean {
  if (tool.alwaysLoad === true) return false  // 显式不延迟
  if (tool.isMcp === true) return true         // MCP 工具全部延迟
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false  // ToolSearch 自身不能延迟！
  return tool.shouldDefer === true             // 看 shouldDefer 标志
}
```

**关键实现：关键词评分算法**

工具名和描述都参与评分，分值设计：

| 匹配位置 | 普通工具分值 | MCP 工具分值 |
|---------|------------|------------|
| 工具名精确部分匹配 | +10 | +12 |
| 工具名包含匹配 | +5 | +6 |
| searchHint 关键词边界匹配 | +4 | +4 |
| 描述中关键词边界匹配 | +2 | +2 |

MCP 工具得分加成的原因：MCP 工具命名规范（`mcp__server__action`），服务器名高度相关，权重更大。

**select: 前缀的特殊处理**：

```typescript
const selectMatch = query.match(/^select:(.+)$/i)
if (selectMatch) {
  const requested = selectMatch[1]!.split(',').map(s => s.trim())
  // 在 deferredTools 和 全部 tools 中都找，already-loaded 的也接受
  // 这样 AI 误用 select: 调用已加载工具时，不会出错
}
```

**结果格式（特殊）**：

```typescript
mapToolResultToToolResultBlockParam(content, toolUseID) {
  return {
    type: 'tool_result',
    tool_use_id: toolUseID,
    content: content.matches.map(name => ({
      type: 'tool_reference',  // 特殊类型！
      tool_name: name,
    })),
  }
}
```

返回 `tool_reference` 而非普通 text——Anthropic API 会把这个特殊类型解析为"展开该工具的完整 schema 插入上下文"。这是协议层特性，不是 Claude Code 自己拼接的。

### 15.2.4 为何是"只读"工具

```typescript
isReadOnly() { return true }
isConcurrencySafe() { return true }
```

读取工具 schema 不修改任何状态，天然是只读操作，可以并发。

---

## 15.3 SkillTool — Slash Command 技能执行器

### 15.3.1 解决什么问题

用户可以在 `.claude/` 目录下写 Markdown 格式的 "Skill"（技能），作为快捷指令（如 `/commit`、`/review-pr`）。这些技能本质是"注入一段上下文 + 指令"的快捷方式。

`SkillTool` 让 AI 能**主动触发**这些技能，而不只是等用户手敲斜杠命令。

**C++ 类比**：就像 C++ 里的 `std::bind` 或函数适配器——把一组已有的操作封装成一个可以被调用的对象，调用时展开执行。

### 15.3.2 两种执行模式

| 模式 | 触发条件 | 行为 |
|------|---------|------|
| **inline（内联）** | 普通 Skill（默认） | 把 Skill 的 prompt 注入为新的 user message，AI 继续在同一对话中处理 |
| **fork（分叉）** | `context: fork` 属性 | 启动独立子 Agent 执行 Skill，子 Agent 完成后返回结果摘要 |

### 15.3.3 何时触发

- 任务场景明确匹配某个已知 Skill 的能力
- 用户描述的需求与某个 Skill 的描述高度吻合
- AI 收到类似 `use the pdf skill` 的暗示

### 15.3.4 核心实现

```
src/tools/SkillTool/SkillTool.ts
```

**输入**：
```typescript
{
  skill: string,    // 技能名，如 "commit" 或 "/commit"
  args?: string,    // 可选参数
}
```

**`validateInput()` 的检查链**：

```
去掉开头的 '/' 
  → 查找本地命令列表
    → 找不到 → 返回 "Unknown skill" 错误
    → 找到但 disableModelInvocation=true → 返回错误
    → 不是 prompt 类型 → 返回错误
    → 通过 ✓
```

**`checkPermissions()` 决策逻辑**：

```
检查 denyRules → 命中 → deny
  ↓ 未命中
检查 allowRules → 命中 → allow
  ↓ 未命中
检查 skillHasOnlySafeProperties → 安全属性 → auto-allow
  ↓ 否则
返回 "ask"（弹出确认对话框）
```

`skillHasOnlySafeProperties()` 维护了一个 safe 属性白名单：
```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'context', 'agent', ...
])
```
**新增属性默认需要权限**——这是安全优先的白名单设计。

**`call()` 执行流程（inline 模式）**：

```
1. processPromptSlashCommand(commandName, args)
   → 解析 Skill 的 Markdown，替换 $ARGUMENTS 等变量
   → 返回 { messages: [...], allowedTools: [...], model?: string }
   
2. 过滤掉 <command-message> 标签（避免重复渲染）

3. 用 tagMessagesWithToolUseID() 给这些新消息打标签
   → 绑定到当前 tool_use_id，确保事务性（工具失败时能清理）
   
4. 返回 { data, newMessages, contextModifier }
   → contextModifier 可修改允许的工具列表、切换模型
```

**`call()` 执行流程（fork 模式）**：

```
1. 创建新 agentId
2. 调用 runAgent() 启动子 Agent
3. 流式接收子 Agent 的消息，汇报进度（onProgress）
4. 子 Agent 结束后提取最终文本（extractResultText）
5. 返回结果摘要，清理 invokedSkills 状态
```

**Model Override（模型覆盖）**：

Skill 可以在 frontmatter 里指定模型：
```markdown
---
model: claude-opus-4
---
```

`contextModifier` 会在调用链上修改模型选择：
```typescript
contextModifier(ctx) {
  if (model) {
    return {
      ...ctx,
      options: {
        ...ctx.options,
        mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel),
      }
    }
  }
}
```

---

## 15.4 Task 系列（V2）— 持久化任务管理

Task V2 是 TodoWriteTool 的升级版，将任务**持久化到磁盘**（`~/.claude/tasks/`），支持多 Agent 共享、任务依赖（blocks/blockedBy）、任务所有者（owner）。

### 15.4.1 系统概述

```
~/.claude/tasks/{task-list-id}/
  ├── 1.json   ← 任务 #1
  ├── 2.json   ← 任务 #2
  └── ...
```

每个任务一个 JSON 文件，原子写入。当多个 Agent 协作时，它们共享同一个任务目录（通过 `getTaskListId()` 获取目录名）。

**状态机**：
```
pending → in_progress → completed
                ↕
              deleted（随时可删）
```

### 15.4.2 TaskCreateTool — 创建任务

```
src/tools/TaskCreateTool/TaskCreateTool.ts
```

**解决的问题**：需要一个标准化的"开工前登记"动作，既能追踪进度，又能触发 hook（钩子）。

**输入**：
```typescript
{
  subject: string,      // 任务标题（祈使句）
  description: string,  // 任务详情
  activeForm?: string,  // 进行时文案（"Fixing..."）
  metadata?: Record<string, unknown>  // 自定义元数据
}
```

**输出**：
```typescript
{ task: { id: string, subject: string } }
```

**执行流程**：

```
1. createTask() → 写入 {id}.json 文件（status: 'pending'）

2. executeTaskCreatedHooks()
   → 运行用户配置的 TaskCreated 钩子（类似 git hook）
   → 如果 hook 报错（blockingError），立即 deleteTask + 抛出异常
   → 这保证了"要么全部成功，要么完全回滚"的原子语义
   
3. setAppState → 自动展开任务面板（expandedView: 'tasks'）
```

**Hook 机制**是亮点：`executeTaskCreatedHooks()` 是一个异步生成器，可配置多个外部脚本，全部通过才算创建成功，任一失败都会回滚。这是"基础设施即代码"的扩展点——用户可以接入 CI/CD、工单系统等。

**触发时机**：
- 接到需要 3 步以上的任务时
- Plan Mode 规划工作时
- 多 Agent 协作时（leader 为 teammates 创建任务）

---

### 15.4.3 TaskGetTool — 按 ID 获取任务

```
src/tools/TaskGetTool/TaskGetTool.ts
```

**解决的问题**：`TaskListTool` 返回的是摘要（只有 subject + status），`TaskGetTool` 返回完整详情，包括 description（需求细节）和依赖关系。

**输入**：
```typescript
{ taskId: string }
```

**输出**：
```typescript
{
  task: {
    id: string,
    subject: string,
    description: string,    // 完整需求
    status: TaskStatus,
    blocks: string[],       // 当前任务阻塞哪些任务
    blockedBy: string[],    // 哪些任务阻塞当前任务
  } | null
}
```

**触发时机**：
- 被分配任务后，读取完整描述
- 在开始执行前，确认 `blockedBy` 列表为空
- 需要了解任务依赖关系时

**isReadOnly = true**：只读，可并发调用。

---

### 15.4.4 TaskListTool — 列出全部任务

```
src/tools/TaskListTool/TaskListTool.ts
```

**解决的问题**：全局视角——"现在有哪些任务？哪些可以认领？哪些被阻塞？"

**输入**：无（`z.strictObject({})`）

**输出**：
```typescript
{
  tasks: Array<{
    id: string,
    subject: string,
    status: 'pending' | 'in_progress' | 'completed',
    owner?: string,          // 谁在做
    blockedBy: string[],     // 依赖过滤（已完成任务不出现）
  }>
}
```

**关键实现：过滤已解决的依赖**

```typescript
const resolvedTaskIds = new Set(
  allTasks.filter(t => t.status === 'completed').map(t => t.id)
)

// blockedBy 只保留未完成的任务 ID
blockedBy: task.blockedBy.filter(id => !resolvedTaskIds.has(id)),
```

这样，当依赖任务完成后，被阻塞任务的 `blockedBy` 自动变空——UI 层无需额外轮询。

**内部任务过滤**：
```typescript
const allTasks = (await listTasks(taskListId)).filter(
  t => !t.metadata?._internal  // 过滤内部任务（系统用，不展示给 AI）
)
```

**多 Agent 协作时的 Teammate 工作流**（在 `isAgentSwarmsEnabled()` 时注入额外 prompt）：
```
1. 完成当前任务后，调用 TaskList 找下一个可用任务
2. 优先认领 ID 最小的任务（ID 越小越可能是上游依赖）
3. 用 TaskUpdate 设置 owner 认领任务
4. 如果全被阻塞，通知 team lead
```

**触发时机**：
- 完成一个任务后，找下一个
- 检查整体进度
- 被分配前，了解全局状态

---

### 15.4.5 TaskUpdateTool — 更新任务状态

```
src/tools/TaskUpdateTool/TaskUpdateTool.ts
```

**解决的问题**：任务的全生命周期管理入口——标记进度、转移所有权、建立依赖、删除任务。

**输入**（都是可选的，按需提供）：
```typescript
{
  taskId: string,           // 必须
  status?: 'pending' | 'in_progress' | 'completed' | 'deleted',
  subject?: string,
  description?: string,
  activeForm?: string,
  owner?: string,
  addBlocks?: string[],     // 增加"被我阻塞"的任务
  addBlockedBy?: string[],  // 增加"阻塞我"的任务
  metadata?: Record<string, unknown | null>,  // null 表示删除该 key
}
```

**几个值得关注的细节**：

**1. 完成钩子（TaskCompleted Hooks）**：
```typescript
if (status === 'completed') {
  // 运行 TaskCompleted 钩子
  // 如果 hook 报 blockingError，拒绝完成（返回错误，任务保持 in_progress）
}
```
这允许用户配置"不跑测试不能标记完成"之类的规则。

**2. 自动设置 owner（多 Agent 场景）**：
```typescript
if (isAgentSwarmsEnabled() && status === 'in_progress' && !existingTask.owner) {
  updates.owner = getAgentName()  // 自动把自己设为 owner
}
```

**3. 通过邮箱通知新 owner（多 Agent 场景）**：
```typescript
if (updates.owner && isAgentSwarmsEnabled()) {
  await writeToMailbox(updates.owner, {
    type: 'task_assignment',
    taskId, subject, description, assignedBy
  })
}
```
任务转让时，通过文件系统邮箱（`~/.claude/tasks/{team}/{owner}/inbox/`）异步通知新主人。

**4. 验证 Nudge（同 TodoWriteTool）**：
全部任务完成、≥3 个任务、无验证步骤 → 返回结果追加提示。

**触发时机**：
- 开始一个任务前：`status: 'in_progress'`
- 完成一个任务后：`status: 'completed'`
- 任务不再需要：`status: 'deleted'`
- 建立任务依赖：`addBlocks` / `addBlockedBy`
- 多 Agent 分配任务：设置 `owner`

---

### 15.4.6 TaskStopTool — 停止后台任务进程

```
src/tools/TaskStopTool/TaskStopTool.ts
```

**解决的问题**：`BashTool` 可以在后台启动长时间运行的进程（如 `npm run dev &`），有时需要中途停止它。`TaskStopTool`（曾名 `KillShell`）就是这个"Kill 按钮"。

**注意**：这里的"Task"是指**运行时 Task（后台进程）**，不同于 TaskCreate/Update/List 里管理的"待办事项 Task"——命名有点歧义，但它们是两个不同系统。

**输入**：
```typescript
{
  task_id?: string,   // 后台任务的运行时 ID
  shell_id?: string,  // 已废弃的旧参数名（兼容老版本）
}
```

**`validateInput()` 三重检查**：
1. `task_id` 不能为空
2. 在 `AppState.tasks` 中能找到对应任务
3. 任务状态必须是 `'running'`（不能 kill 已停止的）

**执行流程**：
```
stopTask(id, { getAppState, setAppState })
  → 发送 SIGTERM / SIGKILL
  → 更新任务状态为 'stopped'
  → 返回 { taskId, taskType, command }
```

**触发时机**：
- 后台进程不再需要（如开发服务器）
- 后台进程响应超时需要重启
- 用户明确要求停止某个后台任务

**兼容性设计**：
```typescript
aliases: ['KillShell']  // 保留旧名字，老的 transcript 重放时不会报错
```

---

## 15.5 Team 系列 — 多 Agent 协作团队

### 15.5.1 系统概述

Team 系列支持"一个 Leader + N 个 Teammate"的协作模型：

```
Leader（主 Agent）
  ├── 调用 TeamCreate → 创建团队
  ├── 调用 TaskCreate → 创建任务列表
  ├── 调用 AgentTool → 启动 Teammates
  │     Teammate A ──→ TaskUpdate(owner: A)
  │     Teammate B ──→ TaskUpdate(owner: B)
  ├── 收到 Teammates 完成通知（自动投递到收件箱）
  └── 调用 TeamDelete → 解散团队
```

团队数据存在磁盘：
```
~/.claude/teams/{team-name}/config.json   ← 团队配置
~/.claude/tasks/{team-name}/              ← 共享任务列表
```

### 15.5.2 TeamCreateTool — 创建团队

```
src/tools/TeamCreateTool/TeamCreateTool.ts
```

**解决的问题**：初始化多 Agent 协作的"公共基础设施"——团队配置、共享任务目录、Leader 的身份注册。

**输入**：
```typescript
{
  team_name: string,      // 团队名
  description?: string,   // 团队目标描述
  agent_type?: string,    // Leader 的角色（如 "researcher"）
}
```

**输出**：
```typescript
{
  team_name: string,
  team_file_path: string,   // ~/.claude/teams/{name}/config.json
  lead_agent_id: string,    // 格式：team-lead@{team_name}
}
```

**执行流程**：

```
1. 检查是否已有团队 → 一个 Leader 只能管一个团队
   
2. generateUniqueTeamName() → 防止重名
   └── 若名字冲突，生成随机词组（如 "swift-eagle"）
   
3. 生成 leadAgentId = "team-lead@{teamName}"（确定性 ID）

4. 写入 team config JSON（包含成员列表、Leader 信息）

5. resetTaskList() + ensureTasksDir() → 创建独立任务目录

6. setLeaderTeamName() → 告诉 getTaskListId() 使用团队目录

7. 更新 AppState.teamContext → UI 层知道已进入团队模式
```

**Leader 不设 `CLAUDE_CODE_AGENT_ID`**：
Leader 不是普通 Teammate，不应触发 `isTeammate()=true`（否则会开始轮询收件箱，导致死循环）。Leader 的 ID 通过 `appState.teamContext` 管理，而非环境变量。

**触发时机**：
- 用户要求多 Agent 并行工作
- 任务复杂到一个 Agent 难以独立完成（前后端并行开发、多模块重构等）

---

### 15.5.3 TeamDeleteTool — 解散团队

```
src/tools/TeamDeleteTool/TeamDeleteTool.ts
```

**解决的问题**：团队任务完成后，清理磁盘上的团队配置和任务文件，释放资源，重置状态。

**输入**：无参数（从 `AppState.teamContext` 自动获取团队名）

**输出**：
```typescript
{
  success: boolean,
  message: string,
  team_name?: string
}
```

**安全检查**：
```typescript
const activeMembers = nonLeadMembers.filter(m => m.isActive !== false)
if (activeMembers.length > 0) {
  return { success: false, message: `Cannot cleanup team with ${activeMembers.length} active member(s)...` }
}
```

**不能强行解散有活跃成员的团队**——必须先发 `shutdown_request` 消息让 Teammate 优雅退出，再调用 `TeamDelete`。

**完整清理序列**：
```
1. 检查活跃成员（isActive !== false）
2. cleanupTeamDirectories(teamName)
   → 删除 ~/.claude/teams/{name}/
   → 删除 ~/.claude/tasks/{name}/
3. unregisterTeamForSessionCleanup() → 会话结束时不重复清理
4. clearTeammateColors() → 重置颜色分配（下次创建团队从头开始）
5. clearLeaderTeamName() → getTaskListId() 回退到 session ID
6. AppState.teamContext = undefined
7. AppState.inbox.messages = [] → 清空未读消息
```

**触发时机**：
- 所有 Teammates 完成工作并退出后
- 收到 Teammate 的 `shutdown_request` 响应后
- 用户明确要求结束多 Agent 协作

---

## 15.6 综合对比：V1 vs V2 任务系统

| 维度 | `TodoWriteTool`（V1） | Task 系列（V2） |
|------|--------------------|----------------|
| 存储位置 | `AppState`（内存） | `~/.claude/tasks/`（磁盘） |
| 跨 Agent 共享 | ❌ 不支持 | ✅ 共享目录 |
| 任务依赖 | ❌ 没有 | ✅ blocks/blockedBy |
| 任务所有者 | ❌ 没有 | ✅ owner 字段 |
| 任务 Hook | ❌ 没有 | ✅ TaskCreated/TaskCompleted |
| 操作粒度 | 全量替换 | 单任务 CRUD |
| 启用条件 | `!isTodoV2Enabled()` | `isTodoV2Enabled()` |
| 适用场景 | 单 Agent 简单任务 | 多 Agent 协作 / 复杂项目 |

---

## 15.7 调用时序总结

### 单 Agent 场景（V1）

```
用户: "帮我重构这个函数，然后加测试，最后检查文档"
  ↓
AI → TodoWriteTool([
  { id:"1", content:"Refactor function", status:"pending" },
  { id:"2", content:"Add tests", status:"pending" },
  { id:"3", content:"Check docs", status:"pending" },
])

AI 开始任务1 → TodoWriteTool(更新id:1 为 in_progress)
AI 完成任务1 → TodoWriteTool(更新id:1 为 completed, id:2 为 in_progress)
...
```

### 多 Agent 场景（V2）

```
用户: "用两个 Agent 并行实现前后端"
  ↓
Leader → TeamCreateTool({ team_name: "fullstack-team" })
Leader → TaskCreateTool("Implement backend API")
Leader → TaskCreateTool("Implement frontend UI")
Leader → AgentTool(subagent_type: "general", name: "backend-agent", team_name: ...)
Leader → AgentTool(subagent_type: "general", name: "frontend-agent", team_name: ...)

backend-agent → TaskListTool()
backend-agent → TaskUpdateTool({ taskId: "1", status: "in_progress" })
backend-agent → [执行开发工作]
backend-agent → TaskUpdateTool({ taskId: "1", status: "completed" })
backend-agent → [发送 idle 消息]

frontend-agent → [并行执行同样流程]

Leader → [收到双方完成通知]
Leader → TeamDeleteTool()
```

### ToolSearch 场景

```
AI 想调用 BashTool（首次）：
  AI → ToolSearchTool({ query: "select:Bash" })
  系统返回 BashTool 的完整 JSONSchema（tool_reference 类型）
  AI 现在有了 schema，调用 BashTool(...)
```

### SkillTool 场景

```
用户说"帮我提交代码"：
  AI → SkillTool({ skill: "commit" })
  
  inline 模式：
    系统把 CLAUDE/commit.md 的内容注入为 user message
    AI 读取其中的提交规范，然后执行 git commit 相关操作
  
  fork 模式（若 Skill 有 context: fork）：
    系统启动子 Agent 执行 commit Skill
    子 Agent 返回结果
    AI 收到摘要
```

---

## 15.8 设计哲学提炼

**1. "工具即接口，不是实现"**  
Task/Team 工具只是接口层，底层是 `utils/tasks.ts`、`utils/swarm/teamHelpers.ts` 等模块。工具负责验证、权限、UI；实际逻辑在 utils 层。

**2. "全局替换 vs 单条 CRUD"**  
TodoWriteTool 选择了简单的全量替换（`todos[key] = newTodos`），牺牲并发性换取简单性；Task V2 改为单条文件，支持多 Agent 并发读写。

**3. "事务性钩子"**  
TaskCreate 先写文件，再执行 hook，hook 失败则删文件——这是典型的"补偿事务"（Compensating Transaction）模式，类似数据库的 rollback。

**4. "提示注入（Verification Nudge）"**  
TodoWriteTool 和 TaskUpdateTool 都在工具返回结果中内嵌元指令，在恰当时机提醒 AI 补做验证。这是一种精准的、时间窗口限定的"提示注入"，比在 System Prompt 里写死的规则更灵活。

**5. "邮箱模型通信"**  
多 Agent 间不通过共享内存通信，而是通过磁盘上的邮箱目录（`~/.claude/tasks/{team}/{agent}/inbox/`）。这是 Actor 模型的简化实现——每个 Agent 有独立的邮箱，异步投递，无锁。

---

## 本章源码位置速查

| 工具 | 核心文件 |
|------|---------|
| `TodoWriteTool` | `src/tools/TodoWriteTool/TodoWriteTool.ts` |
| `ToolSearchTool` | `src/tools/ToolSearchTool/ToolSearchTool.ts` |
| `SkillTool` | `src/tools/SkillTool/SkillTool.ts` |
| `TaskCreateTool` | `src/tools/TaskCreateTool/TaskCreateTool.ts` |
| `TaskGetTool` | `src/tools/TaskGetTool/TaskGetTool.ts` |
| `TaskListTool` | `src/tools/TaskListTool/TaskListTool.ts` |
| `TaskUpdateTool` | `src/tools/TaskUpdateTool/TaskUpdateTool.ts` |
| `TaskStopTool` | `src/tools/TaskStopTool/TaskStopTool.ts` |
| `TeamCreateTool` | `src/tools/TeamCreateTool/TeamCreateTool.ts` |
| `TeamDeleteTool` | `src/tools/TeamDeleteTool/TeamDeleteTool.ts` |
| 任务工具函数 | `src/utils/tasks.ts` |
| 团队辅助函数 | `src/utils/swarm/teamHelpers.ts` |
| 延迟工具判定 | `src/tools/ToolSearchTool/prompt.ts` — `isDeferredTool()` |
