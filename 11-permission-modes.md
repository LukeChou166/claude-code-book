# 第11章 权限模式详解：从 Plan 到 Auto

> **前置知识**：第3章 Agentic Loop、第5章 权限系统  
> **关键文件**：`src/types/permissions.ts`、`src/utils/permissions/PermissionMode.ts`、`src/utils/permissions/permissionSetup.ts`、`src/utils/permissions/getNextPermissionMode.ts`、`src/utils/permissions/yoloClassifier.ts`、`src/tools/ExitPlanModeTool/`、`src/tools/EnterPlanModeTool/`

---

## 11.1 为什么需要多种模式

用户和 AI 之间的"信任程度"天然不是二元的。考虑以下场景：

- **探索阶段**：用户只想看 AI 的思路，不想让它动任何文件
- **日常协作**：用户愿意让 AI 写文件，但想审批每条 shell 命令
- **熟悉任务**：用户已经了解任务风险，不想被频繁打断
- **CI/CD 环境**：根本没有人在场，需要全自动运行

Cursor 用 **Ask / Agent / Plan** 这样按"意图"划分的模式解决这个问题。Claude Code 选择了不同的维度——按**权限粒度**划分，从"什么都不允许"到"什么都允许"形成一个完整的连续谱。

本章将深入每种模式的实现原理。

---

## 11.2 五种权限模式全景

