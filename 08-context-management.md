# 第8章 上下文管理与自动压缩

> **前置知识**：第4章 Agentic Loop、第3章 QueryEngine  
> **关键文件**：`src/services/compact/autoCompact.ts`、`src/services/compact/compact.ts`、`src/utils/tokens.ts`

---

## 8.1 解决什么问题

Claude 的上下文窗口是有限的（约 200k token）。一个复杂的编程任务可能需要：
- 读取数十个文件（每个可能 2,000-10,000 token）
- 多轮工具调用（每次工具结果都加入历史）
- AI 长篇分析（每次回复 2,000-5,000 token）

经过 20-30 轮对话，token 使用量可能达到 150k+，接近上限。然后 API 报 `prompt_too_long` 错误，整个会话崩溃——这是用户无法接受的体验。

更微妙的问题是：
- 如果简单地**截断**历史，关键需求（"第 1 条消息里用户要求的目标"）可能被删除
- 如果用 AI **摘要**，摘要不准确会导致后续决策错误
- 如果**完全不管**，token 用完了只能重新开始会话

---

## 8.2 业界的做法

| 方案 | 代表 | 优点 | 缺点 |
|------|------|------|------|
| 简单截断（固定窗口） | 大多数早期 chatbot | 实现简单 | 丢失关键早期上下文 |
| 滑动窗口 | 一些 Agent 框架 | 保留最新内容 | 同样丢失关键早期内容 |
| 摘要 + 最新 N 轮 | LangChain ConversationSummaryMemory | 保留早期要点 | 摘要失真，且需要额外 API 调用 |
| RAG 检索 | 许多 LLM 应用 | 理论上无限历史 | 对编程精确性要求高，向量检索不可靠 |

---

## 8.3 本项目的多层压缩策略

Claude Code 实现了一个**多层防线**的上下文管理体系：

```
防线 1：工具结果大小限制（maxResultSizeChars）
    ↓ 如果仍然太大
防线 2：微压缩（MicroCompact）
    ↓ 如果仍然太大
防线 3：工具结果预算（Tool Result Budget）
    ↓ 如果仍然太大  
防线 4：自动压缩（AutoCompact）
    ↓ 如果压缩失败或被禁用
防线 5：阻塞限制（Blocking Limit）
    └── 显示"上下文已满，请使用 /compact"提示
```

---

## 8.4 防线 1：工具结果大小限制

在第5章中已经介绍：每个工具有 `maxResultSizeChars` 属性，超过阈值的结果写入临时文件。这是"在问题发生前"的预防。

`GrepTool` 的 `head_limit` 默认 250 行，防止大型代码库搜索产生几万行结果。

---

## 8.5 防线 2：微压缩（MicroCompact）

微压缩处理**特别大的单个工具结果**：当某个工具调用的结果本身非常大（比如读取了一个 10,000 行的文件），微压缩会把它替换为更简洁的版本，同时保留关键信息。

```typescript
// src/query.ts 第 412-426 行
queryCheckpoint('query_microcompact_start')
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages
// pendingCacheEdits: 对于缓存微压缩，
// 推迟边界消息直到收到 API 响应（这样能获得真实的 cache_deleted_input_tokens）
const pendingCacheEdits = feature('CACHED_MICROCOMPACT')
  ? microcompactResult.compactionInfo?.pendingCacheEdits
  : undefined
queryCheckpoint('query_microcompact_end')
```

微压缩的特点是**轻量**：它不调用 Claude API，而是通过规则（比如只保留文件的前 N 行 + 后 N 行）来压缩，快速且无额外成本。

---

## 8.6 防线 3：工具结果预算（Tool Result Budget）

```typescript
// src/query.ts 第 369-394 行
// 在微压缩之前，强制执行每条消息的工具结果大小预算
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements
    ? records => void recordContentReplacement(records, toolUseContext.agentId)
    : undefined,
  new Set(
    toolUseContext.options.tools
      .filter(t => !Number.isFinite(t.maxResultSizeChars))
      .map(t => t.name),
  ),
)
```

这一层对**聚合**的工具结果大小进行预算控制：即使每个单独的结果都在限额内，如果所有工具结果加起来太大，也会触发整体压缩，将最旧的工具结果替换为引用（"结果已保存到文件"）。

---

## 8.7 防线 4：自动压缩（AutoCompact）—— 最重要的防线

