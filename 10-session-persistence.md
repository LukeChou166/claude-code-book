# 第10章 会话持久化与恢复

> **前置知识**：第2章 QueryEngine、第3章 Agentic Loop  
> **关键文件**：`src/utils/sessionStorage.ts`（约5100行）、`src/commands/resume/`、`src/types/logs.ts`

---

## 10.1 解决什么问题

AI 编程助手处理的任务往往不能在一次会话中完成：

1. **用户中途关闭终端**：第二天重新打开，所有上下文都没了，只能重新解释需求
2. **网络断开**：API 调用超时，会话崩溃，几分钟的上下文全部丢失
3. **多设备使用**：在家的机器开始了一个任务，到公司换另一台机器想继续
4. **Bug 复现**：用户报告"Claude 之前帮我写的代码有问题"，需要找回原来的对话
5. **审计需求**：企业用户需要知道 AI 在代码库里做了什么操作

这些需求要求会话数据被持久化到磁盘，并且可以精确恢复。

---

## 10.2 业界的做法

**ChatGPT/Claude.ai 网页版**：对话存储在服务端，用户可以浏览历史对话。
- 优点：自动同步，多设备可用
- 缺点：隐私问题，用户数据上传到服务端；无法访问本地文件系统的上下文

**VS Code Extension**：通常不持久化对话历史，每次打开新窗口就是全新会话。

**LangChain**：通过 `VectorStoreRetrieverMemory` 等将对话摘要存储在向量数据库，实现"长期记忆"。
- 优点：理论上无限历史
- 缺点：向量检索不够精确，对于编程任务可能丢失关键细节

---

## 10.3 本项目的存储结构

会话存储在用户主目录的 `.claude/` 目录下：

```
~/.claude/
├── settings.json              # 全局配置
├── CLAUDE.md                  # 全局记忆文件
└── projects/
    └── <工作目录路径哈希>/
        ├── .claude/
        │   └── settings.local.json  # 项目本地配置
        └── sessions/
            ├── <session-id-1>.jsonl  # 会话日志（JSONL 格式）
            ├── <session-id-2>.jsonl
            └── ...
```

**路径哈希**：将工作目录路径（如 `/Users/luke/project`）转换为 URL 安全的哈希，避免路径中的特殊字符导致文件名问题。

---

## 10.4 JSONL 格式：逐行追加的流式日志

会话日志使用 **JSONL（JSON Lines）** 格式，每行是一个独立的 JSON 对象：

```jsonl
{"type":"summary","summary":"开始分析 utils.ts 中的 bug","timestamp":"2024-01-15T10:00:00.000Z"}
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"帮我修复 utils.ts 里的 null 指针异常"}]},"uuid":"abc123","timestamp":"..."}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"我来看看这个文件..."}]},"uuid":"def456","timestamp":"..."}
{"type":"user","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"xyz789","content":"..."}]},"uuid":"ghi012","timestamp":"..."}
```

**为什么用 JSONL 而不是 JSON**：
1. **流式写入**：每条消息产生时立即追加一行，不需要读取整个文件再重写
2. **崩溃恢复**：即使程序在写入中途崩溃，已经写入的行都是完整的，之后的行只是缺失，不会损坏已有数据
3. **顺序读取**：恢复会话时按行顺序读取，内存使用量低
4. **任意位置截断**：对于大文件，可以从任意行开始读取（`tail -n 100 session.jsonl`）

---

## 10.5 Log Entry 类型系统

`types/logs.ts` 定义了日志中所有可能的 entry 类型：

```typescript
// src/types/logs.ts（简化）
export type Entry =
  | TranscriptMessage        // 完整的消息（用户/助手/工具结果）
  | FileHistorySnapshotMessage  // 文件快照（"工具修改前，文件是这个内容"）
  | ContextCollapseSnapshotEntry  // 上下文折叠快照（内部使用）
  | ContextCollapseCommitEntry    // 上下文折叠确认
  | ContentReplacementEntry   // 内容替换记录（工具结果被截断时的记录）
  | AttributionSnapshotMessage  // 文件归因快照

// 完整的消息
export type TranscriptMessage = {
  type: 'user' | 'assistant'
  message: MessageParam
  uuid: string
  timestamp?: string
  agentId?: AgentId  // 如果是子 Agent 产生的消息
  sessionId?: SessionId
  cwd?: string
  isApiErrorMessage?: boolean
}

// 文件历史快照：记录文件在工具修改之前的内容
export type FileHistorySnapshotMessage = {
  type: 'fileHistorySnapshot'
  filePath: string
  content: string | null  // null 表示文件在此之前不存在
  timestamp: string
}
```