```typescript
// src/types/permissions.ts
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',        // 模式2：自动接受文件编辑
  'bypassPermissions',  // 模式4：完全绕过权限检查
  'default',            // 模式1：默认，危险操作询问
  'dontAsk',            // 模式3.5：不再询问（当前会话）
  'plan',               // 模式0：只读探索，不执行
] as const

// 内部专有模式（仅 Anthropic 内部用户）
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

用 `Shift+Tab` 循环切换时，顺序如下（见 `src/utils/permissions/getNextPermissionMode.ts`）：

```
default  →  acceptEdits  →  plan  →  (bypassPermissions)  →  (auto)  →  default
```

---

## 11.3 模式0：Plan Mode（计划模式）

### 11.3.1 核心思想

Plan Mode 是 Claude Code 最具特色的模式。它的核心约束极其简单：**AI 只能读，不能写**。它的价值在于给用户一个"干跑（dry run）"的机会，在任何真实修改发生之前，先看到 AI 的完整行动方案。

### 11.3.2 进入 Plan Mode 的三种路径

**路径一：用户手动触发 `/plan` 命令**

```typescript
// src/commands/plan/plan.tsx
export async function call(onDone, context, args) {
  const currentMode = appState.toolPermissionContext.mode;

  if (currentMode !== 'plan') {
    // 1. 通知状态管理层（发送遥测、触发动画）
    handlePlanModeTransition(currentMode, 'plan');
    // 2. 更新不可变的权限上下文
    setAppState(prev => ({
      ...prev,
      toolPermissionContext: applyPermissionUpdate(
        prepareContextForPlanMode(prev.toolPermissionContext),
        { type: 'setMode', mode: 'plan', destination: 'session' }
      ),
    }));
  }
}
```

**路径二：AI 主动调用 `EnterPlanMode` 工具**

这是最有趣的路径——AI 自己决定何时需要先计划。工具的提示词（`src/tools/EnterPlanModeTool/prompt.ts`）教 AI 在以下情况下主动进入计划模式：

- 任务有多种合理实现方案
- 需要修改 3 个以上的文件
- 需求模糊，需要先探索
- 用户说"帮我重构"这种宽泛指令

**路径三：CLI 参数 `--permission-mode plan`**

直接以计划模式启动整个会话，适合"只让 AI 分析，永远不执行"的只读工作流。

### 11.3.3 Plan Mode 的内部机制

进入 Plan Mode 时，`prepareContextForPlanMode()` 做了几件关键事：

```typescript
// src/utils/permissions/permissionSetup.ts
export function prepareContextForPlanMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  const currentMode = context.mode;
  if (currentMode === 'plan') return context;

  // 关键：把当前模式存入 prePlanMode，退出时恢复
  return { ...context, prePlanMode: currentMode };
}
```

`prePlanMode` 的作用是"模式快照"——无论用户在什么模式下进入 Plan，退出时都能精确回到原来的模式，而不是统一退回 `default`。

### 11.3.4 Plan Mode 的执行时提示词注入

进入 Plan Mode 后，每轮对话都会向 AI 注入一段强制性的系统提示（`src/utils/messages.ts`）：

```
Plan mode is active. The user indicated that they do not want you to execute 
yet -- you MUST NOT make any edits (with the exception of the plan file 
mentioned below), run any non-readonly tools (including changing configs or 
making commits), or otherwise make any changes to the system. This supercedes 
any other instructions you have received.
```

这段提示的关键字是 **"supercedes any other instructions"**——它的优先级高于用户的其他指令。这防止了"用户不小心说了一句'顺便把这个文件改一下'"导致 Plan Mode 被绕过。

唯一的例外：AI 可以写**计划文件**（`PLAN.md`），这是 Plan Mode 中唯一允许的写操作。

### 11.3.5 两种计划工作流

两种工作流通过同一个入口函数切换，核心是 `messages.ts` 里的 `getPlanModeV2Instructions()`：

```typescript
// src/utils/messages.ts
function getPlanModeV2Instructions(attachment) {
  if (attachment.isSubAgent) return []          // 子 Agent 不注入计划指令

  // 关键分叉：根据 feature flag 选择工作流
  if (isPlanModeInterviewPhaseEnabled()) {
    return getPlanModeInterviewInstructions(attachment)  // 迭代访谈工作流
  }
  // 否则走经典 5 阶段工作流（下面的代码）
  ...
}
```

#### 控制开关

```typescript
// src/utils/planModeV2.ts
export function isPlanModeInterviewPhaseEnabled(): boolean {
  if (process.env.USER_TYPE === 'ant') return true   // 内部用户永远启用

  const env = process.env.CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE
  if (isEnvTruthy(env)) return true                  // 环境变量强制开
  if (isEnvDefinedFalsy(env)) return false           // 环境变量强制关

  return getFeatureValue_CACHED_MAY_BE_STALE(        // GrowthBook 灰度
    'tengu_plan_mode_interview_phase',
    false,
  )
}
```

三级优先级：Anthropic 内部用户（永远开）> 环境变量 > GrowthBook 远程灰度。

#### 工作流 A：经典 5 阶段（`isPlanModeInterviewPhaseEnabled()` = false）

AI 收到的提示词要求分 5 个阶段：

```
Phase 1: 派生最多 N 个 explore agent 并行探索代码库
Phase 2: 派生 plan agent 设计方案
Phase 3: 审查方案，用 AskUserQuestion 补充疑问
Phase 4: 写入计划文件（格式严格，限 40 行，不写背景段落）
Phase 5: 调用 ExitPlanMode 请求审批
```

探索 agent 的数量由订阅套餐决定（`getPlanModeV2AgentCount()`）：Max 20x 和企业版/团队版返回 3，其余返回 1。

#### 工作流 B：迭代访谈（`isPlanModeInterviewPhaseEnabled()` = true，内部用户默认）

AI 收到的提示词是一个循环，而不是线性阶段：

```
### The Loop

Repeat this cycle until the plan is complete:

1. Explore — 读代码，寻找可复用的函数和模式
2. Update the plan file — 每次发现后立即写入，不等到最后
3. Ask the user — 遇到无法从代码推断的决策时，用 AskUserQuestion
```

第一轮有额外约束：**先快速扫几个文件，写出计划骨架，然后立刻问用户问题**——不允许"把代码探索完了再来问"。

计划格式也更宽松：需要包含 Context（为什么改）、推荐方案、文件路径、可复用函数（带行号引用）、验证步骤，没有行数上限。

#### 两种工作流的本质差异

| 维度 | 经典 5 阶段 | 迭代访谈（内部默认） |
|------|-------------|----------------------|
| **探索方式** | 派生 explore agent 并行搜索 | AI 直接读文件，可选用 agent |
| **与用户互动时机** | Phase 3 才问问题 | **第一轮就开始问** |
| **计划文件写入** | Phase 4 统一输出 | **边探索边写入** |
| **计划格式** | 严格：40 行、不写背景 | 宽松：分节，含 Context |

无论哪种工作流，**每轮对话只能以两种方式结束**：`AskUserQuestion`（还有问题）或 `ExitPlanMode`（计划完成）。后续轮次还会注入一段精简提示（`getPlanModeV2SparseInstructions()`）提醒 AI 当前仍在计划模式，并说明应遵循哪种工作流：

```typescript
const workflowDescription = isPlanModeInterviewPhaseEnabled()
  ? 'Follow iterative workflow: explore codebase, interview user, write to plan incrementally.'
  : 'Follow 5-phase workflow.'
