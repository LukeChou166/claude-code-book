# 第1章 整体架构与启动 — 代码问答笔记

本文件记录阅读 `01-agent-architecture.md` 过程中对具体代码的问答，作为正文章节的补充。

---

## Q1：整个代码仓库的目录结构是怎样的？

仓库根目录分三个核心区域：

```
claude-code-sourcemap/
  ├── package/              # npm 发布包（含预编译二进制）
  │   └── vendor/
  │       ├── ripgrep/      # 6 个平台预编译的 rg 二进制
  │       └── audio-capture/
  ├── restored-src/         # 还原后的完整 TypeScript 源码
  │   └── src/              # 约 500+ 个文件
  └── book/                 # 本阅读笔记
```

`restored-src/src/` 下的一级目录对应架构的各个层次：

| 目录 | 层次 | 说明 |
|------|------|------|
| `main.tsx` | 入口 | CLI 入口，约 4700 行 |
| `query.ts` | 会话管理 | Agentic Loop 主循环，约 1730 行 |
| `QueryEngine.ts` | 会话管理 | SDK 容器，约 1300 行 |
| `Tool.ts` | 工具系统 | 工具接口定义，约 793 行 |
| `tools.ts` | 工具系统 | 工具列表装配，约 390 行 |
| `tools/` | 工具系统 | 30+ 个工具的具体实现 |
| `services/` | 基础设施 | API、MCP、压缩、权限等服务 |
| `utils/` | 基础设施 | 约 329 个工具函数文件 |
| `ink/` | UI | 自研终端渲染引擎，约 50 个文件 |
| `components/` | UI | React TUI 组件，约 144 个文件 |
| `tasks/` | 多智能体 | Agent 运行时（本地/远程/同进程） |
| `commands/` | CLI 命令 | `/slash` 命令实现，约 101 个文件 |
| `types/` | 类型定义 | 跨模块共享的 TypeScript 类型 |

---

## Q2：启动流程（`main.tsx`）中与 Agent 相关的部分是怎样的？

启动链路中与 Agent 能力直接相关的环节是**工具装配**和**记忆加载**，它们决定了 Agent "能做什么"和"知道什么"：

```
main.tsx 入口
    │
    ├── 解析命令行参数 (Commander.js)
    ├── init()                 — 基础配置与环境初始化
    ├── getTools(config)       — 按权限/Feature Flag 装配工具列表  ← Agent 能力边界
    ├── 连接 MCP 服务器        — 外部工具接入
    └── launchRepl() / headless — 启动 TUI 或 headless CLI
```

### 工具装配：getTools()

```typescript
// src/tools.ts 第 193-251 行
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool,
    WebSearchTool, AskUserQuestionTool, SkillTool,
    EnterPlanModeTool, ExitPlanModeV2Tool,
    // ant 内部专用
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    // Feature Flag 控制
    ...(SleepTool ? [SleepTool] : []),              // PROACTIVE/KAIROS
    ...(WorkflowTool ? [WorkflowTool] : []),         // WORKFLOW_SCRIPTS
    ...(SnipTool ? [SnipTool] : []),                 // HISTORY_SNIP
    // ...共 30+ 个工具
  ]
}
```

要点：
- Feature Flag 在这里的作用是**控制 Agent 的能力边界**：flag 未开启的工具不会进入工具列表，AI 从一开始就"不知道"这个能力存在
- MCP 服务器连接成功后，其暴露的工具会被适配成 `Tool` 接口合并进列表，对 Agent 而言与内置工具调用方式完全一致
- 最终用 `uniqBy(tools, t => t.name)` 去重，防止重名工具导致 AI 调用混乱

---

## Q3：核心 Agentic Loop（`query.ts`）的结构是什么？

`query.ts` 的核心函数是一个**异步生成器**：

