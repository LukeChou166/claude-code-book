# 第13章 错误处理与恢复机制

> **前置知识**：第2章 QueryEngine、第3章 Agentic Loop  
> **关键文件**：`src/services/api/withRetry.ts`、`src/utils/gracefulShutdown.ts`、`src/utils/cleanupRegistry.ts`、`src/utils/abortController.ts`、`src/utils/errors.ts`、`src/remote/SessionsWebSocket.ts`

---

## 13.1 解决什么问题

AI 编程助手在生产环境中运行，会遭遇各种失败场景：

1. **API 过载**：Anthropic 服务器繁忙，返回 429（速率限制）或 529（过载）
2. **网络闪断**：用户网络不稳定，TCP 连接中途断开，返回 ECONNRESET
3. **Context 超出限制**：积累了大量工具调用，输入 token 超过模型上下文窗口
4. **用户按 Ctrl+C**：需要优雅退出，而不是直接杀死进程（避免终端残留乱码、会话数据丢失）
5. **终端关闭（SIGHUP）**：SSH 断线时需要处理，macOS 甚至不发信号
6. **任务执行中断**：子 Agent 在执行过程中需要被取消
7. **渲染错误**：React 组件抛出异常，需要防止整个 TUI 崩溃

Claude Code 对每一层都有专属的处理策略，形成了一个多层防御体系。

---

## 13.2 业界的做法

| 策略 | 代表实现 | 优点 | 缺点 |
|------|---------|------|------|
| **固定延迟重试** | 简单循环 + `sleep(1s)` | 实现简单 | 所有客户端同时重试，导致惊群效应 |
| **指数退避** | AWS SDK 经典重试 | 减轻服务器压力 | 仍然可能形成"重试波峰" |
| **指数退避 + Jitter** | AWS Architecture Blog 2015 | 防惊群效应，工业标准 | 代码略复杂 |
| **断路器** | Netflix Hystrix | 服务不可用时快速失败 | 需要额外状态机管理 |
| **信号处理** | POSIX signal + cleanup | 系统级，可靠 | 异步信号不安全，需谨慎 |
| **AbortController** | Web 标准 | 树状取消传播 | 需要显式传递 signal，有侵入性 |

Claude Code 综合使用了以上大多数方案，针对不同层次采用不同策略。

---

## 13.3 本项目的做法（deep）

### 13.3.1 层次结构总览

```
┌───────────────────────────────────────────────────────┐
│  React 渲染层：ErrorBoundary（防止 TUI 崩溃）          │
├───────────────────────────────────────────────────────┤
│  API 调用层：withRetry（指数退避 + jitter + 模型回退）  │
├───────────────────────────────────────────────────────┤
│  WebSocket 层：SessionsWebSocket（自动重连心跳）        │
├───────────────────────────────────────────────────────┤
│  任务取消层：AbortController（树状级联取消）            │
├───────────────────────────────────────────────────────┤
│  进程退出层：gracefulShutdown（SIGINT/SIGTERM/SIGHUP）  │
└───────────────────────────────────────────────────────┘
```

每一层只负责自己能处理的错误，上移给上一层处理。

---

### 13.3.2 React 渲染层：两道错误边界

**第一道**：`SentryErrorBoundary`（`src/components/SentryErrorBoundary.ts`）

```typescript
// 静默处理：组件崩溃时直接返回 null（不渲染），不传播给父组件
export class SentryErrorBoundary extends React.Component<Props, State> {
  static getDerivedStateFromError(): State {
    return { hasError: true }
  }
  render(): React.ReactNode {
    if (this.state.hasError) {
      return null  // 静默处理，这个组件从界面消失，但其他组件正常
    }
    return this.props.children
  }
}
```

这用于包裹非关键组件（如各种 UI 小部件）。某个小组件崩溃，不影响整体。

**第二道**：`App.tsx` 的 Ink 根组件（`src/ink/components/App.tsx`）

```typescript
// 整个应用根组件的错误处理：
static getDerivedStateFromError(error: Error) {
  return { error }
}

override componentDidCatch(error: Error) {
  // 渲染崩溃 → 触发优雅退出，而不是进程闪退
  this.handleExit(error)
}
```

根组件的错误边界捕获到任何渲染错误，调用 `handleExit(error)` 启动优雅退出流程，确保终端被清理干净。