```

### 11.3.6 ExitPlanMode：用户审批流程

`ExitPlanMode` 工具（V2 版本）本身**不接受计划内容作为参数**，它只是一个"我写完了，来看看"的信号。系统从计划文件中读取内容展示给用户。

用户在审批弹窗（`ExitPlanModePermissionRequest`）中有以下选项：

| 选项 | 含义 | 退出后的权限模式 |
|------|------|-----------------|
| `yes-default-keep-context` | 批准，保持当前对话上下文 | `default`（需逐一确认） |
| `yes-accept-edits` | 批准，清空上下文后开始 | `acceptEdits`（文件编辑自动通过） |
| `yes-accept-edits-keep-context` | 批准，保持上下文 | `acceptEdits` |
| `yes-bypass-permissions` | 批准，全自动执行 | `bypassPermissions` |
| `yes-resume-auto-mode` | 批准，用 AI 分类器决策 | `auto`（内部用户） |
| `ultraplan` | 发送到云端 CCR 执行 | 远程会话 |
| `no` | 拒绝，AI 继续在 Plan Mode 修改 | `plan`（保持） |

用户批准时，计划文件内容被嵌入 tool_result 发回给 AI：

```
User has approved your plan. You can now start coding.

## Approved Plan:
<plan content here>
```

这告诉 AI："计划被批准了，现在可以开始执行了"。

### 11.3.7 Plan Mode + 多 Agent 协作

当 `COORDINATOR_MODE` 启用且任务可并行时，`ExitPlanMode` 的审批信息里还会提示：

```
If this plan can be broken down into multiple independent tasks, 
consider using the TeamCreate tool to create a team and parallelize the work.
```

AI 可以在计划批准后立即拆分任务，派生多个子 Agent 并行执行（详见第6章）。

---

### 11.3.8 Plan Mode 的三道防线：如何保证模型不乱执行

Plan Mode 的核心约束是"只能读不能写"。但大模型不是强规则型的——它可能"不小心"调用了 `FileEditTool`。系统用**三道独立的防线**确保模型无法在用户不知情的情况下写入文件，每道防线都无法被模型绕过。

#### 防线一：Prompt 层的强制指令（最表层）

模型进入 Plan Mode 后，`EnterPlanModeTool.mapToolResultToToolResultBlockParam()` 会把指令作为 tool_result 直接打回给模型：

```typescript
// src/tools/EnterPlanModeTool/EnterPlanModeTool.ts 第 103-118 行
// 进入 plan mode 后，tool_result 里的内容：
`Entered plan mode. You should now focus on exploring the codebase...

Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.`
```

同时，每轮对话系统都会通过 `getPlanModeV2Instructions()` 重复注入一段强制提示（`messages.ts` 第 3227 行）：

> **"Plan mode is active... you MUST NOT make any edits... This supercedes any other instructions you have received."**

关键字是 **"supercedes any other instructions"**——防止用户消息里夹杂"顺便改一下这个文件"之类的 prompt injection。

**但这层防线依赖模型合作**，如果模型"不听话"仍调用写操作工具，就到了第二道防线。

#### 防线二：权限系统层的硬拦截（核心防线）

当 mode = `'plan'` 时，`checkWritePermissionForTool()`（`filesystem.ts`）**有意跳过了 plan mode 的自动允许逻辑**：

```typescript
// src/utils/permissions/filesystem.ts 第 1360-1375 行
// 3. If in acceptEdits mode, allow all writes in cwd
if (toolPermissionContext.mode === 'acceptEdits' && isInWorkingDir) {
  return { behavior: 'allow', ... }
}
// ↑ 只检查 acceptEdits，故意不检查 plan
// → plan mode 下写操作走到最后一步，必然返回 'ask'

// 5. Default to asking for permission
return {
  behavior: 'ask',   // ← 弹出用户确认框，不自动执行
  message: `Claude requested permissions to write to ${path}...`,
}
```

完整调用链：

```
FileEditTool.call()
    ↓
