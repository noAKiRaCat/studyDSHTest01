# LangGraph 02 · Graph API 实战

> 从零搭一张能跑、能分支、能循环的图。

---

## 1. 定义状态

```python
from typing import Annotated
import operator
from typing_extensions import TypedDict


class ResearchState(TypedDict):
    question: str
    sources: Annotated[list[str], operator.add]   # 多节点写入时自动合并
    draft: str
    review: str
    iterations: int
```

**设计原则**：

- 只放**跨节点共享**的数据；节点内部临时变量不要塞进 state。
- 需要累积的字段用 `operator.add` 或自定义 reducer。
- 字段尽量扁平，嵌套过深会让状态更新难以推理。

---

## 2. 定义节点

```python
def search_node(state: ResearchState):
    results = web_search(state["question"])
    return {"sources": results}          # 只返回要更新的字段


def write_node(state: ResearchState):
    draft = llm.invoke(
        f"根据以下资料回答问题：{state['question']}\n\n"
        + "\n".join(state["sources"])
    ).content
    return {"draft": draft, "iterations": state.get("iterations", 0) + 1}


def review_node(state: ResearchState):
    verdict = llm.invoke(f"评价这份草稿，只回答 通过/需修改：{state['draft']}").content
    return {"review": verdict.strip()}
```

### 节点返回值规则

| 返回 | 含义 |
|---|---|
| `{"key": value}` | 更新指定字段 |
| `None` | 不更新 |
| 抛异常 | 图执行失败（可配重试） |

---

## 3. 连接边

```python
from langgraph.graph import StateGraph, START, END

builder = StateGraph(ResearchState)

builder.add_node("search", search_node)
builder.add_node("write", write_node)
builder.add_node("review", review_node)

builder.add_edge(START, "search")
builder.add_edge("search", "write")
builder.add_edge("write", "review")
```

### 3.1 条件边

```python
def after_review(state: ResearchState) -> str:
    if state["review"] == "通过" or state["iterations"] >= 3:
        return "done"
    return "revise"


builder.add_node("revise", write_node)
builder.add_conditional_edges(
    "review",
    after_review,
    {"done": END, "revise": "revise"},
)
builder.add_edge("revise", "review")
```

`add_conditional_edges` 的第三个参数是**映射表**：router 返回值 → 目标节点。也可以省略映射，直接返回节点名。

### 3.2 并行分支

```python
builder.add_edge("search", "search_web")
builder.add_edge("search", "search_papers")
builder.add_edge("search", "search_news")

builder.add_edge(["search_web", "search_papers", "search_news"], "merge")
```

多入边表示「全部完成后才继续」（join 语义）。

---

## 4. 编译与运行

```python
graph = builder.compile()
result = graph.invoke({"question": "什么是 MCP？", "sources": [], "iterations": 0})
print(result["draft"])
```

带持久化：

```python
from langgraph.checkpoint.memory import InMemorySaver

graph = builder.compile(checkpointer=InMemorySaver())
config = {"configurable": {"thread_id": "research-1"}}
graph.invoke(inputs, config)
```

---

## 5. Command：把「更新状态 + 跳转」合二为一

普通节点只能更新状态，跳转由边决定。`Command` 让节点自己决定去哪：

```python
from langgraph.types import Command


def triage(state: ResearchState) -> Command:
    kind = classify(state["question"])
    if kind == "simple":
        return Command(update={"draft": "直接回答..."}, goto=END)
    return Command(update={"sources": []}, goto="search")
```

适合**动态路由**场景，比条件边更内聚。

---

## 6. Send：动态并行（Map-Reduce）

需要「按数据条数动态派发」时用 `Send`：

```python
from langgraph.types import Send


def fan_out(state: ResearchState):
    return [Send("analyze", {"doc": d}) for d in state["documents"]]


builder.add_conditional_edges("split", fan_out, ["analyze"])
```

每个 `Send` 创建一个并行实例，各自独立执行后汇总。

---

## 7. 子图

把一张图当作另一张图的节点：

```python
sub = sub_builder.compile()
builder.add_node("summarize", sub)
```

要求：子图的输入/输出状态**与父图兼容**（或显式做映射）。

```python
builder.add_node("summarize", sub, input_schema=SummaryInput, output_schema=SummaryOutput)
```

---

## 8. 完整示例：研究 + 审校循环

```python
from typing import Annotated
import operator
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    question: str
    sources: Annotated[list, operator.add]
    draft: str
    ok: bool
    rounds: int


def search(state: State):
    return {"sources": search_web(state["question"])}


def write(state: State):
    text = llm.invoke(
        f"问题：{state['question']}\n资料：{state['sources']}\n写出答案"
    ).content
    return {"draft": text, "rounds": state.get("rounds", 0) + 1}


def review(state: State):
    ok = "通过" in llm.invoke(f"这份答案合格吗？{state['draft']}").content
    return {"ok": ok}


def route(state: State):
    if state["ok"] or state["rounds"] >= 3:
        return END
    return "write"


b = StateGraph(State)
b.add_node("search", search)
b.add_node("write", write)
b.add_node("review", review)

b.add_edge(START, "search")
b.add_edge("search", "write")
b.add_edge("write", "review")
b.add_conditional_edges("review", route, {END: END, "write": "write"})

graph = b.compile()
print(graph.invoke({"question": "解释 Pregel 模型", "sources": [], "rounds": 0})["draft"])
```

---

## 9. 控制最大步数

```python
graph.invoke(inputs, {"recursion_limit": 25})
```

超过限制会抛 `GraphRecursionError`。**生产环境必须设**，否则条件边写错就是无限循环烧钱。

---

## 10. 调试技巧

```python
# 打印每一步
for step in graph.stream(inputs, stream_mode="updates"):
    print(step)

# 看最终状态与中断情况
state = graph.get_state(config)
print(state.values, state.next, state.tasks)

# 看历史
for s in graph.get_state_history(config):
    print(s.config["configurable"]["checkpoint_id"], s.next)
```

---

## 11. 常见坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 状态没合并 | 忘了 reducer | `Annotated[list, operator.add]` |
| 条件边报 `InvalidUpdateError` | router 返回了未声明的目标 | 检查映射表 |
| 并行分支互相覆盖 | 用了覆盖语义 | 用追加 reducer |
| 子图状态不匹配 | schema 不一致 | 用 `input_schema` 转换 |
| 图跑飞 | 无终止条件 | 加 `recursion_limit` 与明确退出条件 |

---

## 下一步

- [03 持久化与中断](<03-持久化与中断.md>)
- [04 流式、容错与子图](<04-流式容错与子图.md>)