**调用链**：

```
任意组件 throw Error
    ↓
React reconciler 向上冒泡
    ↓
SentryErrorBoundary.getDerivedStateFromError()  ← 先被就近的边界捕获
    ↓ 若边界内部也出错
App.componentDidCatch(error)
    ↓
this.handleExit(error) → gracefulShutdown()
```

---

### 13.3.3 API 调用层：withRetry（核心重试引擎）

`withRetry` 是整个项目最复杂的错误处理模块（约 400 行），以异步生成器实现：

**函数签名**：

```170:178:restored-src/src/services/api/withRetry.ts
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (
    client: Anthropic,
    attempt: number,
    context: RetryContext,
  ) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
```

参数说明：

- `getClient`：**惰性**获取 API 客户端，每次认证失败后重新调用，拿到新 token
- `operation`：真正的 API 调用逻辑（调用方提供）
- `options.signal`：AbortSignal，外部取消时立即停止重试
- 返回：`AsyncGenerator<SystemAPIErrorMessage, T>`
  - `yield SystemAPIErrorMessage`：每次重试时向 UI 发送"正在重试第 N 次"的消息，让用户看到进度
  - `return T`：最终成功的结果

**指数退避 + Jitter（防惊群效应）**：

```530:548:restored-src/src/services/api/withRetry.ts
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }

  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

延迟计算公式：`min(500ms × 2^(attempt-1), 32000ms) + random(0~25%)`

各次重试的实际延迟范围：

| 重试次数 | 基础延迟 | jitter 范围 | 实际范围 |
|---------|---------|------------|---------|
| 1 | 500ms | 0~125ms | 500~625ms |
| 2 | 1000ms | 0~250ms | 1000~1250ms |
| 3 | 2000ms | 0~500ms | 2000~2500ms |
| 4 | 4000ms | 0~1000ms | 4000~5000ms |
| 5+ | 8000~32000ms | 0~25% | 8000~40000ms |

**优先级最高**：如果服务器返回了 `Retry-After` 响应头，直接用服务器指定的等待时间，完全覆盖本地计算结果。

**可重试的错误类型**（`shouldRetry` 函数）：

| 状态码 | 场景 | 重试策略 |
|-------|------|---------|
| 401 | Token 过期 | 重试 + 强制刷新 Token |
| 403 `OAuth token revoked` | 另一个进程刷新了 token | 重试 + 强制刷新 Token |
| 408 | 请求超时 | 重试 |
| 409 | 锁超时 | 重试 |
| 429 | 速率限制 | 重试（非订阅用户） |
| 529 | 服务过载 | 重试（前景查询源）|
| `ECONNRESET`/`EPIPE` | TCP 连接重置 | 重试 + 禁用 Keep-Alive |

**不重试的情况**：
- Mock 错误（内部测试用）
- 400 客户端错误（除 context overflow 的特殊处理外）
- 429 达到最大重试次数且用户是订阅用户

**快速模式（Fast Mode）降级**：

```typescript
// 快速模式下遇到 429/529 的特殊处理
if (wasFastModeActive && !isPersistentRetryEnabled() && error.status === 429 || is529Error(error)) {
  const retryAfterMs = getRetryAfterMs(error)
  
  if (retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
    // 等待时间短：保持快速模式重试（保护 prompt cache）
    await sleep(retryAfterMs)
    continue
  }
  
  // 等待时间长：进入冷却期，切换到标准速度模型
  triggerFastModeCooldown(Date.now() + cooldownMs, 'rate_limit')
  retryContext.fastMode = false
  continue
}
```

快速模式会用同一个模型名（如 `claude-3-5-haiku-20241022-fast`）调用，以利用 prompt cache。遇到限流后如果等待时间短，保持快速模式重试；如果等待时间长，切换到标准速度模型（`claude-3-5-haiku-20241022`），避免长时间阻塞用户。

**模型回退机制**：

```326:352:restored-src/src/services/api/withRetry.ts
      // Track consecutive 529 errors
      if (
        is529Error(error) &&
        (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
          (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))
      ) {
        consecutive529Errors++
        if (consecutive529Errors >= MAX_529_RETRIES) {
          // Check if fallback model is specified
          if (options.fallbackModel) {
            logEvent('tengu_api_opus_fallback_triggered', {
              original_model: options.model,
              fallback_model: options.fallbackModel,
            })
            // Throw special error to indicate fallback was triggered
            throw new FallbackTriggeredError(
              options.model,
              options.fallbackModel,
            )
          }
```

`MAX_529_RETRIES = 3`，连续 3 次 529 后：
1. 如果配置了备用模型（`fallbackModel`）→ 抛出 `FallbackTriggeredError`，调用方切换模型重试
2. 如果没有备用模型 → 抛出 `CannotRetryError`，向用户展示错误信息

**Context 溢出自动修复**：

```384:427:restored-src/src/services/api/withRetry.ts
      if (error instanceof APIError) {
        const overflowData = parseMaxTokensContextOverflowError(error)
        if (overflowData) {
          const { inputTokens, contextLimit } = overflowData
          const safetyBuffer = 1000
          const availableContext = Math.max(
            0,
            contextLimit - inputTokens - safetyBuffer,
          )
          // ...
          const adjustedMaxTokens = Math.max(
            FLOOR_OUTPUT_TOKENS,
            availableContext,
            minRequired,
          )
          retryContext.maxTokensOverride = adjustedMaxTokens
          // continue → 用调整后的 maxTokens 重试
          continue
        }
      }
```

当 API 报告"输出 token 超过可用空间"时，自动计算 `adjustedMaxTokens = contextLimit - inputTokens - 1000`，将其记入 `retryContext.maxTokensOverride`，下次请求时用这个值覆盖 `max_tokens`，无需用户介入。

**持久重试模式（无人值守）**：

```typescript
// CLAUDE_CODE_UNATTENDED_RETRY=1 启用
// 适用于 CI/CD 等无人值守环境：429/529 无限重试，最大退避 5 分钟
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000  // 5 分钟
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000  // 6 小时（用于 window-based 限流等待）
const HEARTBEAT_INTERVAL_MS = 30_000

// 等待期间每 30s yield 一次心跳消息（防止宿主环境判定进程空闲）
while (remaining > 0) {
  yield createSystemAPIErrorMessage(error, remaining, attempt, maxRetries)
  const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)
  await sleep(chunk)
  remaining -= chunk
}
```

---

### 13.3.4 WebSocket 层：SessionsWebSocket 自动重连

远程会话（`claude assistant` 模式）通过 WebSocket 连接 Anthropic 服务器，`SessionsWebSocket.ts` 管理连接生命周期：

**重连决策逻辑**（`src/remote/SessionsWebSocket.ts`）：

```typescript
const RECONNECT_DELAY_MS = 2000
const MAX_RECONNECT_ATTEMPTS = 5
const PING_INTERVAL_MS = 30000
const MAX_SESSION_NOT_FOUND_RETRIES = 3

// WebSocket 关闭时的处理
onClose(closeCode) {
  // 永久关闭码：不重连
  if (PERMANENT_CLOSE_CODES.has(closeCode)) {  // 4003 = unauthorized
    this.callbacks.onClose?.()
    return
  }
  
  // 4001 = session not found（压缩期间可能瞬态出现）
  if (closeCode === 4001) {
    this.sessionNotFoundRetries++
    if (this.sessionNotFoundRetries > MAX_SESSION_NOT_FOUND_RETRIES) {
      this.callbacks.onClose?.()
      return
    }
    // 延迟递增重试（1×2s, 2×2s, 3×2s）
    this.scheduleReconnect(RECONNECT_DELAY_MS * this.sessionNotFoundRetries)
    return
  }
  
  // 普通断线：最多重试 5 次，固定 2s 延迟
  if (previousState === 'connected' && this.reconnectAttempts < MAX_RECONNECT_ATTEMPTS) {
    this.reconnectAttempts++
    this.scheduleReconnect(RECONNECT_DELAY_MS)
  }
}
```

**心跳机制**：

```typescript
// 每 30s 发一次 ping，保持 WebSocket 连接存活
// 防止中间网络设备（负载均衡器、NAT）因为连接空闲而关闭它
private startPingInterval(): void {
  this.pingInterval = setInterval(() => {
    if (this.ws && this.state === 'connected') {
      this.ws.ping?.()
    }
  }, PING_INTERVAL_MS)
}
```

**重连调度**：

```typescript
private scheduleReconnect(delay: number): void {
  this.callbacks.onReconnecting?.()  // 通知 UI 显示"重连中..."
  this.reconnectTimer = setTimeout(() => {
    void this.connect()
  }, delay)
}

// 强制立即重连（重置所有计数器）
reconnect(): void {
  this.reconnectAttempts = 0
  this.sessionNotFoundRetries = 0
  this.close()
  this.reconnectTimer = setTimeout(() => {
    void this.connect()
  }, 500)  // 500ms 后重连（给旧连接时间关闭）
}
```

---

### 13.3.5 任务取消层：AbortController 树状级联

Claude Code 大量使用 `AbortController` 来取消正在进行的操作。`src/utils/abortController.ts` 提供了父子级联的工具函数：

```typescript
// 创建子 AbortController：父中止时，子自动中止
export function createChildAbortController(
  parent: AbortController,
  maxListeners?: number,
): AbortController {
  const child = createAbortController(maxListeners)
  
  if (parent.signal.aborted) {
    // 父已经中止，子立即中止
    child.abort(parent.signal.reason)
    return child
  }
  
  // 用 WeakRef 避免循环引用导致内存泄漏
  // C++ 类比：相当于 std::weak_ptr<AbortController>
  const weakChild = new WeakRef(child)
  const weakParent = new WeakRef(parent)
  const handler = propagateAbort.bind(weakParent, weakChild)
  
  // 父中止 → 传播给子
  parent.signal.addEventListener('abort', handler, { once: true })
  
  // 子中止 → 移除父的监听器（避免内存泄漏）
  child.signal.addEventListener(
    'abort',
    removeAbortHandler.bind(weakParent, new WeakRef(handler)),
    { once: true },
  )
  
  return child
}
```

**输入输出示例**：

```typescript
// 输入（场景 A）：创建父子关系
const parent = new AbortController()
const child = createChildAbortController(parent)

// 操作：父中止
parent.abort('user cancelled')

// 输出（场景 A）：子自动中止
console.log(child.signal.aborted)  // → true
console.log(child.signal.reason)   // → 'user cancelled'

// 输入（场景 B）：父已经中止
const alreadyAborted = new AbortController()
alreadyAborted.abort('already done')
const child2 = createChildAbortController(alreadyAborted)

// 输出（场景 B）：子立即中止
console.log(child2.signal.aborted)  // → true（立即）
```

**AbortError 统一检测**：

```typescript
// src/utils/errors.ts
// 三种来源的中止错误，统一检测
export function isAbortError(e: unknown): boolean {
  return (
    e instanceof AbortError           // 自定义中止错误
    || e instanceof APIUserAbortError  // Anthropic SDK 的中止错误
    || (e instanceof Error && e.name === 'AbortError')  // 原生 WebAPI AbortError
  )
}
```

在 `catch` 块里用 `isAbortError(e)` 区分"正常中止"和"真实错误"，避免把用户主动取消当成错误记录：

```typescript
try {
  await longRunningOperation(signal)
} catch (e) {
  if (isAbortError(e)) {
    // 用户主动取消，正常退出，不上报错误
    return
  }
  // 真正的错误，上报并处理
  logError(e)
  throw e
}
```

---

### 13.3.6 进程退出层：gracefulShutdown（最后的防线）

`setupGracefulShutdown()` 在程序启动时调用一次（`memoize` 确保只执行一次），注册三个信号处理器：

```256:297:restored-src/src/utils/gracefulShutdown.ts
  process.on('SIGINT', () => {
    // SIGINT = Ctrl+C
    if (process.argv.includes('-p') || process.argv.includes('--print')) {
      return  // print 模式有自己的处理器，这里跳过
    }
    void gracefulShutdown(0)
  })
  process.on('SIGTERM', () => {
    // SIGTERM = kill 命令 / systemd 关闭服务
    void gracefulShutdown(143) // 128 + 15
  })
  if (process.platform !== 'win32') {
    process.on('SIGHUP', () => {
      // SIGHUP = 终端关闭 / SSH 断线
      void gracefulShutdown(129) // 128 + 1
    })

    // macOS 特殊处理：终端关闭时撤销 TTY fd，不发 SIGHUP
    // 每 30s 检查 stdout/stdin 是否还可用
    if (process.stdin.isTTY) {
      orphanCheckInterval = setInterval(() => {
        if (!process.stdout.writable || !process.stdin.readable) {
          clearInterval(orphanCheckInterval)
          void gracefulShutdown(129)  // 孤儿进程，触发退出
        }
      }, 30_000)
      orphanCheckInterval.unref()  // 不阻止进程自然退出
    }
  }
```

**`gracefulShutdown` 完整执行流程**：

```391:522:restored-src/src/utils/gracefulShutdown.ts
export async function gracefulShutdown(
  exitCode = 0,
  reason: ExitReason = 'other',
  options?: { getAppState?; setAppState?; finalMessage? },
): Promise<void> {
  if (shutdownInProgress) return  // 防止并发多次触发
  shutdownInProgress = true

  // 步骤 0：查询 SessionEnd Hook 的超时配置
  const sessionEndTimeoutMs = getSessionEndHookTimeoutMs()

  // 步骤 1：设置 failsafe 定时器（兜底保证最终一定退出）
  // 预算 = max(5s, hook 超时 + 3.5s 其他操作余量)
  failsafeTimer = setTimeout(
    code => {
      cleanupTerminalModes()
      printResumeHint()
      forceExit(code)
    },
    Math.max(5000, sessionEndTimeoutMs + 3500),
    exitCode,
  )
  failsafeTimer.unref()

  // 步骤 2：立即清理终端模式（同步）
  // 必须最先执行：即使后续步骤被 SIGKILL，终端也已经被清理
  cleanupTerminalModes()  // 禁用鼠标追踪、退出 alt-screen、显示光标等
  printResumeHint()       // 打印 "Resume with: claude --resume <id>"

  // 步骤 3：运行所有注册的 cleanup 函数（最多等 2s）
  await Promise.race([
    runCleanupFunctions(),
    new Promise((_, reject) => setTimeout(() => reject(new CleanupTimeoutError()), 2000))
  ])

  // 步骤 4：执行 SessionEnd hooks（用户配置的退出钩子）
  await executeSessionEndHooks(reason, {
    signal: AbortSignal.timeout(sessionEndTimeoutMs),
    timeoutMs: sessionEndTimeoutMs,
  })

  // 步骤 5：记录启动性能数据
  profileReport()

  // 步骤 6：发送 cache 驱逐提示（让推理服务器回收 prompt cache）
  if (lastRequestId) {
    logEvent('tengu_cache_eviction_hint', { scope: 'session_end', last_request_id: lastRequestId })
  }

  // 步骤 7：刷新 analytics 数据（最多等 500ms，超时丢弃）
  await Promise.race([
    Promise.all([shutdown1PEventLogging(), shutdownDatadog()]),
    sleep(500),
  ])

  // 步骤 8：强制退出
  forceExit(exitCode)
}
```

**failsafe 的设计哲学**：每一步都可能挂起（网络 I/O、MCP 连接关闭等），failsafe 定时器保证了**即使所有步骤都卡住，最终一定会退出**。各步骤的超时是递减的：cleanup 最多 2s，analytics 最多 500ms，failsafe 总预算是它们之和加 3.5s 余量。

**cleanupRegistry 模块注册机制**：

```1:26:restored-src/src/utils/cleanupRegistry.ts
const cleanupFunctions = new Set<() => Promise<void>>()

export function registerCleanup(cleanupFn: () => Promise<void>): () => void {
  cleanupFunctions.add(cleanupFn)
  return () => cleanupFunctions.delete(cleanupFn)  // 返回注销函数
}

export async function runCleanupFunctions(): Promise<void> {
  await Promise.all(Array.from(cleanupFunctions).map(fn => fn()))
}
```

任何需要清理资源的模块（MCP 连接、WebSocket、文件句柄）都可以调用 `registerCleanup(fn)` 注册。返回的注销函数在组件/模块卸载时调用，避免在进程退出时重复执行已经清理过的资源。

**forceExit 的防御式设计**：

```193:232:restored-src/src/utils/gracefulShutdown.ts
function forceExit(exitCode: number): never {
  // 清除 failsafe 定时器（已经到达退出点，不需要它了）
  if (failsafeTimer !== undefined) {
    clearTimeout(failsafeTimer)
    failsafeTimer = undefined
  }
  try {
    process.exit(exitCode)
  } catch (e) {
    // process.exit() 在 TTY 关闭时会抛出 EIO 错误（Bun 的行为）
    // 在测试环境 re-throw；生产环境用 SIGKILL（不尝试 flush stdout）
    if (process.env.NODE_ENV === 'test') {
      throw e
    }
    process.kill(process.pid, 'SIGKILL')
  }
}
```

**cleanupTerminalModes 做了什么**（`src/utils/gracefulShutdown.ts` 第 59-136 行）：

终端 TUI 运行期间，会激活大量"特殊模式"。如果进程不干净退出，这些模式会残留，导致终端看起来"坏了"（鼠标乱跳、键盘输入不显示、颜色异常）。`cleanupTerminalModes` 按顺序发送 escape 序列来清理：

1. 禁用鼠标追踪（`DISABLE_MOUSE_TRACKING`）——最先执行，给终端时间处理
2. 卸载 Ink 实例（退出 alt-screen buffer）
3. 禁用 Kitty 键盘协议（`DISABLE_KITTY_KEYBOARD`）
4. 禁用修饰键追踪（`DISABLE_MODIFY_OTHER_KEYS`）
5. 禁用焦点事件上报（`DFE`）
6. 禁用括号粘贴模式（`DBP`）
7. 显示光标（`SHOW_CURSOR`）
8. 清除 iTerm2 进度条
9. 清除标签页状态、终端标题

---

### 13.3.7 MCP 连接层的特殊错误处理

MCP 服务器的连接错误有其特殊性（`src/services/mcp/client.ts`）：

**会话过期检测**：

```typescript
// HTTP 传输的 MCP Server，会话过期表现为 "HTTP 404 + JSON-RPC -32001"
if ((transportType === 'http' || transportType === 'claudeai-proxy') && 
    isMcpSessionExpiredError(error)) {
  closeTransportAndRejectPending('session expired')
  // 下次调用时会以新的 session ID 重新连接
}
```

**终端连接错误计数**：

```typescript
const MAX_ERRORS_BEFORE_RECONNECT = 3
if (isTerminalConnectionError(error.message)) {
  consecutiveConnectionErrors++
  if (consecutiveConnectionErrors >= MAX_ERRORS_BEFORE_RECONNECT) {
    consecutiveConnectionErrors = 0
    closeTransportAndRejectPending('max consecutive terminal errors')
    // 强制重建连接
  }
}
```

---

### 13.3.8 错误分类：不同错误要不同对待

`src/services/api/errors.ts` 提供了错误分类器：

```typescript
export function classifyAPIError(error: unknown): string {
  if (error instanceof APIError && error.status === 429) return 'rate_limit'
  if (error instanceof APIError && (error.status === 529 || error.message?.includes('"type":"overloaded_error"'))) return 'server_overload'
  if (error instanceof APIConnectionError) return 'connection_error'
  if (error instanceof APIUserAbortError) return 'user_abort'
  // ... 其他分类
  return 'unknown'
}
```

分类的目的：
- `user_abort`：正常取消，不上报错误，不计入失败统计
- `rate_limit`/`server_overload`：临时性，可重试
- `connection_error`：网络问题，可重试
- 其他：不可重试，展示错误信息给用户

---

## 13.4 失败路径与排障

### 症状：API 调用一直重试，长时间不成功

**定位**：
1. 先看 `src/services/api/withRetry.ts` 的 `logEvent('tengu_api_retry', {...})`，找到重试原因（status 字段）
2. 如果是 429，检查 `Retry-After` 响应头是否异常大
3. 如果是 529，检查是否触发了模型回退（`tengu_api_opus_fallback_triggered` 事件）

**修复**：
- 429 频繁：检查 API 配额，考虑降级到低速模型
- 529 频繁：考虑开启 `CLAUDE_CODE_UNATTENDED_RETRY=1`（无人值守模式）

### 症状：退出后终端显示乱码，鼠标跳动

**定位**：`cleanupTerminalModes()` 没有正确执行，可能是：
1. 进程被 `kill -9` 强制杀死（绕过了所有信号处理）
2. `gracefulShutdown` 在 `cleanupTerminalModes()` 之前被 failsafe 杀死

**修复**：在终端里输入 `reset` 命令恢复终端状态。

### 症状：WebSocket 重连次数耗尽后会话断开

**定位**：`src/remote/SessionsWebSocket.ts` 的 `MAX_RECONNECT_ATTEMPTS = 5`。连续 5 次重连失败后，`onClose?.()` 被调用，连接永久关闭。

**修复**：调用 `reconnect()` 方法重置计数器并强制重连：

```typescript
// 手动触发重连
sessionWebSocket.reconnect()
```

---

## 13.5 优势与不足

### 优势

**分层防御，各司其职**：每一层只处理自己能处理的错误。API 层的重试不影响 WebSocket 层；进程退出处理不影响 API 层的 AbortController。层间边界清晰。

**Jitter 防惊群效应**：API 重试的随机抖动是工业标准（AWS Architecture Blog 2015），防止大规模使用时多个客户端同时重试给服务器造成"重试波峰"。

**failsafe 保证最终一定退出**：即使所有清理步骤都挂起，failsafe 定时器保证进程最终退出，不会成为"僵尸进程"。

**孤儿进程检测**：macOS 关闭终端时不发 SIGHUP，这是经典坑。每 30s 检查 stdout/stdin 可写性，能够捕获这个场景。

**AbortController 树状传播**：父取消自动取消子，且用 `WeakRef` 避免内存泄漏，是对原生 API 的正确使用示范。

### 不足

**withRetry.ts 约 400 行，职责过多**：包含了指数退避、jitter、Token 刷新、Fast Mode 降级、模型回退、Context 溢出修复、无人值守模式等逻辑，阅读和维护难度高。按单一职责原则，这些应该拆分成多个独立模块。

**WebSocket 固定重试次数无法自定义**：`MAX_RECONNECT_ATTEMPTS = 5` 硬编码，在网络极不稳定的环境下可能不够。应该作为配置项暴露。

**没有熔断器（Circuit Breaker）模式**：当前的重试逻辑会一直重试直到 `maxRetries`。如果服务完全宕机，用户需要等待完整的重试时间（最长约 5 分钟），才能看到错误信息。熔断器可以在检测到多次失败后立即告知用户，而不是继续等待。

**gracefulShutdown 步骤是固定的**：如果某些场景需要跳过某些清理步骤（比如快速退出模式），无法配置。

---

## 13.6 在源码中找到这些逻辑

| 功能 | 文件 | 关键函数 |
|------|------|---------|
| API 重试核心 | `src/services/api/withRetry.ts` | `withRetry()` —— 约 400 行 |
| 退避延迟计算 | `src/services/api/withRetry.ts:530` | `getRetryDelay()` |
| 错误是否可重试 | `src/services/api/withRetry.ts` | `shouldRetry()` |
| 529 重试来源白名单 | `src/services/api/withRetry.ts:62` | `FOREGROUND_529_RETRY_SOURCES` |
| 模型回退错误 | `src/services/api/withRetry.ts:160` | `FallbackTriggeredError` |
| 连接错误详情提取 | `src/services/api/errorUtils.ts` | `extractConnectionErrorDetails()` |
| WebSocket 重连 | `src/remote/SessionsWebSocket.ts` | `onClose()`, `scheduleReconnect()` |
| 优雅退出主流程 | `src/utils/gracefulShutdown.ts` | `gracefulShutdown()` |
| 信号处理注册 | `src/utils/gracefulShutdown.ts:237` | `setupGracefulShutdown()` |
| 终端模式清理 | `src/utils/gracefulShutdown.ts:59` | `cleanupTerminalModes()` |
| 强制退出 | `src/utils/gracefulShutdown.ts:193` | `forceExit()` |
| 清理函数注册表 | `src/utils/cleanupRegistry.ts` | `registerCleanup()`, `runCleanupFunctions()` |
| AbortController 工具 | `src/utils/abortController.ts` | `createChildAbortController()` |
| AbortError 检测 | `src/utils/errors.ts` | `isAbortError()` |
| React 错误边界 | `src/components/SentryErrorBoundary.ts` | `getDerivedStateFromError()` |
| MCP 会话过期检测 | `src/services/mcp/client.ts` | `isMcpSessionExpiredError()` |