checkPermissions()（工具权限检查）
    ↓
checkWritePermissionForTool()
    ↓
mode === 'plan' → 跳过自动允许逻辑
    ↓
behavior: 'ask' → 弹出"是否允许写操作？"对话框
    ↓
用户点"否" → 工具被拒绝，代码不会被写入
```

**即使模型调用了 `FileEditTool`，系统在执行前会弹出确认框。用户拒绝即可，代码不会被写入。** 这道防线不依赖模型自觉，是硬编码的权限拦截。

#### 防线三：ExitPlanMode 必须用户手动确认（出口锁）

`ExitPlanModeV2Tool` 的 `checkPermissions()` 返回 `behavior: 'ask'`（第 233-238 行），意味着**必须用户手动点击批准，才能退出 Plan Mode 进入执行阶段**：

```typescript
// src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts 第 233-238 行
// For non-teammates, require user confirmation to exit plan mode
return {
  behavior: 'ask' as const,
  message: 'Exit plan mode?',
  updatedInput: input,
}
```

只有批准后，系统才把计划文本打包成执行令，通过 `initialMessage` 机制启动全新执行会话（`ExitPlanModePermissionRequest.tsx` 第 374-387 行）：

```typescript
setAppState(prev => ({
  ...prev,
  initialMessage: {
    message: createUserMessage({
      content: `Implement the following plan:\n\n${currentPlan}...`
    }),
    clearContext: true,  // ← 清空对话上下文，开启全新执行会话
    mode,
    allowedPrompts
  }
}))
```

**没有这条消息，模型永远停留在 Plan Mode，无法自行切换到执行阶段。**

#### 唯一例外：plan file 本身可以写

`checkEditableInternalPath()`（`filesystem.ts` 第 1479 行）允许写入计划文件：

```typescript
// Plan files for current session — 仅此一个例外
if (isSessionPlanFile(normalizedPath)) {
  return {
    behavior: 'allow',
    decisionReason: { type: 'other', reason: 'Plan files for current session are allowed for writing' },
  }
}
```

#### 三道防线总结

```
EnterPlanMode 触发
        ↓
┌─────────────────────────────────────────────────────────┐
│ 防线 1（Prompt 层）                                      │
│ tool_result 里注入 "DO NOT write files" 指令            │
│ 每个 turn 重复注入（supercedes any other instructions） │
│ → 告诉模型不要写文件（依赖模型合作）                    │
└─────────────────────────────────────────────────────────┘
        ↓  如果模型仍调用 FileEditTool
┌─────────────────────────────────────────────────────────┐
│ 防线 2（权限系统层）                                     │
│ checkWritePermissionForTool() 跳过 plan mode 的自动允许 │
│ 返回 behavior: 'ask' → 弹出用户确认框                   │
│ → 用户点"否"即可阻止，代码不会被写入                    │
└─────────────────────────────────────────────────────────┘
        ↓  如果模型想"溜出" plan mode 去执行
┌─────────────────────────────────────────────────────────┐
│ 防线 3（出口锁）                                         │
│ ExitPlanMode 的 checkPermissions 返回 'ask'              │
│ 必须用户手动批准计划，才能切换回执行模式                 │
│ 只有批准后，系统才发送 "Implement the plan" 执行令       │
└─────────────────────────────────────────────────────────┘
```

---

### 11.3.9 执行阶段的步骤保证：软约束而非强执行

#### 计划批准后，模型不一定按步骤执行

用户批准计划退出 Plan Mode 后，系统**没有任何代码能强制模型必须按 1→2→3 步骤顺序执行**。模型可能跳步骤、合并步骤、或换顺序。

模型收到的执行令是（`ExitPlanModeV2Tool.ts` 第 481-490 行）：

```
User has approved your plan. You can now start coding. Start with updating your todo list if applicable

Your plan has been saved to: ${filePath}
You can refer back to it if needed during implementation.

