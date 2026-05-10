# 第3章 QueryEngine：对话会话引擎

> **前置知识**：第1章整体架构、第2章启动流程  
> **关键文件**：`src/QueryEngine.ts`（约1300行）、`src/query.ts`、`src/types/message.ts`

---

## 3.1 解决什么问题

有了 AI 模型 API，要构建编程助手，最简单的做法是：
1. 收到用户消息
2. 调用 API，传入消息历史
3. 显示回复

但这个简单模式无法处理：

- **工具调用**：AI 说"我要读这个文件"，谁来真正执行读取？执行完后结果怎么送回 AI？
- **流式输出**：API 是流式的，如何边流边显示，同时还能让外部程序监听？
- **会话状态**：多轮对话中，消息历史如何维护？谁来决定何时压缩？
- **子智能体**：当主 AI 启动子 Agent 处理子任务，状态如何隔离？
- **SDK 对外暴露**：外部程序（IDE 插件、CI 工具）如何嵌入 Claude Code 作为库使用？
- **成本追踪**：如何统计每次调用消耗的 token？

`QueryEngine` 类就是解决这些问题的核心会话容器。

---

## 3.2 业界的做法

**LangChain** 的方式：
```python
# LangChain 风格（示意）
chain = LLMChain(llm=claude, memory=ConversationMemory())
response = chain.run("帮我修 bug")
```
优点：简单直观。
缺点：`Memory` 对象是独立的，与 Chain 松耦合，多实例时容易状态不同步；流式支持是后来加的，API 不统一。

**OpenAI Assistants API** 的方式：
```python
# OpenAI 风格（示意）
thread = client.beta.threads.create()
run = client.beta.threads.runs.create(thread_id=thread.id, assistant_id=...)
# 轮询等待
while run.status != 'completed':
    run = client.beta.threads.runs.retrieve(run.id)
```
优点：服务端托管状态，客户端无需管理。
缺点：黑盒，延迟高（每次工具结果都要 round-trip），无法自定义循环逻辑。

**Claude Code 的 QueryEngine** 介于两者之间：
- 客户端托管状态（可自定义、可调试）
- 流式异步生成器（低延迟、可嵌入）

---

## 3.3 QueryEngine 类的设计

### 一个 QueryEngine 对应一次对话

```typescript
// src/QueryEngine.ts 第 175-207 行（注释+代码）

/**
 * QueryEngine owns the query lifecycle and session state for a conversation.
 * It extracts the core logic from ask() into a standalone class that can be
 * used by both the headless/SDK path and (in a future phase) the REPL.
 *
 * One QueryEngine per conversation. Each submitMessage() call starts a new
 * turn within the same conversation. State (messages, file cache, usage, etc.)
 * persists across turns.
 */
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]         // 完整消息历史
  private abortController: AbortController   // 用于中止进行中的请求
  private permissionDenials: SDKPermissionDenial[]  // 被拒绝的权限记录
  private totalUsage: NonNullableUsage        // 累计 token 用量
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache       // 文件读取缓存
  private discoveredSkillNames = new Set<string>()  // 本轮发现的技能名
  private loadedNestedMemoryPaths = new Set<string>() // 已加载的 CLAUDE.md 路径

  constructor(config: QueryEngineConfig) {
    this.config = config
    this.mutableMessages = config.initialMessages ?? []
    this.abortController = config.abortController ?? createAbortController()
    this.permissionDenials = []
    this.readFileState = config.readFileCache
    this.totalUsage = EMPTY_USAGE
  }
  // ...
}
```

注意 `mutableMessages` 是私有的，跨多次 `submitMessage()` 调用持续存在。这是与无状态的 `query.ts` 函数最大的区别：QueryEngine **有状态**，`query.ts` **无状态**。

### QueryEngineConfig：配置项汇总

```typescript
// src/QueryEngine.ts 第 130-173 行
export type QueryEngineConfig = {
  cwd: string                         // 工作目录
  tools: Tools                        // 可用工具列表
  commands: Command[]                 // 可用命令列表
  mcpClients: MCPServerConnection[]   // MCP 服务器连接
  agents: AgentDefinition[]           // 可用的 Agent 类型
  canUseTool: CanUseToolFn            // 权限检查函数
  getAppState: () => AppState         // 读取全局 UI 状态
  setAppState: (f: (prev: AppState) => AppState) => void  // 更新 UI 状态
  initialMessages?: Message[]         // 用于恢复会话
  readFileCache: FileStateCache       // 文件内容缓存（LRU）
  customSystemPrompt?: string         // 完全替换默认系统提示
  appendSystemPrompt?: string         // 追加到默认系统提示后
  userSpecifiedModel?: string         // 用户指定的模型
  fallbackModel?: string              // 降级模型
  thinkingConfig?: ThinkingConfig     // 扩展思考配置
  maxTurns?: number                   // 最大轮次（防无限循环）
  maxBudgetUsd?: number               // 最大费用限制
  taskBudget?: { total: number }      // 任务 token 预算
  jsonSchema?: Record<string, unknown> // 强制结构化输出
  // ...
}
```

