# 第4章 工具系统 — 代码问答笔记

本文件记录阅读 `04-tool-system.md` 过程中对具体代码的问答，作为正文章节的补充。

---

## Q1：为什么不直接用 BashTool 调 ripgrep，而要单独封装 GrepTool？

**核心答案**：两者能"做到同一件事"，但对系统的成本完全不同。封装不是为了能力，而是为了**安全、可预测性和 token 成本**。

### 1. 安全边界：硬编码 vs 依赖 AI 判断

GrepTool 的 `checkPermissions()` 调用 `checkReadPermissionForTool()`，有 8 步硬编码安全检查：

```
步骤 1：拦截 UNC 路径（\\server\share）→ 防止访问网络资源
步骤 2：拦截可疑 Windows 路径（备用数据流、短文件名等）
步骤 3：先查显式 deny 规则
步骤 4：再查显式 ask 规则
步骤 6：在工作目录内 → 自动允许
步骤 7：内部路径（session memory、plan 文件）→ 允许
步骤 8：有显式 allow 规则 → 允许
兜底：要求用户确认
```

**关键**：GrepTool 的工作目录限制是**编译进去的行为，AI 无法通过任何 prompt 绕过**。但 AI 可能被诱导让 BashTool 执行 `rg pattern ~/.ssh/`，因为 BashTool 依赖 AI 分类器或用户弹窗。

### 2. 输出后处理：GrepTool 替模型做了很多"体力活"

直接跑 `rg` 返回原始绝对路径，可能几千行。GrepTool 自动做了四件事：

| 后处理 | 代码 | 节省成本 |
|--------|------|---------|
| 绝对路径 → 相对路径 | `toRelativePath()` | 每个路径少 20-40 个 token |
| 按修改时间排序 | `Promise.allSettled(stat)` + sort | 最相关文件排在前面 |
| 结果截断（head_limit=250） | `applyHeadLimit()` | 防止塞满 context window |
| 返回结构化字段 | `numFiles`、`numLines`、`numMatches` | 模型不用自己解析计数 |

### 3. 参数防注入：`-` 开头的 pattern

```typescript
// src/tools/GrepTool/GrepTool.ts 第 378-384 行
if (pattern.startsWith('-')) {
  args.push('-e', pattern)  // 用 -e 显式标记为 pattern，不是选项
} else {
  args.push(pattern)
}
```

### 4. 用户 ignore 规则自动注入

```typescript
const ignorePatterns = normalizePatternsToPath(
  getFileReadIgnorePatterns(appState.toolPermissionContext),
  getCwd(),
)
for (const ignorePattern of ignorePatterns) {
  const rgIgnorePattern = ignorePattern.startsWith('/')
    ? `!${ignorePattern}`
    : `!**/${ignorePattern}`
  args.push('--glob', rgIgnorePattern)
}
```

### 总结对比

| 维度 | `BashTool("rg ...")` | `GrepTool` |
|------|---------------------|-----------|
| 安全边界 | 依赖 AI 分类器或用户确认 | 编译进权限规则，AI 无法绕过 |
| 工作目录限制 | 无自动限制 | 自动检查 `pathInAllowedWorkingPath()` |
| ignore 规则 | 不注入 | 自动注入用户配置 |
| 路径格式 | 绝对路径原样返回 | 自动转相对路径 |
| 结果截断 | 无保护（可能塞满 context） | 默认 250 行 + 分页提示 |
| token 成本 | 高 | 低 |

---

## Q2：`import { z } from 'zod/v4'` 是什么？

这是 TypeScript 的模块导入语句，对应 C++ 的 `#include`。**zod** 是运行时数据验证库，`z` 是它的主命名空间，用来描述工具输入参数的格式并在运行时校验。

---

## Q3：`() =>` 箭头函数语法

C++/TypeScript 对照：

| C++ | TypeScript |
|-----|-----------|
| `[]() { return x; }` | `() => x` |
| `[capture](int a) { return a+1; }` | `(a) => a + 1` |

---

