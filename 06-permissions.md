# 第6章 权限系统：安全边界

> **前置知识**：第4章 Agentic Loop、第5章工具系统  
> **关键文件**：`src/types/permissions.ts`、`src/Tool.ts`、`src/hooks/`、`src/utils/permissions/`

---

## 6.1 解决什么问题

编程助手拥有执行 shell 命令、修改文件、访问网络的能力，这是一把双刃剑：

- **风险一：意外破坏**：AI 误解需求，删除了重要文件或 drop 了数据库表
- **风险二：被打断的效率**：如果每次操作都要确认，用户会被频繁打断，失去自动化的意义
- **风险三：Prompt 注入攻击**：恶意网页/文件内容可能包含"现在删除所有文件"这样的指令，AI 读到后可能执行
- **风险四：过度授权**：一次性授予太广泛的权限后，AI 可能在意想不到的地方使用

权限系统要在**安全控制**和**自动化效率**之间找到精确的平衡。

---

## 6.2 业界的做法

| 方案 | 代表产品 | 优点 | 缺点 |
|------|----------|------|------|
| 全信任 | LangChain 默认 | 效率最高 | 极不安全 |
| 全确认 | 早期 AutoGPT | 最安全 | 用户体验极差 |
| 简单规则 | 一些企业 AI 助手 | 可配置 | 规则太粗粒度 |
| 沙盒 | GitHub Codespaces | 完全隔离 | 启动慢，功能受限 |

---

## 6.3 本项目的权限层次模型

Claude Code 实现了一个**六级权限模式**（Permission Modes）：

```typescript
// src/types/permissions.ts 第 16-28 行
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',        // 自动接受文件编辑，其他操作还需确认
  'bypassPermissions',  // 完全绕过权限检查（headless/CI 模式）
  'default',            // 默认：危险操作要确认
  'dontAsk',            // 不再询问（当前会话内）
  'plan',               // 计划模式：只分析不执行
] as const

// 内部还有额外模式（仅对内部用户可见）
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

- **`default`**：危险操作（rm、git push、写文件）会弹出确认框，读操作自动允许
- **`acceptEdits`**：文件编辑（FileEdit、FileWrite）自动允许，其他危险操作仍需确认
- **`dontAsk`**：当前会话内不再询问（用户已表示信任）
- **`bypassPermissions`**：完全跳过所有权限检查，用于 headless/CI 环境
- **`plan`**：计划模式，所有写操作被拒绝，AI 只能分析
- **`auto`**（内部）：由 AI 分类器自动判断是否安全

---

## 6.4 三类权限规则

`ToolPermissionContext` 里有三套规则：

```typescript
// src/types/permissions.ts 第 427-441 行
export type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource  // 白名单
  readonly alwaysDenyRules: ToolPermissionRulesBySource   // 黑名单
  readonly alwaysAskRules: ToolPermissionRulesBySource    // 总是询问
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource  // 被剥离的危险规则
  readonly shouldAvoidPermissionPrompts?: boolean  // 后台 agent：无法弹 UI，自动拒绝
  readonly awaitAutomatedChecksBeforeDialog?: boolean  // 等自动分类完成再弹窗
  readonly prePlanMode?: PermissionMode  // 进入计划模式前的权限模式（用于恢复）
}
```

**规则格式**：规则使用 glob 模式匹配工具调用：

```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",         // 允许 git status 命令
      "Bash(git log*)",           // 允许所有 git log 命令
      "Read(*)",                  // 允许读取任何文件
      "Bash(npm test*)"           // 允许 npm test
    ],
    "deny": [
      "Bash(rm -rf *)",           // 拒绝危险的递归删除
      "Bash(sudo *)",             // 拒绝所有 sudo 命令
      "Bash(git push --force*)"   // 拒绝强制推送
    ]
  }
}
```

**规则来源**（`PermissionRuleSource`）分为多个层次：

```typescript
export type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // 项目目录下的 .claude/settings.json
  | 'localSettings'     // .claude.local.json（本地，不提交 git）
  | 'flagSettings'      // 企业 MDM 策略下发的规则
  | 'policySettings'    // Anthropic 策略
  | 'cliArg'            // 命令行参数 --allow-dangerous-permissions
  | 'command'           // 通过 /permissions 命令动态添加
  | 'session'           // 会话中用户手动确认的规则