**`FileHistorySnapshotMessage`** 是一个特别有价值的设计：每次 FileEditTool 或 FileWriteTool 执行前，会把文件当前内容快照到 JSONL 日志中。这使得：
1. **undo/撤销**：`/rewind` 命令可以找到最近的文件快照，恢复文件内容
2. **变更归因**：可以查看是哪一步操作修改了文件
3. **diff 展示**：比较快照与当前内容，展示"这次会话中文件的所有变更"

---

## 10.6 会话恢复：/resume 命令

用户可以通过 `/resume <session-id>` 命令恢复历史会话：

```typescript
// src/commands/resume/（简化）
export async function handleResumeCommand(
  sessionId: SessionId,
  options: ResumeOptions,
): Promise<Message[]> {
  
  // 1. 找到对应的 JSONL 文件
  const transcriptPath = getTranscriptPath(sessionId)
  
  // 2. 读取所有日志条目
  const entries = await readJSONLFile(transcriptPath)
  
  // 3. 重建消息历史
  const messages: Message[] = []
  for (const entry of entries) {
    if (entry.type === 'user' || entry.type === 'assistant') {
      messages.push(deserializeMessage(entry))
    }
  }
  
  // 4. 重放关键状态：
  //    - 最新的文件内容（从 fileHistorySnapshot 重建）
  //    - 权限规则（从会话中批准的规则恢复）
  //    - 上下文压缩边界（compactBoundary 之后的消息）
  
  return messages
}
```

恢复后，`initialMessages` 参数传给 QueryEngine，下一条用户消息会在完整的历史上下文中继续处理。

---

## 10.7 会话元数据写入

每次消息产生后，消息立即追加到 JSONL 文件，而不是会话结束时批量写入：

```typescript
// src/utils/sessionStorage.ts 第 77-112 行（简化）
export async function appendMessageToTranscript(
  message: Entry,
  sessionId: SessionId,
  agentId: AgentId,
): Promise<void> {
  const path = getTranscriptPath(sessionId, agentId)
  
  // 确保目录存在
  await mkdir(dirname(path), { recursive: true })
  
  // 逐行追加（O(1) 操作，不读取现有文件）
  await fsAppendFile(path, JSON.stringify(sortLogEntries(message)) + '\n')
}
```

注意 `sortLogEntries(message)` 在序列化前对消息的 key 进行排序，确保相同内容的消息序列化结果一致（方便 diff 和调试）。

---

## 10.8 会话 Ingress：上报到 Anthropic 服务器

`sessionIngress.ts` 负责将会话数据同步到 Anthropic 服务器（用于训练、调试）：

```typescript
// src/services/api/sessionIngress.ts（简化）
export async function reportSessionMessage(
  message: TranscriptMessage,
): Promise<void> {
  if (!isSessionIngressEnabled()) return
  
  // 非阻塞：不等待服务器响应
  // 如果网络不可用或服务器错误，静默失败
  fetch(SESSION_INGRESS_ENDPOINT, {
    method: 'POST',
    body: JSON.stringify(sanitizeForIngress(message)),  // 脱敏处理
  }).catch(() => {
    // 上报失败不影响正常使用
  })
}
```

这个功能默认只在 `ant` 内部用户（Anthropic 员工）启用，外部用户的数据不会上报（通过 feature flag 控制）。

---

## 10.9 会话隔离：子 Agent 的日志

子 Agent 的消息也被记录，但在文件路径中包含了 `agentId`，与主会话的日志分开存储：

