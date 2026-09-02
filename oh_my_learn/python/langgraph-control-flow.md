# LangGraph 控制流：条件边、并行、`Send` 与 `Command`

> 本文从“节点做什么、边决定去哪、状态传递什么”出发，解释 LangGraph 中静态边、条件边、动态并行和状态聚合的关系。

## 这组知识解决什么问题

普通 Python 程序可以直接用 `if`、`for` 和函数调用控制流程。LangGraph 把这些控制关系显式组织成图，适合需要循环、并行、暂停恢复或可观测执行过程的 Agent。

本文重点回答：

- `State`、`Node`、`Edge` 分别负责什么；
- `add_edge()` 和 `add_conditional_edges()` 有什么区别；
- 条件边返回多个节点时如何并行；
- 任务数量运行时才确定时，为什么需要 `Send`；
- 多个 worker 同时写状态时，为什么需要 reducer；
- `Command` 与条件边应该如何选择；
- 循环、并发和副作用有哪些常见风险。

## 一、先建立心智模型

```text
State：节点之间共享和传递的数据
Node：读取 State，执行业务逻辑，返回状态更新
Edge：决定一个节点完成后去哪里
Reducer：多个并行更新写入同一字段时的合并规则
```

最简化的图如下：

```text
START → Node A → Node B → END
           │
           └── 返回对 State 的更新
```

需要特别区分两件事：

- 节点负责“做什么”；
- 路由函数和边负责“下一步去哪”。

路由函数通常不是一个业务节点。它读取当前状态并返回目标节点名，本身不承担主要业务处理。

## 二、最小 `StateGraph`

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    message: str


def append_a(state: State) -> dict:
    """读取旧状态，返回需要更新的字段。"""
    return {"message": state["message"] + " -> A"}


def append_b(state: State) -> dict:
    return {"message": state["message"] + " -> B"}


builder = StateGraph(State)
builder.add_node("append_a", append_a)
builder.add_node("append_b", append_b)

# 静态边：A 完成后一定执行 B。
builder.add_edge(START, "append_a")
builder.add_edge("append_a", "append_b")
builder.add_edge("append_b", END)

graph = builder.compile()
result = graph.invoke({"message": "start"})

print(result["message"])
# start -> A -> B
```

节点不需要返回完整 State，只需返回本次修改的字段。框架根据 State 定义和 reducer 规则生成下一份状态。

## 三、静态边与条件边

### 静态边：目标固定

```python
builder.add_edge("validate", "save")
```

意思是 `validate` 完成后一定进入 `save`。

### 条件边：运行时决定目标

```python
from typing import Literal


class ScoreState(TypedDict):
    score: int
    result: str


def route_score(state: ScoreState) -> Literal["passed", "failed"]:
    """路由函数只返回下一节点的逻辑名称。"""
    return "passed" if state["score"] >= 60 else "failed"


builder.add_conditional_edges(
    "check",
    route_score,
)
```

如果希望业务结果和节点名称解耦，可以使用 `path_map`：

```python
def route_score(state: ScoreState) -> Literal["PASS", "FAIL"]:
    return "PASS" if state["score"] >= 60 else "FAIL"


builder.add_conditional_edges(
    "check",
    route_score,
    {
        "PASS": "passed_node",
        "FAIL": "failed_node",
    },
)
```

路由函数也可以返回 `END`，表示流程直接结束。这正是 ReAct Agent 中“继续调用工具还是结束回答”的常见表达方式：

```text
agent ──需要工具──→ tools ──→ agent
   │
   └──无需工具────→ END
```

## 四、并行与 super-step

一条路径可以分叉到多个节点：

```text
        ┌→ search ─┐
router ─┤          ├→ summarize
        └→ rag ────┘
```

同一阶段中可运行的节点构成一个 super-step。只有当前阶段的节点完成并提交状态更新后，依赖它们的下一阶段才能继续。

条件路由返回多个节点名时，表达的是“选择多个已注册节点”：

```python
def choose_sources(state):
    targets = []

    if state["need_search"]:
        targets.append("search")

    if state["need_rag"]:
        targets.append("rag")

    return targets


builder.add_conditional_edges("router", choose_sources)
```

这适用于候选节点集合在建图时已经确定，只是每次选择不同组合的场景。

## 五、`Send`：运行时动态创建 N 份任务

假设状态中有任意数量的主题：

```python
{"topics": ["Java", "MySQL", "Redis"]}
```

如果希望同一个 `worker` 分别处理每个主题，任务数量直到运行时才知道。此时使用 `Send`：

```python
from langgraph.types import Send