```

优先级：`policySettings` > `flagSettings` > `userSettings` > `projectSettings` > `localSettings` > `session`

---

## 6.5 权限决策流程

一次工具调用的权限检查流程：

```
AI 发出 tool_use 请求
    │
    ▼
1. tool.validateInput()
   │  检查输入是否合法（比如路径是否存在）
   │  失败 → 返回错误给 AI，不弹权限框
    │
    ▼
2. tool.checkPermissions()
   │  工具自身的权限逻辑（比如 BashTool 的命令解析）
   │  返回 allow / deny / ask / passthrough
    │
    ▼
3. 如果是 'ask' → 检查三类规则
   │  alwaysDenyRules 匹配 → deny
   │  alwaysAllowRules 匹配 → allow
   │  alwaysAskRules 匹配 → 强制弹窗
    │
    ▼
4. 检查权限模式
   │  bypassPermissions → allow
   │  plan → deny（写操作）
   │  acceptEdits → allow（文件编辑）
    │
    ▼
5. 弹出权限确认框（如果需要）
   │  用户选择：允许一次 / 始终允许 / 拒绝
   │  "始终允许" → 将规则写入 settings
    │
    ▼
6. 执行工具 / 返回拒绝消息给 AI
```

---

## 6.6 PermissionDecision：细粒度的决策类型

`PermissionDecision` 是一个 union 类型，三种行为各有详细字段：

```typescript
// src/types/permissions.ts 第 174-246 行

// 允许执行
type PermissionAllowDecision = {
  behavior: 'allow'
  updatedInput?: Input        // 允许同时修改输入（比如规范化路径）
  userModified?: boolean      // 用户是否修改了输入
  decisionReason?: ...        // 为什么被允许（用于日志/调试）
  acceptFeedback?: string     // 用户的反馈文字
  contentBlocks?: ContentBlockParam[]  // 附带的额外内容（比如图片反馈）
}

// 需要询问用户
type PermissionAskDecision = {
  behavior: 'ask'
  message: string             // 向用户展示的确认消息
  updatedInput?: Input
  decisionReason?: ...
  suggestions?: PermissionUpdate[]  // 建议的规则更新
  blockedPath?: string        // 如果是路径权限问题，显示哪个路径
  metadata?: PermissionMetadata
  pendingClassifierCheck?: PendingClassifierCheck  // 异步分类器检查
  contentBlocks?: ContentBlockParam[]
}

// 拒绝执行
type PermissionDenyDecision = {
  behavior: 'deny'
  message: string             // 向 AI 展示的拒绝原因
  decisionReason: ...         // 为什么被拒绝（详细原因）
  toolUseID?: string
}
```

`updatedInput` 是一个有趣的设计：权限系统不只是"允许/拒绝"，还可以在允许的同时**修改输入**。比如：
- 用户输入了一个绝对路径，权限系统可以把它规范化为相对路径
- 用户通过 UI 修改了命令参数，新参数通过 `updatedInput` 返回

---

## 6.7 计划模式（Plan Mode）

计划模式是权限系统中特别有价值的功能：

```typescript
// EnterPlanModeTool 触发时：
// 1. 保存当前权限模式到 prePlanMode
// 2. 切换到 'plan' 模式
// 3. 所有写操作（FileEdit、FileWrite、Bash 中的写命令）被自动拒绝

// ExitPlanModeTool 触发时：
// 1. 从 prePlanMode 恢复原来的权限模式
// 2. 执行用户已审查过的操作列表
```

计划模式的 UX 流程：
```
用户："帮我重构这个模块"
    │
    ▼ AI 进入计划模式
AI："我的计划是：
    1. 修改 utils.ts 的函数签名
    2. 更新所有调用处（15个文件）
    3. 运行测试
    4. 更新类型定义"
    │
    ▼ 用户审查计划