## Q4：`lazySchema` 懒加载原理

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined        // 相当于 static T* instance = nullptr
  return () => (cached ??= factory()) // 相当于 if (!instance) instance = new T()
}
```

`??=` = 空值合并赋值，等价 C++ 的 `if (ptr == nullptr) ptr = new T()`。

---

## Q5：`lazySchema` 是否对应 C++ 单例？

**是的，本质就是懒汉式单例**，用闭包持有 `cached` 变量（相当于 `private static`）。每次调用 `lazySchema(factory)` 产生独立的单例，互不干扰。

---

## Q6：`z.strictObject` 是干嘛的？

- **输入**：描述"数据长什么样"的配置（字段名 → 校验规则）
- **输出**：一个 Schema 对象，同时用于运行时校验 + 编译时类型推断

`strictObject` vs `object`：传入任何未定义字段直接报错（`object` 会静默忽略）。用 strict 是为了防止模型乱传参数。

---

## Q7：skill 太多会把上下文塞满，项目有什么处理方式？

三层防护：

1. **skill 本身不进上下文**：`prompt()` 只返回名字列表，调用时才注入 `SKILL.md` 全文
2. **描述截断预算控制**：`SKILL_BUDGET_CONTEXT_PERCENT = 0.01`（上下文窗口的 1%）；超出预算时先截断非内置 skill 描述，极端情况退化为只显示名字
3. **addInvokedSkill 保活**：触发后内容注册到状态，压缩（Compact）时不丢失，压缩后自动重注入
4. **fork 模式**：skill 配置 `context: 'fork'` 时在独立子 Agent 里运行，有自己的 token 预算，只把摘要返回给主对话

---

## Q8：SkillTool.ts 完整 C++ 对照注释

```typescript
// ========================================================
// SkillTool.ts — "skill 执行工具"
//
// 架构定位：
//   这是一个"元工具"（meta-tool）。它本身不做具体事情，
//   而是负责加载并执行用户定义的 skill（即 SKILL.md 文件）。
//   类比 C++ 中的插件加载器：动态加载 .so/.dll 并调用其入口函数。
//
// 两种执行模式（类比两种插件执行方式）：
//   1. inline 模式（默认）：把 SKILL.md 内容注入主对话，模型继续处理
//      → 类比：直接 dlopen + 在当前线程执行
//   2. fork 模式：在独立子 Agent 里运行，返回摘要
//      → 类比：fork() 出子进程执行，只拿返回值
// ========================================================

import { feature } from 'bun:bundle'
// feature() 是 Bun 的编译时特性开关，类似 C++ 的 #ifdef
// feature('EXPERIMENTAL_SKILL_SEARCH') 为 false 时，相关代码被 tree-shake 掉

// --------------------------------------------------------
// getAllCommands — 获取所有可用 skill 列表
// --------------------------------------------------------
// C++ 类比：枚举所有已注册插件的函数
// MCP skills 是通过网络协议加载的外部 skill，需要单独合并
async function getAllCommands(context: ToolUseContext): Promise<Command[]> {
  // 只包含 loadedFrom === 'mcp' 的 MCP skill，不包含普通 MCP prompts
  // （防止模型通过猜名字调用不可发现的 prompts）
  const mcpSkills = context.getAppState().mcp.commands.filter(
    cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp',
  )
  if (mcpSkills.length === 0) return getCommands(getProjectRoot())

  // 用 uniqBy 去重（相当于 C++ set<Command> 按 name 去重后转 vector）
  const localCommands = await getCommands(getProjectRoot())
  return uniqBy([...localCommands, ...mcpSkills], 'name')
}

