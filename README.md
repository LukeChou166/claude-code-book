# 以Claude Code 为例讲述agent设计哲学

> **面向读者**： 对agent感兴趣的同学
> **总体思路**： agent的本质是根据环境(State)，采取一些行为(Tool)，得到一些反馈(ToolOutput)，从而不断调整自己的策略(Learning)，最终实现目标(Goal), llm具有随机的特性，由于采取了llm作为决策中心，harness系统为了降低这种随机性，从刚才那几个组件中，设计出很多确定性的步骤。

agent系统与强化学习对比：
![alt text](image.png)


| 模块对比 |定义 |基于llm的agent | 基于强化学习的agent |
|------|------|------|----------|
| agent | 如何输出action｜llm | 训练的模型 |
| Reward | 做的好不好|反思/验证 | reward model |
| 环境 | 智能体交互对象|个人电脑 | 游戏世界 | 
| 状态 | 智能体能感知到的世界|环境+记忆系统| 通过特征工程进行representation | 
| 动作 |能采取的动作 | tool | 定义好的action |
> **代码路径**：`restored-src/src/`（反编译还原版本）

---

## 章节目录

| 章节 | 主题 | 核心文件 |
|------|------|----------|
| [第1章](./01-architecture-overview.md) | 整体架构总览 | `main.tsx`, `query.ts`, `Tool.ts` |
| [第2章](./02-startup-and-init.md) | 启动流程与初始化 | `main.tsx`, `entrypoints/init.ts` |
| [第3章](./03-query-engine.md) | QueryEngine 对话引擎 | `QueryEngine.ts` |
| [第4章](./04-agentic-loop.md) | Agentic Loop 智能体核心循环 | `query.ts`, `services/tools/toolOrchestration.ts` |
| [第5章](./05-tool-system.md) | 工具系统：让 AI 拥有手 | `Tool.ts`, `tools.ts`, `tools/GrepTool/` |
| [第6章](./06-permissions.md) | 权限系统：安全边界 | `types/permissions.ts`, `hooks/` |
| [第7章](./07-task-orchestration.md) | 任务编排 | `tools/AgentTool/`, `tasks/` |
| [第8章](./08-context-management.md) | 上下文管理与自动压缩 | `services/compact/`, `utils/tokens.ts` |
| [第9章](./09-memory-system.md) | 记忆系统：让 AI 跨会话学习 | `memdir/`, `services/SessionMemory/` |
| [第10章](./10-mcp-integration.md) | MCP 协议集成 | `services/mcp/` |
| [第11章](./11-session-persistence.md) | 会话持久化与恢复 | `utils/sessionStorage.ts`, `QueryEngine.ts` |
| [第13章](./13-state-management.md) | 状态管理：极简自研 Store | `state/store.ts`, `state/AppStateStore.ts`, `state/AppState.tsx` |
| [第14章](./14-error-recovery.md) | 错误处理与恢复机制 | `services/api/withRetry.ts`, `utils/gracefulShutdown.ts`, `utils/abortController.ts` |
| [第15章](./15-security-layers.md) | 5层安全体系：纵深防御 | `constants/prompts.ts`, `types/permissions.ts`, `utils/sandbox/`, `tools/EnterPlanModeTool/`, `utils/hooks.ts` |
| [第16章](./16-tools-catalog.md) | 工具目录详解：TodoWrite、ToolSearch、Skill、Task/Team 系列 | `tools/TodoWriteTool/`, `tools/ToolSearchTool/`, `tools/SkillTool/`, `tools/TaskCreateTool/` 等 |

---

## 如何阅读本书

如果你是完全的初学者，建议按顺序从第1章读起。如果你对某个特定主题感兴趣，可以直接跳到对应章节——每章开头都会说明前置知识。

本书大量引用代码片段，格式如下：

- **行号引用**（项目中真实存在的代码）：直接显示文件路径和行号范围
- **示意代码**（为了说明概念而简化的伪代码）：用普通代码块包裹

---

## 项目概述

Claude Code 是 Anthropic 官方发布的 AI 编程助手 CLI 工具，其核心是一个**客户端驱动的 Agentic Loop**：

```
用户输入 → 构建 System Prompt → 调用 Claude API（流式）
    ↑                                        ↓
    └── 工具执行结果 ←← AI 返回 tool_use 指令
```

它与 Copilot 式补全工具最大的区别在于：Claude Code 不只是"预测下一行代码"，而是真正地**理解任务、规划步骤、执行操作、验证结果**，形成完整的编程智能体。