def assign_workers(state):
    """为每个 topic 构造一份 worker 的局部输入。"""
    return [
        Send("worker", {"topic": topic})
        for topic in state["topics"]
    ]
```

运行时形成：

```text
planner
   │
   ├→ worker({topic: "Java"})
   ├→ worker({topic: "MySQL"})
   └→ worker({topic: "Redis"})
              │
              ↓
           summary
```

两种并行的区别：

| 方式 | 含义 | 适用场景 |
| --- | --- | --- |
| 返回 `['search', 'rag']` | 执行多个不同的已注册节点 | 分支种类固定 |
| 返回多个 `Send('worker', input)` | 同一个节点执行多次，每次输入不同 | 动态 map / worker 任务 |

## 六、为什么动态 worker 需要 reducer

三个 worker 都返回：

```python
{"results": [result]}
```

如果 `results` 没有合并规则，框架无法知道并行写入应该覆盖、相加还是报错。可以通过 `Annotated` 声明 reducer：

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict


class OverallState(TypedDict):
    topics: list[str]

    # 多个 worker 返回的列表按加法合并，而不是相互覆盖。
    results: Annotated[list[str], operator.add]

    summary: str


class WorkerState(TypedDict):
    # worker 只需要看到自己的局部输入。
    topic: str


def worker(state: WorkerState) -> dict:
    return {"results": [f"已处理：{state['topic']}"]}


def summarize(state: OverallState) -> dict:
    return {"summary": "；".join(state["results"])}
```

关键理解：

```text
Send 解决“如何拆出 N 份动态任务”
Reducer 解决“N 份结果如何安全合并”
```

reducer 还应尽量满足可预测的合并性质。并行任务完成顺序可能不同，如果业务要求固定顺序，应在结果中携带索引，再在汇总节点显式排序。

## 七、完整的 Orchestrator–Worker 示例

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send


class OverallState(TypedDict):
    topics: list[str]
    results: Annotated[list[tuple[int, str]], operator.add]
    summary: str


class WorkerState(TypedDict):
    index: int
    topic: str


def plan(state: OverallState) -> dict:
    # 实际项目中，topics 可以由模型或业务逻辑生成。
    return {"topics": ["Java", "MySQL", "Redis"]}


def dispatch(state: OverallState):
    # 一个 worker 节点被动态调用多次，每次拥有独立输入。
    return [
        Send("worker", {"index": index, "topic": topic})
        for index, topic in enumerate(state["topics"])
    ]


def worker(state: WorkerState) -> dict:
    # 携带 index，避免依赖并行任务的完成顺序。
    result = f"{state['topic']} 的处理结果"
    return {"results": [(state["index"], result)]}


def synthesize(state: OverallState) -> dict:
    ordered = [text for _, text in sorted(state["results"])]
    return {"summary": "\n".join(ordered)}


builder = StateGraph(OverallState)
builder.add_node("plan", plan)
builder.add_node("worker", worker)
builder.add_node("synthesize", synthesize)

builder.add_edge(START, "plan")

# plan 后根据 topics 动态创建 worker 任务。
builder.add_conditional_edges("plan", dispatch, ["worker"])

# 同一 super-step 的 worker 完成后，进入汇总节点。
builder.add_edge("worker", "synthesize")
builder.add_edge("synthesize", END)

graph = builder.compile()
result = graph.invoke({"topics": [], "results": [], "summary": ""})
print(result["summary"])
```

不同 LangGraph 版本的类型标注和可选参数可能变化，实际运行前应以项目所安装版本的 API 文档为准。

## 八、`Command`：节点更新状态的同时决定下一步

条件边通常把业务逻辑和路由逻辑拆开：

```text
node 返回状态更新 → router 读取状态 → 决定下一节点
```

`Command` 可以让节点同时返回状态更新和跳转目标：

```python
from typing import Literal
from langgraph.types import Command


def validate(state) -> Command[Literal["save", "repair"]]:
    if state["valid"]:
        return Command(
            update={"status": "valid"},
            goto="save",
        )

    return Command(
        update={"status": "invalid"},
        goto="repair",
    )