// --------------------------------------------------------
// executeForkedSkill — fork 模式执行 skill
// --------------------------------------------------------
// C++ 类比：fork() + execve() 在子进程运行插件，父进程 waitpid() 拿结果
//
// 关键：skill 在独立子 Agent 里运行，有自己的 token 预算（own token budget）。
// 执行过程中产生的所有中间消息不污染主对话，只返回最终摘要文本。
async function executeForkedSkill(
  command,      // skill 定义（从 SKILL.md 加载的 Command 对象）
  commandName,  // skill 名字，如 "pdf"
  args,         // 用户传的参数，如 "convert doc.docx"
  context,      // 工具执行上下文（包含 AppState、权限等）
  canUseTool,   // 权限检查回调
  parentMessage,// 触发此次调用的父消息（用于追踪调用链）
  onProgress?,  // 进度回调（可选），用于在 UI 上显示子 Agent 的中间动作
): Promise<ToolResult<Output>> {

  // 准备独立子 Agent 的执行上下文
  // modifiedGetAppState：修改后的状态访问器（隔离子 Agent 的状态）
  // baseAgent：子 Agent 的定义（agentType、系统 prompt 等）
  // promptMessages：传给子 Agent 的初始消息
  // skillContent：skill 的 SKILL.md 内容（用于 addInvokedSkill 注册）
  const { modifiedGetAppState, baseAgent, promptMessages, skillContent } =
    await prepareForkedCommandContext(command, args || '', context)

  // 如果 skill 指定了 effort（精力投入级别），合并到 agent 定义
  // 类比：设置子进程的调度优先级
  const agentDefinition = command.effort !== undefined
    ? { ...baseAgent, effort: command.effort }
    : baseAgent

  // 收集子 Agent 产生的所有消息（执行完后会清空，释放内存）
  const agentMessages: Message[] = []

  try {
    // runAgent 是一个 async generator（异步生成器）
    // 类比 C++ 的 co_yield：每产生一条消息就 yield 出来
    // for await...of 相当于循环接收子进程发来的事件
    for await (const message of runAgent({
      agentDefinition,
      promptMessages,
      toolUseContext: { ...context, getAppState: modifiedGetAppState },
      canUseTool,
      isAsync: false,
      querySource: 'agent:custom',
      model: command.model as ModelAlias | undefined,
      availableTools: context.options.tools,
      override: { agentId },
    })) {
      agentMessages.push(message)

      // 如果有进度回调，把子 Agent 的工具调用动作实时推送给 UI
      // 类比：把子进程的 stdout 实时 pipe 给父进程显示
      if ((message.type === 'assistant' || message.type === 'user') && onProgress) {
        const normalizedNew = normalizeMessages([message])
        for (const m of normalizedNew) {
          const hasToolContent = m.message.content.some(
            c => c.type === 'tool_use' || c.type === 'tool_result',
          )
          if (hasToolContent) {
            onProgress({
              toolUseID: `skill_${parentMessage.message.id}`,
              data: { message: m, type: 'skill_progress', prompt: skillContent, agentId },
            })
          }
        }
      }
    }

    // 从所有消息里提取最终结果文本（相当于读取子进程的 stdout 最终输出）
    const resultText = extractResultText(agentMessages, 'Skill execution completed')

    // 主动清空消息数组，释放内存
    // C++ 类比：delete[] agentMessages; agentMessages = nullptr;
    agentMessages.length = 0

    return {
      data: {
        success: true,
        commandName,
        status: 'forked',  // 标记为 fork 模式
        agentId,
        result: resultText,  // 只返回摘要，不返回全部中间过程
      },
    }
  } finally {
    // finally 块：无论成功/失败都执行（类似 C++ RAII 析构）
    // 清理该子 Agent 注册的 skill 内容，防止内存泄漏
    clearInvokedSkillsForAgent(agentId)
  }
}

// --------------------------------------------------------
// inputSchema — 工具的输入参数格式定义
// --------------------------------------------------------
// 类比 C++ 的函数签名 + 参数校验：
//   void call(const std::string& skill, const std::optional<std::string>& args)
// 只有两个字段，非常简单：skill 名字 + 可选参数
export const inputSchema = lazySchema(() =>
  z.object({
    skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
    args: z.string().optional().describe('Optional arguments for the skill'),
  }),
)

// --------------------------------------------------------
// outputSchema — 工具的输出格式（两种可能的输出结构）
// --------------------------------------------------------
// C++ 类比：std::variant<InlineResult, ForkedResult>
// 用 z.union 表示"返回值可能是两种结构之一"
export const outputSchema = lazySchema(() => {
  // inline 模式的输出：skill 已加载，后续由模型直接处理
  const inlineOutputSchema = z.object({
    success: z.boolean(),
    commandName: z.string(),
    allowedTools: z.array(z.string()).optional(),  // skill 允许使用的工具子集
    model: z.string().optional(),                  // skill 指定的模型覆盖
    status: z.literal('inline').optional(),
  })

  // fork 模式的输出：子 Agent 已完成，这是摘要结果
  const forkedOutputSchema = z.object({
    success: z.boolean(),
    commandName: z.string(),
    status: z.literal('forked'),
    agentId: z.string(),
    result: z.string(),  // 子 Agent 的执行结果摘要
  })

  return z.union([inlineOutputSchema, forkedOutputSchema])
})

