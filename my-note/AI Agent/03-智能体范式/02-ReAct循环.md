---
title: ReAct 循环
tags:
  - AI-Agent
  - ReAct
  - LLM
sources:
  - https://waylandz.com/ai-agent-book/第02章-ReAct循环/
  - https://datawhalechina.github.io/hello-agents/#/./chapter4/第四章%20智能体经典范式构建
---

# ReAct 循环：核心内容与代码示例

> ReAct 的核心不是让模型一次性给出答案，而是让它每次只决定下一步：**推理（Reason）→ 行动（Act）→ 观察（Observe）**，重复循环，直到任务完成或触发停止条件。

相关笔记：[[01-智能体经典范式|智能体经典范式]] · [[03-Plan-and-Solve|Plan-and-Solve]] · [[04-Reflection|Reflection]]

## 一、ReAct 是什么

ReAct 由 **Reasoning + Acting** 组成，是 AI Agent 最基础的运行模式：

```text
用户目标
   ↓
Reason：根据目标和已有观察，判断下一步做什么
   ↓
Act：调用工具执行一个动作
   ↓
Observe：客观记录工具结果
   ↓
判断是否结束 ── 否 ──→ 回到 Reason
   │
   是
   ↓
最终答案
```

普通对话模型倾向于根据已有知识一次性生成完整答案；ReAct 则强制模型停下来，通过搜索、读文件、调用 API 或执行代码获取外部事实，再根据结果调整下一步。

ReAct 并不保证答案正确。它真正带来的变化是：把不可检查的“直接猜测”，变成带有工具结果和中间状态的“查找、验证、修正”。

### 形式化描述

在第 `t` 轮，模型策略 `π` 根据原始问题 `q` 和此前的行动—观察轨迹，生成当前决策：

$$
(th_t, a_t) = \pi(q, (a_1,o_1), \ldots, (a_{t-1},o_{t-1}))
$$

工具或环境执行动作后返回观察：

$$
o_t = T(a_t)
$$

其中 `th_t` 是本轮决策摘要，`a_t` 是动作，`o_t` 是外部环境返回的事实。新的 `(a_t, o_t)` 被加入状态，成为下一轮推理的输入。

## 二、为什么需要 ReAct

一次性生成存在两个典型问题：

| 问题 | 表现 | 后果 |
| --- | --- | --- |
| 信息可能过时 | 只依赖训练数据 | 无法回答最新价格、天气、系统状态等问题 |
| 无法中途验证 | 生成时不主动检查事实 | 容易编造细节，而且难以定位错误来源 |

ReAct 的主要价值：

1. **获取新信息**：通过工具访问模型参数之外的实时数据。
2. **动态纠错**：工具结果不符合预期时，可以换关键词、换工具或回退。
3. **过程可追踪**：每轮动作和观察都有记录，便于调试和审计。

与其他方式相比：

| 方式 | 特点 | 局限 |
| --- | --- | --- |
| 仅推理（Chain of Thought） | 能分步骤分析 | 无法主动获取新信息 |
| 仅调用工具 | 执行动作直接 | 可能不知道为何调用、接下来做什么 |
| ReAct | 推理与行动交替 | 需要额外控制循环、成本和停止条件 |

## 三、循环的三个阶段

### 1. Reason：判断下一步

输入是用户目标和已有观察，输出是下一步行动决策。

```text
输入：用户目标 + 最近的观察记录
输出：下一步行动及其理由
```

关键原则：**每轮只决定一个关键步骤**。过早规划太多步，后续计划容易建立在尚未验证的假设上。

### 2. Act：执行动作

把推理阶段选出的动作交给相应工具执行，例如：

- 向用户澄清信息
- 搜索网页
- 读取或写入文件
- 调用 HTTP API
- 查询数据库
- 执行代码或测试

在开发和调试初期，每轮只执行一个关键动作更容易定位故障；流程稳定后，可以对互不依赖的动作进行并行化。

### 3. Observe：记录结果