## Approved Plan:
${plan}
```

计划全文被注入为执行令的一部分——相当于给模型一份"地图"，但**没有导航系统强制按路线走**。

#### 四层软性约束

系统用四个层面降低乱执行的概率：

| 层面 | 机制 | 关键代码位置 |
|------|------|-------------|
| **计划全文注入** | 把完整计划原文发回模型作为执行令 | `ExitPlanModeV2Tool.ts` 第 488 行 |
| **TodoWrite 软性建议** | 执行令含 "Start with updating your todo list **if applicable**" | 非强制 |
| **文件持久化** | 计划写入磁盘，模型可随时用 FileReadTool 回读 | "Your plan has been saved to: ${filePath}" |
| **兜底提醒** | 连续 N 个 turn 没调 TodoWrite，系统自动插入 `todo_reminder` attachment | `attachments.ts` 第 3296-3312 行 |

#### TodoWrite 不一定被执行

执行令里的关键词是 **`if applicable`**——这是软性建议，不是强制指令：

```typescript
// src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts 第 483 行
content: `User has approved your plan. You can now start coding. Start with updating your todo list if applicable`
```

模型可以直接调 `FileEditTool` 开始写代码，完全跳过 TodoWrite，系统不会阻止。

**更深的问题**：即使模型想维护 todos，`clearContext: true` 会导致 todos 隐性丢失。

默认批准路径触发 `clearContext: true`，调用 `clearConversation()`，该函数执行 `regenerateSessionId()`（`conversation.ts` 第 203 行）生成全新 session ID。

todos 的存储 key 是 `sessionId`（`TodoWriteTool.ts` 第 67 行）：

```typescript
// src/tools/TodoWriteTool/TodoWriteTool.ts 第 65-67 行
async call({ todos }, context) {
  const appState = context.getAppState()
  const todoKey = context.agentId ?? getSessionId()  // ← key = 当前 sessionId
```

`clearConversation` 的 `setAppState` 返回值（第 168-190 行）没有显式重置 `todos: {}`——todos 数据还在 `AppState.todos[旧sessionId]` 里。但新会话用新 sessionId 查询，拿到的是空数组。**对执行阶段的模型来说，todos 等于被清空了**，必须从头重建。

#### 为什么是软约束而不是硬保证

这是大模型应用的根本限制：Plan Mode 里的"步骤"是给模型看的**文本**，不是像操作系统调度器那样的**强制执行序列**。系统能做的是通过 prompt 工程和 TodoWrite 工具把乱执行的概率降低，但无法提供 ACID 事务般的"步骤完整性保证"。

Anthropic 内部正在开发 `VerifyPlanExecution` 工具（`tools.ts` 第 91-95 行），在计划执行完成后验证计划覆盖度：

```typescript
// src/tools.ts 第 91-95 行
const VerifyPlanExecutionTool =
  process.env.CLAUDE_CODE_VERIFY_PLAN === 'true'
    ? require('./tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.js').VerifyPlanExecutionTool
    : null
```

外部版本 `CLAUDE_CODE_VERIFY_PLAN` 通过死代码消除被移除，说明这是正在开发中的功能——Anthropic 也意识到了这个缺口。

#### 本节小结

| 问题 | 答案 |
|------|------|
| Plan Mode 下模型能偷偷写文件吗？ | **不能**，权限系统在执行前弹出确认框 |
| 用户批准后模型一定按步骤执行吗？ | **不一定**，系统只有软约束（文本建议 + 兜底提醒） |
| 批准后一定会调 TodoWrite 吗？ | **不一定**，`if applicable` 是软性建议；且 clearContext 导致旧 todos 对新会话不可见 |
| Anthropic 知道执行验证的缺口吗？ | **知道**，`VerifyPlanExecution` 工具正在开发中，外部版本尚未开放 |

---

## 11.4 模式1：Default Mode（默认模式）

### 11.4.1 核心行为

Default 模式是最接近"有条件信任"的模式：

- **只读操作**（`FileRead`、`Glob`、`Grep`）：自动允许
- **文件编辑**（`FileEdit`、`FileWrite`）：弹窗询问
- **Shell 命令**：根据命令解析结果决定
  - 纯读操作命令（`ls`、`cat`、`git status` 等）：自动允许
  - 写操作命令（`rm`、`git commit` 等）：弹窗询问

### 11.4.2 Bash 命令的"只读检测"

BashTool 有一个精心维护的**只读命令白名单**（`src/tools/BashTool/readOnlyValidation.ts`），涵盖超过 100 个安全命令：

```typescript
const READONLY_COMMANDS = [
  'ls', 'cat', 'head', 'tail', 'grep', 'find', 'pwd', 'echo',
  'git status', 'git log', 'git diff', 'git show', 'git branch',
  'npm list', 'yarn list',
  // ... 100+ 更多
]
```

对于不在白名单的命令，还会进行**基于 AST 的安全分析**（使用 tree-sitter 解析 bash 语法树），检测：
- 命令注入（shell 替换、管道链）
- 重定向到文件（`>`、`>>`）
- 执行任意脚本（`eval`、`source`）

### 11.4.3 工作目录边界

Default 模式下，文件操作被严格限制在**授权目录**内：

```typescript
// src/utils/permissions/filesystem.ts
// 只有 cwd 和通过 /add-dir 添加的目录才能访问
// 尝试访问 /etc/passwd 会被硬性拒绝，无法通过规则绕过
```

这是一条**硬边界**——即使用户配置了 `alwaysAllowRules`，也无法跨越工作目录边界。

---

## 11.5 模式2：Accept Edits（接受编辑模式）

### 11.5.1 核心行为

Accept Edits 模式在 Default 基础上，自动批准**所有文件编辑操作**：

- `FileEdit`、`FileWrite`、`NotebookEdit`：**自动允许**（无需确认）
- 同时，以下 Bash 文件系统命令也自动允许（`src/tools/BashTool/modeValidation.ts`）：

```typescript
const ACCEPT_EDITS_ALLOWED_COMMANDS = [
  'mkdir',   // 创建目录
  'touch',   // 创建空文件
  'rm',      // 删除文件/目录
  'rmdir',   // 删除空目录
  'mv',      // 移动/重命名
  'cp',      // 复制
  'sed',     // 流编辑（允许 -i 原地修改）
] as const
```

注意：`sed` 命令的允许是有条件的——即使在 Accept Edits 模式下，`sed` 的危险操作（`w`/`W` 写入文件、`e`/`E` 执行命令）仍然会被拦截（`src/tools/BashTool/sedValidation.ts`）。

### 11.5.2 适用场景

Accept Edits 是"我信任这个 AI 写代码，但还是想对非文件操作保持警惕"的最佳平衡点。典型使用场景：

- **重构代码**：文件修改不需要确认，但 `git push` 这样的命令还是要审批
- **批量格式化**：自动执行大量文件的格式化修改

---

## 11.6 模式3：Bypass Permissions（绕过权限模式）

### 11.6.1 核心行为

Bypass Permissions 完全跳过所有权限检查，等价于"我完全信任 AI"。

**启用条件**：必须显式指定 `--dangerously-skip-permissions`（CLI 参数）或在设置中允许。

**适用场景**：无人值守的 CI/CD 环境。

### 11.6.2 安全防线

即使在 Bypass Permissions 模式下，也有两个无法绕过的安全机制：

1. **工作目录边界**：仍然无法访问授权目录之外的文件
2. **Yolo 分类器（可选）**：Anthropic 内部有一个额外的 AI 安全分类器，即使在 bypass 模式下也能对可疑操作发出警告

### 11.6.3 Yolo 分类器

这是 Claude Code 对全自动执行风险的安全补偿机制（`src/utils/permissions/yoloClassifier.ts`）：

```typescript
// 两阶段分类器设计
type TwoStageMode = 'both' | 'fast' | 'thinking'

// 第一阶段（fast）：轻量模型，0-50ms
// - 判断常见安全操作（git status、读文件等）→ 直接允许
// - 判断明显危险操作（rm -rf /、curl | sh）→ 直接阻止
// - 不确定 → 进入第二阶段

// 第二阶段（thinking）：使用 Extended Thinking，500-2000ms
// - 分析完整会话历史和当前操作的上下文
// - 评估意图是否与用户最初的任务一致
// - 输出：shouldBlock: boolean + reason: string
```

**分类器的快捷路径**（避免不必要的推理）：

```typescript
// src/utils/permissions/permissions.ts
// 快捷路径 1：如果这个操作在 acceptEdits 模式下会被允许，直接通过
const acceptEditsResult = await tool.checkPermissions(parsedInput, { mode: 'acceptEdits' });
if (acceptEditsResult.behavior === 'allow') {
  return { behavior: 'allow', ... };
}

// 快捷路径 2：工具在安全白名单中（FileRead、Glob、Grep 等），直接通过
if (classifierDecisionModule.isAutoModeAllowlistedTool(tool.name)) {
  return { behavior: 'allow', ... };
}

// 以上都不满足 → 调用 classifyYoloAction()
```

---

## 11.7 模式4：Auto Mode（自动模式，内部特性）

Auto Mode 是 `TRANSCRIPT_CLASSIFIER` Feature Flag 控制的内部模式，目前仅对 Anthropic 内部用户（`USER_TYPE === 'ant'`）开放。它代表了 Claude Code 对"智能权限决策"的最新探索。

### 11.7.1 核心思想

Auto Mode 不是简单地"允许一切"，而是用**另一个 AI 实例**实时评估每个操作的安全性，取代人工确认。用户设定意图（"允许写测试文件"、"不允许推送到远程"），分类器在此边界内自主决策。

### 11.7.2 进入 Auto Mode 的条件

```typescript
// src/utils/permissions/getNextPermissionMode.ts
function canCycleToAuto(ctx: ToolPermissionContext): boolean {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const gateEnabled = isAutoModeGateEnabled();     // GrowthBook 远程开关
    const can = !!ctx.isAutoModeAvailable && gateEnabled;
    return can;
  }
  return false;
}
```

两个条件缺一不可：
1. `ctx.isAutoModeAvailable`：启动时检查（模型是否支持、账户是否开启）
2. `isAutoModeGateEnabled()`：GrowthBook 实时远程开关（支持熔断）

### 11.7.3 危险权限剥离

进入 Auto Mode 时，有一个重要的安全动作——**剥离用户已预配置的危险权限规则**：

```typescript
// src/utils/permissions/permissionSetup.ts
export function stripDangerousPermissionsForAutoMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  // 把允许执行任意代码的规则从 alwaysAllowRules 中移除
  // 保存到 strippedDangerousRules，退出 Auto Mode 时恢复
  // 这防止用户配置的宽泛规则（如 Bash(*））绕过分类器
}
```

举例：如果用户之前配置了 `Bash(*)` 规则（允许任何 bash 命令），在 Auto Mode 下这条规则会被临时移除，强制所有 bash 命令经过分类器评估。

### 11.7.4 Auto Mode + Plan Mode 的组合

`shouldPlanUseAutoMode()` 允许在 Plan Mode 下也启用 Auto Mode 的分类器语义：

```typescript
// 用户可以在计划阶段就使用分类器自动决策
// 这意味着 Plan Mode 里的读操作也会经过分类器
// 目的：让分类器学习任务上下文，为后续执行阶段做准备
```

### 11.7.5 Auto Mode 的连续拒绝保护

一个防止分类器"过于谨慎"而陷入死循环的机制：

```typescript
// src/utils/permissions/denialTracking.ts
type DenialTrackingState = {
  consecutiveDenials: number;   // 连续拒绝次数
  lastDenialToolName: string;
}

