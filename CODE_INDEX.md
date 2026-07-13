# 代码索引

> 本索引按主题分类列出所有重要的源码位置，方便直接定位到感兴趣的代码段。

---

## 入口与启动

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 程序主入口 | `src/main.tsx` | CLI 入口，启动 REPL 或 headless 模式 |
| 并行预热 | `src/main.tsx:1-30` | `startMdmRawRead()` + `startKeychainPrefetch()` |
| MDM 策略读取 | `src/utils/settings/mdm/rawRead.ts` | macOS plutil / Windows 注册表读取 |
| Keychain 预取 | `src/utils/secureStorage/keychainPrefetch.ts` | OAuth token + API Key 并行读取 |
| 启动性能打点 | `src/utils/startupProfiler.ts` | `profileCheckpoint()` + `profileReport()` |
| 工具初始化 | `src/tools.ts` | `getTools()` 按 feature flag 组装工具列表 |

---

## 核心引擎

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 会话容器（SDK 模式） | `src/QueryEngine.ts:184` | `QueryEngine` 类，约1300行 |
| QueryEngine 配置 | `src/QueryEngine.ts:130` | `QueryEngineConfig` 类型，所有可配置项 |
| 提交消息方法 | `src/QueryEngine.ts:209` | `submitMessage()` —— SDK 核心方法 |
| Agentic Loop 入口 | `src/query.ts` | `query()` 异步生成器，约1730行 |
| 循环主体 | `src/query.ts:241` | `queryLoop()` —— 实际的 while(true) 循环 |
| Token 警告状态 | `src/services/compact/autoCompact.ts:93` | `calculateTokenWarningState()` |

---

## 工具系统

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| Tool 接口定义 | `src/Tool.ts:362` | 约 40 个方法/属性的完整接口 |
| 工具构建工厂 | `src/Tool.ts:757` | `buildTool()` —— fail-safe 默认值注入 |
| 工具执行上下文 | `src/Tool.ts:180` | `ToolUseContext` 类型，约150行 |
| 工具并发调度 | `src/services/tools/toolOrchestration.ts` | `partitionToolCalls()` 批次分组 |
| 并发执行 | `src/services/tools/toolOrchestration.ts:152` | `runToolsConcurrently()` |
| GrepTool 完整实现 | `src/tools/GrepTool/GrepTool.ts:160` | 包含 ripgrep 参数组装 |
| BashTool | `src/tools/BashTool/BashTool.ts` | Shell 命令执行 |
| FileEditTool | `src/tools/FileEditTool/FileEditTool.ts` | 精确文件编辑 |
| FileReadTool | `src/tools/FileReadTool/FileReadTool.ts` | 文件内容读取 |
| AgentTool | `src/tools/AgentTool/AgentTool.ts` | 启动子 Agent |
| TodoWriteTool | `src/tools/TodoWriteTool/TodoWriteTool.ts` | V1 会话任务清单（全量替换） |
| ToolSearchTool | `src/tools/ToolSearchTool/ToolSearchTool.ts` | 延迟工具 schema 加载器 |
| SkillTool | `src/tools/SkillTool/SkillTool.ts` | slash-command 技能执行器（inline/fork 双模式）|
| TaskCreateTool | `src/tools/TaskCreateTool/TaskCreateTool.ts` | V2 创建持久化任务 |
| TaskGetTool | `src/tools/TaskGetTool/TaskGetTool.ts` | V2 按 ID 获取任务详情 |
| TaskListTool | `src/tools/TaskListTool/TaskListTool.ts` | V2 列出全部任务（过滤已解决依赖）|
| TaskUpdateTool | `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | V2 更新任务状态/属性/依赖 |
| TaskStopTool | `src/tools/TaskStopTool/TaskStopTool.ts` | 停止后台运行的进程任务 |
| TeamCreateTool | `src/tools/TeamCreateTool/TeamCreateTool.ts` | 创建多 Agent 协作团队 |
| TeamDeleteTool | `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | 解散团队并清理磁盘资源 |
| 延迟工具判定 | `src/tools/ToolSearchTool/prompt.ts` — `isDeferredTool()` | 判断某个工具是否需要延迟加载 |
| 团队辅助函数 | `src/utils/swarm/teamHelpers.ts` | team config 读写、目录清理 |
| 任务工具函数 | `src/utils/tasks.ts` | Task 的 CRUD、blockedBy 管理 |

