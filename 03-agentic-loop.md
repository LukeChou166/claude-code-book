# 第3章 Agentic Loop：智能体核心循环

> **前置知识**：第1章 整体架构与启动、第2章 QueryEngine  
> **关键文件**：`src/query.ts`（约1730行）、`src/services/tools/toolOrchestration.ts`、`src/services/tools/toolExecution.ts`

---

## 3.1 解决什么问题

这是整个项目最关键的一章。

**Agentic Loop**（智能体循环）指的是：AI 不只回答一次，而是**反复推理 → 行动 → 观察**，直到任务完成。

具体需要解决以下问题：

1. **工具调用的协调**：AI 说"我要同时读三个文件"，这三个读取应该并行还是串行？
2. **中断与恢复**：用户在工具执行中途按 Ctrl+C，如何优雅中止？
3. **错误处理**：工具执行失败时，应该告诉 AI，让 AI 决定如何继续，还是直接报错退出？
4. **Token 超限**：当上下文接近模型的 token 上限时，如何自动压缩而不丢失关键信息？
5. **流式模型响应**：Claude API 是流式的，如何在收到部分响应时就开始执行工具？
6. **Thinking 模式**：Claude 3.7+ 支持"扩展思考"，其 token 流有特殊规则，如何处理？
7. **降级模型**：当主模型调用失败时，自动切换备用模型并保持一致性。

---

## 3.2 业界的做法

**OpenAI Assistants API** 将 agentic loop 托管在服务端——客户端只需轮询 run 的状态。
- 优点：客户端代码简单
- 缺点：服务端黑盒，工具结果每次都要 round-trip 到服务端（增加延迟）；无法精细控制循环逻辑

**ReAct 框架**（Reasoning + Acting）通过 prompt 让 AI 输出 `Thought:`、`Action:`、`Observation:` 文本来"模拟"循环：
- 优点：不需要 Function Calling 支持，任何模型都可用
- 缺点：纯文本协议脆弱，AI 可能随时"偏离格式"；效率低（每个步骤都是一个完整 API 调用）

**LangChain AgentExecutor**：
- 优点：抽象层次高，易上手
- 缺点：并发工具执行支持差，错误恢复逻辑简单，不适合复杂生产场景

---

## 3.3 本项目的做法：本地驱动的流式循环

`query.ts` 中的 `queryLoop()` 实现了完全在**客户端驱动**的 Agentic Loop，约 1700 行代码。

### 核心结构

```typescript
// src/query.ts 第 241-310 行
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<...> {

  // 不可变参数（整个循环期间不变）
  const { systemPrompt, userContext, canUseTool, maxTurns, ... } = params

  // 可变跨迭代状态（用单一 state 对象管理）
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    autoCompactTracking: undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    turnCount: 1,
    pendingToolUseSummary: undefined,
    transition: undefined,   // ← 记录"上次为什么 continue"，主要用于测试断言
  }

  // 主循环
  while (true) {
    const { messages, toolUseContext, ... } = state

    // 1. 上下文准备（压缩、微压缩）
    // 2. 调用 Claude API（流式）
    // 3. 处理流式响应（提取 tool_use、助手文本）
    // 4. 执行工具
    // 5. 将工具结果追加到消息历史
    // 6. 决定：继续还是结束？
  }
}
```

### 关键设计：不可变参数 vs 可变状态

```typescript
// 不可变参数 ← 用 const 解构
const { systemPrompt, userContext, canUseTool, ... } = params

// 可变状态 ← 用 let state 管理
let state: State = { messages, toolUseContext, ... }

// 在 continue 点更新状态（整体替换，而非修改字段）
state = {
  ...state,
  messages: [...messages, ...newMessages],
  toolUseContext: updatedContext,
  transition: { reason: 'tool_use' },
}
continue  // 进入下一轮
```

这种设计避免了一个常见问题：在一次迭代中修改了下一次迭代需要读取的字段，导致顺序依赖 bug。通过整体替换 `state` 对象，每次迭代开始时拿到的是一致的状态快照。

---

## 3.4 工具并发执行

当 AI 返回多个 tool_use 块时，`toolOrchestration.ts` 负责协调执行：

### 分批策略

