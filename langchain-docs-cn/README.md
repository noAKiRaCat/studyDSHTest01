# LangChain / LangGraph / Deep Agents 中文文档

> 依据官方文档（docs.langchain.com，Python 版）整理翻译，覆盖 **LangChain v1**、**LangGraph v1**、**Deep Agents SDK** 三条产品线。
> 译文以「能读懂、能上手、能查」为目标：概念解释 + 完整代码 + 常见坑，代码注释已中文化。

---

## 目录

### 0. 先读这篇

- [三者关系与选型](<00-三者关系与选型.md>)：LangChain、LangGraph、Deep Agents 分别解决什么问题，怎么选。

### 一、LangChain（高层 Agent 框架）

| 篇目 | 内容 |
|---|---|
| [01 快速开始与安装](<langchain/01-快速开始与安装.md>) | 安装、首个 Agent、项目结构 |
| [02 模型与消息](<langchain/02-模型与消息.md>) | `init_chat_model`、多 Provider、消息类型、多模态 |
| [03 工具（Tools）](<langchain/03-工具.md>) | `@tool`、参数模式、运行时上下文、动态工具 |
| [04 Agent 与 create_agent](<langchain/04-Agent与create_agent.md>) | Agent 循环、状态、调用方式、结构化输出 |
| [05 中间件（Middleware）](<langchain/05-中间件.md>) | 钩子模型、内置中间件、自定义中间件 |
| [06 记忆、流式与多智能体](<langchain/06-记忆流式与多智能体.md>) | 短期/长期记忆、事件流、子代理、Handoff、路由 |

### 二、LangGraph（底层编排运行时）

| 篇目 | 内容 |
|---|---|
| [01 总览与核心概念](<langgraph/01-总览与核心概念.md>) | 状态机模型、State/Node/Edge、两种 API |
| [02 Graph API 实战](<langgraph/02-GraphAPI实战.md>) | `StateGraph`、条件边、Reducer、Command、循环 |
| [03 持久化与中断](<langgraph/03-持久化与中断.md>) | Checkpointer、Thread、Store、`interrupt` 人机协同 |
| [04 流式、容错与子图](<langgraph/04-流式容错与子图.md>) | `stream`、重试与超时、子图、时间旅行、部署 |

### 三、Deep Agents（开箱即用的 Agent 运行时骨架）

| 篇目 | 内容 |
|---|---|
| [01 总览与快速开始](<deepagents/01-总览与快速开始.md>) | `create_deep_agent`、内置能力全景 |
| [02 定制、后端与权限](<deepagents/02-定制后端与权限.md>) | 虚拟文件系统、后端类型、声明式权限 |
| [03 子代理与上下文工程](<deepagents/03-子代理与上下文工程.md>) | `task` 工具、技能、记忆、摘要与上下文卸载 |
| [04 生产实践、沙箱与人机协同](<deepagents/04-生产实践沙箱与人机协同.md>) | 沙箱、`interrupt_on`、容错、部署 |

---

## 快速对照

| 维度 | LangChain | LangGraph | Deep Agents |
|---|---|---|---|
| 定位 | 高层 Agent 框架 | 低层编排运行时 | 成品 Agent 骨架 |
| 核心 API | `create_agent` | `StateGraph` / `@entrypoint` | `create_deep_agent` |
| 抽象程度 | 中 | 低（完全可控） | 高（开箱即用） |
| 适合 | 常规工具调用 Agent | 复杂有状态工作流 | 长任务、研究型 Agent |
| 依赖关系 | 构建在 LangGraph 之上 | 独立可用 | 构建在 LangChain + LangGraph 之上 |

---

## 阅读建议

```text
只想快速做出一个 Agent      → 00 → LangChain 01/04
需要复杂流程与人工审核       → 00 → LangGraph 01/02/03
需要长任务、文件操作、子代理  → 00 → Deep Agents 01/03
要做生产部署                → 各自最后一篇 + LangSmith 可观测性
```

---

## 术语对照表

| 英文 | 中文 | 说明 |
|---|---|---|
| Agent | 智能体 | 模型在循环中调用工具直到完成 |
| Harness | 骨架 / 运行装置 | Agent 循环之外的一切：提示词、工具、中间件 |
| Middleware | 中间件 | 在模型调用前后插入逻辑的钩子 |
| State | 状态 | 图执行过程中共享的数据结构 |
| Node | 节点 | 图中的一个处理步骤 |
| Edge | 边 | 节点之间的跳转关系 |
| Checkpointer | 检查点存储器 | 保存图执行快照，支持恢复与回溯 |
| Thread | 线程 | 一次会话的标识（`thread_id`） |
| Interrupt | 中断 | 暂停执行等待人工输入 |
| Subagent | 子代理 | 由主代理派生、独立上下文的代理 |
| Backend | 后端 | 虚拟文件系统的存储实现 |
| Skill | 技能 | 按需加载的领域知识包 |
| Sandbox | 沙箱 | 隔离的代码执行环境 |

---

*本目录由官方文档翻译整理，示例代码保留官方 API 名称，注释已译为中文。日期：2026-11-02*
