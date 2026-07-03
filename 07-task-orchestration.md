# 第7章 任务编排（Task Orchestration）

> **前置知识**：第4章 Agentic Loop、第5章工具系统  
> **关键文件**：`src/tools/AgentTool/`、`src/tasks/`、`src/tools/AgentTool/built-in/`

---

## 7.1 解决什么问题

单个 AI 面对复杂任务有两个根本瓶颈：

1. **上下文窗口有限**（Claude 约 200k token）：一个大型代码库可能有数百万行，单次无法全部读入
2. **串行执行太慢**：分析前端、分析后端、跑测试——如果都由同一个 Agent 顺序完成，速度无法接受

任务编排（Task Orchestration）要解决的就是：**如何把一个复杂任务分解、分配给多个执行单元，使它们并行或协作完成，最终汇总结果**。

Claude Code 的编排模型围绕三个关键抽象构建：

| 抽象 | 文件 | 作用 |
|------|------|------|
| `AgentTool` | `src/tools/AgentTool/` | 以"调用工具"的方式启动子 Agent |
| `Task` | `src/tasks/` | 描述子 Agent 的执行环境和生命周期 |
| 内置 Agent | `src/tools/AgentTool/built-in/` | 预定义的专业化子 Agent（Explore / General / Verification）|

---

## 7.2 编排层次：从单次工具调用到 Swarm

Claude Code 的任务编排有四个层次，从简单到复杂：

```
┌─────────────────────────────────────────────────┐
│  Layer 4: Swarm / Coordinator Mode               │
│  多个 Teammate 持续运行，共享团队上下文           │
├─────────────────────────────────────────────────┤
│  Layer 3: Plan Mode 两阶段编排                   │
│  先规划（只读），审批后再执行                     │
├─────────────────────────────────────────────────┤
│  Layer 2: AgentTool 单次子 Agent                 │
│  主 Agent 动态启动子 Agent，拿到结果后继续        │
├─────────────────────────────────────────────────┤
│  Layer 1: 并行工具调用                           │
│  同一个 Agent 同时发起多个工具调用（无子 Agent）  │
└─────────────────────────────────────────────────┘
```

本章重点讲 Layer 2~4（Layer 1 属于 Agentic Loop 章节的内容）。

---

## 7.3 核心设计：AgentTool——工具即编排

### 为什么不用专门的编排协议？

业界常见方案的共同问题：

- **CrewAI / LangGraph**：开发者预先定义 Agent 拓扑，AI 不透明、不能动态决策
- **AutoGen**：Agent 互相"聊天"，消息传递开销大，聊天记录很快填满上下文

Claude Code 的核心设计决策是：**将启动子 Agent 这件事本身，设计为一个普通工具**。

AI 不需要学习特殊的"编排协议"，只需要像调用 `Bash` 或 `GlobTool` 一样调用 `AgentTool`：

```typescript
// 主 Agent 的工具调用（从 AI 视角看，和调用其他工具没有区别）
AgentTool({
  task: "搜索所有包含 TODO 注释的文件，统计数量并列出关键的 TODO",
  subagent_type: "general-purpose",
})
```

**从外部看**：`AgentTool` 是一个接受字符串、返回字符串的普通工具  
**从内部看**：它启动了一个完整的 `queryLoop`，有自己的消息历史、工具集和系统提示

这个设计使得 AI 可以**自然地、动态地**决定是否需要子 Agent，而无需框架强制。

### AgentTool 的执行流程

```
主 Agent 调用 AgentTool(task)
    │
    ▼
创建 LocalAgentTask（独立 QueryEngine 实例）
    │
    ▼
子 Agent 进入自己的 queryLoop
    ├── 调用工具（受 disallowedTools 约束）
    ├── 产生消息历史（独立，不影响主 Agent）
    └── 完成时返回文字摘要
    │
    ▼
主 Agent 收到摘要，继续执行下一步
```

---

## 7.4 Task 抽象：描述执行环境

`tasks/` 目录定义了四种任务类型，覆盖不同的执行场景：

```typescript
// src/tasks/types.ts（简化）
type Task = {
  type: 'local-agent' | 'remote-agent' | 'in-process-teammate' | 'local-shell'
  id: string
  status: 'running' | 'completed' | 'failed' | 'cancelled'
}
```