```typescript
// src/query.ts 第 219-228 行
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

调用方通过 `for await (const event of query(...))` 消费流式事件。内部 `queryLoop()` 是真正的 while 循环：

```
queryLoop() while(true):
  ①  构建消息列表（附带系统提示、记忆文件）
      ↓
  ②  调用 Claude API（流式）
      ↓
  ③  接收 AI 回复（可能包含 tool_use 块）
      ↓
  ④  执行工具（partitionToolCalls → 并发/串行）
      ↓
  ⑤  把 tool_result 放回消息队列
      ↓
  ⑥  检查 stop_reason：
        end_turn    → 退出循环，返回 Terminal
        tool_use    → 回到 ②
        token 过多  → 触发 autoCompact，回到 ②
```

**循环状态对象**：每次迭代间共享的可变状态由 `State` 类型承载：

```typescript
// src/query.ts 第 204-217 行
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number    // max_output_tokens 错误恢复次数（最多 3 次）
  hasAttemptedReactiveCompact: boolean
  turnCount: number
  transition: Continue | undefined        // 上一次为何继续（调试用）
}
```

---

## Q4：工具系统（`Tool.ts` + `tools.ts`）如何组织？

### 工具接口关键方法

`ToolDef` 接口约 40 个方法/属性，核心如下：

| 方法/属性 | 作用 |
|-----------|------|
| `call()` | 工具执行主体 |
| `inputSchema` | Zod Schema，AI 输入参数验证 |
| `outputSchema` | Zod Schema，工具输出格式 |
| `checkPermissions()` | 权限检查，返回 allow/ask/deny |
| `validateInput()` | 参数合法性验证（路径存在等） |
| `isReadOnly()` | 是否只读（影响并发调度） |
| `isConcurrencySafe()` | 是否可与其他工具并发执行 |
| `renderToolUseMessage()` | TUI 展示调用过程 |
| `mapToolResultToToolResultBlockParam()` | 将执行结果转为 API 格式 |

**fail-safe 默认值**：`buildTool()` 工厂注入保守默认值，遵循"安全优先"原则：

```typescript
// src/Tool.ts 第 757+ 行
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // 默认不安全（最坏结果：变慢，不是竞态）
  isReadOnly: () => false,         // 默认会写（最坏结果：串行，不是数据损坏）
}
```

### 工具装配（`getAllBaseTools()`）

```typescript
// src/tools.ts 第 193-251 行
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    // 内置搜索工具（ant 嵌入版本使用 bfs/ugrep 替代，此两工具不包含）
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool,
    WebSearchTool, AskUserQuestionTool, SkillTool,
    EnterPlanModeTool, ExitPlanModeV2Tool,
    // ant 内部专用
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    // Feature Flag 控制
    ...(SleepTool ? [SleepTool] : []),              // PROACTIVE/KAIROS
    ...(WorkflowTool ? [WorkflowTool] : []),         // WORKFLOW_SCRIPTS
    ...(SnipTool ? [SnipTool] : []),                 // HISTORY_SNIP
    // ...共 30+ 个工具
  ]
}
```

### 工具并发调度（`toolOrchestration.ts`）

```
partitionToolCalls(toolUseMessages)
  → 分批：
      [只读工具批次] → runToolsConcurrently()  (最多 10 个并发，CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY)
      [写入工具批次] → runToolsSerially()       (串行，避免竞态)
  → 两批次之间串行，批次内部并发
```

`isReadOnly()=true` 的工具（GrepTool、GlobTool、FileReadTool）可并发执行，写操作（FileEditTool、BashTool）串行执行。

---

## Q5：权限系统的 5+2 种权限模式是什么？

```typescript
// src/types/permissions.ts 第 16-36 行
const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',       // 只自动接受文件编辑，其他操作仍需确认
  'bypassPermissions', // 完全绕过权限检查（危险！适用于完全可信的自动化环境）
  'default',           // 默认：写操作需用户确认
  'dontAsk',           // 全部自动允许（"我知道风险，不要再问我"）
  'plan',              // 计划模式：只读+规划，不执行任何写操作
] as const