```typescript
// src/services/tools/toolOrchestration.ts 第 91-116 行
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc: Batch[], toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const isConcurrencySafe = tool?.isConcurrencySafe(parsedInput) ?? false

    // 连续的并发安全工具 → 合并为一批
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      // 新建一批
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

**示例**：AI 同时要求以下操作：
```
[FileRead("a.ts"), FileRead("b.ts"), FileRead("c.ts"), FileEdit("a.ts"), FileRead("d.ts")]
```

被分成三个批次：
| 批次 | 工具 | 执行方式 |
|------|------|----------|
| 批次1（并发安全） | FileRead(a), FileRead(b), FileRead(c) | **并发**执行 |
| 批次2（不安全） | FileEdit(a) | **串行**执行 |
| 批次3（并发安全） | FileRead(d) | **并发**执行 |

**优势**：FileEdit 完成后，FileRead(d) 读到的是最新版本的文件。如果全部并发，FileRead(d) 可能读到 FileEdit 之前的状态。

### 并发实现

```typescript
// src/services/tools/toolOrchestration.ts 第 152-177 行
async function* runToolsConcurrently(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void> {
  yield* all(
    toolUseMessages.map(async function* (toolUse) {
      toolUseContext.setInProgressToolUseIDs(prev =>
        new Set(prev).add(toolUse.id),
      )
      yield* runToolUse(toolUse, ...)
      markToolUseAsComplete(toolUseContext, toolUse.id)
    }),
    getMaxToolUseConcurrency(), // 默认 10，可通过环境变量配置
  )
}
```

`all()` 是一个辅助函数，将多个异步生成器并发运行，限制最大并发数为 10（可通过 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 环境变量调整）。

---

## 3.5 流式响应处理

API 响应是流式的，每个 chunk 可能包含：
- 文本 token（AI 的回复文字）
- tool_use 块的开始/内容/结束
- 思考（thinking）块

Claude Code 支持**流式工具执行**（Streaming Tool Execution）：不等 AI 回复完全结束，一旦收到完整的 tool_use 块，立即开始执行对应工具：

```typescript
// src/query.ts 第 561-568 行
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

**好处**：如果 AI 回复包含 3 个工具调用，传统做法是等 AI 全部说完再依次执行；流式执行允许在 AI 还在"说第 2 个工具"时，第 1 个工具已经开始执行了。

---

## 3.6 Thinking 模式的特殊规则

注释里有一段带有幽默感的"圣经"风格警告：

```typescript
// src/query.ts 第 151-163 行
/**
 * The rules of thinking are lengthy and fortuitous. They require plenty of thinking
 * of most long duration and deep meditation for a wizard to wrap one's noggin around.
 *
 * The rules follow:
 * 1. A message that contains a thinking or redacted_thinking block must be part of a query whose max_thinking_length > 0
 * 2. A thinking block may not be the last message in a block
 * 3. Thinking blocks must be preserved for the duration of an assistant trajectory (a single turn, or if that turn includes a tool_use block then also its subsequent tool_result and the following assistant message)
 *
 * Heed these rules well, young wizard. For they are the rules of thinking, and
 * the rules of thinking are the rules of the universe. If ye does not heed these
 * rules, ye will be punished with an entire day of debugging and hair pulling.
 */
```

Claude 的"扩展思考"功能在响应中产生两种特殊块：
- `thinking`：AI 的内部推理文字（可见）
- `redacted_thinking`：加密的内部推理（API 不透传内容，只透传占位符）

**规则 1**：包含 thinking 块的历史消息发回 API 时，必须开启 `max_thinking_length > 0`。否则 API 报错。

**规则 2**：thinking 块不能是消息内容的最后一个块（必须后面跟着文字或 tool_use）。

**规则 3**：在一个"助手轨迹"（tool_use + tool_result + 下一个 assistant response 的整体）期间，thinking 块必须被完整保留，不能被压缩删除。

违反这些规则不会有编译错误，API 会在运行时报错，而且错误信息往往不直观，调试极其痛苦——这就是为什么注释用如此夸张的语气来警示维护者。

---

## 3.7 max_output_tokens 恢复机制

当 Claude 生成的回复超过 `max_output_tokens` 限制时（比如写了特别长的代码），API 返回带有 `apiError: 'max_output_tokens'` 的消息。

Claude Code 不直接报错，而是实现了"恢复循环"：

```typescript
// src/query.ts 第 164-179 行
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

function isWithheldMaxOutputTokens(
  msg: Message | StreamEvent | undefined,
): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

**恢复过程**：
1. 检测到 `max_output_tokens` 错误
2. 把截断的部分回复保留在上下文中
3. 重新调用 API，告诉 Claude "你刚才回复被截断了，请继续"
4. 最多重试 3 次（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`）

**关键细节**：这个错误消息会被"扣留"（withheld），不立即发给 SDK 调用方。因为有些 SDK 消费者（比如 IDE 插件）在看到任何 `error` 字段时会直接终止会话——如果在恢复成功后才透出结果，消费者完全感知不到这次截断。

---

## 3.8 降级模型（Fallback Model）

当主模型（比如 `claude-opus-4-5`）调用失败时，自动切换到 `fallbackModel`（比如 `claude-sonnet-4-5`）。

```typescript
// src/query.ts（简化）
for await (const message of deps.callModel({
  model: currentModel,
  fallbackModel,
  onStreamingFallback: () => {
    streamingFallbackOccured = true
  },
  // ...
}))
```

降级触发时，之前已经收到的部分响应（可能包含 thinking 块）会被用"墓碑"（tombstone）标记为无效：

```typescript
if (streamingFallbackOccured) {
  // 把孤立的消息替换为墓碑
  for (const msg of assistantMessages) {
    yield { type: 'tombstone' as const, message: msg }
  }
  // 清空已收集的工具调用
  assistantMessages.length = 0
  toolUseBlocks.length = 0
}
```

这确保了降级后的对话状态是一致的——不会出现来自 opus 的 thinking 块与来自 sonnet 的 tool_use 块混在一起的情况（这会导致 API 报错，因为不同模型的 thinking 块格式不同）。

---

## 3.9 Stop Hooks：自定义循环终止逻辑

当 AI 决定结束（没有 tool_use，纯文本回复）时，Claude Code 会运行"Stop Hooks"：

```typescript
// src/query/stopHooks.ts（简化）
export async function handleStopHooks(
  messages: Message[],
  toolUseContext: ToolUseContext,
): Promise<'continue' | 'stop'> {
  // 运行用户配置的 stop hook 脚本
  const result = await executeStopHooks(messages, toolUseContext)
  
  if (result === 'continue') {
    // Stop hook 要求继续循环（比如说"测试还没通过"）
    return 'continue'
  }
  return 'stop'
}
```

Stop Hooks 让用户可以在 AI 想要结束时插入自定义检查，比如：
- "如果最后的 Bash 命令返回了非零 exit code，要求 AI 继续修复"
- "如果 TODO 列表还有未完成项，要求 AI 继续处理"

这是一个强大的扩展点：用户可以用 shell 脚本表达"任务完成的标准"，而不是靠 AI 自己判断。

---

## 3.10 终止条件总结

循环退出有以下情况：

| 情况 | Terminal 值 | 说明 |
|------|-------------|------|
| AI 无 tool_use 且 stop hooks 通过 | `{ reason: 'success' }` | 正常完成 |
| 达到最大轮次 | `{ reason: 'max_turns' }` | 防止无限循环 |
| token 超过阻塞限制 | `{ reason: 'blocking_limit' }` | 需要手动 /compact |
| 用户中止（Ctrl+C） | 抛出 `APIUserAbortError` | 优雅退出 |
| 不可恢复错误 | 抛出错误 | 传播给上层 |
| Stop Hook 不通过 | 继续循环（不终止） | 重新执行 |

---

## 3.11 优势与不足

### 优势

**客户端驱动**：工具在本地执行，没有额外的 round-trip 延迟。相比 OpenAI Assistants API，每次工具调用节省 200-500ms 网络延迟。

**智能并发执行**：只读工具并发，写操作串行，兼顾了速度和安全性。

**流式工具执行**：不等 AI 全部说完就开始执行工具，对于包含多个工具调用的复杂任务，能显著减少总耗时。

**max_output_tokens 自动恢复**：长代码生成不会因截断而失败，用户无感知。

**Stop Hooks 扩展点**：用户可以定义任务完成标准，Claude 的"我觉得我做完了"和用户的"测试通过了才算完"两种定义可以共存。

### 不足

**`queryLoop` 约 1700 行**，是整个项目最难读的单个函数。状态机逻辑、压缩逻辑、错误恢复逻辑、流式处理逻辑全部耦合在一个函数里。注释详尽但仍然难以完整理解所有的 `continue` 跳转目标。

**无超时保证**：如果工具执行无限阻塞（比如 `BashTool` 执行了一个死循环程序），理论上循环会永久阻塞。虽然 `BashTool` 有自己的超时机制，但这不是 loop 层面的通用保障。

**无重试策略**：工具执行失败时，循环直接把错误信息报告给 AI，让 AI 决定下一步。AI 可能选择重试（浪费时间）或放弃（任务未完成）。没有类似指数退避的智能重试策略。