这个配置类型体现了 Claude Code 的设计哲学：**可配置性优于约定**。每个重要的行为都可以在实例化时定制，而不是硬编码。

---

## 3.4 submitMessage：一次轮次的生命周期

`submitMessage()` 是 SDK 对外暴露的核心方法，返回异步迭代器：

```typescript
// src/QueryEngine.ts 第 209-212 行
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown>
```

一次 `submitMessage()` 调用的内部流程：

```
submitMessage("帮我修 bug")
    │
    ├─ 1. 清理本轮状态
    │      this.discoveredSkillNames.clear()
    │      setCwd(cwd)
    │
    ├─ 2. 构建 ToolUseContext（工具执行上下文）
    │      {
    │        options: { tools, commands, mcpClients, ... },
    │        abortController,
    │        readFileState,
    │        getAppState,
    │        setAppState,
    │        messages: this.mutableMessages,
    │        setInProgressToolUseIDs,
    │        setResponseLength,
    │        ...
    │      }
    │
    ├─ 3. 构建系统提示
    │      fetchSystemPromptParts(cwd, tools, ...)
    │      + loadMemoryPrompt()   // CLAUDE.md
    │      + userContext          // 动态上下文（目录、git status等）
    │
    ├─ 4. 处理用户输入
    │      processUserInput()     // 解析斜杠命令、文件引用等
    │
    ├─ 5. 调用 query() 驱动 Agentic Loop
    │      for await (const event of query({
    │        messages: this.mutableMessages,
    │        systemPrompt,
    │        toolUseContext,
    │        ...
    │      })) {
    │        // 转换 event 为 SDKMessage 并 yield 给调用方
    │        yield toSDKMessage(event)
    │        // 更新 mutableMessages
    │        this.mutableMessages = updateMessages(event)
    │      }
    │
    └─ 6. 会话持久化
           flushSessionStorage()  // 写入磁盘
           recordTranscript()     // 记录对话日志
```

### 包装 canUseTool 追踪权限拒绝

```typescript
// src/QueryEngine.ts 第 243-260 行（简化）
// Wrap canUseTool to track permission denials
const wrappedCanUseTool: CanUseToolFn = async (
  tool, input, toolUseContext, assistantMessage, toolUseID
) => {
  const result = await canUseTool(tool, input, toolUseContext, ...)
  if (result.behavior === 'deny') {
    this.permissionDenials.push({
      toolName: tool.name,
      toolInput: input,
      // ...
    })
  }
  return result
}
```

这让 SDK 调用方可以查询哪些工具调用被权限系统拒绝了，方便调试。

---

## 3.5 消息类型系统

`types/message.ts` 定义了完整的消息 union 类型：

```typescript
// 完整的 Message 联合类型
type Message =
  | UserMessage           // 用户输入
  | AssistantMessage      // AI 回复（可含 tool_use 块）
  | AttachmentMessage     // 文件内容附件（CLAUDE.md、图片等）
  | SystemMessage         // 系统消息（用于 UI 展示，不发给 API）
  | ProgressMessage       // 工具执行进度（不发给 API）
  | TombstoneMessage      // 被删除消息的占位符（用于 token 追踪）
  | ToolUseSummaryMessage // 工具调用的摘要（压缩后）
  | SystemCompactBoundaryMessage  // 压缩边界标记
```

这里有几种特殊的消息类型值得深入说明：

**TombstoneMessage**：当某条消息被从上下文中移除（压缩）时，用一个"墓碑"代替它，保存原来消息的 token 数量。这样即使消息内容消失了，token 统计仍然准确。

```typescript
type TombstoneMessage = {
  type: 'tombstone'
  uuid: string
  tokenCount: number  // 被替换的原始消息的 token 数
}
```

**SystemCompactBoundaryMessage**：标记一次压缩操作发生的位置，是一个"审计日志"。从这个标记之前的所有消息都已经被摘要替代；这个标记之后的消息是最新的完整消息。

`getMessagesAfterCompactBoundary()` 函数利用这个标记，快速获取"上次压缩之后的"消息列表，发送给 API。

---

## 3.6 QueryEngine 与 query.ts 的关系

这两个模块的分工非常清晰：