// 如果分类器连续拒绝太多次，触发 shouldFallbackToPrompting()
// 回退到手动弹窗，让用户介入
```

这防止了一种场景：分类器对某类操作持续拒绝，AI 陷入不断重试的循环，用户无法推进任务。

---

## 11.8 权限决策的完整调用链

一次工具调用的完整权限决策流程，比第5章的描述更详细：

```
AI 发出 tool_use 请求
    │
    ▼
1. 规则层检查（checkRuleBasedPermissions）
   │  1a. alwaysDenyRules 匹配 → deny（立即返回）
   │  1b. alwaysAskRules 匹配（整个工具） → ask
   │  1c. tool.checkPermissions()（工具自定义逻辑）
   │      BashTool：解析命令，检查只读白名单
   │      FileEditTool：检查路径是否在工作目录内
   │  1d. 如果工具返回 deny → deny
   │  1e. 如果是 safetyCheck 类型的 ask → 直接 ask（不可绕过）
   │  1f. alwaysAllowRules 匹配 → allow（规则显式允许）
    │
    ▼
2. 模式层决策（hasPermissionsToUseToolInner 续）
   │  bypassPermissions → allow（跳过所有检查）
   │  plan → deny（如果是写操作）
   │  acceptEdits → allow（如果是文件操作或 ACCEPT_EDITS 命令）
    │
    ▼
