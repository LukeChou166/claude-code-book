# 第4章 工具系统：让 AI 拥有手

> **前置知识**：第1章 整体架构与启动、第3章 Agentic Loop  
> **关键文件**：`src/Tool.ts`（约793行）、`src/tools.ts`、`src/tools/GrepTool/GrepTool.ts`

---

## 4.1 解决什么问题

大语言模型本质上是一个**文本生成器**——它只能输入文字，输出文字。要让 AI 真正地操作代码（读文件、写文件、执行命令），需要一套机制把 AI 的"文字意图"转化为真实的系统操作。

具体问题：
1. **能力注册**：有哪些工具？每个工具能做什么？如何将能力描述给 AI？
2. **参数校验**：AI 传来的参数可能格式错误（比如 JSON 字段名拼错），如何安全解析？
3. **权限控制**：工具有危险操作（删除文件、执行任意命令），如何征得用户同意？
4. **结果序列化**：工具执行结果（可能很大）如何高效地送回给 AI？
5. **UI 渲染**：工具执行时，终端如何显示进度和结果？
6. **并发安全**：哪些工具可以并发执行？哪些必须串行？
7. **工具懒加载**：工具描述本身消耗 token，如何按需加载？

---

## 4.2 业界的做法

**OpenAI Function Calling**：开发者用 JSON Schema 描述工具，API 自动解析 AI 的函数调用。简单，但：
- 没有内置运行时参数验证
- 没有权限控制机制
- 工具结果必须一次性返回（不支持流式/进度更新）
- UI 渲染完全靠开发者自己实现

**LangChain Tool**：
- 优点：抽象层次高，有 `BaseTool` 可继承
- 缺点：关注点（权限、UI、并发安全）散落在不同地方，没有统一接口

---

## 4.3 Tool 接口：统一的关注点容器

`Tool.ts` 定义了一个极其完整的 `Tool` 接口（约 350 行），把所有关注点统一在一个地方：

```typescript
// src/Tool.ts 第 362-695 行（简化展示关键方法）
export type Tool<Input, Output, P extends ToolProgressData> = {
  // ── 基础信息 ──────────────────────────────────────
  readonly name: string          // 工具名（用于 AI 调用）
  aliases?: string[]             // 历史名（向后兼容）
  searchHint?: string            // 工具搜索关键词（3-10个词）

  // ── 核心执行 ──────────────────────────────────────
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  readonly inputSchema: Input    // Zod schema（运行时验证 + TS 类型推断）

  // ── 能力声明 ──────────────────────────────────────
  isConcurrencySafe(input): boolean  // 可否并发执行？
  isEnabled(): boolean               // 当前是否可用？
  isReadOnly(input): boolean         // 是否只读（不修改状态）？
  isDestructive?(input): boolean     // 是否不可逆（删除等）？
  interruptBehavior?(): 'cancel' | 'block'  // 用户中断时如何处理？

  // ── 权限控制 ──────────────────────────────────────
  validateInput?(input, context): Promise<ValidationResult>  // 输入合法性检查
  checkPermissions(input, context): Promise<PermissionResult>  // 权限检查
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>  // 规则匹配

  // ── 上下文窗口管理 ────────────────────────────────
  maxResultSizeChars: number  // 结果超过此值时持久化到磁盘

  // ── UI 渲染（React 组件）─────────────────────────
  renderToolUseMessage(input, options): React.ReactNode      // 展示"AI 调用了什么"
  renderToolResultMessage?(content, ...): React.ReactNode    // 展示执行结果
  renderToolUseProgressMessage?(...): React.ReactNode        // 执行中进度
  renderToolUseRejectedMessage?(...): React.ReactNode        // 被拒绝时展示
  renderToolUseErrorMessage?(...): React.ReactNode           // 执行出错时展示
  renderGroupedToolUse?(...): React.ReactNode | null         // 多个并发调用分组展示

  // ── 文本提取（用于搜索索引）──────────────────────
  extractSearchText?(out: Output): string

  // ── 分析/安全分类 ─────────────────────────────────
  toAutoClassifierInput(input): unknown  // 用于自动安全分类器的输入

  // ── AI 可读描述 ───────────────────────────────────
  prompt(options): Promise<string>   // 完整的工具描述（发给 AI）
  description(input, options): Promise<string>  // 简短描述
  userFacingName(input): string  // 用户界面展示的名字

  // ── API 序列化 ────────────────────────────────────
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
}
```

这个接口涵盖了一个工具从"被 AI 发现"到"执行结果展示在终端"的完整生命周期。

---

## 4.4 关键属性深度解析

### 4.4.1 `inputSchema`：Zod 驱动的双重保障

所有工具的输入参数用 Zod schema 定义：

```typescript
// src/tools/GrepTool/GrepTool.ts 第 33-90 行
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('The regular expression pattern to search for'),
    path: z.string().optional().describe('File or directory to search in'),
    output_mode: z.enum(['content', 'files_with_matches', 'count'])
      .optional()
      .describe('Output mode: "content" shows matching lines...'),
    '-B': semanticNumber(z.number().optional()).describe('Lines before match'),
    '-A': semanticNumber(z.number().optional()).describe('Lines after match'),
    '-i': semanticBoolean(z.boolean().optional()).describe('Case insensitive'),
    head_limit: semanticNumber(z.number().optional()).describe(
      'Limit output to first N lines. Defaults to 250. Pass 0 for unlimited.'
    ),
    // ...
  }),
)
```

**Zod 的两个角色**：

1. **运行时验证**：当 AI 调用工具时，参数会经过 `inputSchema.safeParse(aiInput)` 验证。如果参数格式错误（比如 `output_mode: "invalid_value"`），会返回清晰的错误消息，而不是崩溃。

