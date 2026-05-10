# 第1章 整体架构总览

> **前置知识**：了解 TypeScript 基础语法，知道什么是 CLI 工具  
> **关键文件**：`src/main.tsx`、`src/query.ts`、`src/Tool.ts`、`src/tools.ts`

---

## 1.1 解决什么问题

在 Claude Code 出现之前，AI 辅助编程的主流形态是**代码补全**（GitHub Copilot 等）：用户在 IDE 里写代码，AI 在光标处预测下一行。这种模式有天然的局限：

1. **单点感知**：AI 只看到当前打开的几个文件，无法理解整个项目的依赖关系。
2. **无工具调用**：AI 无法真正执行命令、读取文件、运行测试——它只能"预测文本"。
3. **无多轮交互**：Copilot 式补全是一次性的，缺乏对话式推理能力。
4. **上下文窗口太小**：早期 Copilot 只能感知约 512 个 token 的上下文，对于大型项目几乎无效。

Claude Code 的目标是构建一个**真正的编程智能体（Coding Agent）**：它能读文件、写文件、执行 Shell、调用外部工具、跨会话记忆，并且在没有用户干预的情况下完成从"我要修这个 Bug"到"代码已经 commit 并且测试通过"的完整任务链。

---

## 1.2 业界的做法

**代码补全派**（GitHub Copilot, Cursor Tab）：
- 优点：低延迟，适合"快速补全"场景
- 缺点：无任务规划能力，无法跨文件操作

**对话助手派**（ChatGPT, Claude.ai 网页版）：
- 优点：可以多轮交互，可以"思考"
- 缺点：无法真正操作文件系统，只能输出代码片段让用户手动复制

**早期 Agentic 工具**（AutoGPT, BabyAGI）：
- 优点：自主规划和执行
- 缺点：工程质量差，容易死循环，缺乏安全机制，对大代码库无能为力

**竞争者**（Cursor Agent, Copilot Workspace）：
- 优点：集成在 IDE 中，有文件系统访问
- 缺点：闭源，可扩展性差，不支持自定义工具

---

## 1.3 本项目的做法：四层架构

读完代码，可以归纳出一个清晰的四层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                      第四层：用户界面                         │
│   TUI (ink/React)  │  SDK (QueryEngine API)  │  CLI (print) │
├─────────────────────────────────────────────────────────────┤
│                      第三层：会话管理                         │
│           QueryEngine  ←→  query.ts (主循环)                 │
├─────────────────────────────────────────────────────────────┤
│                    第二层：工具与智能体                        │
│  BashTool  FileReadTool  GrepTool  AgentTool  SkillTool  MCP│
├─────────────────────────────────────────────────────────────┤
│                      第一层：基础设施                         │
│  Anthropic API │ 权限系统 │ 上下文管理 │ 存储 │ 分析/遥测     │
└─────────────────────────────────────────────────────────────┘
```

每一层的核心职责：

| 层级 | 关键模块 | 核心职责 |
|------|----------|----------|
| 用户界面 | `main.tsx`, `ink/` | 命令行交互、React 组件渲染到终端 |
| 会话管理 | `QueryEngine.ts`, `query.ts` | 管理对话状态、驱动智能体循环 |
| 工具与智能体 | `tools/`, `tasks/` | 实现 AI 可调用的每个能力 |
| 基础设施 | `services/`, `utils/` | API 调用、权限、压缩、存储 |

---

## 1.4 信息流：一次用户输入如何变成代码修改

以用户输入 `"帮我修复 src/utils.ts 里的 bug"` 为例，整个请求链路：

```
1. 用户按下回车
   │
   ▼
2. main.tsx: processUserInput()
   │  解析命令/普通消息
   │  判断是否是 /slash 命令
   │
   ▼
3. QueryEngine.ts: processUserTurn()
   │  构造 UserMessage
   │  加载记忆文件（CLAUDE.md）
   │  构建 systemPrompt
   │
   ▼