3. 还是 ask？进入交互层
   │  dontAsk 模式 → 直接 deny（不打断用户）
   │  auto 模式 → 调用 classifyYoloAction()：
   │    ├─ 快捷路径：acceptEdits 检查 → allow
   │    ├─ 快捷路径：工具白名单检查 → allow
   │    └─ 完整分类：两阶段 AI 分类器 → allow 或 deny
   │  default 模式 → 弹出权限确认框
    │
    ▼
4. 权限确认框（React 组件）
   │  用户点"允许一次" → allow（不保存规则）
   │  用户点"始终允许" → allow + 将规则写入 settings.json
   │  用户点"拒绝" → deny + 可选原因反馈
   │  用户修改命令 → allow + updatedInput（新命令）
    │
    ▼
5. 执行工具 / 返回拒绝消息给 AI
```

---

## 11.9 关键设计决策解析

### 为什么权限上下文是不可变的（DeepImmutable）

```typescript
// src/types/permissions.ts
export type ToolPermissionContext = DeepReadonly<{
  mode: PermissionMode;
  alwaysAllowRules: ToolPermissionRulesBySource;
  // ...
}>
```

`ToolPermissionContext` 被设计为完全不可变的数据结构。这不是"最佳实践"，而是安全防线：

- 工具不能在执行过程中提升自己的权限
- 并发工具调用之间不会互相干扰权限状态
- 权限变更必须通过 `applyPermissionUpdate()` 产生新对象，留下可追踪的变更历史

### 为什么 prePlanMode 在退出时才清除

```typescript
// 退出 Plan Mode 时：
if (fromMode === 'plan' && toMode !== 'plan' && context.prePlanMode) {
  return { ...context, prePlanMode: undefined };  // 才清除
}
```

`prePlanMode` 在整个 Plan Mode 期间都保留，这允许系统在任何时刻"知道"Plan Mode 之前的状态，而不只是在退出的瞬间。这对 Auto Mode + Plan Mode 组合尤其重要——系统需要知道是否应该在退出 Plan Mode 时恢复 Auto Mode 的分类器。

### 为什么 Bypass Permissions 的进入需要显式同意

```typescript
// 用户第一次切换到 bypassPermissions 时，弹出警告对话框
// 用户必须勾选 "我了解风险" 才能进入
// 这个同意记录在 settings.skipDangerousModePermissionPrompt
```

这防止了用户通过手误（多按了几下 Shift+Tab）进入危险模式。

---

## 11.10 在源码中找到这些逻辑

| 功能 | 文件 | 关键函数 |
|------|------|---------|
| 模式定义与显示 | `src/utils/permissions/PermissionMode.ts` | `PERMISSION_MODE_CONFIG` |
| 模式循环逻辑 | `src/utils/permissions/getNextPermissionMode.ts` | `getNextPermissionMode()` |
| 模式切换副作用 | `src/utils/permissions/permissionSetup.ts` | `transitionPermissionMode()` |
| Plan Mode 上下文准备 | `src/utils/permissions/permissionSetup.ts` | `prepareContextForPlanMode()` |
| 完整权限决策流 | `src/utils/permissions/permissions.ts` | `hasPermissionsToUseToolInner()` |
| Bash 模式验证 | `src/tools/BashTool/modeValidation.ts` | `checkPermissionMode()` |
| Bash 只读命令白名单 | `src/tools/BashTool/readOnlyValidation.ts` | `COMMAND_ALLOWLIST` |
| Yolo 分类器 | `src/utils/permissions/yoloClassifier.ts` | `classifyYoloAction()` |
| Plan Mode 提示词 | `src/utils/messages.ts` | `getPlanModeV2Instructions()` |
| ExitPlanMode 工具 | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | `call()` |
| EnterPlanMode 工具 | `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` | `call()` |
| 退出 Plan 权限弹窗 | `src/components/permissions/ExitPlanModePermissionRequest/` | `buildPermissionUpdates()` |
| Auto Mode 危险规则剥离 | `src/utils/permissions/permissionSetup.ts` | `stripDangerousPermissionsForAutoMode()` |
| 连续拒绝保护 | `src/utils/permissions/denialTracking.ts` | `shouldFallbackToPrompting()` |