// --------------------------------------------------------
// SkillTool 对象定义 — 实现 Tool 接口
// --------------------------------------------------------
// C++ 类比：class SkillTool : public ITool { ... }
// buildTool() 相当于注册工具并做一些框架级包装
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: SKILL_TOOL_NAME,              // 工具名，模型用这个名字调用
  searchHint: 'invoke a slash-command skill',  // ToolSearchTool 搜索时用的提示词
  maxResultSizeChars: 100_000,        // 结果最大字符数（防止爆 context）

  // description：模型调用时显示的简短描述（动态生成）
  description: async ({ skill }) => `Execute skill: ${skill}`,

  // prompt()：工具的完整说明文档（注入到 system prompt）
  // memoize 过：同一 cwd 只读取一次，缓存结果
  prompt: async () => getPrompt(getProjectRoot()),

  // -------------------------------------------------
  // validateInput — 输入合法性校验（在权限检查之前）
  // C++ 类比：bool validate(const Input& input)
  // -------------------------------------------------
  async validateInput({ skill }, context): Promise<ValidationResult> {
    const trimmed = skill.trim()
    if (!trimmed) {
      return { result: false, message: `Invalid skill format: ${skill}`, errorCode: 1 }
    }

    // 兼容用户输入 "/commit"（有斜杠前缀）
    const normalizedCommandName = trimmed.startsWith('/') ? trimmed.substring(1) : trimmed

    // 实验性功能：远程 skill（ant 内部用）
    // 以 "_canonical_" 开头的是从远程服务器加载的 skill
    if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
      const slug = remoteSkillModules!.stripCanonicalPrefix(normalizedCommandName)
      if (slug !== null) {
        const meta = remoteSkillModules!.getDiscoveredRemoteSkill(slug)
        if (!meta) {
          return {
            result: false,
            message: `Remote skill ${slug} was not discovered in this session...`,
            errorCode: 6,
          }
        }
        return { result: true }
      }
    }

    const commands = await getAllCommands(context)
    const foundCommand = findCommand(normalizedCommandName, commands)

    // skill 不存在
    if (!foundCommand) {
      return { result: false, message: `Unknown skill: ${normalizedCommandName}`, errorCode: 2 }
    }

    // skill 禁止了模型调用（某些 skill 只能用户手动触发）
    if (foundCommand.disableModelInvocation) {
      return {
        result: false,
        message: `Skill ${normalizedCommandName} cannot be used with ${SKILL_TOOL_NAME} tool...`,
        errorCode: 4,
      }
    }

    // 必须是 prompt 类型的 skill（不能是普通命令）
    if (foundCommand.type !== 'prompt') {
      return {
        result: false,
        message: `Skill ${normalizedCommandName} is not a prompt-based skill`,
        errorCode: 5,
      }
    }

    return { result: true }
  },

  // -------------------------------------------------
  // checkPermissions — 权限检查
  // C++ 类比：bool hasPermission(const std::string& commandName)
  //
  // 优先级（从高到低）：
  //   deny 规则 > 远程canonical skill（自动允许）> allow 规则 > 安全属性检查 > 询问用户
  // -------------------------------------------------
  async checkPermissions({ skill, args }, context): Promise<PermissionDecision> {
    const commandName = skill.trim().replace(/^\//, '')
    const permissionContext = context.getAppState().toolPermissionContext
    const commands = await getAllCommands(context)
    const commandObj = findCommand(commandName, commands)

    // 规则匹配函数：支持精确匹配（"commit"）和前缀通配（"review:*"）
    const ruleMatches = (ruleContent: string): boolean => {
      const normalizedRule = ruleContent.replace(/^\//, '')
      if (normalizedRule === commandName) return true
      if (normalizedRule.endsWith(':*')) {
        return commandName.startsWith(normalizedRule.slice(0, -2))
      }
      return false
    }

    // 1. 先检查 deny 规则（黑名单优先）
    const denyRules = getRuleByContentsForTool(permissionContext, SkillTool as Tool, 'deny')
    for (const [ruleContent, rule] of denyRules.entries()) {
      if (ruleMatches(ruleContent)) {
        return { behavior: 'deny', message: 'Skill execution blocked by permission rules', decisionReason: { type: 'rule', rule } }
      }
    }

    // 2. 远程 canonical skill：直接允许（deny 规则已在上面检查过）
    if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
      const slug = remoteSkillModules!.stripCanonicalPrefix(commandName)
      if (slug !== null) {
        return { behavior: 'allow', updatedInput: { skill, args }, decisionReason: undefined }
      }
    }

    // 3. 检查 allow 规则（白名单）
    const allowRules = getRuleByContentsForTool(permissionContext, SkillTool as Tool, 'allow')
    for (const [ruleContent, rule] of allowRules.entries()) {
      if (ruleMatches(ruleContent)) {
        return { behavior: 'allow', updatedInput: { skill, args }, decisionReason: { type: 'rule', rule } }
      }
    }

    // 4. 安全属性检查：只包含安全属性的 skill 自动允许
    // （防止新增属性绕过权限——不在白名单里的属性默认需要用户确认）
    if (commandObj?.type === 'prompt' && skillHasOnlySafeProperties(commandObj)) {
      return { behavior: 'allow', updatedInput: { skill, args }, decisionReason: undefined }
    }

    // 5. 兜底：询问用户
    // suggestions 里提供两个快捷建议：精确允许 或 前缀允许
    return {
      behavior: 'ask',
      message: `Execute skill: ${commandName}`,
      suggestions: [
        { type: 'addRules', rules: [{ toolName: SKILL_TOOL_NAME, ruleContent: commandName }], behavior: 'allow', destination: 'localSettings' },
        { type: 'addRules', rules: [{ toolName: SKILL_TOOL_NAME, ruleContent: `${commandName}:*` }], behavior: 'allow', destination: 'localSettings' },
      ],
      updatedInput: { skill, args },
      metadata: commandObj ? { command: commandObj } : undefined,
    }
  },

  // -------------------------------------------------
  // call — 实际执行 skill
  // C++ 类比：ToolResult call(const Input& input, Context& ctx)
  //
  // 此时 validateInput 已经通过，参数合法性有保证。
  // -------------------------------------------------
  async call({ skill, args }, context, canUseTool, parentMessage, onProgress?) {
    const commandName = skill.trim().replace(/^\//, '')

    // 实验性：加载远程 skill
    if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
      const slug = remoteSkillModules!.stripCanonicalPrefix(commandName)
      if (slug !== null) {
        return executeRemoteSkill(slug, commandName, parentMessage, context)
      }
    }

    const commands = await getAllCommands(context)
    const command = findCommand(commandName, commands)

    recordSkillUsage(commandName)  // 记录使用频次，用于 skill 排名

    // ---- fork 模式 ----
    // skill 在 SKILL.md frontmatter 里配置了 context: fork
    if (command?.type === 'prompt' && command.context === 'fork') {
      return executeForkedSkill(command, commandName, args, context, canUseTool, parentMessage, onProgress)
    }

    // ---- inline 模式（默认）----
    // processPromptSlashCommand：
    //   1. 读取 SKILL.md 内容
    //   2. 替换 $ARGUMENTS 占位符
    //   3. 处理 !command 内嵌命令
    //   4. 把结果包装成 UserMessage
    const { processPromptSlashCommand } = await import('src/utils/processUserInput/processSlashCommand.js')
    const processedCommand = await processPromptSlashCommand(commandName, args || '', commands, context)

    if (!processedCommand.shouldQuery) {
      throw new Error('Command processing failed')
    }

    const allowedTools = processedCommand.allowedTools || []
    const model = processedCommand.model

    // 获取父消息的 tool_use_id，用于关联新消息和这次工具调用
    const toolUseID = getToolUseIDFromParentMessage(parentMessage, SKILL_TOOL_NAME)

    // tagMessagesWithToolUseID：给新消息打上 sourceToolUseID 标记
    // 作用：在工具调用结束前，这些消息保持"暂时性"状态（不被压缩）
    const newMessages = tagMessagesWithToolUseID(
      processedCommand.messages.filter(/* 过滤掉内部命令消息标签 */),
      toolUseID,
    )

    return {
      data: { success: true, commandName, allowedTools: allowedTools.length > 0 ? allowedTools : undefined, model },
      newMessages,  // 这些消息会被插入主对话（SKILL.md 内容就在这里）

      // contextModifier：修改后续对话的执行上下文（类比修改子进程的环境变量）
      contextModifier(ctx) {
        let modifiedContext = ctx

        // 如果 skill 限制了可用工具，更新权限上下文的 alwaysAllowRules
        if (allowedTools.length > 0) {
          modifiedContext = {
            ...modifiedContext,
            getAppState() {
              const appState = modifiedContext.getAppState()
              return {
                ...appState,
                toolPermissionContext: {
                  ...appState.toolPermissionContext,
                  alwaysAllowRules: {
                    ...appState.toolPermissionContext.alwaysAllowRules,
                    command: [...new Set([
                      ...(appState.toolPermissionContext.alwaysAllowRules.command || []),
                      ...allowedTools,
                    ])],
                  },
                },
              }
            },
          }
        }

        // 如果 skill 指定了模型（如 model: opus），切换主循环使用的模型
        // 注意：保留 [1m] 后缀（200k context 标记），防止意外降级
        if (model) {
          modifiedContext = {
            ...modifiedContext,
            options: {
              ...modifiedContext.options,
              mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel),
            },
          }
        }

        // 如果 skill 指定了 effort 级别，覆盖当前设置
        if (effort !== undefined) {
          modifiedContext = {
            ...modifiedContext,
            getAppState() {
              return { ...modifiedContext.getAppState(), effortValue: effort }
            },
          }
        }

        return modifiedContext
      },
    }
  },

  // -------------------------------------------------
  // mapToolResultToToolResultBlockParam
  // 把结果转成 Anthropic API 要求的格式
  // C++ 类比：serialize(result) → JSON
  // -------------------------------------------------
  mapToolResultToToolResultBlockParam(result: Output, toolUseID: string): ToolResultBlockParam {
    // fork 模式：返回摘要文本
    if ('status' in result && result.status === 'forked') {
      return {
        type: 'tool_result',
        tool_use_id: toolUseID,
        content: `Skill "${result.commandName}" completed (forked execution).\n\nResult:\n${result.result}`,
      }
    }

    // inline 模式：只告诉模型"skill 已加载"，实际内容在 newMessages 里
    return {
      type: 'tool_result',
      tool_use_id: toolUseID,
      content: `Launching skill: ${result.commandName}`,
    }
  },
})