2. **TypeScript 类型推断**：`z.infer<typeof inputSchema>` 自动生成 TS 类型。`call()` 函数的 `args` 参数是强类型的——访问 `args.pattern` 是 `string`，访问 `args.output_mode` 是 `'content' | 'files_with_matches' | 'count' | undefined`，如果写错字段名，编译时就会报错。

**`lazySchema`**：注意 schema 被包在 `lazySchema()` 中。这是为了避免工具模块被加载时立即初始化 Zod schema（Zod 对象的初始化有一定开销），改为第一次实际用到时才构建。

**`semanticNumber` 和 `semanticBoolean`**：AI 有时会把数字作为字符串传来（`"5"` 而不是 `5`），或布尔值作为字符串（`"true"` 而不是 `true`）。这两个辅助函数扩展了 Zod 的解析逻辑，让工具对这种情况更宽容。

### 4.4.2 `isConcurrencySafe` vs `isReadOnly`

这两个方法名字相近但含义不同：

| 方法 | 含义 | 返回 true 的条件 |
|------|------|-----------------|
| `isReadOnly(input)` | 这个调用是否不会修改任何文件系统/环境状态？ | 纯读取操作 |
| `isConcurrencySafe(input)` | 这个具体调用是否能与其他工具并发执行？ | 读取操作，且不依赖当前写操作的结果 |

它们是独立的：
- `FileReadTool`：`isReadOnly = true`，`isConcurrencySafe = true`
- `GrepTool`：`isReadOnly = true`，`isConcurrencySafe = true`
- `BashTool("ls .")`: `isReadOnly = true`，`isConcurrencySafe = false`（因为 ls 看到的目录状态可能被同时执行的写操作影响）
- `FileEditTool`：`isReadOnly = false`，`isConcurrencySafe = false`

### 4.4.3 `interruptBehavior`

```typescript
interruptBehavior?(): 'cancel' | 'block'
```

当用户在工具执行中途输入新消息时：
- `'cancel'`：立即停止工具，丢弃结果。适用于可以中途停止的长时间操作（如 `BashTool` 执行慢速命令）
- `'block'`：继续执行，新消息等工具完成后再处理。适用于写操作（中断可能导致文件损坏）

没有实现此方法时默认 `'block'`（更保守，更安全）。

### 4.4.4 `maxResultSizeChars`：防止上下文爆炸

```typescript
// src/Tool.ts 第 459-468 行
maxResultSizeChars: number
/**
 * Maximum size in characters for tool result before it gets persisted to disk.
 * When exceeded, the result is saved to a file and Claude receives a preview
 * with the file path instead of the full content.
 *
 * Set to Infinity for tools whose output must never be persisted (e.g. Read,
 * where persisting creates a circular Read→file→Read loop and the tool
 * already self-bounds via its own limits).
 */
```

当工具返回的内容超过阈值（比如 20,000 字符），内容会被写入磁盘临时文件，AI 收到的是文件路径和预览，而不是全部内容。

**为什么 FileReadTool 是 Infinity**：如果 FileReadTool 的输出也被写入文件，AI 调用 FileReadTool 去读这个结果文件，就会无限循环。`FileReadTool` 通过自己的 `limit` 参数（按行数限制）来控制输出大小，所以不需要外部的大小限制。

**GrepTool** 的阈值是 `20_000` 字符。如果搜索结果超过 20K 字符（比如在大型代码库中搜索一个常见单词），结果会被保存到临时文件，AI 收到的是 `"Results saved to /tmp/grep_result_xxxx.txt. Preview: ..."` 这样的消息。

### 4.4.5 `buildTool`：fail-safe 默认值工厂