// 内部模式（用户不可直接设置）：
// 'auto'   — 自动分类器模式（TRANSCRIPT_CLASSIFIER feature flag）
// 'bubble' — 子 Agent 冒泡权限给父 Agent
```

**权限规则来源与优先级**（高 → 低）：

```typescript
type PermissionRuleSource =
  | 'policySettings'   // 企业 MDM 策略（最高，用户无法覆盖）
  | 'cliArg'           // 命令行参数 --allowedTools
  | 'projectSettings'  // .claude/settings.json（项目级）
  | 'userSettings'     // ~/.claude/settings.json（用户级）
  | 'localSettings'    // .claude/settings.local.json（本地，不提交 git）
  | 'flagSettings'     // Feature Flag 控制
  | 'command'          // /slash 命令临时修改
  | 'session'          // 当次会话内（最低）
```

每次工具调用前执行三步检查：
1. **静态 deny 规则**：匹配 `alwaysDenyRules`，直接拒绝
2. **自动分类器**（`auto` 模式）：AI 模型判断操作安全性
3. **用户交互确认**：展示 UI 弹窗，等待用户 allow/deny/remember

---

## Q6：MCP 集成（`services/mcp/client.ts`）支持哪些传输方式？

MCP（Model Context Protocol）是工具扩展的标准协议，`client.ts` 约 3400 行，是整个 MCP 子系统核心。

**支持 4 种传输方式**：

```typescript
// src/services/mcp/client.ts 第 8-17 行
import { StdioClientTransport }          // stdio（本地子进程，最常用）
import { SSEClientTransport }            // SSE（旧版 HTTP 流）
import { StreamableHTTPClientTransport } // Streamable HTTP（MCP 新标准）
// + WebSocket（内部使用）
```

**关键功能模块**：

| 文件 | 功能 |
|------|------|
| `MCPConnectionManager.tsx` | React 组件，管理所有连接的生命周期 |
| `MCPTool.ts` | 将 MCP Server 工具适配为 Claude Code 的 Tool 接口 |
| `auth.ts` | MCP Server 的 OAuth 2.0 流程（PKCE） |
| `envExpansion.ts` | 展开 `${{ secrets.KEY }}` 和 1Password 秘密引用 |
| `elicitationHandler.ts` | MCP 2.0 新特性：Server 向用户请求额外输入 |
| `officialRegistry.ts` | Anthropic 认可的 MCP Server 目录 |
| `mcpOutputStorage.ts` | 大二进制内容（图片等）存磁盘，避免塞满上下文 |

---

## Q7：上下文压缩系统如何防止超长会话崩溃？

### 自动压缩触发阈值（`autoCompact.ts`）

```typescript
// src/services/compact/autoCompact.ts 第 62-80 行
const AUTOCOMPACT_BUFFER_TOKENS = 13_000
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000

getAutoCompactThreshold(model) =
  contextWindow(model) - MAX_OUTPUT_TOKENS_FOR_SUMMARY - 13_000
```

当 token 数超过阈值时触发压缩，防止 API 报 `prompt_too_long` 错误。

### 压缩流程（`compact.ts`）

```
compactConversation()
  ① 调用 Claude API，生成现有会话的摘要（最多 20,000 output tokens）
  ② 用摘要替换历史消息（插入 CompactBoundaryMessage 作为分界标记）
  ③ 恢复最近读取的文件内容（最多 5 个，POST_COMPACT_MAX_FILES_TO_RESTORE = 5）
  ④ 清理 tool_result 中的大内容（toolResultBudget）
```

### 多种压缩策略

| 策略 | 触发条件 | 文件 |
|------|----------|------|
| 自动压缩（proactive） | token 超过阈值 | `autoCompact.ts` |
| 响应式压缩（reactive） | API 报 `prompt_too_long` | `reactiveCompact.ts` |
| 微压缩（microcompact） | 单个工具结果太大 | `microCompact.ts` |
| 历史微截断（snip） | `HISTORY_SNIP` feature flag | `snipCompact.ts` |

**电路熔断**：连续压缩失败 3 次（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`）后停止重试，避免无效 API 调用浪费（背景数据：曾有会话连续失败 3272 次，每天浪费约 25 万次 API 调用）。