// --------------------------------------------------------
// skillHasOnlySafeProperties — 安全属性白名单检查
// --------------------------------------------------------
// C++ 类比：bool isSafe(const Command& cmd) { ... }
//
// 设计原则："默认不信任"——只有在白名单里的属性才是安全的。
// 这样未来新增属性时，不需要修改这里，新属性默认需要用户确认。
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  'name', 'description', 'hasUserSpecifiedDescription', 'isEnabled', 'isHidden',
  'aliases', 'isMcp', 'argumentHint', 'whenToUse', 'paths', 'version',
  'disableModelInvocation', 'userInvocable', 'loadedFrom', 'immediate',
  'userFacingName',
])

function skillHasOnlySafeProperties(command: Command): boolean {
  for (const key of Object.keys(command)) {
    if (SAFE_SKILL_PROPERTIES.has(key)) continue
    // 不在白名单的属性，如果有意义的值 → 不安全
    const value = (command as Record<string, unknown>)[key]
    if (value === undefined || value === null) continue
    if (Array.isArray(value) && value.length === 0) continue
    if (typeof value === 'object' && !Array.isArray(value) && Object.keys(value).length === 0) continue
    return false  // 有未知的有效属性 → 不安全，需要用户确认
  }
  return true
}
```

---

## prompt.ts 补充：skill 列表如何控制 token 预算

```typescript
// --------------------------------------------------------
// SKILL_BUDGET_CONTEXT_PERCENT = 0.01
// --------------------------------------------------------
// skill 名字列表只能用上下文窗口的 1%
// 200k token 窗口 × 1% × 4字符/token = 8000 字符
// 这是防止 skill 列表本身把上下文塞满的"安全阀"