```typescript
// src/Tool.ts 第 757-791 行
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,   // 保守：假设不安全
  isReadOnly: (_input?) => false,           // 保守：假设会写
  isDestructive: (_input?) => false,
  checkPermissions: (input) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }), // 默认允许
  toAutoClassifierInput: (_input?) => '',   // 默认不参与安全分类
  userFacingName: (_input?) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

`buildTool` 的设计遵循 **fail-safe 原则**：
- `isConcurrencySafe` 默认 `false` → 假设需要串行，不会因为忘记声明并发安全而导致竞态
- `isReadOnly` 默认 `false` → 假设会写，不会因为忘记声明只读而跳过权限检查
- `checkPermissions` 默认 `allow` → 工具本身的权限逻辑优先，通用权限系统兜底

---

## 4.5 GrepTool 深度解析

`GrepTool` 是使用最频繁的代码搜索工具，底层调用 [ripgrep](https://github.com/BurntSushi/ripgrep)：

```typescript
// src/tools/GrepTool/GrepTool.ts 第 310-441 行（简化）
async call({ pattern, path, glob, output_mode, head_limit, ... }, { abortController, getAppState }) {
  const absolutePath = path ? expandPath(path) : getCwd()
  const args = ['--hidden']   // 包含隐藏文件

  // 排除版本控制目录（.git, .svn, .hg...）
  for (const dir of VCS_DIRECTORIES_TO_EXCLUDE) {
    args.push('--glob', `!${dir}`)
  }

  // 限制行长度，防止 base64/压缩内容污染结果
  args.push('--max-columns', '500')

  // 根据参数组装 ripgrep 命令
  if (output_mode === 'files_with_matches') args.push('-l')
  if (case_insensitive) args.push('-i')
  if (multiline) args.push('-U', '--multiline-dotall')

  // 如果 pattern 以 - 开头，用 -e 防止被当作命令行选项
  if (pattern.startsWith('-')) {
    args.push('-e', pattern)
  } else {
    args.push(pattern)
  }

  const results = await ripGrep(args, absolutePath, abortController.signal)

  // 对 files_with_matches 模式，按文件修改时间排序
  // 最近修改的文件排在前面，更可能是用户关注的
  const sortedMatches = results
    .map((_, i) => [_, stats[i].mtimeMs] as const)
    .sort((a, b) => b[1] - a[1])  // 降序（最新在前）
    .map(_ => _[0])

  // 应用 head_limit，截断过长结果
  const { items: finalMatches, appliedLimit } = applyHeadLimit(sortedMatches, head_limit)

  // 绝对路径 → 相对路径，节省 token
  const relativeMatches = finalMatches.map(toRelativePath)

  return { data: { filenames: relativeMatches, numFiles: relativeMatches.length } }
}
```

几个值得注意的细节：

**排除 VCS 目录**：`.git`、`.svn`、`.hg`、`.bzr`、`.jj`（Jujutsu）、`.sl`（Sapling）——后两个是比较新的版本控制系统，说明代码库在持续跟进新工具。

**按修改时间排序**：用户最关注的通常是最近修改的文件，这个优化让 AI 更快找到相关文件。

**路径转相对路径**：把 `/Users/luke/project/src/utils.ts` 转成 `src/utils.ts`，节省上下文 token。

**默认结果数量限制**：`head_limit` 默认 250，防止大型代码库的广泛搜索填满上下文窗口（250 行约 2,500 token）。

---

### 4.5.1 为什么不直接用 `Bash` 调 ripgrep，而要单独封装 GrepTool？

这是一个很自然的问题。既然 BashTool 可以执行任意命令，`rg -rn pattern src/` 也能搜索，为什么还要封装 GrepTool？

答案是：**两者能"做到同一件事"，但对系统的成本完全不同**。封装不是为了能力，而是为了**安全、可预测性和 token 成本**。

#### 1. 安全边界：工作目录限制在 GrepTool 里是自动的

GrepTool 的 `checkPermissions()` 调用 `checkReadPermissionForTool()`，后者有一套完整的 8 步安全检查：

```typescript
// src/utils/permissions/filesystem.ts 第 1030-1193 行（简化）
export function checkReadPermissionForTool(tool, input, toolPermissionContext) {
  // 步骤 1：拦截 UNC 路径（\\server\share）防止访问网络资源
  if (path.startsWith('\\\\') || path.startsWith('//')) → behavior: 'ask'
  
  // 步骤 2：拦截可疑 Windows 路径（备用数据流、短文件名等）
  if (hasSuspiciousWindowsPathPattern(path)) → behavior: 'ask'
  
  // 步骤 3：先查显式 deny 规则
  if (matchingRuleForInput(path, 'read', 'deny')) → behavior: 'deny'
  
  // 步骤 4：再查显式 ask 规则
  if (matchingRuleForInput(path, 'read', 'ask')) → behavior: 'ask'
  
  // 步骤 6：在工作目录内 → 自动允许
  if (pathInAllowedWorkingPath(path, toolPermissionContext)) → behavior: 'allow'
  
  // 步骤 7：内部路径（session memory、plan 文件）→ 允许
  // 步骤 8：有显式 allow 规则 → 允许
  
  // 兜底：要求用户确认
  → behavior: 'ask'
}
```

用 BashTool 跑 `rg pattern ~/` 呢？BashTool 会把整个命令交给 **Bash 安全分类器**（AI 二次判断）或直接弹确认框——这比 GrepTool 多了一轮 AI 推理，慢且贵。

更关键的是：**GrepTool 的工作目录限制是编译进去的行为，不依赖 AI 的判断**。AI 不可能通过某种 prompt 让 GrepTool 搜索 `~/.ssh/`，但它可能被诱导让 BashTool 执行 `rg pattern ~/.ssh/`。

#### 2. 输出后处理：GrepTool 替模型做了很多"体力活"

直接跑 `rg` 返回的是原始文本，模型要自己处理：

```
/Users/luke/project/src/utils/auth.ts:42:  const token = getToken()
/Users/luke/project/src/utils/crypto.ts:18:  return hash(password)
...（可能 5000 行）
```

GrepTool 自动处理了四件事：

| 后处理 | GrepTool 的做法 | 用 Bash 的代价 |
|--------|----------------|---------------|
| **绝对路径 → 相对路径** | `toRelativePath()` 自动转换 | 模型要消耗 token 处理每个长路径 |
| **按修改时间排序** | `Promise.allSettled(stat)` + 排序 | 模型无法知道哪些文件更相关 |
| **结果截断（head_limit）** | 默认 250 行，超出则附带分页提示 | Bash 结果可能塞满整个 context |
| **结果汇总** | 返回 `numFiles`、`numLines`、`numMatches` 结构化字段 | 模型要自己解析和计数 |

以 `files_with_matches` 模式为例，GrepTool 返回给模型的是：

```
Found 3 files
src/utils/auth.ts
src/utils/crypto.ts
src/hooks/useAuth.ts
```

而不是原始的绝对路径列表。节省 token，也降低了模型误读路径的概率。

#### 3. 参数防注入：pattern 以 `-` 开头的处理

ripgrep 的命令行解析有一个边缘情况：如果 `pattern` 以 `-` 开头（比如用户想搜 `-deprecate`），ripgrep 会把它当成命令行选项而不是搜索词，导致出错。

GrepTool 在 `call()` 里处理了这个：

```typescript
// src/tools/GrepTool/GrepTool.ts 第 378-384 行
if (pattern.startsWith('-')) {
  args.push('-e', pattern)  // 用 -e 显式标记为 pattern，不是选项
} else {
  args.push(pattern)
}
```

AI 通过 BashTool 直接调 `rg` 时，很可能不知道这个边缘情况，导致偶发性的失败（而且错误消息很难懂）。

#### 4. 用户配置的 ignore patterns 自动注入

GrepTool 会读取用户在权限设置里配置的文件忽略规则，并自动追加到 ripgrep 参数中：

```typescript
// src/tools/GrepTool/GrepTool.ts 第 411-427 行
const ignorePatterns = normalizePatternsToPath(
  getFileReadIgnorePatterns(appState.toolPermissionContext),
  getCwd(),
)
for (const ignorePattern of ignorePatterns) {
  // 非绝对路径需要加 **/ 前缀，因为 ripgrep 的 gitignore 规则是相对于 cwd 的
  const rgIgnorePattern = ignorePattern.startsWith('/')
    ? `!${ignorePattern}`
    : `!**/${ignorePattern}`
  args.push('--glob', rgIgnorePattern)
}
```

用 BashTool 直接调 `rg` 不会经过这个逻辑，用户配置的 ignore 规则就被绕过了。

#### 4. 可中断性：搜索可以被用户打断

GrepTool 把 `abortController.signal` 传入 `ripGrep()` 调用：

```typescript
const results = await ripGrep(args, absolutePath, abortController.signal)
```

当用户在大型代码库搜索时输入了新消息，搜索可以立即取消（GrepTool 的 `interruptBehavior` 默认 `'cancel'`）。用 BashTool 的话，BashTool 的中断行为是 `'cancel'`，但中断的粒度是整个 bash 进程，而不是 ripgrep 进程——结果可能没那么干净。

#### 总结：封装解决的是"不可见成本"

| 对比维度 | `BashTool("rg ...")` | `GrepTool` |
|---------|---------------------|-----------|
| 安全边界 | 依赖 AI 分类器或用户确认 | 编译进权限规则，AI 无法绕过 |
| 工作目录限制 | 无自动限制 | 自动检查 `pathInAllowedWorkingPath()` |
| ignore 规则 | 不注入 | 自动注入用户配置 |
| 路径格式 | 绝对路径原样返回 | 自动转相对路径 |
| 结果排序 | 无 | 按修改时间排序（最相关在前） |
| 结果截断 | 无保护（可能塞满 context） | 默认 250 行 + 分页提示 |
| 参数安全 | `-` 开头的 pattern 可能出错 | 自动加 `-e` 保护 |
| 可观测性 | 原始文本 | 结构化 `numFiles`、`numLines` |
| token 成本（每次搜索） | 高（路径长 + 无截断） | 低（相对路径 + 截断） |

---

## 4.5.2 AskUserQuestionTool：AI 主动向用户提问

`AskUserQuestionTool` 是 Claude Code 中专门用于**人机交互决策**的工具——当 AI 遇到关键选择点，不确定该采取哪条路时，它会暂停执行流，向用户提出结构化的多选题，等待用户回答后再继续。

### 工具定位与触发场景

这个工具的 `DESCRIPTION` 字段是：

```
Asks the user multiple choice questions to gather information, clarify ambiguity,
understand preferences, make decisions or offer them choices.
```

翻译过来就是：**在执行过程中收集信息、消除歧义、了解偏好、做出决策或为用户提供选择。**

具体来说，AI 应该在以下情况下触发这个工具（来自 `ASK_USER_QUESTION_TOOL_PROMPT`）：

1. **收集用户偏好或需求**：执行一个任务之前，需要确认用户想要哪种实现方式（例如：用 TypeScript 还是 JavaScript？）
2. **澄清模糊指令**：用户的请求存在多种合理解读（例如："重构这个文件"可能意味着拆分成多个小文件，也可能意味着优化内部逻辑）
3. **在执行过程中获取决策**：遇到关键分叉路口（例如：发现有冲突，询问保留哪一方）
4. **向用户提供选择方向**：把几种可行方案呈现给用户，让用户决定走哪条路

**特殊的 Plan Mode 限制**：在计划模式（Plan Mode）中，此工具只能用于**澄清需求或选择方案**，不能用来询问"计划是否可以批准"——那是 `ExitPlanModeTool` 的职责。而且，问题中不能提及"the plan"，因为 UI 在用户调用 `ExitPlanMode` 之前不会显示计划内容。

### 工具的核心特征

**`shouldDefer: true`**：这个工具被标记为延迟加载，不会出现在初始 system prompt 中，需要通过 ToolSearch 搜到之后才能使用。这说明它属于"按需工具"——只有在 AI 预见到需要提问时才会加载。

**`requiresUserInteraction(): true`**：这是 `AskUserQuestion` 最关键的标记。返回 `true` 表明此工具需要真实用户在终端前**实时操作**。Claude Code 框架在调度工具时会检查这个方法，凡是 `requiresUserInteraction()` 为 `true` 的工具，都会被以下场景过滤掉：

- **Channel 模式（Telegram/Discord）**：`isEnabled()` 在检测到 `--channels` 参数时直接返回 `false`，因为消息通道的对话是异步的，没有人坐在键盘前等待多选题
- **无人值守的批处理流**：`interactiveHandler.ts` 会跳过所有 `requiresUserInteraction()` 工具

**`checkPermissions()` 永远返回 `'ask'`**：

```typescript
// src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx 第 182-188 行
async checkPermissions(input) {
  return {
    behavior: 'ask' as const,
    message: 'Answer questions?',
    updatedInput: input
  }
}
```

这是这个工具设计中最有意思的地方：它没有"自动允许"的路径。每次 AI 调用此工具，**一定**会触发权限弹框，把问题呈现给用户。这不是安全限制，而是功能本身——工具的目的就是让用户介入。

**`isReadOnly(): true` 且 `isConcurrencySafe(): true`**：提问这个行为本身不修改任何文件系统状态，也不依赖外部写操作的结果，可以与其他只读工具并发执行。

### 输入结构的精妙设计

```typescript
// 一次调用可以问 1-4 个问题（批量提问，减少打断次数）
questions: z.array(questionSchema()).min(1).max(4)