| 任务类型 | 描述 | 适用场景 |
|----------|------|----------|
| `LocalAgentTask` | 当前机器运行的子 Agent | 大多数日常任务（默认） |
| `RemoteAgentTask` | 远程服务器上的 Agent | 需要特殊环境（GPU、隔离沙盒）的任务 |
| `InProcessTeammateTask` | 同进程持续运行的"队友" | Swarm 协作模式 |
| `LocalShellTask` | 后台 Shell 进程（非 AI） | 长时间运行的 Shell 命令 |

### LocalAgentTask：标准子 Agent

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx（简化）
export class LocalAgentTask {
  private queryEngine: QueryEngine

  constructor(config: LocalAgentTaskConfig) {
    // 每个子任务有独立的 QueryEngine 实例，完全隔离的会话状态
    this.queryEngine = new QueryEngine({
      ...config,
      customSystemPrompt: config.agentSystemPrompt,  // 子 Agent 的专用系统提示
    })
  }

  async *run(task: string): AsyncGenerator<SDKMessage> {
    yield* this.queryEngine.submitMessage(task)
  }
}
```

---

## 7.5 上下文隔离机制

子 Agent 的上下文隔离通过 `ToolUseContext` 实现：

```typescript
// src/Tool.ts 第 245-248 行
agentId?: AgentId  // 只有子 Agent 才有这个字段
agentType?: string // 子 Agent 的类型名
```

子 Agent 创建时，会构造一个独立的 `ToolUseContext`：

```typescript
const subagentContext: ToolUseContext = {
  ...parentContext,
  agentId: generateAgentId(),  // 独立的 Agent ID

  // 子 Agent 的 setAppState 是 no-op——不影响主 Agent 的 UI 状态
  setAppState: (_fn) => { /* 空实现 */ },

  // 但 setAppStateForTasks 仍可影响全局基础设施（到达根 AppState）
  setAppStateForTasks: parentContext.setAppStateForTasks,

  messages: [],  // 独立的消息历史
}
```

隔离的含义：
- 子 Agent 的工具调用和消息历史**不会出现**在主 Agent 的对话中
- 子 Agent **无法直接修改**主 Agent 的 UI 状态
- 但可通过 `setAppStateForTasks` 注册后台任务（这个通道穿透到根 AppState）

---

## 7.6 内置专业化 Agent

`tools/AgentTool/built-in/` 目录下预定义了三个 Agent，每个针对不同的编排场景：

### 7.6.1 Explore Agent：只读代码探索

```typescript
// src/tools/AgentTool/built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases...',

  disallowedTools: [
    AGENT_TOOL_NAME,         // 禁止启动子 Agent（防止无限递归）
    FILE_EDIT_TOOL_NAME,     // 禁止编辑文件
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],

  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 不需要项目规范（纯探索，不修改代码）
}
```

**设计决策**：

- `model: 'haiku'`（对外部用户）：探索任务是高频、快速的搜索，haiku 速度快约 2-3 倍、成本低约 20 倍
- `omitClaudeMd: true`：省去 CLAUDE.md 里的编码规范，节省 token，也避免规范对只读任务产生误导
- `disallowedTools` 包含 `AGENT_TOOL_NAME`：防止 Explore Agent 再启动子 Agent，避免指数级嵌套

### 7.6.2 General Purpose Agent：通用执行者

```typescript
// src/tools/AgentTool/built-in/generalPurposeAgent.ts
export const GENERAL_PURPOSE_AGENT: BuiltInAgentDefinition = {
  agentType: 'general-purpose',
  whenToUse: 'General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks.',
  tools: ['*'],  // 通配符：拥有和主 Agent 相同的工具集
  // model 字段故意省略，使用 getDefaultSubagentModel() 动态决定
  getSystemPrompt: getGeneralPurposeSystemPrompt,
}
```

`tools: ['*']` 通配符意味着通用 Agent 有完整能力——可以修改文件、运行命令、甚至再启动子 Agent。

### 7.6.3 Verification Agent：对抗性验证专家

这是三个内置 Agent 里**设计最复杂**的一个（系统提示约 130 行），专门在任务完成后试图"破坏"实现，输出 `VERDICT: PASS / FAIL / PARTIAL`。

```typescript
// src/tools/AgentTool/built-in/verificationAgent.ts
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  color: 'red',        // UI 显示红色（表示严格审查）
  background: true,    // 后台运行，不阻塞主 Agent 回复用户

  disallowedTools: [
    AGENT_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,   // 严格禁止修改项目文件
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],

  model: 'inherit',  // 继承主 Agent 模型等级（不降级到 haiku）

  // 实验性：每次工具调用前强制追加的提示，防止 Agent "忘记"自己是验证者
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or create files IN THE PROJECT DIRECTORY...',
}
```

**触发时机**（`whenToUse` 字段）：
> "Invoke after non-trivial tasks (3+ file edits, backend/API changes, infrastructure changes). Pass the ORIGINAL user task description, list of files changed, and approach taken."

即：AI 实现完功能后，**在回复用户之前**，主 Agent 调用 Verification Agent 做独立验证——类似自动化 Code Review。

**系统提示的核心设计思想**：

1. **破坏者思维，而非确认者思维**  
   > "Your job is not to confirm the implementation works — it's to **try to break it**."

2. **反规避条款**：明确列出 AI 容易犯的「自我欺骗」借口：
   - "The code looks correct based on my reading" → 读代码不是验证，必须**运行**
   - "The implementer's tests already pass" → 实现者也是 LLM，要独立验证
   - "This would take too long" → 不是你的决定

3. **按变更类型分层的验证策略**：

   | 变更类型 | 验证策略 |
   |----------|----------|
   | 前端变更 | 启动 dev server + 浏览器自动化截图点击 |
   | 后端 API | curl 验证响应 schema，不只看 status code |
   | 数据库迁移 | 必须测试 down 迁移（可逆性验证）|
   | 重构 | 公开 API surface 不能变化 |

4. **对抗性探针（Adversarial Probes）**：
   - 并发：并行请求命中同一资源（数据竞争）
   - 边界：0、-1、空字符串、MAX_INT、Unicode
   - 幂等性：同一突变请求发两次
   - 孤儿操作：引用不存在的 ID

5. **强制输出格式**：没有命令执行证据的 PASS 直接被拒绝

   ```
   ### Check: [验证的内容]
   **Command run:** [实际执行的命令]
   **Output observed:** [终端输出原文，禁止改写]
   **Result: PASS** 或 **FAIL**
   ```

   最终必须以 `VERDICT: PASS`、`VERDICT: FAIL` 或 `VERDICT: PARTIAL` 结束（精确字符串，供调用方解析）。

---

## 7.7 查询链跟踪（Query Chain Tracking）

编排链上的所有 Agent 通过 `QueryChainTracking` 关联：

```typescript
// src/Tool.ts
export type QueryChainTracking = {
  chainId: string  // 同一用户请求的所有 Agent 共享同一个 chainId
  depth: number    // 嵌套深度（主 Agent = 0，子 Agent = 1，子子 Agent = 2）
}
```

每次启动子 Agent 时递增 depth：

```typescript
// src/query.ts
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,  // 递增
    }
  : {
      chainId: deps.uuid(),  // 新调用链的起点
      depth: 0,
    }
