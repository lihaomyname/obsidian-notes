---
title: Plan-and-Solve
aliases:
  - 先规划后执行
tags:
  - AI-Agent
  - Planning
source: https://datawhalechina.github.io/hello-agents/#/./chapter4/第四章%20智能体经典范式构建
---

# Plan-and-Solve：先规划，后执行

> Plan-and-Solve 把复杂任务拆成两个阶段：规划器生成完整、结构化的步骤，执行器携带历史状态逐步完成每个子任务。

相关笔记：[[01-智能体经典范式|智能体经典范式]] · [[02-ReAct循环|ReAct]] · [[04-Reflection|Reflection]]

## 一、核心流程

```text
原始任务
   ↓
Plan：拆成有序、可执行的步骤
   ↓
Solve：按顺序执行步骤，并记录每步结果
   ↓
汇总最终答案
```

形式化地，规划模型根据问题 `q` 生成计划：

$$
P = \pi_{plan}(q) = (p_1, p_2, \ldots, p_n)
$$

执行第 `i` 个步骤时，同时参考原问题、完整计划和此前结果：

$$
s_i = \pi_{solve}(q, P, (s_1, \ldots, s_{i-1}))
$$

## 二、与 ReAct 的区别

| ReAct | Plan-and-Solve |
| --- | --- |
| 每轮根据最新观察决定下一步 | 开始时先生成完整计划 |
| 适合路径未知、环境动态的任务 | 适合结构稳定、可提前分解的任务 |
| 局部适应能力强 | 全局目标一致性强 |
| 可能缺少长远视角 | 初始计划错误可能向后传播 |

Plan-and-Solve 不是要求永远“严格执行错误计划”。生产系统最好允许执行失败或环境变化时触发重新规划。

## 三、适用场景

- 多步数学与逻辑问题
- 报告结构设计与分章节撰写
- 代码模块、类和函数的分步实现
- 数据处理、迁移和发布工作流

任务应当满足两个条件：可以拆成相对独立的子任务，并且步骤之间的依赖关系比较稳定。

## 四、最小 Python 示例

```python
from dataclasses import dataclass, field


@dataclass
class PlanState:
    question: str
    plan: list[str]
    results: list[str] = field(default_factory=list)


class PlanAndSolveAgent:
    def __init__(self, planner, executor):
        self.planner = planner
        self.executor = executor

    def run(self, question: str) -> str:
        plan = self.planner.plan(question)
        if not plan:
            raise RuntimeError("规划器没有生成有效计划")

        state = PlanState(question=question, plan=plan)

        for index, step in enumerate(plan):
            result = self.executor.solve_step(
                question=question,
                full_plan=plan,
                previous_results=state.results,
                current_step=step,
            )
            state.results.append(result)

        return self.executor.synthesize(state)
```

相比把历史拼成一大段字符串，结构化保存 `step`、`status`、`result` 和 `error` 更便于恢复、重试与监控。

## 五、规划提示词

```text
你是任务规划器。请把用户目标拆成按顺序执行的子任务。

要求：
1. 每一步必须具体、可执行、可验收。
2. 明确步骤之间的输入输出依赖。
3. 不要在规划阶段伪造执行结果。
4. 如果信息不足，把“向用户澄清”列为第一步。

输出 JSON：
{
  "goal": "最终目标",
  "steps": [
    {"id": 1, "task": "...", "depends_on": [], "acceptance": "..."}
  ]
}
```

## 六、执行器需要的上下文

每一步至少应包含：

- 原始任务：防止执行器忘记最终目标
- 完整计划：理解当前步骤的位置
- 已完成步骤及结果：提供必要输入
- 当前步骤：限制本轮只处理一项任务
- 验收标准：判断当前步骤是否真正完成

## 七、局限与改进

### 初始计划可能错误

增加计划评审，或执行中发现前提不成立时返回 `requires_replan=True`。

### 错误会沿依赖链传播

每一步执行后先校验结果，再交给下游步骤；关键步骤可使用 Reflection 或外部测试验证。

### 计划过细导致成本过高

合并简单且强相关的步骤，仅对风险高、依赖明确的节点单独调用模型。

### 计划过粗无法执行

每一步都应有明确动作、产物和验收条件，避免“分析问题”“完成开发”这类模糊描述。

## 八、落地检查清单

- [ ] 计划步骤有明确顺序和依赖
- [ ] 每一步具体、可执行、可验收
- [ ] 执行器始终知道原始目标和当前步骤
- [ ] 历史结果使用结构化状态保存
- [ ] 步骤失败后可以重试、跳过或重新规划
- [ ] 设置最大步骤数、预算和总超时
- [ ] 最终答案经过统一汇总，而非机械返回最后一步文本

## 参考

- [Hello-Agents：第四章智能体经典范式构建](https://datawhalechina.github.io/hello-agents/#/./chapter4/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%99%BA%E8%83%BD%E4%BD%93%E7%BB%8F%E5%85%B8%E8%8C%83%E5%BC%8F%E6%9E%84%E5%BB%BA)
- [Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning](https://arxiv.org/abs/2305.04091)