// 每个问题有 2-4 个选项（必须提供有意义的选项，"Other" 会自动添加）
options: z.array(questionOptionSchema()).min(2).max(4)

// 支持多选（choices are not mutually exclusive 时）
multiSelect: z.boolean().default(false)
```

**批量提问的好处**：如果 AI 有多个问题要问，一次 `AskUserQuestion` 调用可以最多问 4 个，而不是调用 4 次工具。这样用户只会被打断一次，而不是被连续弹出 4 个对话框。

**唯一性约束**：`UNIQUENESS_REFINE` 校验确保同一次调用中每个问题的文本不重复、每个问题内部选项的 label 不重复——防止 AI 生成混乱的重复选项。

**`preview` 字段（可选高级功能）**：每个选项可以携带一个 `preview`，当用户把焦点移动到该选项时，UI 切换为左右两栏布局——左边是选项列表，右边是当前聚焦选项的预览内容（支持 Markdown 或 HTML 片段）。这适用于需要**可视化对比**的场景，比如展示两种不同的代码实现方案。

### 结果如何回传给 AI

用户选择完成后，框架通过 `mapToolResultToToolResultBlockParam()` 把答案格式化为：

```
User has answered your questions: "Which library should we use?"="React Query".
You can now continue with the user's answers in mind.
```

如果用户选了带 preview 的选项并留了注释（notes），结果里还会附带：

```
"Which approach?"="Option A" selected preview:
<div>...option A的HTML预览...</div> user notes: 需要兼容旧版本
```

这些信息会作为 `tool_result` 消息插入对话历史，AI 在下一轮就能看到用户的选择，据此继续执行。

### 与普通交互的对比

| 对比维度 | AI 直接在消息里问问题 | AskUserQuestionTool |
|---------|---------------------|-------------------|
| 结构化程度 | 自由文本，用户可能用任意方式回答 | 结构化多选，答案格式确定 |
| 批量性 | 每次只能问一个问题 | 一次最多问 4 个问题 |
| UI 体验 | 显示在聊天气泡中 | 专用的选择对话框（支持键盘导航） |
| 可选项约束 | 无法约束，用户可能偏题 | 强制选一个（或多个）预设项，可输入 Other |
| 预览功能 | 无法内联展示代码/UI 对比 | 可附带可视化预览辅助决策 |

---

## 4.6 工具懒加载（ToolSearchTool）

当工具很多时，把所有工具描述都放进 system prompt 会消耗大量 token。`ToolSearchTool` 实现了工具的"按需发现"：

```typescript
// 工具被标记为延迟加载
export const GlobTool = buildTool({
  name: 'Glob',
  shouldDefer: true,    // ← 不出现在初始 system prompt
  // ...
})
```

AI 需要先"搜索"工具，才能使用：

```
// AI 的工具调用序列（有 ToolSearch 时）
1. ToolSearch({ query: "read file content" })
   → 返回：FileReadTool 的完整描述
