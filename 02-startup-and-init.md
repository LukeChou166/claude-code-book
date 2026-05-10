# 第2章 启动流程与初始化

> **前置知识**：第1章整体架构  
> **关键文件**：`src/main.tsx`、`src/entrypoints/init.ts`、`src/utils/secureStorage/keychainPrefetch.ts`

---

## 2.1 解决什么问题

"启动"看起来简单，但一个成熟的编程助手在接受第一条用户消息之前，需要做大量准备工作：

1. **加载用户配置**：API Key、偏好设置、工作目录的 `CLAUDE.md` 记忆文件、MCP 服务器地址
2. **身份验证**：OAuth 登录 / API Key 有效性检查
3. **预取外部数据**：GrowthBook 功能开关（A/B 测试）、企业 MDM 策略
4. **扫描工作环境**：当前 git 仓库、项目类型识别
5. **连接外部服务**：所有配置的 MCP 服务器
6. **初始化工具系统**：按 feature flag 决定启用哪些工具

这些操作有些可以并行，有些有依赖顺序，有些会在 macOS/Windows 上特别慢（Keychain、注册表读取）。不好的设计会导致冷启动超过 1 秒，让用户体验变差。

---

## 2.2 业界的做法

**VS Code 插件**的启动通常是：
1. 同步读取 `settings.json`（阻塞）
2. 按需懒加载功能模块（第一次使用时才加载）
3. 通过 Extension Host 进程隔离插件，避免主进程阻塞

这种做法的问题：
- 大量"懒加载"推迟到第一次使用时，造成**首次响应延迟**（用户感知到明显的卡顿）
- Keychain/Credential Manager 读取如果在同步路径上，会阻塞整个 UI 线程

**Node.js 传统 CLI** 工具（如 ESLint、TypeScript CLI）通常是顺序初始化：
```
读配置 → 解析参数 → 执行
```
简单，但没有利用并行机会。

---

## 2.3 本项目的做法：并行预热（Prefetch First）

`main.tsx` 的最开头几行揭示了整个启动策略：

```typescript
// src/main.tsx 第 1-20 行
// These side-effects must run before all other imports:
// 1. profileCheckpoint marks entry before heavy module evaluation begins
// 2. startMdmRawRead fires MDM subprocesses (plutil/reg query) so they run in
//    parallel with the remaining ~135ms of imports below
// 3. startKeychainPrefetch fires both macOS keychain reads (OAuth + legacy API
//    key) in parallel — isRemoteManagedSettingsEligible() otherwise reads them
//    sequentially via sync spawn inside applySafeConfigEnvironmentVariables()
//    (~65ms on every macOS startup)
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';

profileCheckpoint('main_tsx_entry');
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';

startMdmRawRead();
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';

startKeychainPrefetch();
```

注释里写得非常清楚，这里有三层设计：

### 设计一：启动性能打点（profileCheckpoint）

```typescript
profileCheckpoint('main_tsx_entry');
// ... 初始化工作 ...
profileCheckpoint('init_complete');
profileReport(); // 输出各阶段耗时
```

`startupProfiler.ts` 记录每个阶段的时间戳，用于监控启动性能回归。在生产版本中可以通过环境变量关闭，不影响正常使用。

### 设计二：MDM 策略并行读取

```typescript
startMdmRawRead();
```

MDM（Mobile Device Management）是企业设备管理的策略系统。在 macOS 上，读取 MDM 策略需要调用 `plutil` 子进程；在 Windows 上需要读取注册表（`reg query`）。这些操作耗时约 50-100ms，且完全独立于其他初始化工作。

`startMdmRawRead()` 在后台启动这个子进程，然后立即返回。后续需要 MDM 策略时（比如检查是否允许某个功能），再 `await` 等待结果。

在模块加载期间（约 135ms 的 TypeScript import 链），MDM 读取已经在后台跑了很长时间，等到真正需要结果时，几乎不需要额外等待。

### 设计三：Keychain 并行预取

```typescript
startKeychainPrefetch();
```

macOS Keychain 读取需要调用 `security` 命令（系统二进制），是同步 spawn——如果串行读取两个 key（OAuth token + 旧版 API Key），需要 65ms × 2 = 130ms。

`startKeychainPrefetch()` 立即并行启动两次读取：
- 读取 OAuth token（用于新版认证）
- 读取旧版 API Key（用于向后兼容）

后续需要时调用 `ensureKeychainPrefetchCompleted()`，由于已经并行启动，实际等待时间只有 65ms（最慢那个的时间），而不是 130ms。

**节省时间：约 65ms**（在每次 macOS 启动时）

---

## 2.4 Feature Flags 与 Dead-Code Elimination

```typescript
// src/main.tsx 第 74-82 行
const coordinatorModeModule = feature('COORDINATOR_MODE') 
  ? require('./coordinator/coordinatorMode.js') 
  : null;

const assistantModule = feature('KAIROS') 
  ? require('./assistant/index.js') 
  : null;
```

`COORDINATOR_MODE` 是多智能体协调器，`KAIROS` 是助手模式。

这里有两层好处：

**编译时**：Bun 构建时如果 `COORDINATOR_MODE = false`，整个 `require('./coordinator/coordinatorMode.js')` 表达式及其依赖会被从打包产物中删除——外部用户拿到的二进制文件里甚至不存在这些代码，减小包体积。