// formatCommandsWithinBudget — 预算内格式化 skill 列表
// 三级降级策略（类比内存分配失败的降级）：
//   1. 全描述（full desc）：总长 ≤ 预算 → 完整显示
//   2. 截断描述（truncated desc）：均分剩余预算，截断非内置 skill 的描述
//   3. 仅名字（names only）：极端情况，每条只显示 "- skill-name"
//
// 内置 skill（source === 'bundled'）永远不被截断，优先保留
```

---

## Q9：Claude Code 在超大代码库下，如何理解自然语言需求、找到相关代码、保证写出正确结果？

> 涉及源码：`src/constants/prompts.ts` · `src/utils/claudemd.ts` · `src/context.ts` · `src/memdir/findRelevantMemories.ts` · `src/native-ts/file-index/index.ts` · `src/tools/LSPTool/LSPTool.ts`

**核心答案**：不是"全量读代码"，而是**六层机制叠加**，让 Claude 像人类工程师一样"边探索边理解边写"。

---

### 第一层：启动时注入环境上下文（System Prompt）

```typescript
// src/constants/prompts.ts — getSystemPrompt() 构建内容
return [
  getSimpleIntroSection(),      // "你是 Claude Code，帮助软件工程任务"
  getSimpleDoingTasksSection(), // 行为准则：先读代码再改、并行工具调用
  getUsingYourToolsSection(),   // 工具指南：GrepTool > bash grep
  envInfo,                      // 当前目录 + 是否 git 仓库 + 平台信息
  claudeMd,                     // ← CLAUDE.md 项目说明书（关键！）
  MEMORY.md,                    // ← 自动记忆索引
]
```

`envInfo` 告诉 Claude 基础环境：
```xml
<env>
Working directory: /Users/me/my-project
Is directory a git repo: Yes
Platform: darwin
</env>
```

> **C++ 类比**：就像给 GDB 传 `--directory` 参数，让调试器知道源码根目录。

---

### 第二层：CLAUDE.md — 项目说明书（开发者手动写的先验知识）

```typescript
// src/utils/claudemd.ts — 加载顺序（优先级由低到高）：
// 1. /etc/claude-code/CLAUDE.md    — 企业级管理员配置
// 2. ~/.claude/CLAUDE.md           — 用户私人全局配置
// 3. CLAUDE.md / .claude/CLAUDE.md — 项目级（check-in 到 git）
// 4. .claude/rules/*.md            — 按主题拆分（可按路径作用域限定）
// 5. CLAUDE.local.md               — 用户私人项目配置（gitignore）
```

典型 CLAUDE.md 内容：
```markdown
## Architecture
- Frontend: React + TypeScript, entry at src/pages/
- Backend: FastAPI, routes in api/routes/