2. FileRead({ path: "src/utils.ts" })
   → 返回：文件内容
```

**好处**：工具 schema 通常每个约 300-500 token。40 个工具全部加载 ≈ 12,000-20,000 token。如果 80% 的工具在大多数对话中用不到，懒加载可以节省 10,000 token 以上的 prompt 大小。

**代价**：每次使用新工具需要一次额外的 ToolSearch 调用，增加约 1-2 秒延迟。

`alwaysLoad: true` 标记的工具永远不延迟加载：

```typescript
export const BashTool = buildTool({
  name: 'Bash',
  alwaysLoad: true,   // ← 总是出现在初始 prompt
  // ...
})
```

---

## 4.7 完整工具列表

```typescript
// src/tools.ts（完整工具清单，简化）
import { AgentTool }         from './tools/AgentTool/'      // 启动子智能体
import { AskUserQuestion }   from './tools/AskUserQuestionTool/'  // 向用户提问
import { BashTool }          from './tools/BashTool/'       // 执行 shell 命令
import { BriefTool }         from './tools/BriefTool/'      // 生成简报
import { ConfigTool }        from './tools/ConfigTool/'     // 读取/修改配置
import { EnterPlanModeTool } from './tools/EnterPlanModeTool/'  // 进入计划模式
import { EnterWorktreeTool } from './tools/EnterWorktreeTool/'  // 进入 worktree
import { ExitPlanModeV2Tool }from './tools/ExitPlanModeTool/'   // 退出计划模式
import { ExitWorktreeTool }  from './tools/ExitWorktreeTool/'   // 退出 worktree
import { FileEditTool }      from './tools/FileEditTool/'   // 精确编辑文件
import { FileReadTool }      from './tools/FileReadTool/'   // 读取文件内容
import { FileWriteTool }     from './tools/FileWriteTool/'  // 写入新文件
import { GlobTool }          from './tools/GlobTool/'       // 文件模式匹配
import { GrepTool }          from './tools/GrepTool/'       // 正则搜索（ripgrep）
import { LSPTool }           from './tools/LSPTool/'        // 语言服务器协议
import { ListMcpResourcesTool } from './tools/ListMcpResourcesTool/'  // 列出 MCP 资源
import { NotebookEditTool }  from './tools/NotebookEditTool/'  // Jupyter Notebook 编辑
import { ReadMcpResourceTool }  from './tools/ReadMcpResourceTool/'   // 读取 MCP 资源
import { SkillTool }         from './tools/SkillTool/'      // 执行用户技能
import { TaskCreateTool }    from './tools/TaskCreateTool/' // 创建待办任务
import { TaskGetTool }       from './tools/TaskGetTool/'    // 查询任务
import { TaskListTool }      from './tools/TaskListTool/'   // 列出任务
import { TaskOutputTool }    from './tools/TaskOutputTool/' // 子任务输出
import { TaskStopTool }      from './tools/TaskStopTool/'   // 停止任务
import { TaskUpdateTool }    from './tools/TaskUpdateTool/' // 更新任务
import { TodoWriteTool }     from './tools/TodoWriteTool/'  // 管理 todo 列表
import { ToolSearchTool }    from './tools/ToolSearchTool/' // 搜索工具
import { TungstenTool }      from './tools/TungstenTool/'   // 内部特殊工具
import { WebFetchTool }      from './tools/WebFetchTool/'   // 抓取网页
import { WebSearchTool }     from './tools/WebSearchTool/'  // 网页搜索
// ...
// 条件加载（feature flag）
const SleepTool = feature('PROACTIVE') ? require(...) : null
const REPLTool = process.env.USER_TYPE === 'ant' ? require(...) : null
const cronTools = feature('AGENT_TRIGGERS') ? [...] : []
```

---

## 4.8 工具的完整生命周期：从定义到执行

> **以 GrepTool 为例，走一遍完整流程**

### 4.8.1 第一阶段：工具序列化为 API Schema

每次向 Claude API 发起请求前，工具列表需要转成 API 能理解的 JSON 格式。这由 `toolToAPISchema()` 完成：

```typescript
// src/utils/api.ts 第 119-134 行
export async function toolToAPISchema(
  tool: Tool,
  options: {
    getToolPermissionContext: () => Promise<ToolPermissionContext>
    tools: Tools
    agents: AgentDefinition[]
    deferLoading?: boolean
    cacheControl?: { type: 'ephemeral'; ... }
  },
): Promise<BetaToolUnion>
```

以 `GrepTool` 为例，这个函数产出如下 JSON：

```json
{
  "name": "Grep",
  "description": "Fast content search tool that works with any codebase size...",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string", "description": "The regular expression pattern to search for" },
      "path":    { "type": "string", "description": "File or directory to search in" },
      "output_mode": { "type": "string", "enum": ["content", "files_with_matches", "count"] },
      "-i": { "type": "boolean", "description": "Case insensitive search" },
      "head_limit": { "type": "number", "description": "Limit output to first N lines. Defaults to 250." }
    },
    "required": ["pattern"]
  }
}
```

**关键细节**：

1. **`tool.prompt()` 生成 description**：不是硬编码字符串，而是每次调用 `tool.prompt()` 动态生成，可以根据当前工作目录、权限模式等状态调整描述内容。

2. **Zod schema → JSON Schema**：`zodToJsonSchema(tool.inputSchema)` 把 Zod 对象转成 JSON Schema 格式。`lazySchema` + `WeakMap` 缓存确保每次 API 请求复用同一份对象，不重复计算：

```typescript
// src/utils/zodToJsonSchema.ts 第 11-20 行
// toolToAPISchema() 每次请求都调用，每 turn 可能 60-250 次
// Tool schema 用 lazySchema() 包裹，保证同一 ZodTypeAny 引用
// 用 WeakMap 按对象身份缓存，避免重复计算
const cache = new WeakMap<ZodTypeAny, JsonSchema7Type>()