```

**chainId** 用于日志关联：一次用户请求触发的所有 Agent 日志都带同一个 `chainId`，可以完整重建整条调用链。

**depth** 用于防止无限嵌套：超过阈值可报警或截断，防止 Agent 无休止地启动子 Agent。

---

## 7.8 Swarm 模式（InProcessTeammate）

Layer 4 的编排是 **Swarm 模式**：多个 Teammate 在同一进程中持续运行，共享团队上下文，由 Leader 调度任务。

与普通 `AgentTool`（一次性调用）的核心区别是：**Teammate 有持久化身份，空闲后可接受新任务，而不是完成就销毁**。

### TeammateIdentity：持久化身份

```typescript
// src/tasks/InProcessTeammateTask/types.ts
export type TeammateIdentity = {
  agentId: string           // "researcher@my-team"（name@teamName 格式）
  agentName: string         // "researcher"
  teamName: string          // 所属团队
  color?: string            // 终端输出颜色（便于区分多个 Teammate）
  planModeRequired: boolean // 是否要求 Plan Mode 审批后才执行
  parentSessionId: string   // Leader 的 session ID
}
```

`InProcessTeammateTaskState` 存储在 `AppState` 中，包含以下关键字段：

| 字段 | 说明 |
|------|------|
| `awaitingPlanApproval` | 是否正在等待用户审批执行计划 |
| `permissionMode` | 独立的权限模式（可通过 Shift+Tab 单独切换）|
| `isIdle` | 当前是否空闲，等待新任务 |
| `shutdownRequested` | 是否已请求优雅退出 |
| `onIdleCallbacks` | Idle 时的回调列表（Leader 用回调等待，而非轮询）|

### 消息上限与真实内存事故

```typescript
// UI 镜像的消息上限，仅用于 zoomed transcript 对话框
export const TEAMMATE_MESSAGES_UI_CAP = 50