```
QueryEngine.ts（状态层）
    ├── 持有 mutableMessages[]
    ├── 持有 totalUsage（token 统计）
    ├── 持有 readFileState（文件缓存）
    ├── 提供 SDK 接口（submitMessage, getMessages, getUsage...）
    ├── 负责构建 systemPrompt
    ├── 负责会话持久化
    └── 调用 query.ts
           │
           ▼
    query.ts（执行层）
         ├── 实现 Agentic Loop（while(true) 循环）
         ├── 管理单次用户请求中的多轮 AI 交互
         ├── 驱动工具调用和结果反馈
         ├── 处理 token 超限、自动压缩
         └── 无跨请求状态（每次调用都是全新的）
```

**关键设计原则**：`query.ts` 是**无状态的纯函数（生成器）**，给定相同的输入，产生相同的输出流。这使得：
1. `query.ts` 可以独立测试，无需构造完整的 QueryEngine
2. 子 Agent 可以复用 `query.ts`，只是传入不同的 `ToolUseContext`
3. 状态的所有权明确（在 QueryEngine 中），不散落在多处

---

## 3.7 SDK 使用示例

QueryEngine 被设计为可以被外部程序嵌入。以下是 SDK 模式的使用方式（简化示意）：

```typescript
import { QueryEngine } from 'claude-code'

const engine = new QueryEngine({
  cwd: process.cwd(),
  tools: getDefaultTools(),
  commands: [],
  mcpClients: [],
  agents: [],
  canUseTool: async () => ({ behavior: 'allow' }),
  getAppState: () => initialAppState,
  setAppState: (fn) => { /* 更新状态 */ },
  readFileCache: createFileStateCache(),
})

// 发送消息并流式接收结果
for await (const event of engine.submitMessage("帮我写一个排序算法")) {
  if (event.type === 'assistant') {
    // 渐进显示 AI 回复
    process.stdout.write(extractText(event.message.content))
  } else if (event.type === 'tool_result') {
    // 显示工具执行结果
    console.log(`[工具: ${event.toolName}]`, event.result)
  }
}

// 查询统计
console.log('Token 用量:', engine.getUsage())
```

---

## 3.8 token 追踪与成本管理

```typescript
// src/cost-tracker.ts（简化）
let totalUsage = {
  input_tokens: 0,
  output_tokens: 0,
  cache_read_input_tokens: 0,
  cache_creation_input_tokens: 0,
}

export function updateUsage(usage: Usage): void {
  totalUsage.input_tokens += usage.input_tokens
  totalUsage.output_tokens += usage.output_tokens
  // ...
}

export function getTotalCost(): number {
  const inputCost = totalUsage.input_tokens * INPUT_PRICE_PER_TOKEN
  const outputCost = totalUsage.output_tokens * OUTPUT_PRICE_PER_TOKEN
  const cacheReadCost = totalUsage.cache_read_input_tokens * CACHE_READ_PRICE_PER_TOKEN
  const cacheCreationCost = totalUsage.cache_creation_input_tokens * CACHE_CREATION_PRICE_PER_TOKEN
  return inputCost + outputCost + cacheReadCost + cacheCreationCost
}
```

特别注意：Claude API 支持 **Prompt Cache**（提示词缓存）。如果系统提示相同，第二次调用时缓存命中，`cache_read_input_tokens` 的价格远低于普通 `input_tokens`（约 10 倍便宜）。Claude Code 的代码会追踪这四种不同价格的 token 类型，计算出真实的 API 费用。

---

## 3.9 优势与不足

### 优势

**SDK/REPL 统一架构**：QueryEngine 既可以在 REPL（交互式终端）中使用，也可以以 SDK 模式嵌入外部程序，实现了"一套核心逻辑，两种使用方式"。

**状态/执行分离**：QueryEngine 管理状态，query.ts 执行逻辑，边界清晰，各自可独立测试。

**精细的 token 追踪**：区分 input/output/cache_read/cache_creation 四种 token 类型，成本计算准确。

**可恢复的会话**：通过 `initialMessages` 可以恢复历史会话，配合 `sessionStorage` 实现会话持久化。

### 不足

**QueryEngine.ts 有约 1300 行代码**，承担了太多职责：状态管理、SDK 适配、成本追踪、会话持久化、权限拦截。是一个轻度的"上帝类"（God Class）。

**ToolUseContext 字段过多**：`Tool.ts` 中 `ToolUseContext` 类型定义约 150 行，包含大量可选字段（`?:`），不同使用场景下部分字段会缺失。这种"隐式约定"（某些场景下某些字段必须存在）很难从类型系统中表达清楚，容易在新增使用场景时出错。

更好的做法可能是将 `ToolUseContext` 拆分为多个更小的接口，用组合而非单一大接口来表达不同场景的需求。