export function zodToJsonSchema(schema: ZodTypeAny): JsonSchema7Type {
  const hit = cache.get(schema)
  if (hit) return hit
  const result = toJSONSchema(schema) as JsonSchema7Type
  cache.set(schema, result)
  return result
}
```

3. **`cacheControl` 标记用于 Prompt Caching**：工具列表末尾的工具会打上 `cache_control: { type: 'ephemeral' }` 标记，告诉 Claude API 在此处设置 KV 缓存断点，下次请求如果工具列表不变，可以直接命中缓存，节省大量 token 计算费用。

4. **`defer_loading` 懒加载标记**：对于 `shouldDefer: true` 的工具（如 GlobTool），schema 里会带 `defer_loading: true`，API 收到后不把这个工具的完整描述放入模型的上下文，只有模型调用 ToolSearch 发现并请求后才注入。

---

### 4.8.2 第二阶段：模型返回 `tool_use` block

API 以流式方式返回响应。当模型决定调用 GrepTool 时，返回的 JSON 流里包含：

```json
{
  "type": "tool_use",
  "id": "toolu_01AbCdEf...",
  "name": "Grep",
  "input": {
    "pattern": "TODO",
    "path": "src/",
    "output_mode": "files_with_matches"
  }
}
```

这个 block 在 `query.ts` 的 Agentic Loop 中被收集到 `toolUseBlocks` 数组。等模型的流式响应结束（或遇到 `stop_reason: 'tool_use'`）后，进入工具执行阶段。

---

### 4.8.3 第三阶段：调度——串行还是并发？

`runTools()` 先调用 `partitionToolCalls()` 把工具列表分批：

```typescript
// src/services/tools/toolOrchestration.ts 第 91-116 行
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc, toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const parsedInput = tool?.inputSchema.safeParse(toolUse.input)
    
    // 先 safeParse，再问工具"这次调用是否并发安全"
    const isConcurrencySafe = parsedInput?.success
      ? Boolean(tool?.isConcurrencySafe(parsedInput.data))
      : false
    
    // 连续的 concurrencySafe=true 工具合并进同一个批次
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

假设模型同时调用了三个工具：

```
[GrepTool("TODO"), GrepTool("FIXME"), FileEditTool("fix.ts")]
```

分批结果：
```
批次1: [GrepTool("TODO"), GrepTool("FIXME")]  → isConcurrencySafe=true → 并发执行
批次2: [FileEditTool("fix.ts")]               → isConcurrencySafe=false → 串行执行
```

---

### 4.8.4 第四阶段：权限检查 → 执行 → 结果回填

