# LangChain 04 · Agent 与 create_agent

> `create_agent` 是整个 v1 的中心。理解它的参数，就理解了 LangChain 的能力边界。

---

## 1. 什么是 Agent

```text
Agent = Model + Harness
Harness = 系统提示词 + 工具 + 中间件
```

Agent 的本质是一个循环：

```text
while 任务未完成:
    响应 = 模型(消息历史, 工具列表)
    if 响应.tool_calls:
        结果 = 执行工具(响应.tool_calls)
        消息历史 += 结果
    else:
        break
```

---

## 2. 最简用法

```python
from langchain.agents import create_agent

agent = create_agent(model="openai:gpt-5.5", tools=tools)
result = agent.invoke({"messages": [{"role": "user", "content": "你好"}]})
```

`create_agent` 返回的是**编译好的 LangGraph 图**，因此具备 `invoke` / `stream` / `ainvoke` / `astream` 全套方法。

---

## 3. 参数全表

| 参数 | 类型 | 说明 |
|---|---|---|
| `model` | str / BaseChatModel | 模型标识或实例 |
| `tools` | list | 工具列表（函数、`@tool`、dict） |
| `system_prompt` | str / SystemMessage | 系统提示词 |
| `middleware` | list | 中间件栈 |
| `response_format` | schema | 结构化输出 |
| `state_schema` | TypedDict 子类 | 自定义状态字段 |
| `context_schema` | dataclass / TypedDict | 每次运行的上下文 |
| `checkpointer` | BaseCheckpointSaver | 持久化对话历史 |
| `store` | BaseStore | 长期记忆存储 |
| `name` | str | 图名（多代理时用于标识） |

---

## 4. 状态（State）

Agent 的状态是一个 TypedDict：

```python
from langchain.agents import AgentState

AgentState = {
    "messages": list[BaseMessage],   # 内置字段，只追加
}
```

### 4.1 自定义状态字段

```python
from langchain.agents import AgentState, create_agent


class MyState(AgentState):
    user_id: str
    call_count: int


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[],
    state_schema=MyState,
)
```

### 4.2 在工具与中间件中读写状态

```python
from langchain.tools import tool, ToolRuntime


@tool
def bump_counter(runtime: ToolRuntime) -> str:
    """把调用计数加一。"""
    count = runtime.state.get("call_count", 0) + 1
    return f"当前计数：{count}"


from langchain.agents.middleware import after_model


@after_model
def record(state, runtime):
    return {"call_count": state.get("call_count", 0) + 1}
```

> 中间件返回的 dict 会被**合并**进状态；`messages` 字段是追加语义，其他字段是覆盖语义。

---

## 5. 调用方式

### 5.1 `invoke`（同步、一次性）

```python
result = agent.invoke({"messages": [{"role": "user", "content": "查天气"}]})
print(result["messages"][-1].content)
```

### 5.2 `stream`（流式）

```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "写一段介绍"}]},
    stream_mode="messages",
):
    print(chunk)
```

`stream_mode` 取值：

| 值 | 输出内容 |
|---|---|
| `"values"` | 每一步的完整状态 |
| `"updates"` | 每一步的增量 |
| `"messages"` | 逐 token 的消息片段 |
| `"custom"` | 自定义事件 |
| `"debug"` | 调试信息 |

### 5.3 多轮对话

```python
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.utils.uuid import uuid7

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[],
    checkpointer=InMemorySaver(),   # 必须配置检查点
)

config = {"configurable": {"thread_id": str(uuid7())}}

agent.invoke({"messages": [{"role": "user", "content": "我叫小明"}]}, config=config)
agent.invoke({"messages": [{"role": "user", "content": "我叫什么？"}]}, config=config)
# 第二次调用能记住「小明」
```

**关键**：`thread_id` 相同的调用共享历史；不同 `thread_id` 相互隔离。

### 5.4 异步

```python
result = await agent.ainvoke({"messages": [...]})

async for event in agent.astream({"messages": [...]}, stream_mode="messages"):
    ...
```

---

## 6. 每次运行的上下文（Context）

`config` 管**会话级**（谁的历史），`context` 管**单次运行级**（本次的密钥、用户 ID、开关）：

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver


@dataclass
class Context:
    user_id: str
    tenant: str


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_my_orders],
    context_schema=Context,
    checkpointer=InMemorySaver(),
)

agent.invoke(
    {"messages": [{"role": "user", "content": "我的订单"}]},
    config={"configurable": {"thread_id": "t1"}},
    context=Context(user_id="u_1", tenant="acme"),
)
```

在工具/中间件里通过 `runtime.context` 访问。

---

## 7. 结构化输出

```python
from pydantic import BaseModel
from langchain.agents import create_agent


class Answer(BaseModel):
    summary: str
    confidence: float


agent = create_agent(model="openai:gpt-5.5", tools=tools, response_format=Answer)
result = agent.invoke({"messages": [{"role": "user", "content": "总结 AI 趋势"}]})
print(result["structured_response"])
```

---

## 8. 能力扩展：靠中间件叠加

`create_agent` 本身极简，能力通过 `middleware` 叠加：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ModelRetryMiddleware,
    ToolRetryMiddleware,
    PIIMiddleware,
    HumanInTheLoopMiddleware,
    SummarizationMiddleware,
)

agent = create_agent(
    model="openai:gpt-5.5",
    tools=tools,
    middleware=[
        ModelRetryMiddleware(max_retries=3),          # 模型失败重试
        ToolRetryMiddleware(max_retries=2),           # 工具失败重试
        PIIMiddleware("email"),                       # 邮箱脱敏
        SummarizationMiddleware(max_tokens=4000),     # 历史压缩
        HumanInTheLoopMiddleware(interrupt_on={"write_file": True}),  # 敏感操作审批
    ],
)
```

---

## 9. 作为子图嵌入

给 Agent 起名后，可以作为多代理系统中的节点：

```python
researcher = create_agent(model="openai:gpt-5.5", tools=[search], name="researcher")
writer = create_agent(model="openai:gpt-5.5", tools=[], name="writer")

# 在 LangGraph 里作为节点使用
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
```

---

## 10. 常见坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 多轮对话丢历史 | 没配 `checkpointer` 或 `thread_id` 变了 | 两者都要 |
| `context` 取到 None | 没传 `context_schema` | 声明 schema 并传值 |
| 自定义状态字段被覆盖 | 用了非追加语义 | 需要累加就用 `Annotated[..., operator.add]` |
| 死循环调用工具 | 工具总返回「需要继续」 | 加最大迭代数或让工具给出终止信号 |

---

## 下一步

- [05 中间件](<05-中间件.md>)：真正把 Agent 变强的地方
- [06 记忆、流式与多智能体](<06-记忆流式与多智能体.md>)