```

选择建议：

- 路由是图结构的一部分、希望集中查看时，用 `add_conditional_edges()`；
- 状态更新和下一步业务上不可分割时，可以使用 `Command`；
- 一个节点尽量只采用一种清晰的路由机制；
- 不要同时保留指向其他节点的静态边，又用 `Command(goto=...)`，否则可能产生额外分支。

`Command` 的 `goto` 也可以携带 `Send`，把“更新状态”和“动态派发任务”组合起来。不过如果普通条件边已经表达得足够清楚，不必为了使用高级 API 而增加复杂度。

## 九、循环与终止条件

Agent 常见的 Evaluator–Optimizer 流程天然是循环：

```text
generate → evaluate ──合格──→ END
    ↑          │
    └──不合格──┘
```

循环必须同时具备业务终止条件和运行时上限：

```python
def route_after_evaluate(state):
    if state["score"] >= 80:
        return END

    if state["attempts"] >= 3:
        return END

    return "generate"
```

调用图时还可以配置递归/步数上限，作为防止意外死循环的最后保护。具体配置字段以当前 LangGraph 版本为准。

## 十、生产环境容易踩的坑

### 1. 并行写同一个字段却没有 reducer

表现为更新冲突、结果覆盖或执行报错。先明确该字段是覆盖、追加、求和还是自定义合并。

### 2. 认为并行结果天然有顺序

worker 完成顺序不稳定。需要稳定顺序时携带 index，在汇总节点排序。

### 3. 把路由函数写成重业务节点

路由函数应快速、确定地选择路径。昂贵的模型调用和有副作用操作应放到正式节点里，便于重试和观测。

### 4. 静态边与动态路由混用

同一节点既有静态出边，又通过 `Command` 或条件边跳转，可能让多个后继同时执行。建图时应检查所有出边。

### 5. 动态 worker 数量无限制

输入包含 1000 个任务时，`Send` 可能快速放大模型、数据库或第三方 API 压力。应设置并发上限、批次、超时和速率限制。

### 6. 并行 worker 执行不可重入副作用

发送消息、扣费和写数据库等操作需要幂等键。图恢复或节点重试不能导致副作用重复发生。

### 7. 把 State 当作任意全局变量

State 应保持字段清晰、可序列化并尽量精简。大型文档和工具输出更适合保存在外部存储，在 State 中留下引用。

## 十一、如何选择控制流

| 需求 | 选择 |
| --- | --- |
| A 完成后永远执行 B | `add_edge()` |
| 根据状态在固定节点中选择一个 | `add_conditional_edges()` |
| 根据状态选择多个固定节点 | 条件路由返回多个节点名 |
| 运行时创建 N 个同类任务 | 多个 `Send()` |
| 合并并行结果 | State 字段 reducer |
| 节点内同时更新状态并跳转 | `Command(update=..., goto=...)` |
| 反复生成和评价 | 条件边或 `Command` 形成循环 |
| 动态任务最后统一汇总 | `Send` + reducer + synthesizer |

## 十二、学习检查题

1. 路由函数和普通节点有什么职责差异？
2. 返回 `['search', 'rag']` 与返回两个 `Send('worker', ...)` 有什么区别？
3. 为什么并行 worker 更新 `results` 时需要 reducer？
4. 为什么不能依赖 worker 的完成顺序？
5. 同一个节点同时配置静态边和 `Command(goto=...)` 可能发生什么？
6. 如果任务数量达到几百个，需要增加哪些运行时保护？

可以尝试把完整示例改成：输入若干 URL，动态派发 worker 提取标题，最后按照输入顺序汇总。先用纯 Python 字符串模拟结果，不需要真的访问网络。

## 总结

LangGraph 控制流可以浓缩为以下关系：

```text
Node 做事
Edge 选路
State 传值
Reducer 合并并行更新
Send 动态派发同类任务
Command 在节点内部同时更新状态和选路
```

先掌握 `State + reducer + add_conditional_edges()`，再学习 `Send` 和 `Command`，会比直接堆叠高级 API 更容易建立稳定的图执行心智模型。

## 参考资料

- [原始分享对话：Send 结合条件边使用](https://chatgpt.com/share/6a97d7a0-6b74-83e9-8585-8fe5f404637a)
- [LangGraph：Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [LangGraph Reference：Send](https://reference.langchain.com/python/langgraph/types/Send)
- [LangGraph Reference：Command](https://reference.langchain.com/python/langgraph/types/Command)
- [LangGraph Reference：add_conditional_edges](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_conditional_edges)

> API 的可选参数和版本号变化较快。本文保留分享对话中的稳定机制，没有把对话里出现的具体版本号、超时参数或最新版本能力当成跨版本保证。
