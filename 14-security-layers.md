# 第14章 5层安全体系：AI 的纵深防御

> **前置知识**：第5章 权限系统、第11章 权限模式  
> **关键文件**：`src/constants/prompts.ts`、`src/types/permissions.ts`、`src/utils/sandbox/sandbox-adapter.ts`、`src/tools/EnterPlanModeTool/`、`src/utils/hooks.ts`、`src/QueryEngine.ts`

---

## 14.1 为什么需要分层防御

给 AI 一把锤子，它会把每个问题都当成钉子。

AI 编程助手有能力执行 shell 命令、修改文件、访问网络——这是它的力量，也是它的危险。单一的安全机制天然脆弱：AI 可能被精心构造的 Prompt 绕过规则（Prompt 注入），配置可能出错，用户可能误操作。

Claude Code 的答案是**纵深防御**（Defense in Depth）：5 个相互独立的安全层，每一层都尽量独立失效，即使其中某一层被绕过，其余层仍然有效。

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: AI 端安全约束 (System Prompt)                       │
│   "执行前确认"、"优先可逆操作"、"不暴露密钥"               │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: 权限规则 (Permission Rules)                         │
│   应用层 allow/deny/ask 规则，支持 Bash/Glob/Edit 等工具     │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: 沙箱隔离 (OS-level Sandbox)                         │
│   sandbox-exec (macOS) / bubblewrap (Linux) 强制约束         │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: 计划模式 (Plan Mode)                                 │
│   只读探索阶段，AI 先理解再动手                               │
├─────────────────────────────────────────────────────────────┤
│ Layer 5: Hooks & 预算上限                                     │
│   外部审计钩子 + token/成本硬上限                             │
└─────────────────────────────────────────────────────────────┘
```

每一层防御的**攻击面**不同，**实施主体**不同，**失效模式**也不同。下面逐层解析。

---

## 14.2 Layer 1：AI 端安全约束（System Prompt）

### 14.2.1 这层防御的本质

Layer 1 不是代码，而是**写给 AI 的行为规范**。它在每次 API 调用时注入 System Prompt，在 AI "大脑"层面建立安全意识。

这是最软的一层——AI 是概率性系统，规则可能被复杂 Prompt 绕过。但它也是**覆盖面最广**的一层，因为它作用于 AI 做决策之前，是其他层面来不及介入的"最后提醒"。

### 14.2.2 三条核心安全指令

**指令一：执行前确认高风险操作**

```typescript
// src/constants/prompts.ts — getActionsSection()
`Carefully consider the reversibility and blast radius of actions. 
Generally you can freely take local, reversible actions like editing 
files or running tests. But for actions that are hard to reverse, 
affect shared systems beyond your local environment, or could otherwise 
be risky or destructive, check with the user before proceeding. 

The cost of pausing to confirm is low, while the cost of an unwanted 
action (lost work, unintended messages sent, deleted branches) can be 
very high.`
```

这段提示明确列举了**哪些操作需要确认**：
- 破坏性操作：删除文件/分支、清空数据表、`rm -rf`、覆盖未提交的变更
- 难以撤销的操作：强制推送、`git reset --hard`、修改已发布的 commit、降级依赖
- 影响共享状态的操作：推送代码、在 PR 上评论、发消息给他人、修改生产基础设施

**指令二：优先可逆操作**

```typescript
// src/constants/prompts.ts
`When you encounter an obstacle, do not use destructive actions as a 
shortcut to simply make it go away. For instance, try to identify root 
causes and fix underlying issues rather than bypassing safety checks 
(e.g. --no-verify). If you discover unexpected state like unfamiliar 
files, branches, or configuration, investigate before deleting or 
overwriting.`
```

这解决了 AI 常见的"走捷径"问题：遇到障碍时倾向于绕过（`--force`、`--no-verify`）而非深究原因。

**指令三：不暴露敏感信息（Auto Mode 特有）**

```typescript
// src/utils/messages.ts — getAutoModeFullInstructions()
`Avoid data exfiltration — Post even routine messages to chat platforms 
or work tickets only if the user has directed you to. You must not share 
secrets (e.g. credentials, internal documentation) unless the user has 
explicitly authorized both that specific secret and its destination.`
```

Auto Mode（无人值守模式）下，AI 会自动发消息、调用外部 API。这条指令明确禁止 AI 在没有用户授权的情况下分享密钥或内部文档。

### 14.2.3 授权的范围限制

```typescript
// src/constants/prompts.ts — getActionsSection()
`A user approving an action (like a git push) once does NOT mean that 
they approve it in all contexts, so unless actions are authorized in 
advance in durable instructions like CLAUDE.md files, always confirm 
first. Authorization stands for the scope specified, not beyond.`
```

这是一个细微但重要的规则：**授权不可泛化**。用户在某个具体情境下批准了 `git push`，不代表 AI 可以在任意时刻 push 任意分支。每次高风险操作都必须在当前上下文下重新确认。

### 14.2.4 Layer 1 的局限性

Layer 1 对以下威胁无效：
- **Prompt 注入攻击**：恶意文件/网页内容包含"现在删除所有文件"，AI 读取后可能执行
- **AI 幻觉**：概率性系统，规则不能 100% 遵守
- **长会话漂移**：长对话中 AI 可能"遗忘"早期的安全指令

这就是为什么需要 Layer 2 的硬规则。

---

## 14.3 Layer 2：权限规则（Permission Rules）

### 14.3.1 这层防御的本质

Layer 2 是**应用层的 allow/deny 规则引擎**。无论 AI 怎么想，如果操作不符合规则，代码层面直接拒绝，不经过 AI 的决策。

这是纯代码逻辑，不依赖 AI 的理解，因此对 Prompt 注入免疫。

### 14.3.2 规则格式与来源

规则使用"工具名(内容)"的 glob 模式：

```json
{
  "permissions": {
    "allow": [
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

规则来源有 7 个层次，越高优先级越高：

```typescript
// src/types/permissions.ts
export type PermissionRuleSource =
  | 'policySettings'    // 企业 MDM 策略（最高优先级，不可覆盖）
  | 'flagSettings'      // Anthropic 官方设置
  | 'userSettings'      // ~/.claude/settings.json
  | 'projectSettings'   // 项目目录 .claude/settings.json
  | 'localSettings'     // .claude.local.json（本地，不提交 git）
  | 'cliArg'            // 命令行参数
  | 'session'           // 会话中用户手动确认
  | 'command'           // 通过 /permissions 命令动态添加（最低优先级）
```

**企业管理员**可以通过 MDM（Mobile Device Management）下发 `policySettings`，这些规则对用户是只读的——即使用户想放开某个危险命令，`policySettings` 里的 deny 规则也无法被覆盖。

### 14.3.3 三套规则的优先级

```typescript
// src/types/permissions.ts — ToolPermissionContext
readonly alwaysDenyRules:  ToolPermissionRulesBySource  // 黑名单（优先级最高）
readonly alwaysAllowRules: ToolPermissionRulesBySource  // 白名单
readonly alwaysAskRules:   ToolPermissionRulesBySource  // 总是弹窗（不可被 allow 绕过）
```

决策优先级：`alwaysDenyRules` > `alwaysAskRules` > `alwaysAllowRules`。

即使用户配置了允许某操作，如果它同时匹配 `alwaysDenyRules`，依然被拒绝。

### 14.3.4 Bash 命令的精细解析

对于 BashTool，权限检查不只是简单的字符串匹配。系统内置了 **100+ 只读命令白名单**以及基于 **AST 的语法树安全分析**：

```typescript
// src/tools/BashTool/readOnlyValidation.ts
const READONLY_COMMANDS = [
  'ls', 'cat', 'head', 'tail', 'grep', 'find', 'pwd', 'echo',
  'git status', 'git log', 'git diff', 'git show', 'git branch',
  'npm list', 'yarn list',
  // ... 100+ 更多
]
```

对于不在白名单的命令，系统使用 tree-sitter 解析 bash 语法树，检测：
- 命令注入（shell 替换 `$(...)` 或反引号）
- 重定向到文件（`>`、`>>`）
- 执行任意脚本（`eval`、`source`）
- 访问环境变量敏感路径（如 `/proc/*/environ`，可能暴露密钥）

```typescript
// src/utils/bash/ast.ts
for (const r of cmd.redirects) {
  if (r.target.includes('/proc/') && PROC_ENVIRON_RE.test(r.target)) {
    return {
      ok: false,
      reason: 'Accesses /proc/*/environ which may expose secrets',
    }
  }
}
```

### 14.3.5 不可变的权限上下文

```typescript
// src/types/permissions.ts
export type ToolPermissionContext = DeepReadonly<{
  mode: PermissionMode;
  alwaysAllowRules: ToolPermissionRulesBySource;
  // ...
}>
```

`ToolPermissionContext` 是完全不可变的（`DeepReadonly`）。工具执行过程中**无法修改权限状态**——这从类型系统层面封死了权限提升攻击。权限变更必须通过 `applyPermissionUpdate()` 产生新对象。

### 14.3.6 Layer 2 的局限性

- 规则语法（glob 模式）对普通用户不友好，配置门槛高
- 无法感知操作的**语义上下文**——同一个 `git push` 命令，推到个人分支和推到 `main` 的风险完全不同
- 这就是为什么需要 Layer 3 的 OS 级强制隔离

---

## 14.4 Layer 3：OS 级沙箱隔离（Sandbox）

### 14.4.1 这层防御的本质

Layer 3 是**操作系统层面的强制访问控制**。即使 AI 绕过了前两层（Prompt 注入成功 + 规则被配置错误），沙箱依然会在内核层面拦截危险操作。

沙箱对进程完全不透明——被沙箱化的 bash 进程"以为"自己在正常执行，但实际上文件写入被静默阻止，网络请求被路由到受限代理。

### 14.4.2 两种实现：macOS vs Linux

```
macOS：sandbox-exec（Apple 的强制访问控制框架）
Linux：bubblewrap (bwrap) + seccomp 过滤
```

Claude Code 使用 `@anthropic-ai/sandbox-runtime` 包封装这两种实现，通过 `sandbox-adapter.ts` 与 Claude CLI 的设置系统集成：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
import {
  SandboxManager as BaseSandboxManager,
} from '@anthropic-ai/sandbox-runtime'

// 覆盖 BaseSandboxManager，注入 Claude CLI 特有逻辑
export const SandboxManager: ISandboxManager = {
  initialize,
  isSandboxingEnabled,
  wrapWithSandbox,      // 核心：将 shell 命令包裹在沙箱中执行
  cleanupAfterCommand,  // 清理 bwrap 产生的挂载残留文件
  // ...
}
```

每次执行 bash 命令时，`Shell.ts` 调用 `SandboxManager.wrapWithSandbox()` 把命令字符串包裹成沙箱命令：

```typescript
// src/utils/Shell.ts
if (shouldUseSandbox) {
  SandboxManager.cleanupAfterCommand()
}
```

### 14.4.3 三维度沙箱约束

**维度一：文件系统约束**

```typescript
// src/utils/sandbox/sandbox-adapter.ts — convertToSandboxRuntimeConfig()
const allowWrite: string[] = ['.', getClaudeTempDir()]  // 默认只允许写当前目录
const denyWrite: string[] = []
const denyRead: string[]  = []
```

- `allowOnly`（写入白名单）：只有列表中的路径才能被写入
- `denyWithinAllow`（白名单内的例外）：在可写路径里再屏蔽某些子目录
- `denyOnly`（读取黑名单）：禁止读取特定路径（如 `~/.ssh/`、`~/.aws/`）

特别保护：`.claude/skills` 目录被明确加入写入黑名单——AI 自定义技能文件不能被 AI 自己修改：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
// Block writes to .claude/skills in both original and current working directories.
// Skills have the same privilege level (auto-discovered, auto-loaded, full 
// Claude capabilities) so they need the same OS-level sandbox protection.
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
```

**维度二：网络约束**

沙箱可以配置**域名白名单**，限制 AI 只能访问指定的网络域名：

```typescript
// settings.json 配置示例
{
  "sandbox": {
    "network": {
      "allowedDomains": ["api.github.com", "registry.npmjs.org"],
      "deniedHosts": ["internal-corp-server.example.com"]
    }
  }
}
```

网络请求通过沙箱内部的 HTTP 代理路由，代理检查域名是否在白名单内。企业可以配置 `shouldAllowManagedSandboxDomainsOnly()`，只允许通过 MDM 策略下发的域名，防止 AI 访问内部服务器或泄露数据到外部。

**维度三：沙箱违规报告**

当命令尝试违规操作时，沙箱不会静默失败，而是通过 `SandboxViolationStore` 收集违规信息，并将这些信息标注到 stderr 输出中：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
annotateStderrWithSandboxFailures(command: string, stderr: string): string
```

这让 AI 能"感知"到沙箱阻止了什么，从而改变策略，而不是陷入死循环。

### 14.4.4 沙箱的特殊安全规则

**防止 Git 沙箱逃逸**：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.
const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const gitFile of bareGitRepoFiles) {
  // ...
  denyWrite.push(p)  // 阻止写入这些文件
}
```

攻击者可能在 cwd 中放置伪造的 Git 仓库文件，让 `git` 命令触发 `core.fsmonitor` 钩子，借此逃出沙箱。Claude Code 专门处理了这个攻击向量。

**防止 Root 权限滥用**：

```typescript
// src/setup.ts
if (
  process.getuid() === 0 &&         // 以 root 运行
  process.env.IS_SANDBOX !== '1' && // 不在沙箱中
  !isEnvTruthy(process.env.CLAUDE_CODE_BUBBLEWRAP)
) {
  console.error(
    `--dangerously-skip-permissions cannot be used with root/sudo 
     privileges for security reasons`
  )
  process.exit(1)
}
```

以 root 身份运行时，`bypassPermissions` 模式直接被禁止——即使是完全信任的 CI 环境，也不允许以 root + 无权限检查的组合运行。

### 14.4.5 Layer 3 的局限性

- 依赖 OS 特性，Windows 原生不支持（`bwrap`/`sandbox-exec` 是 POSIX 专有）
- 需要安装依赖（`bubblewrap`、`socat`、`seccomp`），依赖缺失时沙箱静默降级——这是一个已知的"安全足枪"，代码注释中明确提到：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
// Fix for #34044: previously isSandboxingEnabled() silently returned false
// when dependencies were missing, giving users zero feedback that their
// explicit security setting was being ignored. This is a security footgun.
```

---

## 14.5 Layer 4：计划模式（Plan Mode）

### 14.5.1 这层防御的本质

Layer 4 是一个**行为约束层**：在 AI 真正开始动手之前，强制进入一个"只读探索"的阶段。

和前三层不同，Plan Mode 不是"阻止危险操作"，而是**提前暴露风险**——让用户在损失发生之前看到并审批 AI 的完整行动方案。

### 14.5.2 Plan Mode 的核心机制

进入 Plan Mode 时，系统向每轮对话注入一段不可绕过的强制提示：

```
// src/utils/messages.ts
Plan mode is active. The user indicated that they do not want you to 
execute yet -- you MUST NOT make any edits (with the exception of the 
plan file mentioned below), run any non-readonly tools (including 
changing configs or making commits), or otherwise make any changes to 
the system. This supercedes any other instructions you have received.
```

关键词是 **"supercedes any other instructions"**——Plan Mode 的权重高于用户的其他所有指令，防止用户无意中说"顺便把这个文件改一下"而解除 Plan Mode。

同时，应用层权限检查也会拒绝所有写操作：

```
权限决策流 (Plan Mode):
  任何写操作 → checkPermissions() → mode === 'plan' → deny
```

双重锁：**AI 主观层面**（System Prompt 约束）和**代码执行层面**（权限拒绝）同时生效。

### 14.5.3 AI 主动触发 Plan Mode

Plan Mode 不只是用户手动触发的保护机制——AI 被提示词引导在以下情况下**主动进入计划模式**：

```
// src/tools/EnterPlanModeTool/prompt.ts 描述的场景
- 任务有多种合理实现方案
- 需要修改 3 个以上的文件
- 需求模糊，需要先探索
- 用户说"帮我重构"这种宽泛指令
```

这是 Layer 1（AI 的安全意识）和 Layer 4（计划模式约束）的协同——AI 不仅被告知"危险操作要小心"，还有一个明确的工具（`EnterPlanMode`）让它在不确定时主动降到安全的只读状态。

### 14.5.4 Plan Mode 的 "Dry Run" 价值

```
用户："帮我重构这个 1000 行的模块"
    │
    ▼ AI 进入 Plan Mode，派生 Explore Agent 扫描代码
AI 输出 PLAN.md：
    1. 将 UserService 拆分为 UserRepo + UserDomain
    2. 更新 17 个调用处（列出具体文件和行号）
    3. 迁移 3 个 SQL 查询
    4. 运行测试套件验证
    │
    ▼ 用户审查：
用户："第3步可以先跳过，先跑通基础重构" → AI 调整计划
用户："好，执行吧" → 退出 Plan Mode → AI 开始执行（已获批准）
```

用户在任何实际修改发生之前就看到了完整的影响范围，并有机会调整方向。

### 14.5.5 Layer 4 的局限性

- 需要用户主动理解并审查计划，对初级用户的认知负担较重
- 计划文件（`PLAN.md`）本身可能写得过于笼统，用户难以判断风险
- 批准计划不等于批准"计划以外"的事情——AI 执行中可能发现新情况，需要临时超出计划范围

---

## 14.6 Layer 5：Hooks & 预算上限

### 14.6.1 这层防御的本质

Layer 5 是**外部审计与量化约束**。它不阻止任何具体操作，而是：

1. **Hooks**：在工具执行前后注入自定义脚本，实现外部审计、日志、合规检查
2. **预算上限**：对 token 消耗和美元成本设置硬上限，防止无限循环和意外超支

### 14.6.2 Hooks：可编程的外部审计

用户在 `.claude/settings.json` 中配置 Hook 脚本：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/bash-audit.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/notify-slack.sh"
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/verify-tests.sh"
        }]
      }
    ]
  }
}
```

**Hook 类型**：

| 类型 | 触发时机 | 能做什么 |
|------|----------|----------|
| `PreToolUse` | 工具执行前 | 检查输入、阻止执行、记录日志 |
| `PostToolUse` | 工具执行后 | 检查输出、触发额外操作、通知 |
| `Stop` | AI 准备结束时 | 验证任务完成、要求继续 |
| `SubagentStop` | 子 Agent 结束时 | 汇总结果 |
| `Notification` | 显示通知时 | 自定义通知渠道 |

**Hook 返回值的含义**：

| 返回码 | stdout | 含义 |
|--------|--------|------|
| `0` | 空 | 继续执行（通过） |
| `0` | 有内容 | 继续执行，将 stdout 作为信息告知 AI |
| 非 `0` | 有内容 | **阻止执行**，将 stdout 作为拒绝原因发给 AI |
| 任意 | JSON 格式 | 结构化控制（修改输入、设置权限决策） |

Hook 是纯外部脚本，不需要修改 Claude Code 源码，即可实现：
- 企业合规审计（记录所有命令到中央日志）
- 危险命令二次确认（`rm -rf` 时触发 Slack 通知）
- 自动化质量门禁（`Stop` Hook 检查测试是否通过，失败则要求 AI 继续）

### 14.6.3 美元成本上限（maxBudgetUsd）

```typescript
// src/QueryEngine.ts
// Check if USD budget has been exceeded
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield {
    type: 'result',
    subtype: 'error_max_budget_usd',
    is_error: true,
    errors: [`Reached maximum budget ($${maxBudgetUsd})`],
  }
  return  // 立即终止循环
}
```

通过 CLI 参数设置：
```bash
claude --max-budget-usd 5.00 "重构整个代码库"
```

一旦累计 API 调用成本超过 5 美元，循环立即终止，返回错误。这防止了 AI 因误解任务而无限循环消耗大量 token。

成本信息还会以 attachment 形式注入到对话中，让 AI 感知剩余预算：

```typescript
// src/utils/attachments.ts — getMaxBudgetUsdAttachment()
{
  type: 'budget_usd',
  used: usedCost,
  total: maxBudgetUsd,
  remaining: remainingBudget,
}
```

### 14.6.4 Token 预算（task_budget）

除了美元上限，还有 token 维度的约束：

```bash
claude --task-budget 500000 "分析整个代码库"
```

这是一个 API 侧的参数（`output_config.task_budget`），直接传给 Claude API。当 AI 接近 token 上限时，会收到提示并主动归纳总结，而非突然被截断。

```typescript
// src/constants/prompts.ts — token_budget 系统提示
`When the user specifies a token target (e.g., "+500k", "spend 2M tokens"), 
your output token count will be shown each turn. Keep working until you 
approach the target — plan your work to fill it productively. The target 
is a hard minimum, not a suggestion.`
```

Token 预算有双重作用：既是**成本约束**，也是**任务完整性保障**——AI 被要求在预算内做尽可能完整的工作，而不是草草收场。

### 14.6.5 Layer 5 的局限性

- Hooks 是 shell 脚本，需要用户有脚本编写能力才能充分利用
- 成本上限是事后检查（每次 API 调用后），无法阻止**单次**昂贵的 API 调用
- 没有**操作数量**上限（比如"最多修改 N 个文件"），只有成本上限

---

## 14.7 五层防御的协同关系

五层防御覆盖了不同的威胁场景：

| 威胁场景 | 被哪层防御阻止 |
|----------|---------------|
| AI 误解需求，删除了重要文件 | Layer 1（确认提示）+ Layer 4（Plan Mode 预览） |
| 恶意文件包含"删除所有内容"的 Prompt 注入 | Layer 2（deny 规则）+ Layer 3（沙箱文件系统约束） |
| 用户误配置允许了危险命令 | Layer 3（沙箱）+ Layer 5（Hooks 审计） |
| AI 无限循环消耗资源 | Layer 5（成本上限） |
| AI 试图访问 `~/.ssh/id_rsa` | Layer 2（工作目录边界）+ Layer 3（沙箱读取黑名单） |
| AI 尝试推送代码到生产分支 | Layer 1（确认提示）+ Layer 2（deny 规则）+ Layer 5（PreToolUse Hook 拦截）|
| 企业合规：需要审计所有 AI 操作 | Layer 5（Hooks 记录日志）|

没有任何单层能覆盖所有场景，但五层的组合提供了很高的安全覆盖度。

---

## 14.8 与业界方案的对比

| 维度 | Claude Code | LangChain/AutoGPT | GitHub Copilot |
|------|------------|-------------------|----------------|
| AI 端安全提示 | ✅ 完整的 System Prompt 行为规范 | ⚠️ 依赖开发者自定义 | ❌ 不适用（只做补全） |
| 应用层规则引擎 | ✅ 7层来源、3类规则、glob 匹配 | ⚠️ 无标准方案 | ❌ 不适用 |
| OS 级沙箱 | ✅ sandbox-exec/bubblewrap | ❌ 无 | ❌ 不适用 |
| Dry Run 模式 | ✅ Plan Mode | ❌ 无 | ❌ 不适用 |
| 外部审计钩子 | ✅ Hooks 系统 | ⚠️ 无标准方案 | ❌ 不适用 |
| 成本上限 | ✅ maxBudgetUsd + task_budget | ⚠️ 部分支持 | ❌ 不适用 |
| 企业管控 | ✅ MDM 策略（policySettings） | ❌ 无 | ✅ Enterprise 版本 |

Claude Code 在安全体系的**系统性**上明显优于开源框架方案，五层防御都有对应的代码实现，而非依赖约定或文档。

---

## 14.9 对构建自己的 Agent 系统的启示

如果你正在构建一个生产级 AI Agent 系统，这个五层框架提供了很好的参考：

**第一步（必须）**：在 System Prompt 里明确 AI 的行为边界，列举哪些操作需要确认。这是零成本的安全提升。

**第二步（推荐）**：实现一个简单的 allow/deny 规则引擎，覆盖最危险的操作（生产环境写入、推送到主分支等）。

**第三步（按需）**：如果 Agent 需要执行任意 shell 命令，考虑使用容器（Docker）或 VM 隔离，这是 Layer 3 的简化版本。

**第四步（推荐）**：为长任务实现"确认计划后再执行"的两阶段流程，这是最被低估的安全机制——因为它在保护安全的同时也大幅提升了用户信任度。

**第五步（必须）**：设置成本/token 上限，并实现操作日志。无论发生什么，你都需要知道 AI 做了什么。

---

## 14.10 关键源码位置

| 层次 | 功能 | 文件 | 关键函数/位置 |
|------|------|------|--------------|
| Layer 1 | 核心安全提示 | `src/constants/prompts.ts` | `getActionsSection()` |
| Layer 1 | Auto Mode 安全约束 | `src/utils/messages.ts` | `getAutoModeFullInstructions()` |
| Layer 2 | 权限规则类型定义 | `src/types/permissions.ts` | `ToolPermissionContext`（第427行） |
| Layer 2 | Bash 命令安全分析 | `src/utils/bash/ast.ts` | `checkBashReadOnly()` |
| Layer 2 | 文件系统边界检查 | `src/utils/permissions/filesystem.ts` | `pathInAllowedWorkingPath()` |
| Layer 2 | 完整权限决策流 | `src/utils/permissions/permissions.ts` | `hasPermissionsToUseToolInner()` |
| Layer 3 | 沙箱适配层 | `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager.wrapWithSandbox()` |
| Layer 3 | 沙箱配置转换 | `src/utils/sandbox/sandbox-adapter.ts` | `convertToSandboxRuntimeConfig()` |
| Layer 3 | 沙箱配置 UI | `src/components/sandbox/SandboxConfigTab.tsx` | — |
| Layer 4 | 进入计划模式 | `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` | `call()` |
| Layer 4 | 退出计划模式 | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | `call()` |
| Layer 4 | Plan Mode 提示词 | `src/utils/messages.ts` | `getPlanModeV2Instructions()` |
| Layer 5 | Hooks 执行 | `src/utils/hooks.ts` | `executePreToolUseHooks()` |
| Layer 5 | 成本上限检查 | `src/QueryEngine.ts` | 第 972 行，`error_max_budget_usd` |
| Layer 5 | 成本上限 attachment | `src/utils/attachments.ts` | `getMaxBudgetUsdAttachment()` |
| Layer 5 | Token 预算解析 | `src/utils/tokenBudget.ts` | `parseTokenBudget()` |