```typescript
// src/utils/sessionStorage.ts 第 46-60 行（简化）
export function getTranscriptPath(
  sessionId: SessionId,
  agentId?: AgentId,  // undefined = 主 Agent
): string {
  const basePath = getSessionDir(sessionId)
  if (agentId) {
    // 子 Agent 的日志放在子目录中
    return path.join(basePath, 'agents', `${agentId}.jsonl`)
  }
  return path.join(basePath, 'session.jsonl')
}
```

主会话的 JSONL 中只记录"子 Agent 执行完成，结果是 X"，而子 Agent 执行过程中的所有工具调用和推理步骤记录在单独的 JSONL 文件中。这让主会话日志保持清晰，同时保留了调试子 Agent 的完整信息。

---

## 10.10 Rewind 功能（撤销操作）

`/rewind` 命令允许用户撤销最近的文件修改：

```typescript
// src/commands/rewind/（简化）
export async function handleRewindCommand(
  sessionId: SessionId,
  targetTimestamp?: string,  // 回退到指定时间点
) {
  // 1. 读取文件快照历史
  const snapshots = await getFileHistorySnapshots(sessionId)
  
  // 2. 找到目标时间点之前最近的快照
  const targetSnapshots = snapshots.filter(s => s.timestamp <= targetTimestamp)
  
  // 3. 恢复文件到快照时的状态
  for (const snapshot of targetSnapshots) {
    if (snapshot.content === null) {
      // 文件在此时间点不存在 → 删除它
      await unlink(snapshot.filePath).catch(() => {})
    } else {
      // 恢复文件内容
      await writeFile(snapshot.filePath, snapshot.content)
    }
  }
  
  // 4. 更新消息历史，移除目标时间点之后的消息
  // ...
}
```

这使得 Claude Code 具备了"时间机器"功能：用户可以回退到任意历史节点，然后从那里重新开始。

---

## 10.11 会话分析工具（/stats, /cost）

会话日志不只用于恢复，还用于分析：

```typescript
// src/commands/stats/（简化）
export function analyzeSession(sessionId: SessionId) {
  const entries = readJSONL(getTranscriptPath(sessionId))
  
  return {
    totalMessages: entries.filter(e => e.type === 'user' || e.type === 'assistant').length,
    toolCallCount: countToolCalls(entries),
    filesModified: getModifiedFiles(entries),
    totalTokensUsed: sumTokenUsage(entries),
    estimatedCost: calculateCost(entries),
    sessionDuration: getSessionDuration(entries),
  }
}
```

`/cost` 命令展示本次会话的累计 API 费用，帮助用户了解 Claude Code 的使用成本。

---

## 10.12 优势与不足

### 优势

**JSONL 流式写入**：每条消息立即持久化，不依赖会话"正常结束"。程序崩溃或强制杀进程，最多丢失最后一条消息，之前的历史完整保留。

**文件快照为 Rewind 提供基础**：修改文件前自动保存快照，让"撤销操作"成为可能，这是许多 AI 助手缺失的功能。

**子 Agent 日志隔离**：主会话日志干净，子 Agent 的详细日志按需查看，两者都完整保存，方便不同粒度的调试。

**本地存储**：数据保留在用户机器上（`~/.claude/`），不传输到 Anthropic 服务器（除非是 `ant` 用户且显式启用 ingress），满足企业隐私合规需求。

### 不足

**没有数据库，只有文件**：对于有数千个会话的重度用户，在所有会话中搜索（"找到上次修改 config.ts 的对话"）只能线性扫描所有 JSONL 文件，性能很差。一个 SQLite 数据库可以更好地支持这类查询。

**`sessionStorage.ts` 约 5100 行**：混合了读取、写入、格式化、排序、快照、ingress 上报等大量职责，是第二个"上帝文件"。

**没有会话同步**：跨设备同步（比如在 macOS 和 Linux 服务器之间）完全不支持。用户必须手动同步 `~/.claude/` 目录（通过 Dropbox 或 rsync 等）。

**JSONL 文件可能很大**：一个长时间的会话（运行几小时，包含大量文件读取）的 JSONL 可能超过 100MB。没有压缩或归档机制，磁盘占用可能成为问题。