对每个工具，`checkPermissionsAndCallTool()` 按以下顺序处理：

```
1. inputSchema.safeParse(input)    ← Zod 类型验证
        ↓ 失败 → 返回 InputValidationError
        ↓ 成功
2. tool.validateInput?.(input)     ← 工具自定义业务校验（可选）
        ↓ 失败 → 返回 ValidationError
        ↓ 成功
3. runPreToolUseHooks(...)          ← 执行用户配置的 PreToolUse hooks
        ↓ hook 返回 stop → 终止
        ↓ 通过
4. canUseTool(tool, input)          ← 权限检查（规则匹配 / 用户确认弹窗 / AI 分类器）
        ↓ deny → 返回 PermissionDenied
        ↓ allow
5. tool.call(input, context, ...)   ← 实际执行（GrepTool 调用 ripgrep）
        ↓
6. mapToolResultToToolResultBlockParam(result, toolUseID)
        ↓
7. 把 tool_result 追加到消息队列，回传给下一次 API 调用
```

**GrepTool 执行后的结果**（打包成 `tool_result` block 回传给模型）：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01AbCdEf...",
  "content": "src/utils/debug.ts\nsrc/tasks/DreamTask/index.ts\nsrc/commands/btw/btw.tsx"
}
```

模型收到这个结果后，继续下一轮推理，决定下一步操作（读取其中某个文件，或直接回答用户）。

---

### 4.8.5 完整调用链一览

```
getTools()
  └─ 返回 Tool[] 列表

toolToAPISchema(tool)                        ← src/utils/api.ts
  ├─ tool.prompt() → description（动态生成）
  ├─ zodToJsonSchema(tool.inputSchema) → input_schema
  ├─ 打 cache_control（Prompt Caching）
  └─ 返回 BetaToolUnion（发给 Claude API）

── Claude API 推理 ──

assistant message { tool_use: { name, id, input } }

runTools(toolUseBlocks, ...)                 ← src/services/tools/toolOrchestration.ts
  └─ partitionToolCalls()  → Batch[]
       ├─ 并发批次 → runToolsConcurrently()
       └─ 串行批次 → runToolsSerially()
            └─ runToolUse(toolUse, ...)       ← src/services/tools/toolExecution.ts
                 ├─ findToolByName()          ← 按 name 找工具对象
                 └─ checkPermissionsAndCallTool()
                      ├─ inputSchema.safeParse()    ← Zod 验证
                      ├─ tool.validateInput?()      ← 业务验证
                      ├─ runPreToolUseHooks()        ← hooks
                      ├─ canUseTool()               ← 权限
                      └─ tool.call()                ← 真正执行
                           └─ mapToolResultToToolResultBlockParam()
                                └─ 返回 tool_result → 追加到消息队列
```

---

### 4.8.6 `tool.prompt()` 深度解析：description 如何设计

`toolToAPISchema()` 调用 `await tool.prompt(...)` 获取工具的 `description` 字段。**这个字符串直接作为行为指令注入到 Claude 的上下文**——它不是给人类读者的文档，而是写给模型看的使用说明书。

#### GrepTool 最终发出的 description 字符串

以 `GrepTool` 为例，`getDescription()` 返回的字符串（其中 `${GREP_TOOL_NAME}` = `"Grep"`，`${BASH_TOOL_NAME}` = `"Bash"`，`${AGENT_TOOL_NAME}` = `"Agent"`）展开后完整内容：

```
A powerful search tool built on ripgrep

  Usage:
  - ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command. The Grep tool has been optimized for correct permissions and access.
  - Supports full regex syntax (e.g., "log.*Error", "function\s+\w+")
  - Filter files with glob parameter (e.g., "*.js", "**/*.tsx") or type parameter (e.g., "js", "py", "rust")
  - Output modes: "content" shows matching lines, "files_with_matches" shows only file paths (default), "count" shows match counts
  - Use Agent tool for open-ended searches requiring multiple rounds
  - Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping (use `interface\{\}` to find `interface{}` in Go code)
  - Multiline matching: By default patterns match within single lines only. For cross-line patterns like `struct \{[\s\S]*?field`, use `multiline: true`
```

注意几点：
- 开头一行是简短总结（`A powerful search tool built on ripgrep`），模型在选择工具时先扫这里
- 正文是 Markdown 列表，每条都是**可执行的指令**，而不是功能描述
- **包含禁令**：`NEVER invoke grep or rg as a Bash command` ← 这是防止模型绕过工具直接 Shell 调用
- **包含工具间路由**：`Use Agent tool for open-ended searches requiring multiple rounds` ← 告诉模型什么时候应该选别的工具
- **包含转义规则**：Go 语言 `interface{}` 的写法示例 ← 减少模型因 ripgrep 语法不熟而犯的错误

---

#### `prompt()` 方法的三种实现模式

**模式一：静态字符串（最简单）**

```typescript
// 简单工具，描述不随状态变化
async prompt() {
  return getDescription()  // 返回固定字符串
}
```

GrepTool 就是这种模式——description 内容固定，适合行为稳定的工具。

---

**模式二：动态生成（运行时状态驱动）**

`AgentTool` 的 prompt 是最复杂的例子——它需要把当前可用的 Agent 列表嵌入描述中：

```typescript
// src/tools/AgentTool/prompt.ts 第 66-74 行
export async function getPrompt(
  agentDefinitions: AgentDefinition[],  // 运行时才知道有哪些 Agent
  isCoordinator?: boolean,              // 模式不同，描述不同
  allowedAgentTypes?: string[],         // 权限限制影响可用列表
): Promise<string> {
  const effectiveAgents = allowedAgentTypes
    ? agentDefinitions.filter(a => allowedAgentTypes.includes(a.agentType))
    : agentDefinitions
  // ...拼接出不同模式的 prompt
}
```

生成的描述中会内嵌当前可用的 Agent 列表：

```
Available agent types and the tools they have access to:
- explore: Read-only codebase exploration (Tools: Read, Grep, Glob, WebFetch)
- general-purpose: Full task execution (Tools: All tools)
- code-reviewer: Code review and analysis (Tools: Read, Grep, Glob)
```

**但这里有一个性能权衡**：Agent 列表一旦变化（新加 MCP、重载插件、权限模式切换），整个工具 schema 缓存就失效，导致 Prompt Caching 命中率下降。实际上这是 fleet 中 `cache_creation_tokens` 的重要来源之一（约 10.2%）。

解决方案是把动态部分抽离到消息流里（`agent_listing_delta` attachment），让工具 description 本身变成静态字符串：

```typescript
// src/tools/AgentTool/prompt.ts 第 194-198 行
const listViaAttachment = shouldInjectAgentListInMessages()  // feature flag
const agentListSection = listViaAttachment
  ? `Available agent types are listed in <system-reminder> messages in the conversation.`
  : `Available agent types and the tools they have access to:\n${effectiveAgents.map(...)...}`