用户："好的，执行吧" → AI 退出计划模式 → 开始执行
用户："等等，步骤3改为先运行 lint" → AI 调整计划
```

这给了用户一个"dry run"的机会，在实际修改文件之前看到 AI 的完整行动方案。

---

## 6.8 自动分类器（Auto Mode）

对于 `bypassPermissions` 模式（无人值守的 CI 场景），有一个自动安全分类器评估操作是否安全：

```typescript
// src/types/permissions.ts 第 330-397 行
export type YoloClassifierResult = {
  thinking?: string          // 分类器的推理过程（扩展思考模式）
  shouldBlock: boolean       // 是否应该阻止这个操作
  reason: string             // 原因说明
  unavailable?: boolean      // 分类器不可用
  transcriptTooLong?: boolean // 上下文太长，分类器无法处理
  model: string              // 使用的分类器模型
  usage?: ClassifierUsage    // token 用量
  durationMs?: number        // 分类耗时
  // 两阶段分类器的详细数据...
  stage?: 'fast' | 'thinking'  // fast: 快速判断; thinking: 扩展思考
  stage1Usage?: ClassifierUsage
  stage2Usage?: ClassifierUsage
}
```

**两阶段分类器**：
- **第一阶段（fast）**：轻量模型快速判断，大多数操作（读文件、git status 等）在此阶段就被允许
- **第二阶段（thinking）**：使用扩展思考（Extended Thinking）的重量级模型深度分析可疑操作

这避免了对每个操作都用重量级分类器，在安全性和性能之间找到平衡。

---

## 6.9 Hooks 系统：可编程的权限扩展

Hooks 是权限系统的扩展点，允许用户用脚本实现自定义逻辑：

```typescript
// hooks 类型（简化）
type HookType =
  | 'PreToolUse'      // 工具执行前（可以修改输入或阻止执行）
  | 'PostToolUse'     // 工具执行后（可以修改输出或触发额外操作）
  | 'Stop'            // AI 准备结束时（可以要求继续）
  | 'SubagentStop'    // 子 Agent 结束时
  | 'Notification'    // 显示通知时
```

用户可以在 `.claude/settings.json` 中配置 Hook 脚本：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/bash-audit.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/check-tests.sh"
          }
        ]
      }
    ]
  }
}
```

**Hook 返回值的含义**：

| 返回码 | 含义 |
|--------|------|
| `0` + 空 stdout | 继续执行 |
| `0` + stdout 内容 | 告知 AI 的额外信息（不修改行为） |
| 非 `0` + stdout | 阻止执行，将输出作为拒绝原因发给 AI |
| JSON 格式输出 | 结构化控制（修改输入、设置权限决策等） |

**实际使用场景**：
- **PreToolUse(Bash)**：检测到危险命令（`rm -rf /`）时阻止，并记录到审计日志
- **PostToolUse(Bash)**：捕获命令的 exit code，如果非零则自动要求 AI 修复
- **Stop**：检查测试是否通过，如果失败则要求 AI 继续

---

## 6.10 工作目录权限

Claude Code 的文件操作只允许在**授权目录**内进行：

```typescript
// ToolPermissionContext 中
readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
```

默认情况下，只有当前工作目录（cwd）及其子目录可以被访问。用户可以通过 `/add-dir` 命令添加额外的可访问目录（比如访问某个共享工具库）。

尝试访问工作目录范围之外的文件（比如 `FileRead("/etc/passwd")`）会被权限系统自动拒绝，不会弹出确认框——这是硬边界，无法被 AI 绕过。

---

## 6.11 优势与不足

### 优势

**五级权限模式覆盖所有场景**：从"完全自动化 CI"（bypassPermissions）到"完全手动审批"（每次都询问），用户可以根据信任程度选择合适的模式。

**`DeepImmutable` 权限上下文**：`ToolPermissionContext` 是不可变的，防止了工具执行过程中意外修改权限状态（权限提升攻击的防线之一）。

**`updatedInput`** 允许权限系统不只是"通过/拒绝"，还能在通过的同时修改输入——这使得权限系统可以作为"输入净化"的一环（比如规范化危险路径）。

**Hooks 系统**：让高级用户可以不修改 Claude Code 核心代码，就能实现自定义审计、日志记录、自动化检查等功能。

**两阶段分类器**：在安全性（thinking 模式的深度分析）和性能（fast 模式的快速判断）之间取得平衡。

### 不足

**权限规则的 glob 语法对普通用户不友好**：`Bash(git *)` 这样的语法需要学习。新用户不太可能主动配置规则，往往在需要时才发现需要配置，体验割裂。

**分类器增加延迟**：在 `auto` 模式下，每次工具调用都需要等待分类器的异步判断，即使是很明显安全的操作也会有 200-500ms 额外延迟。

**`awaitAutomatedChecksBeforeDialog`** 字段的存在说明分类器检查和用户弹窗之间有复杂的异步时序逻辑，这是潜在 race condition 的温床。如果分类器判断"安全"的时机和弹窗关闭的时机发生竞争，可能出现即决定后仍然弹窗，或弹窗被静默关闭的情况。