AutoCompact 是整个上下文管理体系的核心，它在 token 使用量接近阈值时，自动调用 Claude 生成一个**会话摘要**，然后用摘要替换旧历史。

### 触发条件

```typescript
// src/services/compact/autoCompact.ts 第 62-91 行
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  // 保留 13,000 token 给摘要生成时的新输出
  const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
  return autocompactThreshold
}
```

对于 200k 上下文的 claude-opus-4-5：
- 有效上下文 ≈ 200,000 - 20,000（为模型输出保留）= 180,000 token
- 自动压缩触发点 ≈ 180,000 - 13,000 = 167,000 token
- 警告阈值 ≈ 167,000 - 20,000 = 147,000 token

用户会先看到橙色警告（"上下文还剩 20%"），然后黄色警告，最后触发自动压缩。

### 压缩排除条件

```typescript
// src/services/compact/autoCompact.ts 第 160-200 行
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
): Promise<boolean> {
  // 防止死锁：compact 子 Agent 本身也会触发自动压缩，形成无限递归
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }
  // 内部 feature：context-collapse 的 ctx-agent 不能触发 autocompact
  if (feature('CONTEXT_COLLAPSE') && querySource === 'marble_origami') {
    return false
  }
  if (!isAutoCompactEnabled()) {
    return false
  }
  // ...
}
```

这里的死锁防护很关键：压缩本身也是一次 Claude API 调用，也会积累 token。如果压缩请求也触发压缩，就会无限递归。`querySource` 字段（标识本次调用来自哪个场景）用于打破这个循环。

### 压缩过程

```typescript
// services/compact/compact.ts（简化）
export async function compactConversation(
  messages: Message[],
  toolUseContext: ToolUseContext,
): Promise<CompactionResult> {
  
  // 1. 移除图片（减少压缩请求的 token 消耗）
  const messagesWithoutImages = stripImagesFromMessages(messages)
  
  // 2. 构建压缩提示
  // 提示包含：整个对话历史 + "请生成一个简洁的摘要"
  const compactPrompt = getCompactPrompt()
  
  // 3. 调用 Claude API 生成摘要
  const summary = await queryModelWithStreaming({
    messages: messagesWithoutImages,
    systemPrompt: compactPrompt,
    maxOutputTokens: COMPACT_MAX_OUTPUT_TOKENS,  // 约 20,000
    // 使用比主模型便宜的小模型（比如 haiku）节省成本
  })
  
  // 4. 构建压缩后的消息列表
  // 包含：摘要消息 + 最近 N 轮完整消息（保留"尾部"）
  const postCompactMessages = [
    createCompactBoundaryMessage(),  // 标记压缩发生的位置
    ...summary,                       // AI 生成的摘要
    ...recentMessages,                // 最近几轮未压缩
  ]
  
  // 5. 恢复关键文件内容（最多 5 个最近修改的文件）
  // 防止 AI 在摘要中提到的文件实际内容被丢失
  const filesToRestore = await selectFilesToRestore(messages)
  for (const file of filesToRestore.slice(0, POST_COMPACT_MAX_FILES_TO_RESTORE)) {
    postCompactMessages.push(await generateFileAttachment(file))
  }
  
  return { summaryMessages, attachments, ... }
}
```

### 关键设计：恢复文件内容

```typescript
// src/services/compact/compact.ts
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
```

压缩后，系统会重新注入最近修改的文件内容（最多 5 个文件，每个最多 5,000 token），确保 AI 在压缩后仍然能看到它正在操作的文件的实际内容，而不是依赖摘要中可能不精确的描述。

### 图片剥离

```typescript
// src/services/compact/compact.ts 第 145-199 行（stripImagesFromMessages）
export function stripImagesFromMessages(messages: Message[]): Message[] {
  // 压缩请求时，把图片块替换为 "[image]" 文字标记
  // 原因：图片会让压缩请求本身占用大量 token，
  // 而对于生成文字摘要来说，图片内容通常不是关键信息
}
```

---

## 8.8 防线 5：阻塞限制

如果自动压缩被禁用，或者连续多次压缩失败，token 使用量达到"阻塞限制"时：

