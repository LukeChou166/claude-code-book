# 第12章 状态管理：一个极简自研 Store

> **前置知识**：第1章 整体架构与启动、第3章 Agentic Loop  
> **关键文件**：`src/state/store.ts`、`src/state/AppStateStore.ts`、`src/state/AppState.tsx`、`src/state/onChangeAppState.ts`、`src/state/selectors.ts`

---

## 12.1 解决什么问题

任何有界面的应用都面临同一个挑战：**数据在哪里，谁来更新，谁来读取，更新后如何通知界面刷新？**

Claude Code 的界面是基于 [Ink](https://github.com/vadimdemedes/ink) 构建的终端 TUI，本质上是 React 在终端里跑。它面临的具体问题是：

1. **多组件共享数据**：权限模式、MCP 连接状态、任务列表，需要在几十个组件之间共享
2. **非 React 代码也需要读写状态**：Bash 工具执行器、API 调用层、信号处理器都需要读写状态，它们不在 React 的渲染树里
3. **高频更新不能级联重渲染**：一个工具每秒输出几十行文本，如果每次都让整个组件树重渲染，性能会崩溃
4. **状态变更要触发副作用**：权限模式改变时要通知远程服务器，模型配置改变时要持久化到磁盘

---

## 12.2 业界的做法

| 方案 | 代表 | 优点 | 缺点 |
|------|------|------|------|
| **Redux** | React 生态标配 | 可追踪、DevTools 好用、中间件生态 | 概念多（action/reducer/dispatch）、样板代码多 |
| **Zustand** | 轻量状态库 | 简单、支持 selector 优化、可在 React 外使用 | 需要引入依赖 |
| **Jotai/Recoil** | 原子化状态 | 细粒度更新 | 原子颗粒度设计复杂、依赖图难维护 |
| **React Context** | React 内置 | 零依赖 | Context 值变更会让所有消费者重渲染，无法做 selector 优化 |
| **MobX** | 响应式 | 自动追踪依赖 | 魔法太多、难以理解更新来源 |

Claude Code 选择了**自研极简 Store**，核心思想借鉴自 Zustand：用发布-订阅模式封装一个 Store，通过 `useSyncExternalStore` 钩子连接 React，通过 selector 实现精准订阅。

---

## 12.3 本项目的做法（deep）

### 12.3.1 最底层：35 行的 Store 实现

```1:34:restored-src/src/state/store.ts
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // ← 关键：引用不变则不通知
      state = next
      onChange?.({ newState: next, oldState: prev })  // ← 副作用钩子
      for (const listener of listeners) listener()    // ← 通知所有订阅者
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)  // ← 返回取消订阅函数
    },
  }
}
```

这段代码只有 35 行，却包含了三个关键设计：

1. **`Object.is` 幂等检查**：如果 `setState(prev => prev)` 没有真正改变状态，直接返回，不触发任何通知。C++ 中类似 `if (newVal == oldVal) return;`
2. **`onChange` 副作用钩子**：每次状态真正变更后，先调用 `onChange`（同步），再通知 React 组件（同步）。这是状态持久化和远程同步的入口
3. **`subscribe` 返回取消函数**：调用者拿到取消函数后，在组件卸载时调用它即可，无需记住当初传入的 listener 引用。C++ 中类似 `std::function<void()> unsubscribe = eventBus.on(event, handler);`

### 12.3.2 AppState：全局状态的数据结构

`AppState` 是整个应用的"数据库"，定义在 `src/state/AppStateStore.ts` 中，是一个扁平的大对象。

**设计特点**：使用 `DeepImmutable<{...}>` 包装，确保类型系统强制不可变性：

```89:98:restored-src/src/state/AppStateStore.ts
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  // Optional - only present when ENABLE_AGENT_SWARMS is true
  showTeammateMessagePreview?: boolean
```

以下是核心字段分组（从源码归纳）：

| 分组 | 代表字段 | 说明 |
|------|---------|------|
| **运行时配置** | `settings`, `verbose`, `mainLoopModel` | 用户配置，会持久化到磁盘 |
| **UI 状态** | `expandedView`, `footerSelection`, `statusLineText` | 界面展示状态，不持久化 |
| **任务系统** | `tasks`, `foregroundedTaskId`, `viewingAgentTaskId` | 当前运行的任务 |
| **权限** | `toolPermissionContext` | 当前权限模式和规则（详见第11章） |
| **Bridge/远程** | `replBridgeEnabled`, `remoteConnectionStatus`, 10 个 `replBridge*` 字段 | Claude.ai 网页连接状态 |
| **工具生态** | `mcp.clients`, `mcp.tools`, `plugins.enabled` | MCP 和插件注册表 |
| **推测执行** | `speculation` | AI 预测下一步操作的状态（`idle`/`active`） |
| **通知** | `notifications.current`, `notifications.queue` | 用户消息通知队列 |

**`tasks` 字段为什么在 `DeepImmutable` 外面？**

```158:163:restored-src/src/state/AppStateStore.ts
}> & {
  // Unified task state - excluded from DeepImmutable because TaskState contains function types
  tasks: { [taskId: string]: TaskState }
  // Name → AgentId registry populated by Agent tool when `name` is provided.
  agentNameRegistry: Map<string, AgentId>
```

因为 `TaskState` 包含函数类型（如 `kill: () => void`），而 `DeepImmutable` 会把函数也变成只读，导致类型错误。所以 `tasks` 和 `agentNameRegistry` 被排除在深度不可变之外。

### 12.3.3 Provider 层：React 的连接桥梁

`AppStateProvider` 是整个状态树的根，定义在 `src/state/AppState.tsx`：

```typescript
// src/state/AppState.tsx（简化）
export function AppStateProvider({ children, initialState, onChangeAppState }) {
  // 关键：Store 只创建一次，引用永远不变
  // 这样 <AppStoreContext.Provider value={store}> 的 value 永远不变
  // Provider 从不触发重渲染
  const [store] = useState(() => 
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )
  
  return (
    <HasAppStateContext.Provider value={true}>
      <AppStoreContext.Provider value={store}>
        {children}
      </AppStoreContext.Provider>
    </HasAppStateContext.Provider>
  )
}
```

`HasAppStateContext` 是一个防护层：如果有人不小心嵌套了两层 `AppStateProvider`，会立即抛出错误：

```typescript
// 检测到嵌套时
throw new Error("AppStateProvider can not be nested within another AppStateProvider")
```

### 12.3.4 消费状态：useAppState + selector 精准订阅

这是整个状态管理中最精妙的部分——通过 selector 实现"只有我用到的字段变化时，我才重渲染"：

```typescript
// src/state/AppState.tsx（简化）
export function useAppState(selector) {
  const store = useAppStore()
  // useSyncExternalStore 是 React 18 提供的外部 store 订阅钩子
  // 第二个参数是"当前值的获取器"，React 在每次 re-render 时调用它
  // 如果返回值和上次 Object.is 相等，React 跳过重渲染
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState()),
  )
}
```

**调用示例与效果对比**：

```typescript
// 示例 1：订阅单个字段
// 只有 verbose 变化时重渲染，其他所有字段变化时不重渲染
const verbose = useAppState(s => s.verbose)

// 示例 2：订阅多个独立字段（正确做法）
const model = useAppState(s => s.mainLoopModel)
const mode = useAppState(s => s.toolPermissionContext.mode)

// 示例 3：错误做法！每次调用都返回新对象 → Object.is 永远不等 → 死循环重渲染
// const data = useAppState(s => ({ a: s.verbose, b: s.model })) // ← 禁止！

// 示例 4：订阅子对象引用（正确，因为返回的是已有引用）
const { text, promptId } = useAppState(s => s.promptSuggestion) // good
```

**C++ 类比**：

| TypeScript 写法 | C++ 等价 |
|----------------|---------|
| `useSyncExternalStore(store.subscribe, getSnapshot)` | `观察者模式 + 脏检查` |
| `selector(state)` | `std::function<T(AppState)> selector` |
| `Object.is(prev, next)` | `prev == next`（对象地址比较或值比较） |

### 12.3.5 更新状态：useSetAppState

```typescript
// src/state/AppState.tsx
export function useSetAppState() {
  return useAppStore().setState
}

// 使用示例
const setAppState = useSetAppState()

// 更新单个字段
setAppState(prev => ({ ...prev, verbose: true }))

// 更新嵌套字段（注意：需要展开每一层）
setAppState(prev => ({
  ...prev,
  mcp: {
    ...prev.mcp,
    tools: [...prev.mcp.tools, newTool]
  }
}))
```

这里使用**函数式更新**（传 `prev => newState`），而不是直接传新值，原因是：

1. 避免闭包里的过期值（C++ 中的 stale capture）
2. 支持异步场景：两个并发的 `setAppState` 调用，第二个能看到第一个的结果

### 12.3.6 副作用集中管理：onChangeAppState

每次 AppState 真正变更后，`onChangeAppState` 都会被同步调用。这是整个应用**副作用的单一入口**：

```43:171:restored-src/src/state/onChangeAppState.ts
export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}) {
  // 权限模式变更 → 通知远程服务器（CCR/SDK）
  const prevMode = oldState.toolPermissionContext.mode
  const newMode = newState.toolPermissionContext.mode
  if (prevMode !== newMode) {
    // ...
    notifySessionMetadataChanged({ permission_mode: newExternal })
    notifyPermissionModeChanged(newMode)
  }

  // 模型配置变更 → 持久化到 settings.json
  if (newState.mainLoopModel !== oldState.mainLoopModel && newState.mainLoopModel !== null) {
    updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
    setMainLoopModelOverride(newState.mainLoopModel)
  }

  // expandedView 变更 → 持久化到全局配置（兼容旧字段名）
  if (newState.expandedView !== oldState.expandedView) {
    const showExpandedTodos = newState.expandedView === 'tasks'
    const showSpinnerTree = newState.expandedView === 'teammates'
    saveGlobalConfig(current => ({ ...current, showExpandedTodos, showSpinnerTree }))
  }

  // settings 变更 → 清除认证相关缓存（让新配置立即生效）
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
    clearGcpCredentialsCache()
    // settings.env 变更 → 重新应用环境变量（加法操作，不删除已有的）
    if (newState.settings.env !== oldState.settings.env) {
      applyConfigEnvironmentVariables()
    }
  }
}
```

**为什么把副作用集中在这里？**

注释里写得很清楚（第 50-64 行）：在此之前，权限模式的变更分散在 8 个以上的调用路径里，只有其中 2 个会通知远程服务器。每次新增一个权限变更路径，开发者都需要记得在那里手动发通知，容易遗漏。

现在把副作用集中在 `onChangeAppState` 里，**无论哪个路径触发了状态变更，副作用都会自动执行**。这是"响应式编程"的精髓——不追踪"谁改了"，而是追踪"改成了什么"。

### 12.3.7 非 React 代码如何访问状态

工具执行器（如 BashTool）、API 调用层等非 React 代码，通过 `useAppStateStore()` 获取 Store 对象本身，然后传递给非 React 代码：

```typescript
// React 组件里获取 store
const store = useAppStateStore()

// 传给非 React 函数
await bashTool.execute(command, {
  getAppState: store.getState,
  setAppState: store.setState,
})

// 非 React 代码直接操作状态
function handleBashResult(result, { getAppState, setAppState }) {
  const currentState = getAppState()  // 直接读取，无 hooks
  setAppState(prev => ({ ...prev, ... }))  // 直接更新，无 hooks
}
```

### 12.3.8 局部 Context 状态

除了全局 `AppState`，项目还有若干局部 Context，用于隔离只有少数组件需要的状态：

| Context 文件 | 用途 | 为什么不放 AppState |
|------------|------|------------------|
| `context/mailbox.tsx` | Agent 间消息传递（邮箱） | 生命周期短，只在多 Agent 场景用 |
| `context/voice.tsx` | 语音录制/处理状态 | ant-only feature，条件编译排除 |
| `context/stats.tsx` | 性能计数器/直方图 | 纯诊断用，不影响业务逻辑 |
| `context/modalContext.tsx` | 弹窗尺寸信息 | 纯 UI 计算，无业务意义 |
| `context/fpsMetrics.tsx` | FPS 数据获取器 | 工具函数性质，不是状态 |
| `context/promptOverlayContext.tsx` | Slash 命令建议层 | 专意为读写分离设计 |
| `ink/components/TerminalFocusContext.tsx` | 终端焦点 | Ink 内部逻辑 |

**PromptOverlayContext 的读写分离**是一个值得注意的模式：

```typescript
// 将 data 和 setter 放在不同的 Context
// setter 的 Context 永远不会因数据变化而触发重渲染
// 只有真正需要显示数据的组件才订阅 data Context
const PromptOverlayDataContext = createContext(null)
const PromptOverlaySetterContext = createContext(null)
```

---

## 12.4 优势与不足

### 优势

**零依赖、极简实现**：核心 Store 35 行代码，无需 Redux、Zustand 等第三方包。可以精准控制每个细节，不受第三方库版本升级和 breaking change 的影响。

**Selector 精准订阅**：通过 `Object.is` 比较 selector 返回值，组件只在自己关心的字段变化时重渲染。这是 Zustand 的核心优势，Claude Code 自己实现了一遍，且完全理解每一行的含义。

**非 React 代码无缝集成**：工具执行器、API 调用层都是普通函数，通过 `store.getState`/`store.setState` 直接操作状态，不需要 React hooks 的上下文环境。这在 Redux 里需要 `getState()`/`dispatch()`，在 Zustand 里用 `useStore.getState()`，原理相同。

**副作用集中管理**：`onChangeAppState` 是状态变更副作用的单一入口，消除了"某个路径忘记发通知"的 bug。

**强制不可变性**：`DeepImmutable` 类型包装让 TypeScript 编译器帮你发现试图直接修改状态的错误，在编译期就消灭 bug。

### 不足

**AppState 是"上帝对象"**：约 570 行类型定义，把权限、任务、MCP、Bridge、插件、UI 状态全放在一起。任何新功能都倾向于往这里加字段，随着时间推移会越来越臃肿（已经可以看到这个趋势：computer use 状态、tmux 状态、REPL VM 上下文……）。

**没有时间旅行调试**：Redux 最大的优势是 DevTools 可以录放每一次 action，帮助复现 bug。自研的 Store 没有这个能力，调试状态问题只能靠 console.log。

**无法做到细粒度原子化**：`tasks` 字段是 `{ [taskId: string]: TaskState }` 的大对象。如果某个任务的状态高频更新，所有订阅了 `tasks` 的组件都会重渲染，即使它们只关心某个特定的 taskId。Jotai 的原子化设计可以解决这个问题。

**`onChangeAppState` 是同步瓶颈**：所有副作用都在状态变更后同步执行。如果某个副作用耗时（如写文件），会阻塞后续的组件重渲染。更好的做法是用 `requestAnimationFrame` 或 `useEffect` 做异步副作用。

---

## 12.5 在源码中找到这些逻辑

| 功能 | 文件 | 关键函数/行号 |
|------|------|-------------|
| Store 基础实现 | `src/state/store.ts` | `createStore()` —— 完整 35 行 |
| AppState 类型定义 | `src/state/AppStateStore.ts` | `AppState` 类型（约570行） |
| 默认初始值 | `src/state/AppStateStore.ts` | `getDefaultAppState()` |
| Provider 实现 | `src/state/AppState.tsx` | `AppStateProvider` |
| 订阅 Hook | `src/state/AppState.tsx` | `useAppState(selector)` |
| 更新 Hook | `src/state/AppState.tsx` | `useSetAppState()` |
| 非 React 访问 | `src/state/AppState.tsx` | `useAppStateStore()` |
| 副作用处理 | `src/state/onChangeAppState.ts` | `onChangeAppState()` |
| 外部状态同步回 AppState | `src/state/onChangeAppState.ts` | `externalMetadataToAppState()` |
| 派生状态（计算属性） | `src/state/selectors.ts` | 各 `selectXxx` 函数 |
| Voice 状态（ant-only） | `src/context/voice.tsx` | `VoiceProvider`, `useVoiceState(selector)` |
| Slash 命令建议层 | `src/context/promptOverlayContext.tsx` | 读写分离 Context |