观察阶段只记录工具返回的客观事实，不急于解释或下结论。解释工作留给下一轮 Reason。

推荐使用结构化记录：

```python
observation = {
    "tool": "weather_api",
    "success": True,
    "data": {"city": "北京", "temperature": 18},
    "error": None,
}
```

结构化观察比自然语言更容易判断成功、失败、重复和收敛。

## 四、一个完整示例

任务：排查 API 为什么返回 HTTP 500。

```text
[第 1 轮]
Reason：500 通常是服务端异常，先查看错误日志。
Act：读取应用日志的最近 50 行。
Observe：发现 UserService.java:45 抛出 NullPointerException。

[第 2 轮]
Reason：已有明确堆栈位置，读取附近代码。
Act：读取 UserService.java 第 40～50 行。
Observe：第 45 行调用 user.getProfile().getName()。

[第 3 轮]
Reason：getProfile() 可能返回 null，需要确认调用链并给出修复。
Act：检查 Profile 的获取逻辑和相关测试。
Observe：未设置资料的用户确实会得到 null，且没有对应测试。

[第 4 轮]
Reason：原因和证据充分，可以结束。
Act：输出根因、修复方案和建议测试。
Observe：最终报告完成。
```

这里的关键不是模型列出所有“500 的常见原因”，而是用日志和源码逐步缩小范围，最终给出有证据的结论。

## 五、最小 Python 实现

下面是一个便于理解的框架示例。`reason()` 负责让模型选择下一步，`execute()` 负责调用工具。

```python
from dataclasses import dataclass, field
from typing import Any


@dataclass
class Action:
    # 要调用的工具名称，例如 web_search。
    tool: str
    # 传给工具的结构化参数。
    arguments: dict[str, Any]


@dataclass
class Step:
    # 只保存简短、可审计的决策摘要，不要求完整私有推理。
    thought: str
    action: Action | None
    # 工具执行后再回填观察结果。
    observation: Any = None
    completed: bool = False


@dataclass
class AgentState:
    goal: str
    # default_factory 确保每个 AgentState 都有独立的步骤列表。
    steps: list[Step] = field(default_factory=list)
    final_answer: str | None = None


def react_loop(goal: str, max_iterations: int = 10) -> str:
    # 一个任务对应一份独立状态，避免不同会话互相污染。
    state = AgentState(goal=goal)

    for _ in range(max_iterations):
        # Reason：模型根据目标和历史记录决定下一步
        step = reason(state)
        state.steps.append(step)

        # 模型声称完成后，统一从已收集的证据构造最终答案。
        if step.completed:
            state.final_answer = build_final_answer(state)
            return state.final_answer

        # 未完成却没有动作属于非法状态，应立即暴露问题。
        if step.action is None:
            raise RuntimeError("模型未结束任务，也没有给出动作")

        # Act
        result = execute(step.action)

        # Observe：保留客观工具结果，供下一轮使用
        step.observation = {
            "success": result.success,
            "data": result.data,
            "error": result.error,
        }

    # 达到硬上限时返回已有进度，而不是继续无限循环。
    return build_partial_answer(state, reason="达到最大循环次数")
```

工具分发可以写成：

```python
TOOLS = {
    # 白名单限制 Agent 只能调用已经注册和授权的工具。
    "web_search": web_search,
    "read_file": read_file,
    "run_code": run_code,
}


def execute(action: Action):
    # 根据模型给出的名称从注册表中查找真实函数。
    tool = TOOLS.get(action.tool)
    if tool is None:
        raise ValueError(f"未知工具：{action.tool}")

    # 将结构化参数展开为函数的关键字参数。
    return tool(**action.arguments)
```

生产代码还需要加入超时、预算、权限检查、参数校验、异常处理和日志记录。

## 六、Reason 阶段的提示词示例

可以要求模型输出结构化决策，避免推理结果和实际动作脱节：

