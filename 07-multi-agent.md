# 第7章 多智能体系统：让 AI 协作

> **前置知识**：第4章 Agentic Loop、第5章工具系统  
> **关键文件**：`src/tools/AgentTool/`、`src/tasks/`、`src/tools/AgentTool/built-in/`

---

## 7.1 解决什么问题

单个 AI 的 context window 是有限的（Claude 约 200k token），而一个大型项目可能有数百万行代码。即使 context 够用，单个 AI 串行处理复杂任务也很慢。

多智能体系统要解决：
1. **任务并行化**：多个独立子任务被多个 Agent 同时处理（比如"同时分析前端代码和后端代码"）
2. **上下文隔离**：子 Agent 有独立的消息历史，避免子任务信息污染主 Agent 的上下文
3. **专业化分工**：Explore Agent 专注代码探索，General Agent 负责执行——各司其职
4. **递归任务分解**：主 Agent 把大任务分解为子任务，子 Agent 把子任务继续分解

---

## 7.2 业界的做法

**CrewAI**：Python 框架，定义 Role/Task/Crew，Agent 之间通过消息传递通信。需要开发者显式设计 Agent 拓扑图，对 AI 不透明（AI 不知道多 Agent 的存在）。

**AutoGen**：基于对话的多 Agent 框架，Agent 互相"聊天"。自然，但消息传递开销大，且聊天记录快速填满上下文。

**LangGraph**：用有向图定义 Agent 编排。灵活但需要开发者深度介入，不能让 AI 动态决定是否启动子 Agent。

以上方案的共同问题：需要开发者预先定义 Agent 拓扑，**AI 自身不能动态决策是否和如何使用子 Agent**。

---

## 7.3 本项目的做法：AgentTool——工具即 Agent

Claude Code 的核心创新在于：**将启动子 Agent 这件事本身，设计为一个普通工具（AgentTool）**。

AI 不需要学习特殊的"启动子 Agent"协议，只需要像调用其他工具一样调用 `AgentTool`：

```
// AI 的工具调用（主 Agent 视角）
AgentTool({
  task: "搜索所有包含 TODO 注释的文件，统计数量并列出关键的 TODO",
  // 子 Agent 自动使用 Explore 专用工具集
})
```

从外部看，AgentTool 就是一个工具，返回文字结果；从内部看，它启动了一个完整的 `queryLoop`，有自己的消息历史、工具集和系统提示。

---

## 7.4 内置 Agent 类型

`tools/AgentTool/built-in/` 目录下有两个预定义的 Agent：

### Explore Agent：只读代码探索专家

```typescript
// src/tools/AgentTool/built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases. Use this when you need to quickly find files by patterns, search code for keywords, or answer questions about the codebase. Specify thoroughness: "quick", "medium", or "very thorough".',
  
  // 明确禁止的工具（防止 Explore Agent 意外修改文件）
  disallowedTools: [
    AGENT_TOOL_NAME,         // 不能启动子 Agent（避免无限递归）
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,     // 不能编辑文件
    FILE_WRITE_TOOL_NAME,    // 不能写文件
    NOTEBOOK_EDIT_TOOL_NAME, // 不能编辑 Notebook
  ],
  
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku', // 外部用户用 haiku（速度快、成本低）
  omitClaudeMd: true,  // 不需要项目记忆（纯探索任务）
  
  getSystemPrompt: () => `You are a file search specialist for Claude Code.
=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
Your role is EXCLUSIVELY to search and analyze existing code.
You MUST NOT create, modify, or delete any files.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

NOTE: Make efficient use of tools — spawn multiple parallel tool calls for grepping and reading files.`
}
```

几个关键设计决策：

**使用 haiku 模型**（对外部用户）：探索任务通常是快速搜索，不需要 opus 的深度推理能力。haiku 速度快（约 2-3 倍）、成本低（约 20 倍便宜），非常适合高频的代码探索子任务。