---

## 权限系统

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 权限模式枚举 | `src/types/permissions.ts:16` | 5+2 种权限模式 |
| 权限规则类型 | `src/types/permissions.ts:75` | `PermissionRule` —— 工具+来源+行为 |
| 权限决策类型 | `src/types/permissions.ts:174` | allow / ask / deny 三种 |
| 权限上下文 | `src/types/permissions.ts:427` | `ToolPermissionContext` 不可变对象 |
| Bash 命令权限检查 | `src/utils/permissions/bash.ts` | 命令级别的规则匹配 |
| 文件系统权限检查 | `src/utils/permissions/filesystem.ts` | 工作目录边界检查 |
| 自动分类器结果 | `src/types/permissions.ts:346` | `YoloClassifierResult` —— 两阶段分类 |
| Hooks 执行 | `src/utils/hooks.ts` | `executePreToolUseHooks()` 等 |

---

## 上下文管理与压缩

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 自动压缩触发检查 | `src/services/compact/autoCompact.ts:160` | `shouldAutoCompact()` |
| 自动压缩阈值计算 | `src/services/compact/autoCompact.ts:62` | `getAutoCompactThreshold()` |
| 压缩是否启用 | `src/services/compact/autoCompact.ts:147` | `isAutoCompactEnabled()` |
| 压缩主流程 | `src/services/compact/compact.ts` | `compactConversation()` |
| 图片剥离 | `src/services/compact/compact.ts:145` | `stripImagesFromMessages()` |
| Token 计数（精确） | `src/utils/tokens.ts` | `tokenCountFromLastAPIResponse()` |
| Token 计数（估算） | `src/utils/tokens.ts` | `tokenCountWithEstimation()` |
| 压缩边界消息 | `src/utils/messages.ts` | `createCompactBoundaryMessage()` |
| 文件恢复预算 | `src/services/compact/compact.ts:122` | `POST_COMPACT_MAX_FILES_TO_RESTORE = 5` |
| 工具结果预算 | `src/query.ts:369` | `applyToolResultBudget()` |

---

## MCP 集成

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| MCP 客户端主文件 | `src/services/mcp/client.ts` | 约3400行，MCP 连接和工具调用 |
| 连接管理器 | `src/services/mcp/MCPConnectionManager.tsx` | React 组件管理连接生命周期 |
| 支持的传输类型 | `src/services/mcp/client.ts:8-17` | stdio / SSE / HTTP / WebSocket |
| OAuth 认证 | `src/services/mcp/auth.ts` | MCP Server 的 OAuth 流程 |
| MCP 工具包装 | `src/tools/MCPTool/MCPTool.ts` | 将 MCP 工具适配为 Claude Code 工具 |
| 工具结果截断 | `src/utils/mcpValidation.ts` | `truncateMcpContentIfNeeded()` |
| 二进制内容存储 | `src/utils/mcpOutputStorage.ts` | `persistBinaryContent()` |
| 环境变量展开 | `src/services/mcp/envExpansion.ts` | 支持 Keychain、1Password 引用 |
| Elicitation 处理 | `src/services/mcp/elicitationHandler.ts` | MCP Server 向用户请求输入 |
| 官方服务器目录 | `src/services/mcp/officialRegistry.ts` | Anthropic 认可的 MCP Server 列表 |

---

## 会话持久化

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 会话存储主文件 | `src/utils/sessionStorage.ts` | 约5100行，所有持久化逻辑 |
| 追加消息到日志 | `src/utils/sessionStorage.ts:77` | `appendMessageToTranscript()` |
| 获取日志路径 | `src/utils/sessionStorage.ts` | `getTranscriptPath()` |
| 日志 Entry 类型 | `src/types/logs.ts` | `Entry` union 类型 |
| 文件快照类型 | `src/types/logs.ts` | `FileHistorySnapshotMessage` |
| 恢复会话命令 | `src/commands/resume/` | `/resume` 命令实现 |
| Rewind 命令 | `src/commands/rewind/` | `/rewind` 撤销操作 |
| 会话统计命令 | `src/commands/stats/` | `/stats` 分析会话 |

---