```typescript
// src/query.ts 第 637-648 行
const { isAtBlockingLimit } = calculateTokenWarningState(
  tokenCountWithEstimation(messagesForQuery),
  toolUseContext.options.mainLoopModel,
)
if (isAtBlockingLimit) {
  yield createAssistantAPIErrorMessage({
    content: PROMPT_TOO_LONG_ERROR_MESSAGE,
    error: 'invalid_request',
  })
  return { reason: 'blocking_limit' }
}
```

此时返回给用户一个明确的错误消息："上下文已满，请使用 `/compact` 命令手动压缩，或者开始新会话"。

---

## 8.9 Token 计数：估算与精确值

Claude Code 维护两种 token 计数：

```typescript
// src/utils/tokens.ts（简化）

// 精确值：从 API 响应的 usage 字段获取
export function tokenCountFromLastAPIResponse(messages: Message[]): number {
  const lastAssistantMessage = getLastAssistantMessage(messages)
  return lastAssistantMessage?.apiResponse?.usage?.input_tokens ?? 0
}

// 估算值：在 API 调用之前使用，用于预判是否需要压缩
export function tokenCountWithEstimation(messages: Message[]): number {
  const preciseCount = tokenCountFromLastAPIResponse(messages)
  if (preciseCount > 0) return preciseCount
  
  // 粗略估算：1 token ≈ 4 个字符（英文），中文约 2 个字符
  return roughTokenCountEstimationForMessages(messages)
}
```

为什么需要估算？因为精确的 token 计数只有在调用 API 时才能获得（Anthropic API 在响应的 `usage` 字段里返回实际 token 数）。在决定是否要压缩时，还没有调用 API，只能估算。

---

## 8.10 压缩进度 UI

压缩是一个耗时操作（可能需要 5-30 秒），用户需要知道正在发生什么：

```typescript
// ToolUseContext 中
onCompactProgress?: (event: CompactProgressEvent) => void

type CompactProgressEvent =
  | { type: 'hooks_start'; hookType: 'pre_compact' | 'post_compact' | 'session_start' }
  | { type: 'compact_start' }
  | { type: 'compact_end' }
```

在 REPL 模式下，这会触发终端 UI 显示进度动画："正在压缩会话..." 用户不需要手动操作，只需等待压缩完成。

---

## 8.11 Prompt Cache 与压缩的交互

Claude API 支持 Prompt Cache：如果连续两次调用的系统提示相同，第二次命中缓存，输入 token 费用降低约 90%。

压缩会破坏 Prompt Cache！当消息历史改变（压缩后变短了），再次调用 API 时系统提示的 cache key 不同，缓存失效。

Claude Code 专门处理这个问题：

```typescript
// src/services/api/promptCacheBreakDetection.ts
export function notifyCompaction() {
  // 通知 API 层：下一次请求是压缩后的新对话
  // 不要尝试使用旧的 prompt cache
}
```

---

## 8.12 优势与不足

### 优势

**多层防线**：从工具结果大小控制到自动摘要，每一层都有独立的触发条件和处理方式，形成了纵深防御。没有单点失效。

**恢复关键文件**：压缩后自动重新注入最近修改文件的内容，让 AI 在压缩后继续操作文件时不会"失忆"。

**图片剥离**：压缩请求中移除图片，避免图片内容本身导致压缩请求也失败（套娃式失败）。

**死锁防护**：通过 `querySource` 字段，确保压缩请求自身不会触发自动压缩，防止无限递归。

**用户透明**：自动压缩对用户几乎透明——会话继续，AI 从摘要出发继续工作。只有 UI 角落的 token 指示器会显示上下文窗口占用减少了。

### 不足

**摘要信息损失**：AI 生成的摘要不可避免地会丢失部分信息。如果早期对话中有关键的技术细节（比如"我们决定不用 X 方案因为 Y 原因"），这些细节可能在摘要中被压缩为"讨论了技术方案"，导致 AI 在后续重复已经否定过的方案。

**压缩延迟**：压缩需要调用 Claude API，通常耗时 5-30 秒。在长对话的关键时刻（正在执行重要操作时）触发压缩，会造成较长的等待。虽然有进度提示，但体验仍然不流畅。

**连续压缩失败**的断路器（MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3）在最坏情况下会让用户在 3 次失败后不再自动压缩，而手动 `/compact` 对用户来说也不直观。

**Prompt Cache 失效**：每次压缩都会让 Prompt Cache 失效，导致下一次调用的 token 费用显著增加（约 10 倍）。这在频繁压缩的长会话中会明显影响成本。