4. query.ts: query() → queryLoop()    ← 核心 Agentic Loop
   │
   │  ① 发送消息给 Claude API（流式）
   │  ② 接收 AI 回复（可能含 tool_use 块）
   │  ③ 执行工具：
   │     ├── FileReadTool("src/utils.ts")    → 读取文件内容
   │     ├── GrepTool("error pattern")       → 搜索错误模式
   │     └── FileEditTool("修复内容")         → 写入修复
   │  ④ 把工具结果作为 tool_result 送回 AI
   │  ⑤ AI 决定：继续？还是任务完成？
   │  ⑥ 如果继续，重新执行 ②
   │
   ▼
5. 流式输出到终端 / SDK
```

---

## 1.5 技术选型解读

### TypeScript + Bun

Claude Code 用 [Bun](https://bun.sh/) 作为运行时和打包器，而不是传统的 Node.js + webpack/esbuild。

代码里充斥着这样的写法：

```typescript
// src/query.ts（第 15-20 行）
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? (require('./services/compact/reactiveCompact.js') as typeof import('./services/compact/reactiveCompact.js'))
  : null
const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? (require('./services/contextCollapse/index.js') as typeof import('./services/contextCollapse/index.js'))
  : null
```

`feature('FLAG')` 是 Bun 的 **Dead-Code Elimination (DCE)** 特性：在编译时，如果 `FLAG` 为 `false`，整个三元表达式的 false 分支会被从打包产物中删除，就像 C 语言的 `#ifdef`。

这让 Anthropic 可以：
- 在同一份代码库里维护外部版和内部版（`ant` 用户）功能
- 通过构建标志选择性开启实验特性，不增加生产包体积
- 避免 feature flag 检查在运行时的开销

### React (ink) 驱动终端 UI