## 多智能体

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| AgentTool | `src/tools/AgentTool/AgentTool.ts` | "工具即 Agent"的核心实现 |
| Explore Agent 定义 | `src/tools/AgentTool/built-in/exploreAgent.ts:64` | 只读代码探索专家 |
| 通用 Agent 定义 | `src/tools/AgentTool/built-in/generalPurposeAgent.ts:25` | 全能执行者 |
| 本地 Agent 任务 | `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 本机子 Agent 运行时 |
| 远程 Agent 任务 | `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 跨网络子 Agent |
| 查询链追踪 | `src/Tool.ts:90` | `QueryChainTracking` —— chainId + depth |
| Agent 加载目录 | `src/tools/AgentTool/loadAgentsDir.ts` | 发现用户自定义 Agent |

---

## Feature Flags（关键）

| Flag 名 | 功能 | 所在文件 |
|---------|------|----------|
| `COORDINATOR_MODE` | 多 Agent 协调器模式 | `src/main.tsx` |
| `KAIROS` | 助手（会话摘要）模式 | `src/main.tsx` |
| `HISTORY_SNIP` | 历史微压缩 | `src/QueryEngine.ts:122` |
| `CONTEXT_COLLAPSE` | 上下文折叠功能 | `src/query.ts` |
| `REACTIVE_COMPACT` | 响应式压缩（只压缩 API 报错后）| `src/services/compact/autoCompact.ts` |
| `PROACTIVE` | 主动任务建议 | `src/tools.ts` |
| `AGENT_TRIGGERS` | 定时 Agent 触发器 | `src/tools.ts` |
| `TRANSCRIPT_CLASSIFIER` | 自动权限分类 | `src/types/permissions.ts:35` |
| `STREAMING_TOOL_EXECUTION` | 流式工具执行 | `src/query.ts:561` |
| `CACHED_MICROCOMPACT` | 缓存感知的微压缩 | `src/query.ts:423` |

---

## UI 组件（ink/React TUI）

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| 权限对话框 | `src/components/permissions/` | 各类工具的确认 UI |
| Bash 权限 | `src/components/permissions/BashPermissionRequest/` | Bash 命令确认 |
| 文件编辑权限 | `src/components/permissions/FileEditPermissionRequest/` | 文件修改确认 |
| Prompt 输入框 | `src/components/PromptInput/` | 用户输入组件 |
| 设置面板 | `src/components/Settings/` | 设置页面 |
| 结构化 Diff | `src/components/StructuredDiff/` | 文件变更展示 |
| 代码高亮 | `src/components/HighlightedCode/` | 代码块高亮渲染 |
| MCP 管理 | `src/components/mcp/` | MCP 服务器管理 UI |
| 技能管理 | `src/components/skills/` | Skill 列表和详情 |

---

## 实用工具

| 功能 | 文件路径 |
|------|----------|
| 路径工具 | `src/utils/path.ts` |
| 文件工具 | `src/utils/file.ts` |
| 错误处理 | `src/utils/errors.ts` |
| 日志记录 | `src/utils/log.ts` |
| 配置读取 | `src/utils/config.ts` |
| ripgrep 包装 | `src/utils/ripgrep.ts` |
| 延迟 Schema | `src/utils/lazySchema.ts` |
| Semantic 数字/布尔 | `src/utils/semanticNumber.ts`, `src/utils/semanticBoolean.ts` |
| 消息工具函数 | `src/utils/messages.ts` |
| Token 估算 | `src/services/tokenEstimation.ts` |

---

## 状态管理（第13章）

| 功能 | 文件路径 | 何时查看 |
|------|----------|---------|
| Store 基础实现（35行）| `src/state/store.ts` | 理解发布-订阅核心时 |
| AppState 类型定义（约570行）| `src/state/AppStateStore.ts` | 了解全局状态有哪些字段时 |
| 默认初始值 | `src/state/AppStateStore.ts` — `getDefaultAppState()` | 理解初始状态来源时 |
| Provider + 订阅/更新 Hooks | `src/state/AppState.tsx` | 理解 React 如何连接 Store 时 |
| 副作用集中处理 | `src/state/onChangeAppState.ts` — `onChangeAppState()` | 追查状态变更引发了哪些副作用时 |
| 派生状态（计算属性）| `src/state/selectors.ts` | 需要从 AppState 计算复合值时 |
| Slash 命令建议层（读写分离）| `src/context/promptOverlayContext.tsx` | 学习"分离读写 Context"模式时 |