## Commands
- Test: pytest tests/ -v
- Build: npm run build
```

**这是最关键的一层**：Claude 知道架构、目录结构、测试命令，不需要每次从零探索。

---

### 第三层：工具驱动的主动探索（Agentic Loop）

System prompt 里明确约束：

```typescript
// src/constants/prompts.ts
`In general, do not propose changes to code you haven't read.
 If a user asks about or wants you to modify a file, read it first.`
```

**工作流程示例（修复登录 bug）**：

```
用户: "帮我修复登录 bug"
      │
      ▼ 并行工具调用
  GrepTool("login")     → 找到 src/auth/login.ts
  GrepTool("LoginForm") → 找到 components/Login.tsx
      │
      ▼
  FileReadTool: 读取 src/auth/login.ts
      │
      ▼
  GrepTool("validateToken") → 找到所有调用点
      │
      ▼
  FileEditTool 修复 → BashTool 跑测试 → 验证通过
```

**核心工具组合**：

| 工具 | 作用 | C++ 类比 |
|------|------|---------|
| `GrepTool` | 正则搜索文件内容 | `grep -r "pattern" .` |
| `GlobTool` | 按模式找文件 | `find . -name "*.ts"` |
| `FileReadTool` | 读具体文件内容 | `cat file.ts` |
| `LSPTool` | 跳转定义/找引用/调用链 | IDE 的 Go-to-Definition |
| `BashTool` | 跑测试验证结果 | `make test` |

---

### 第四层：LSPTool — 理解代码语义（比 Grep 更聪明）

纯文本 Grep 找"字符串匹配"，LSPTool 找"语义"：

```typescript
// src/tools/LSPTool/LSPTool.ts — 支持的操作
z.enum([
  'goToDefinition',       // 跳到函数/类的真实定义（跨文件）
  'findReferences',       // 找所有调用点（而非字符串出现点）
  'hover',                // 获取类型信息、文档注释
  'documentSymbol',       // 列出文件里所有符号
  'workspaceSymbol',      // 在整个项目里搜索符号名
  'incomingCalls',        // 谁调用了这个函数（上游）
  'outgoingCalls',        // 这个函数调用了谁（下游）
])
```

**修复 `validateToken` 的完整流程**：
1. `workspaceSymbol("validateToken")` → 定位文件和行号
2. `goToDefinition` → 读实现（跳过 re-export 中间层）
3. `findReferences` → 找所有调用点，确保改动不破坏其他地方
4. `incomingCalls` → 理解上游调用链

> **C++ 类比**：相当于把 `clangd` / `ctags` 能力内置进来，`goToDefinition` 对应 clangd 的交叉引用解析。

---

### 第五层：记忆系统 — 跨会话积累上下文

```typescript
// src/memdir/findRelevantMemories.ts
// 每次对话开始，用 Sonnet 小模型判断哪些记忆和当前查询相关
// 只传记忆文件的"名字 + 描述"（不是内容），便宜又快
const SELECT_MEMORIES_SYSTEM_PROMPT = `
Return filenames for memories clearly useful (up to 5).
Only include memories you are CERTAIN will be helpful.
`
```

**两级加载**：
- **第一级（自动）**：`MEMORY.md` 索引摘要 → 每次对话自动注入 system prompt
- **第二级（按需）**：Sonnet 判断相关性 → 注入 attachment → AI 决定是否读详情

---

### 第六层：FileIndex — 文件路径模糊搜索（用于 `@` 补全）

```typescript
// src/native-ts/file-index/index.ts — 基于 nucleo 算法的 TS 实现
class FileIndex {
  loadFromFileListAsync(fileList: string[]) {
    // 异步构建，每 4ms yield 一次
    // 270k+ 文件也不阻塞 UI（约 120ms 构建完）
  }