```text
你是一个使用 ReAct 循环工作的 Agent。

用户目标：{goal}
最近的观察：{recent_observations}

请只决定下一步，不要提前执行后续步骤。

规则：
1. 如果缺少完成任务所需的关键信息，选择澄清或工具调用。
2. 不得编造工具结果。
3. 如果观察与上一轮高度重复，必须改变策略或说明无法继续。
4. 只有证据足以回答目标时，才能设置 completed=true。

输出 JSON：
{
  "thought_summary": "简短说明选择该动作的原因",
  "completed": false,
  "action": {
    "tool": "工具名称",
    "arguments": {}
  }
}
```

实际系统通常不应要求或保存模型的完整私有推理过程；保留简短、可审计的决策摘要即可。

## 七、什么时候停止

停止条件决定了 Agent 会不会过早结束、无限循环或持续消耗 Token。

| 停止条件 | 含义 | 类型 |
| --- | --- | --- |
| 用户中断 | 用户主动取消任务 | 最高优先级 |
| 任务完成 | 已有足够证据，可以回答目标 | 正常结束 |
| 预算耗尽 | 达到 Token 或费用上限 | 硬护栏 |
| 超时 | 达到端到端时间限制 | 硬护栏 |
| 结果收敛 | 连续观察高度相似，没有新进展 | 提前停止 |
| 最大迭代数 | 达到预设轮数 | 最终兜底 |

注意：即使任务未完成，触发预算、超时或用户取消时也必须停止，并向用户明确说明当前进度和未完成部分。

### Python 停止判断示例

```python
from dataclasses import dataclass
from time import monotonic


@dataclass
class LoopConfig:
    min_iterations: int = 1
    max_iterations: int = 12
    token_budget: int = 20_000
    timeout_seconds: float = 120.0
    observation_window: int = 5


def should_stop(state, config: LoopConfig) -> tuple[bool, str | None]:
    # 用户取消具有最高优先级。
    if state.user_cancelled:
        return True, "用户取消"

    # 预算和超时是不可绕过的硬护栏。
    if state.tokens_used >= config.token_budget:
        return True, "Token 预算耗尽"

    if monotonic() - state.started_at >= config.timeout_seconds:
        return True, "执行超时"

    if state.iteration >= config.max_iterations:
        return True, "达到最大迭代数"

    # 最低轮数用于防止模型未获取任何证据便直接结束。
    if state.iteration < config.min_iterations:
        return False, None

    # 只有任务完成且证据充分，才接受模型的完成声明。
    if state.task_completed and state.has_required_evidence:
        return True, "任务完成"

    # 连续观察没有新信息时提前终止，避免原地打转。
    if observations_converged(state.observations):
        return True, "观察结果收敛，无新进展"

    return False, None
```

## 八、生产环境的关键配置

### `MaxIterations`

防止模型反复使用相同关键词、读取相同结果而无限循环。原文建议生产环境可以从 **10～15 轮**起步，再根据任务复杂度和成本调整。

### `MinIterations`

防止模型第一轮直接声称完成。对于必须查询实时数据的任务，可以要求至少完成一次真实工具调用，而不只是机械地要求固定轮数。

### `ObservationWindow`

只把最近若干条观察原文放进上下文，旧观察压缩成摘要，避免上下文不断膨胀：

```python
def context_observations(observations: list, window: int = 5):
    # 最近结果保留原文，供下一轮精确判断。
    recent = observations[-window:]
    # 更早的结果压缩为摘要，降低上下文成本。
    older = observations[:-window]
    return {
        "older_summary": summarize(older) if older else None,
        "recent": recent,
    }
```

### 预算和超时

预算不是 ReAct 独有的配置，而是所有 Agent 推理模式都需要的公共护栏。通常至少控制：

- 单次任务 Token 上限
- 工具调用次数或费用上限
- 单工具超时
- 整个任务的端到端超时

### 完成前检查证据

模型说“完成”不等于任务真的完成。研究、排障或数据查询任务应检查：

