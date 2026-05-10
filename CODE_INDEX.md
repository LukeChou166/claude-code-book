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