```

---

**模式三：按配额截断（防 token 爆炸）**

`SkillTool` 的 prompt 需要列举所有用户定义的 Skill，但每个用户的 Skill 数量不固定。直接列全会超出 token 预算：

```typescript
// src/tools/SkillTool/prompt.ts 第 29-48 行
export const MAX_LISTING_DESC_CHARS = 250  // 每条 Skill 描述的字符上限

export function getCharBudget(contextWindowTokens?: number): number {
  // 根据模型的上下文窗口大小，按比例分配给 Skill 列表的 token 预算
  if (contextWindowTokens) {
    return Math.floor(
      contextWindowTokens * CHARS_PER_TOKEN * SKILL_BUDGET_CONTEXT_PERCENT
    )
  }
  return DEFAULT_CHAR_BUDGET
}
```

超出预算的 Skill 描述会被截断：`- /my-long-skill: This is a very long description that...…`（末尾加 `…`）。内置（bundled）Skill 永远不截断，因为它们是高优先级的。

---

#### Tool Prompt 设计的五条原则

结合项目中所有工具 prompt 的实际写法，可以总结出以下规律：

| 原则 | 说明 | GrepTool 的体现 |
|------|------|----------------|
| **禁令优先于说明** | 先说"不要做什么"，再说"怎么做" | `NEVER invoke grep as Bash command` 在最前面 |
| **包含工具间路由** | 明确说什么情况下应该用别的工具 | `Use Agent tool for open-ended searches` |
| **给出常见错误的解法** | 预防模型常犯的错 | Go 语言 `{}` 转义示例 |
| **动态内容尽量抽离** | 状态变化的内容不放进 description，放进消息流 | AgentTool 的 agent list 移到 attachment |
| **description 和 inputSchema 互补** | description 讲"何时用/怎么用"，inputSchema 的 `.describe()` 讲"这个参数是什么" | pattern 的 `.describe()` 说参数含义，prompt 说整体用法 |

---

#### `inputSchema` 的 `.describe()` 也是 prompt 的一部分

发给模型的 JSON Schema 里，每个参数的 `description` 来自 Zod 的 `.describe()` 调用：

```typescript
// src/tools/GrepTool/GrepTool.ts 第 58-82 行
'-B': semanticNumber(z.number().optional()).describe(
  'Number of lines to show before each match (rg -B). Requires output_mode: "content", ignored otherwise.',
),
head_limit: semanticNumber(z.number().optional()).describe(
  'Limit output to first N lines/entries... Defaults to 250 when unspecified. Pass 0 for unlimited (use sparingly — large result sets waste context).',
),
```

最终模型拿到的参数描述（`input_schema.properties.head_limit.description`）就是：

```
Limit output to first N lines/entries, equivalent to "| head -N".
Works across all output modes... Defaults to 250 when unspecified.
Pass 0 for unlimited (use sparingly — large result sets waste context).
```

括号里的 `use sparingly — large result sets waste context` 是直接对模型的行为引导——**参数 description 和 tool description 一起构成了完整的行为约束**。

---

## 4.9 优势与不足

### 优势

**统一的关注点容器**：一个工具接口涵盖执行、校验、权限、UI 渲染、并发策略、安全分类——所有工具的这些方面都在同一个地方定义，没有散落在多处的"配套代码"。

**Zod 双重保障**：运行时防止格式错误的 AI 参数导致崩溃；编译时通过类型推断防止工具内部代码出错。

**`buildTool` 工厂的 fail-safe 默认值**：忘记声明某个属性，会得到保守（安全）的行为，而不是不安全的行为。

**工具懒加载**：对于工具密集的会话（比如数据分析），ToolSearch 可以节省 10,000+ token/请求，直接影响成本和响应速度。

**按修改时间排序的搜索结果**：把最相关的文件排在前面，帮助 AI 更快定位问题。

### 不足

**Tool 接口有约 40 个方法/属性**，对于实现新工具的开发者来说，学习曲线很陡。虽然 `buildTool` 提供了默认值，但要写一个"完善的"工具（有良好的 UI、安全分类、错误处理），仍需要实现很多方法。

**React 渲染方法混入业务逻辑**：`renderToolResultMessage`、`renderToolUseMessage` 等 React 组件代码与工具的核心执行逻辑混在同一个文件中，这使得工具定义耦合了 UI 框架。如果将来要在无 UI 的服务端环境运行 Claude Code，这些渲染方法就是不必要的依赖。

**工具 schema 是运行时生成**：由于 Zod schema 是动态的，无法在编译时生成 OpenAPI/JSON Schema 文档，需要专门的工具来提取。