- 是否实际调用了必要工具
- 是否至少获得一条有效观察
- 观察是否能支撑最终结论
- 输出是否满足用户要求的格式和验收条件

## 九、常见陷阱

### 先认识 ReAct 的固有边界

| 局限 | 原因 | 工程应对 |
| --- | --- | --- |
| 依赖模型能力 | 模型可能不遵循格式或选错工具 | 使用结构化输出、参数校验和更稳定的模型 |
| 串行成本较高 | 每轮至少包含一次模型调用和一次工具调用 | 缓存结果，并行执行互不依赖的动作 |
| 提示词脆弱 | 文案变化可能影响动作格式 | 使用 JSON Schema 或原生 Tool Calling |
| 缺少全局视角 | “走一步看一步”可能陷入局部最优 | 复杂任务与 Plan-and-Solve 组合 |

### 1. 无限循环

**表现**：重复搜索、重复读取相同内容。

**处理**：设置最大轮数；比较连续观察的相似度；重复时强制换策略或停止。

```python
def observations_converged(observations: list[str]) -> bool:
    # 少于两次观察时无法判断是否重复。
    if len(observations) < 2:
        return False

    # 阈值越高，只有越相似的结果才会被视为收敛。
    return similarity(observations[-1], observations[-2]) > 0.95
```

### 2. 过早结束

**表现**：没有调用工具就声称已经完成实时查询或排障。

**处理**：设置最低证据要求；必要时要求至少一次有效工具调用。

### 3. Token 爆炸

**表现**：每轮都携带所有历史内容，成本和延迟持续增加。

**处理**：限制观察窗口、压缩旧记录，并设置 Token 预算。

### 4. 推理与动作脱节

**表现**：Reason 选择读取日志，Act 却调用了无关工具。

**处理**：让动作直接来自同一份结构化决策；执行前校验工具名称和参数，不要分别生成互不关联的文本。

### 5. 把 ReAct 当成正确性保证

ReAct 只是提高了获取证据、纠错和追踪的能力。工具可能返回错误数据，模型也可能误读观察。生产系统仍然需要来源校验、权限控制、结果验收和人工介入机制。

### 调试顺序

1. 打印最终发送给模型的完整提示和必要上下文。
2. 保存模型未经解析的原始输出，区分模型错误与解析器错误。
3. 检查工具名、输入参数、返回结构和异常信息。
4. 必要时加入一两个成功的 Thought—Action—Observation 示例。
5. 将温度调低以提高格式稳定性，再评估是否需要更换模型。

## 十、落地检查清单

- [ ] 每轮只选择一个关键的下一步动作
- [ ] 工具输入使用结构化参数并进行校验
- [ ] Observation 保留客观结果，不混入未经验证的结论
- [ ] 设置最大迭代数、Token 预算和超时
- [ ] 检测重复动作或高度相似的观察
- [ ] 限制上下文中的 Observation 数量，压缩旧记录
- [ ] 完成前检查是否有足够证据
- [ ] 中断或失败时输出当前进度、原因和未完成项
- [ ] 记录工具调用、耗时、Token、错误和停止原因

## 十一、一句话总结

**ReAct 是一个受约束的闭环：Agent 根据当前证据只思考下一步，调用工具取得新事实，再依据观察继续或停止；真正让它可用于生产的，不只是循环本身，还有预算、超时、收敛检测和完成验收。**

## 参考

- [第 2 章：ReAct 循环](https://waylandz.com/ai-agent-book/%E7%AC%AC02%E7%AB%A0-ReAct%E5%BE%AA%E7%8E%AF/)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Hello-Agents：第四章 智能体经典范式构建](https://datawhalechina.github.io/hello-agents/#/./chapter4/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%99%BA%E8%83%BD%E4%BD%93%E7%BB%8F%E5%85%B8%E8%8C%83%E5%BC%8F%E6%9E%84%E5%BB%BA)

> 本文综合 Wayland Zhang 与 Datawhale 两份资料整理；Python 示例是面向工程实践的改写，并非原文代码的逐字复制。