  search(query: string, limit: number): SearchResult[] {
    // 模糊匹配：bitmap 快速过滤 + 边界加分 + test 文件降权
    // "lgts" → 匹配 "src/auth/loginTests.ts"
    // 整个搜索 8~15ms（270k 文件规模）
  }
}
```

用于用户输入 `@` 时的文件名实时补全，让用户能精确把文件加入上下文。

---

### 六层机制全景图

```
用户提问："帮我加一个限流功能"
        │
【第1层】System Prompt：CWD、git 环境信息
        │
【第2层】CLAUDE.md：架构/目录/技术栈/规范
        │
【第5层】Memory：召回历史决策（"上次说用 Redis 限流"）
        │
【第3层】Agentic Loop：GrepTool → GlobTool → FileReadTool
        │
【第4层】LSPTool：workspaceSymbol → findReferences → incomingCalls
        │
【第6层】行为约束：写代码 → BashTool 跑测试 → 验证通过
        │
        ▼
  输出正确的限流实现
```

### 设计哲学对照表（C++ 视角）

| 问题 | 朴素方案 | Claude Code 的方案 |
|------|---------|-------------------|
| 代码库太大，全量读不进去 | 截断 | 工具按需读，只读相关文件 |
| 不知道从哪开始找 | 随机猜 | GrepTool + GlobTool 精确定位 |
| 找到文件但不理解语义 | 纯文本理解 | LSPTool 拿到类型/调用链 |
| 每次对话从零开始 | 重复探索浪费 token | Memory 系统跨会话积累 |
| 项目有特殊规范 | 违反规范 | CLAUDE.md 预注入知识 |
| 写完代码不知道对不对 | 看 diff 猜 | BashTool 跑测试验证 |
| 用户提到文件名要快速找 | 全局搜索 | FileIndex 270k 文件 8ms 响应 |

**本质**：不是"把代码库输入给 AI"，而是**给 AI 配备工具集，让它像人类工程师一样边探索边理解边验证**。