**运行时**：即使某个 feature flag 在编译时保留了代码，`feature('FLAG')` 返回 `null` 时，模块不会被 `require` 加载，节省内存和启动时间。

---

## 2.5 完整启动流程图

```
main.tsx 入口
    │
    ├─[并行]─ startMdmRawRead()        // 后台读 MDM 策略
    ├─[并行]─ startKeychainPrefetch()  // 后台读 Keychain
    │
    ├── 解析命令行参数 (Commander.js)
    │
    ├── init()  ←── entrypoints/init.ts
    │   ├── initializeWarningHandler()        // 处理 Node 警告
    │   ├── applyConfigEnvironmentVariables() // 应用配置环境变量
    │   ├── loadRemoteManagedSettings()       // 企业托管设置（等待 MDM）
    │   ├── loadPolicyLimits()                // 功能策略限制
    │   ├── initializeGrowthBook()            // A/B 测试框架
    │   └── ensureKeychainPrefetchCompleted() // 等待 Keychain 预取
    │
    ├── 身份验证检查
    │   ├── checkHasTrustDialogAccepted()    // 首次运行：显示信任对话框
    │   ├── validateForceLoginOrg()          // 企业 SSO 验证
    │   └── OAuth / API Key 有效性检查
    │
    ├── 加载工具系统
    │   └── getTools(config)                 // 按 feature flag 组装工具列表
    │
    ├── 连接 MCP 服务器
    │   └── 并行连接所有配置的 MCP servers
    │       ├── stdio 传输（子进程）
    │       ├── SSE 传输（HTTP 长连接）
    │       └── WebSocket 传输
    │
    ├── 预取其他数据（并行）
    │   ├── prefetchFastModeStatus()         // 快速模式预取
    │   ├── prefetchPassesEligibility()      // Claude Pro 资格检查
    │   └── prefetchOfficialMcpUrls()        // 官方 MCP 服务器列表
    │
    └── 启动 REPL
        └── launchRepl() → ink React 渲染
```

---

## 2.6 工具初始化：getTools()

工具系统在启动时被集中初始化。`tools.ts` 的 `getTools()` 函数按 feature flag 和用户类型决定哪些工具可用：

```typescript
// tools.ts（简化）
export function getTools(config: ToolsConfig): Tools {
  const tools: Tool[] = [
    BashTool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    GlobTool,
    GrepTool,
    // ... 约 40 个工具
  ]

  // 仅 ant（Anthropic 内部）用户可用的工具
  if (process.env.USER_TYPE === 'ant') {
    if (REPLTool) tools.push(REPLTool)
    if (SuggestBackgroundPRTool) tools.push(SuggestBackgroundPRTool)
  }

  // 仅特定 feature flag 开启时可用
  if (SleepTool) tools.push(SleepTool)
  if (cronTools.length) tools.push(...cronTools)
  
  // MCP 动态工具（从连接的 MCP 服务器获取）
  tools.push(...mcpTools)

  return uniqBy(tools, t => t.name)
}
```

注意 `uniqBy(tools, t => t.name)` 去重，防止重名工具导致 AI 调用混乱。

---

## 2.7 CLAUDE.md 记忆文件加载

启动时会扫描工作目录下的 `CLAUDE.md` 文件（以及父目录中的同名文件），作为"项目记忆"注入到系统提示：

```typescript
// utils/memdir/memdir.ts（简化）
export async function loadMemoryPrompt(cwd: string): Promise<string> {
  // 查找 CLAUDE.md 文件的路径层级
  const paths = [
    path.join(cwd, 'CLAUDE.md'),           // 项目级
    path.join(homedir(), 'CLAUDE.md'),     // 用户全局级
    // ... 还有中间目录
  ]
  
  const contents = await Promise.all(
    paths.map(p => readFile(p).catch(() => null))
  )
  
  return contents.filter(Boolean).join('\n\n---\n\n')
}
```

`CLAUDE.md` 的内容会附加到系统提示末尾，告诉 AI 项目的特定约定（比如"这个项目用 bun 不用 npm"、"所有提交必须通过 lint 检查"）。

---

## 2.8 本项目的优势与不足

### 优势

**并行预热设计**减少冷启动延迟：
- MDM 读取（50-100ms）在模块加载期间并行进行
- Keychain 两次读取并行化，节省约 65ms
- GrowthBook、MCP 服务器连接等都是异步非阻塞的

**Bun DCE** 让不同用户版本从同一代码库构建，无需维护多个分支。

**启动性能打点**（`profileCheckpoint`）让团队能监控和优化启动路径，避免性能回归。

### 不足

**MCP 服务器连接阻塞**：启动时需要等待所有 MCP 服务器连接完成（或超时）。如果用户配置了慢速/不稳定的 MCP 服务器（比如需要 SSH 隧道的远程服务器），整个启动会卡住。

当前解决方案是设置连接超时，但超时期间用户看到的是一个挂起的进度提示，体验不友好。更好的方案是"乐观启动"——先启动 REPL，MCP 服务器在后台连接，连接成功后再更新工具列表。

**身份验证逻辑散布**：OAuth、API Key、企业 SSO、MDM 托管设置分散在多个文件中，没有统一的身份验证模块，对新贡献者理解成本高。

**首次运行初始化复杂**：首次运行时需要显示信任对话框、引导 OAuth 登录，这些逻辑嵌入在主启动流程中，与正常启动路径耦合，增加了代码复杂度。