---

## 5层安全体系（第15章）

| 功能 | 文件路径 | 何时查看 |
|------|----------|---------|
| Layer 1 核心安全提示 | `src/constants/prompts.ts` — `getActionsSection()` | 理解 AI 的行为约束时 |
| Layer 1 Auto Mode 安全约束 | `src/utils/messages.ts` — `getAutoModeFullInstructions()` | 排查自动模式是否会泄露信息时 |
| Layer 2 权限上下文（不可变） | `src/types/permissions.ts:427` — `ToolPermissionContext` | 理解权限不可变设计时 |
| Layer 2 Bash 命令 AST 安全分析 | `src/utils/bash/ast.ts` | 排查为何某条 bash 命令被拦截时 |
| Layer 2 文件系统边界检查 | `src/utils/permissions/filesystem.ts` | 排查路径越界访问时 |
| Layer 3 沙箱适配层 | `src/utils/sandbox/sandbox-adapter.ts` | 了解沙箱与 Claude CLI 集成时 |
| Layer 3 沙箱配置转换 | `src/utils/sandbox/sandbox-adapter.ts` — `convertToSandboxRuntimeConfig()` | 自定义文件系统/网络约束时 |
| Layer 3 防 Git 沙箱逃逸 | `src/utils/sandbox/sandbox-adapter.ts:257` | 理解 bareGitRepo 攻击向量防护时 |
| Layer 4 进入/退出计划模式 | `src/tools/EnterPlanModeTool/`, `src/tools/ExitPlanModeTool/` | 理解 Plan Mode 工具实现时 |
| Layer 4 Plan Mode 强制提示词 | `src/utils/messages.ts` — `getPlanModeV2Instructions()` | 了解 Plan Mode 如何约束 AI 行为时 |
| Layer 5 Hooks 执行引擎 | `src/utils/hooks.ts` — `executePreToolUseHooks()` | 排查 Hook 脚本不触发时 |
| Layer 5 成本上限检查 | `src/QueryEngine.ts:972` | 排查会话为何突然终止时 |
| Layer 5 Token 预算解析 | `src/utils/tokenBudget.ts` — `parseTokenBudget()` | 了解 token 预算语法时 |

---

## 错误处理与恢复（第14章）

| 功能 | 文件路径 | 何时查看 |
|------|----------|---------|
| API 重试核心引擎 | `src/services/api/withRetry.ts` — `withRetry()` | 排查 API 调用为何一直重试/失败时 |
| 退避延迟计算（jitter）| `src/services/api/withRetry.ts:530` — `getRetryDelay()` | 理解重试间隔如何计算时 |
| 529 重试来源白名单 | `src/services/api/withRetry.ts:62` — `FOREGROUND_529_RETRY_SOURCES` | 添加新查询源时判断是否应重试 529 |
| 模型回退错误类型 | `src/services/api/withRetry.ts:160` — `FallbackTriggeredError` | 调试 Opus 过载自动切换 Sonnet 时 |
| 连接错误详情提取 | `src/services/api/errorUtils.ts` — `extractConnectionErrorDetails()` | 区分 SSL/TCP 错误时 |
| WebSocket 重连逻辑 | `src/remote/SessionsWebSocket.ts` — `onClose()` | 排查远程会话自动断开时 |
| 优雅退出主流程 | `src/utils/gracefulShutdown.ts` — `gracefulShutdown()` | 理解 Ctrl+C 后发生了什么时 |
| 信号处理注册 | `src/utils/gracefulShutdown.ts:237` — `setupGracefulShutdown()` | 了解支持哪些退出信号时 |
| 终端模式清理序列 | `src/utils/gracefulShutdown.ts:59` — `cleanupTerminalModes()` | 排查退出后终端乱码问题时 |
| 清理函数注册表 | `src/utils/cleanupRegistry.ts` — `registerCleanup()` | 给新模块添加退出清理逻辑时 |
| AbortController 父子级联 | `src/utils/abortController.ts` — `createChildAbortController()` | 实现可取消的长时间操作时 |
| AbortError 统一检测 | `src/utils/errors.ts` — `isAbortError()` | catch 块里区分取消和真实错误时 |
| React 错误边界 | `src/components/SentryErrorBoundary.ts` | 了解 TUI 组件崩溃防护时 |