React 不只用于浏览器。[ink](https://github.com/vadimdemedes/ink) 库把 React 组件渲染到命令行终端。这意味着：
- 终端里的进度条、工具调用展示、权限确认弹窗都是 React 组件
- 有完整的状态管理（`useState`, `useEffect`）和生命周期
- UI 更新是响应式的：当工具执行状态变化，对应的 React 组件自动重渲染

**与浏览器 React 的对比：**

| 维度 | 浏览器 React | Claude Code ink/React |
|------|-------------|----------------------|
| 渲染目标 | DOM（HTML 元素） | 终端（ANSI 字符） |
| 布局引擎 | CSS Flexbox（浏览器内置） | **Yoga**（Facebook 独立 Flexbox 实现） |
| 基础组件 | `<div>`、`<span>` | `<Box>`（≈div）、`<Text>`（≈span） |
| 事件 | 鼠标/键盘（原生 DOM） | 键盘/鼠标（解析 ANSI 转义序列） |
| 输出 | 像素 | ANSI 转义码字符串 |

**渲染原理（四步流水线）：**

1. **自定义 React Reconciler**（`src/ink/reconciler.ts`）：调用 `react-reconciler` 创建自定义渲染器，替换掉 `react-dom`。React 的 diff/更新算法完全复用，只是最终"操作 DOM"变成了"操作 ink 的虚拟节点"。

2. **虚拟节点类型**（`src/ink/dom.ts`）：ink 有自己的"HTML 标签"——`ink-box`（对应 `<Box>`）、`ink-text`（对应 `<Text>`）等，存在于内存的虚拟节点树里。

3. **Yoga 布局引擎**（`src/ink/layout/yoga.ts`）：每个 `<Box>` 节点对应一个 Yoga 节点（Meta 开源的独立 C++ Flexbox 实现，通过 native-ts 绑定调用）。Yoga 计算出每个组件的位置和尺寸，单位是"字符格"。

4. **差量渲染到屏幕**（`src/ink/render-node-to-output.ts`）：渲染器把虚拟节点树遍历一遍，生成字符格矩阵（Screen），然后与上一帧 diff，只把变化的部分写入 stdout（ANSI 转义码控制光标位置和颜色）。稳定帧只更新变化的字符格 O(changed cells)，布局变动帧才全量重绘 O(rows×cols)。

**整体数据流：**
```
React 组件（<Box>, <Text>）
      ↓  React Reconciler（自定义渲染器）
  ink 虚拟 DOM（ink-box, ink-text 节点）
      ↓  Yoga 布局引擎
  每个节点的 (x, y, width, height) 字符坐标
      ↓  render-node-to-output.ts
  字符格矩阵 Screen（每格存字符+颜色+样式）
      ↓  writeDiffToTerminal()（差量 diff）
  ANSI 转义码 → stdout → 终端屏幕
```

Claude Code 完全自研了这套终端渲染引擎（`src/ink/` 目录，约 50 个文件），实现了 Flexbox 布局、鼠标事件、键盘导航、差量渲染、滚动、选中、搜索高亮、主题系统等完整特性。

### 异步生成器（AsyncGenerator）作为流式管道

`query.ts` 里核心的 `query()` 函数签名：

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

它不是普通函数，而是**异步生成器函数**（`async function*`）。调用方用 `for await (const event of query(...))` 来消费流式事件：

```typescript
// 消费方（简化示意）
for await (const event of query(params)) {
  if (event.type === 'assistant') {
    // 流式渲染 AI 回复
    renderText(event.content)
  } else if (event.type === 'tool_result') {
    // 显示工具执行结果
    renderToolResult(event)
  }
}
```

这种设计的好处：
1. **背压控制**：消费者来不及处理时，生产者自动暂停（yield 挂起）
2. **资源复用**：整个智能体循环复用同一个 HTTP 连接的流式传输
3. **透明性**：调用方不需要知道内部是单轮还是多轮——它只关心事件流

### Zod 用于运行时类型安全

项目大量使用 [Zod](https://zod.dev/) 库定义数据 schema：

```typescript
// 工具输入 schema 示例（GrepTool）
const inputSchema = z.strictObject({
  pattern: z.string().describe('The regular expression pattern'),
  path: z.string().optional(),
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  // ...
})
```

Zod 的作用：
1. **运行时验证**：`inputSchema.safeParse(aiInput)` 确保 AI 传来的参数格式正确
2. **TypeScript 类型推断**：`z.infer<typeof inputSchema>` 自动生成 TS 类型
3. **描述即文档**：`.describe()` 注释会作为 tool schema 的一部分发给 AI，告诉 AI 每个参数的含义

---

## 1.6 项目比业界好在哪

| 维度 | 业界做法 | 本项目做法 | 本项目优势 |
|------|----------|------------|------------|
| 工具并发 | 串行执行 | 按只读/写操作智能并发 | 多文件操作快 2-5x |
| 上下文管理 | 手动截断或不管理 | 自动压缩（AI 生成摘要）| 无感知地处理超长会话 |
| 权限控制 | 无 / 全信任 | 五级权限模型 + Hooks | 安全与效率平衡 |
| 工具扩展 | 需要 fork 代码 | MCP 协议 + 插件系统 | 生态开放 |
| SDK 复用 | CLI 工具不可嵌入 | QueryEngine 提供 SDK 接口 | 可被 IDE 插件复用 |
| 特性管理 | 维护多个分支 | Bun DCE + Feature Flag | 一套代码，多个版本 |

## 1.7 本项目的不足

1. **冷启动时间**：即使有并行预热优化，首次启动仍需 300-500ms，在 CI 频繁调用时累积开销显著。
2. **`query.ts` 过度耦合**：约 1700 行的 `queryLoop` 函数把状态机、压缩、错误恢复、token 追踪全部耦合，极难维护。
3. **Tool 接口过于复杂**：约 40 个方法/属性，对新工具开发者来说学习曲线陡峭。
4. **无本地缓存语义索引**：目前的代码搜索依赖 ripgrep（文本搜索），没有持久化的 AST 索引，对于"找到所有实现了接口 X 的类"这类语义查询效率较低。