---

## Q8：多智能体系统（`AgentTool`）如何工作？

`AgentTool.tsx` 是"工具即 Agent"的核心实现：每个 AgentTool 调用都会启动一个独立的 AI 推理循环。

### 三种运行模式

```typescript
// src/tasks/ 目录
LocalAgentTask     // 本机子 Agent（最常用）
RemoteAgentTask    // 远程执行 Agent（teleport 跨机器）
InProcessTeammate  // 同进程内 Agent（Swarms 多 Agent 协作）
```

### Agent 隔离机制

| 机制 | 实现 | 目的 |
|------|------|------|
| 上下文隔离 | 独立的 `messages` 数组 | Agent 间不共享对话历史 |
| 权限继承 | 子 Agent 权限 ≤ 父 Agent | 防止权限提升攻击 |
| Worktree 隔离 | `createAgentWorktree()` | 避免多 Agent 写同一文件冲突 |
| 调用链追踪 | `QueryChainTracking { chainId, depth }` | 防止无限递归嵌套 |

### 内置 Agent 类型

```typescript
// src/tools/AgentTool/built-in/
generalPurposeAgent   // 全能执行者（默认）
exploreAgent          // 只读代码探索专家（无写操作工具）
```

用户还可在 `~/.claude/agents/` 或项目 `.claude/agents/` 目录放置自定义 Agent 定义文件，通过 `loadAgentsDir.ts` 动态发现加载。

### 自动后台化

```typescript
// src/tools/AgentTool/AgentTool.tsx 第 72-77 行
function getAutoBackgroundMs(): number {
  if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS) ||
      getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents', false)) {
    return 120_000;  // 2 分钟后自动转为后台任务
  }
  return 0;
}
```

长时间运行的 Agent 任务可以自动转为后台执行，用户可继续使用主对话。

---

## Q9：会话持久化（`sessionStorage.ts`）如何实现 rewind？

会话采用 **append-only JSONL 格式**持久化到磁盘：每次工具调用、AI 回复都追加一行 JSON，不修改历史记录。

```typescript
// src/utils/sessionStorage.ts 第 77+ 行
async function appendMessageToTranscript(message: Entry): Promise<void>
// 始终追加，从不覆盖已有行
// 崩溃只影响最后一行（可能不完整），之前的记录完整保留
```

**支持的操作**：

| 命令 | 文件 | 功能 |
|------|------|------|
| `/resume` | `src/commands/resume/` | 恢复历史会话 |
| `/rewind` | `src/commands/rewind/` | 回滚到任意时间点 |
| `/stats` | `src/commands/stats/` | 分析 token 用量、费用 |

**文件快照**（`FileHistorySnapshotMessage`）：每次 FileEditTool 执行前后，记录文件内容快照，`/rewind` 时用来恢复文件到对应状态。

---

## Q10：整体架构的关键设计模式总结

| 模式 | 实现 | 目的 |
|------|------|------|
| **AsyncGenerator 流水线** | `query()` 函数 | 背压控制 + 流式渲染 |
| **Bun DCE Feature Flag** | `feature('FLAG')` | 一份代码，多版本产物 |
| **lazySchema** | `lazySchema(() => z.object(...))` | 避免启动时 Schema 实例化开销 |
| **semanticNumber/Boolean** | Zod 包装器 | 允许 AI 用字符串传数字/布尔值 |
| **工具并发分批** | `partitionToolCalls()` | 只读并发，写入串行 |
| **append-only 日志** | `sessionStorage.ts` | 不可变历史，支持 rewind |
| **immutable PermissionContext** | `DeepImmutable<{...}>` | 防止工具执行污染权限状态 |
| **memoize 配置** | `getRipgrepConfig`, `countFilesRoundedRg` | 避免重复系统调用 |
| **fail-safe 默认值** | `buildTool()` 工厂 | 遗漏声明时保守降级，不引发数据损坏 |
| **Promise.allSettled** | 文件 stat 批量查询 | 单个失败不中断整批处理 |
