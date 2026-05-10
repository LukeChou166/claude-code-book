# 第5章 工具系统 — 代码问答笔记

本文件记录阅读 `05-tool-system.md` 过程中对具体代码的问答，作为正文章节的补充。

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

搜索 `-deprecate` 这样的词时，直接传给 `rg` 会被解析为命令行选项出错。AI 直接调 BashTool 时大概率不知道这个细节。

### 4. 用户 ignore 规则自动注入

```typescript
// src/tools/GrepTool/GrepTool.ts 第 411-427 行
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

用户在设置里配置的"不允许读取的文件模式"自动变成 `--glob !xxx` 参数。用 BashTool 直接跑 `rg` 完全绕过这个机制。

### 5. 可中断性

GrepTool 把 `abortController.signal` 传入 `ripGrep()` 调用，用户输入新消息时搜索可以立即取消。

### 总结对比

| 维度 | `BashTool("rg ...")` | `GrepTool` |
|------|---------------------|-----------|
| 安全边界 | 依赖 AI 分类器或用户确认 | 编译进权限规则，AI 无法绕过 |
| 工作目录限制 | 无自动限制 | 自动检查 `pathInAllowedWorkingPath()` |
| ignore 规则 | 不注入 | 自动注入用户配置 |
| 路径格式 | 绝对路径原样返回 | 自动转相对路径 |
| 结果排序 | 无 | 按修改时间排序 |
| 结果截断 | 无保护（可能塞满 context） | 默认 250 行 + 分页提示 |
| 参数安全 | `-` 开头的 pattern 可能出错 | 自动加 `-e` 保护 |
| token 成本 | 高（路径长 + 无截断） | 低（相对路径 + 截断） |

---

## Q2：`import { z } from 'zod/v4'` 是什么？

这是 TypeScript 的模块导入语句，对应 C++ 的 `#include`：

```cpp
// C++ 类比
#include <nlohmann/json.hpp>
using json = nlohmann::json;
```

**zod** 是一个运行时数据验证库，`z` 是它的主命名空间。在这个项目里用来描述工具的输入参数格式：

```typescript
z.strictObject({
  pattern: z.string(),          // pattern 必须是字符串
  path: z.string().optional(),  // path 是可选字符串
  '-i': z.boolean().optional(), // -i 是可选布尔值
})
```

---

## Q3：`lazySchema(() => ...)` 里的 `() =>` 是什么语法？

`() =>` 是 TypeScript/JavaScript 的**箭头函数**，完全对应 C++ 的 lambda 表达式：

| C++ | TypeScript |
|-----|-----------|
| `[]() { return x; }` | `() => x` |
| `[capture](int a) { return a+1; }` | `(a) => a + 1` |
| `auto f = []() {...}` | `const f = () => {...}` |

当函数体只有一个返回表达式时可以省略花括号和 `return`：

```typescript
// 完整写法
const f = () => { return z.strictObject({...}) }

// 简写（等价）
const f = () => z.strictObject({...})
```

所以 `lazySchema(() => z.strictObject({...}))` 就是把一个"待执行的代码块"作为参数传进去，对应 C++：

```cpp
lazySchema([]() {
    return z.strictObject({...});
});
```

---

## Q4：`lazySchema` 懒加载的原理是什么？

实现只有 3 行：

```typescript
// src/utils/lazySchema.ts
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined        // 相当于 static T* instance = nullptr
  return () => (cached ??= factory()) // 相当于 if (!instance) instance = new T()
}
```

`??=` 是"空值合并赋值"：只有当左边是 `null` / `undefined` 时才赋值，等价于 C++ 的 `if (ptr == nullptr) ptr = new T()`。

执行流程：

```
第1次调用 inputSchema()
    ↓
cached === undefined → 执行 z.strictObject({...}) → 存入 cached → 返回
                                                                    ↓
第2次调用 inputSchema()                                        （同一个对象）
    ↓
cached !== undefined → 直接返回 cached（跳过构建）
```

---

## Q5：`lazySchema` 是不是对应 C++ 的单例？

**是的，本质上就是懒汉式单例。**

C++ 懒汉式单例：
```cpp
static T* instance = nullptr;
if (instance == nullptr) {
    instance = new T();  // 第一次才构建
}
return *instance;
```

`lazySchema` 等价物：
```typescript
let cached: T | undefined
return () => (cached ??= factory())
```

**区别**：C++ 单例通常绑定在类上，`lazySchema` 用**闭包**来持有 `cached` 变量，外部访问不到（相当于 `private static`）。每次调用 `lazySchema(factory)` 产生一个**独立的单例**——GrepTool 有自己的 schema 单例，BashTool 有自己的，互不干扰，比传统类单例更灵活。