**`omitClaudeMd: true`**：Explore Agent 不需要 CLAUDE.md 里的项目规范（"不要用 any 类型"、"提交前必须运行 lint"），那些是给修改代码的 Agent 看的。省略它节省了 token，也避免了误导。

**禁止子 Agent 嵌套**：`disallowedTools` 里包含 `AGENT_TOOL_NAME`，防止 Explore Agent 再启动子 Agent，避免无限递归。

### General Purpose Agent：全能执行者

```typescript
// src/tools/AgentTool/built-in/generalPurposeAgent.ts
export const GENERAL_PURPOSE_AGENT: BuiltInAgentDefinition = {
  agentType: 'general-purpose',
  whenToUse: 'General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks.',
  tools: ['*'],  // 拥有所有工具（通配符）
  
  getSystemPrompt: () => `You are an agent for Claude Code.
Given the user's message, use the tools available to complete the task.
Complete the task fully—don't gold-plate, but don't leave it half-done.

Guidelines:
- NEVER create files unless absolutely necessary
- NEVER proactively create documentation files (*.md) or README files
- When complete, respond with a concise report of what was done`
}
```

注意 `tools: ['*']` 通配符，通用 Agent 拥有和主 Agent 相同的工具集。

---

## 7.5 上下文隔离机制

子 Agent 的上下文隔离通过 `ToolUseContext` 实现：

```typescript
// src/Tool.ts 第 245-248 行
agentId?: AgentId  // 只有子 Agent 才有这个字段
agentType?: string // 子 Agent 的类型名
```

子 Agent 创建时，会创建一个独立的 `ToolUseContext`：

```typescript
// 子 Agent 上下文的关键差异
const subagentContext: ToolUseContext = {
  ...parentContext,
  agentId: generateAgentId(),  // 独立的 Agent ID
  
  // 子 Agent 的 setAppState 默认是 no-op（不影响主 Agent 的 UI 状态）
  setAppState: (_fn) => {
    // 空实现：子 Agent 的状态变化不影响主界面
  },
  
  // 但 setAppStateForTasks 仍然可以影响全局基础设施
  setAppStateForTasks: parentContext.setAppStateForTasks,
  
  messages: [],  // 独立的消息历史
}
```

这意味着：
- 子 Agent 的工具调用、消息历史不会出现在主 Agent 的对话中
- 子 Agent 无法直接修改主 Agent 的 UI 状态（比如更新文件缓存）
- 但子 Agent 可以通过 `setAppStateForTasks` 注册后台任务（这个通道到达根 AppState）

---

## 7.6 任务类型：Task 抽象

`tasks/` 目录下有多种任务类型，表示不同执行环境的 Agent：

```typescript
// src/tasks/types.ts（简化）
type Task = {
  type: 'local-agent' | 'remote-agent' | 'in-process-teammate' | 'local-shell'
  id: string
  status: 'running' | 'completed' | 'failed' | 'cancelled'
  // ...
}
```

| 任务类型 | 描述 | 适用场景 |
|----------|------|----------|
| `LocalAgentTask` | 在当前机器/进程中运行的子 Agent | 大多数日常任务 |
| `RemoteAgentTask` | 通过网络运行在远程服务器的 Agent | 需要特殊环境（GPU、沙盒）的任务 |
| `InProcessTeammateTask` | 在同一进程中作为"队友"运行 | Swarm 模式的协同 Agent |
| `LocalShellTask` | 后台 Shell 进程（不是 AI Agent） | 执行长时间 Shell 命令 |

### LocalAgentTask

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx（简化）
export class LocalAgentTask {
  private queryEngine: QueryEngine
  private status: TaskStatus = 'running'
  