export function appendCappedMessage<T>(prev: readonly T[] | undefined, item: T): T[] {
  if (prev === undefined || prev.length === 0) return [item]
  if (prev.length >= TEAMMATE_MESSAGES_UI_CAP) {
    const next = prev.slice(-(TEAMMATE_MESSAGES_UI_CAP - 1))
    next.push(item)
    return next
  }
  return [...prev, item]
}
```

这个 50 条上限来自真实的生产事故（代码注释中有完整记录）：

> 大型 Swarm 会话（whale session `9a990de8`）在 **2 分钟内启动了 292 个 Agent**，峰值 RSS 达 **36.8GB**。  
> 根因：`task.messages` 作为 AppState 的 UI 镜像，持有每条消息的第二份副本，每个并发 Agent 约占 **125MB**。
>
> 修复方案：`task.messages` 只保留最近 50 条（够 UI 展示），完整对话历史保存在 `inProcessRunner` 的本地 `allMessages` 数组和磁盘上的 agent transcript 文件中。

### Swarm 模式的关键能力

```
src/utils/swarm/
├── backends/
│   └── teammateModeSnapshot.ts   # 状态快照：序列化/恢复 Teammate 状态
├── reconnection.ts               # 断线重连：网络中断后恢复协同
└── teammatePromptAddendum.ts     # 系统提示补充：告知 Teammate 其身份和团队信息
```

- **持久化标识**：`agentId`（`name@team` 格式），重启后可被识别
- **Mailbox 通信**：通过 `teamContext.inProcessMailboxes` 传递消息，不依赖 AppState
- **独立权限审批**：每个 Teammate 独立处于 Plan Mode，Leader 等待各自 `awaitingPlanApproval` 清零
- **无轮询等待**：`onIdleCallbacks` 让 Leader 可以 `await idle()` 而不是定期轮询

### feature flag 控制

```typescript
// src/main.tsx
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

Swarm / Coordinator Mode 目前仅对 Anthropic 内部用户（`USER_TYPE === 'ant'`）开启，外部版本通过 Bun DCE 完全剔除。从 36.8GB 峰值 RSS 的教训来看，谨慎控制访问范围是正确的。

---

## 7.9 SkillTool：用户自定义编排单元

除了内置 Agent，用户可以在 `~/.claude/skills/` 目录下创建 `SKILL.md` 文件，定义自己的专业化 Agent。Claude Code 会自动发现并注册为可调用的工具（`SkillTool`）。

```
AgentTool（内置）         SkillTool（用户自定义）
├── Explore Agent         ├── 代码审查 Agent
├── General Agent         ├── 测试生成 Agent
└── Verification Agent    └── 文档生成 Agent
```

两者的执行机制相同，区别在于谁来定义系统提示和工具约束：
- `AgentTool` 内置 Agent：由 Claude Code 团队硬编码在 `built-in/` 目录
- `SkillTool`：由用户在 SKILL.md 中以自然语言描述，框架负责解析和注册

---

## 7.10 编排的权衡与局限

### 优势

**"工具即编排"极其自然**：AI 动态决定是否需要子 Agent，和决定是否调用 `Bash` 没有区别，无需学习新协议。

**上下文隔离**：子 Agent 的探索过程不污染主 Agent 的消息历史，主 Agent 只看到最终摘要，上下文保持简洁。

**专业化成本控制**：Explore Agent 用 haiku（快速便宜），Verification Agent 用 inherit（准确），不同场景用不同模型，降低总成本。

**查询链可追踪**：通过 `chainId + depth` 可以完整重建一次用户请求触发的所有 Agent 调用链，方便调试。

### 局限

**结果只能是文本摘要**：子 Agent 的工作成果通过文字传回主 Agent，大量结构化信息（文件路径列表、代码变更 diff）在转化为文字时可能丢失精度。

**没有框架级并行冲突检测**：两个子 Agent 并行修改同一文件时，主 Agent 需要自己处理合并冲突，框架不提供事务性保证。

**Swarm 模式的内存压力**：在高并发 Swarm 场景（2 分钟 292 个 Agent），峰值内存可达 36.8GB——这是目前最大的工程挑战，已通过消息上限缓解但未根本解决。