  constructor(config: LocalAgentTaskConfig) {
    // 为子 Agent 创建独立的 QueryEngine 实例
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

每个子任务有独立的 QueryEngine 实例，完全隔离的会话状态。

---

## 7.7 查询链跟踪（Query Chain Tracking）

```typescript
// src/Tool.ts 第 90-93 行
export type QueryChainTracking = {
  chainId: string  // 同一用户请求的所有 Agent 共享
  depth: number    // 嵌套深度（主 Agent = 0，子 Agent = 1，子子 Agent = 2）
}
```

**chainId** 用于日志关联：主 Agent 和所有子 Agent 的日志都带同一个 chainId，可以通过这个 ID 将整个调用链的日志串联起来，方便调试。

**depth** 用于防止无限嵌套：如果嵌套深度超过阈值，可以报警或截断，防止 Agent 无休止地启动子 Agent。

在 `query.ts` 中，每次循环迭代都会递增 depth：

```typescript
// src/query.ts 第 347-355 行
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,  // 递增嵌套深度
    }
  : {
      chainId: deps.uuid(),  // 新的调用链
      depth: 0,
    }
```

---

## 7.8 Swarm 模式（多主机 Agent 集群）

代码中存在对 Swarm 模式的暗示：

```typescript
// src/main.tsx（feature flag）
const coordinatorModeModule = feature('COORDINATOR_MODE') 
  ? require('./coordinator/coordinatorMode.js') 
  : null
```

```typescript
// src/utils/swarm/ 目录结构
// swarm/
//   backends/
//     teammateModeSnapshot.ts  // Teammate 模式快照
//   reconnection.ts            // 重连逻辑
//   teammatePromptAddendum.ts  // Teammate 系统提示补充
```

这是一个更高级的功能：在 Swarm 模式下，多个 Claude Code 实例（可能在不同机器上）协同工作，由一个 Coordinator 调度任务。每个实例是一个"Teammate"，共享任务状态和上下文。

这个功能目前仅对 Anthropic 内部用户（`ant`）开启，在外部版本中通过 Bun DCE 完全剔除。

---

## 7.9 Agent 与 Skill 的关系

`SkillTool` 是另一种形式的"封装工具集"：

```typescript
// Skill 是用户在 skills/ 目录下定义的 Agent
// 与 AgentTool 不同：
// - AgentTool 是内置类型（Explore/GeneralPurpose）
// - SkillTool 是用户自定义的专业化 Agent（比如"代码审查 Agent"、"测试生成 Agent"）
```

用户可以在 `~/.claude/skills/` 目录下创建 SKILL.md 文件，定义自己的专业化 Agent，Claude Code 会自动发现并注册为可用的工具。

---

## 7.10 优势与不足

### 优势

**"工具即 Agent"的设计极其优雅**：AI 不需要学习特殊的"启动子 Agent"协议，只是调用一个工具。这使得 AI 可以自然地、动态地决定是否需要子 Agent，而不是被框架强制。

**上下文隔离**：子 Agent 的操作不污染主 Agent 的消息历史，主 Agent 只看到子 Agent 的最终结果摘要，而不是子 Agent 探索过程中的所有中间步骤。这让主 Agent 的上下文保持简洁。

**专业化分工**：Explore Agent 用 haiku 模型（便宜、快速），主 Agent 用 opus 模型（强大）。这种差异化使用减少了总成本。

**查询链追踪**：通过 chainId 和 depth，可以完整重建一次用户请求触发的所有 Agent 调用链，方便调试和性能分析。

### 不足

**目前没有 Agent 间直接通信机制**：只有主-子的层级关系，没有对等（peer-to-peer）的 Agent 通信。如果需要多个 Agent 协同完成一个任务（比如一个 Agent 负责前端，另一个负责后端），只能通过主 Agent 中转。

**子 Agent 的结果是文本摘要**：子 Agent 的工作成果通过文字摘要传回主 Agent。大量结构化信息（文件路径、代码变更）在转化为文字时可能丢失精度。更好的方案可能是通过结构化数据格式（JSON）传递结果。

**并行子 Agent 的状态合并**：当两个子 Agent 并行修改了同一个文件，主 Agent 需要手动处理合并冲突（AI 不一定能正确处理），这是多 Agent 并行执行的根本挑战，当前没有框架级别的解决方案。
