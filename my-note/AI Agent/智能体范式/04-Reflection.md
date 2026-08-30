---
title: Reflection
aliases:
  - 智能体反思范式
tags:
  - AI-Agent
  - Reflection
source: https://datawhalechina.github.io/hello-agents/#/./chapter4/第四章%20智能体经典范式构建
---

# Reflection：执行、反思与优化

> Reflection 是一种事后自我校正循环：先生成初稿，再让评审角色指出具体问题，最后依据反馈生成修订稿，直到通过验收或达到迭代上限。

相关笔记：[[01-智能体经典范式|智能体经典范式]] · [[02-ReAct循环|ReAct]] · [[03-Plan-and-Solve|Plan-and-Solve]]

## 一、核心循环

```text
Execution：生成初稿
    ↓
Reflection：检查事实、逻辑、效率和遗漏
    ↓
Refinement：根据反馈生成修订稿
    ↓
通过验收？── 否 → 再次 Reflection
    │
    是
    ↓
最终结果
```

假设 `O_i` 是第 `i` 版输出，反思模型生成反馈：

$$
F_i = \pi_{reflect}(Task, O_i)
$$

优化模型结合原任务、上一版输出和反馈生成新版：

$$
O_{i+1} = \pi_{refine}(Task, O_i, F_i)
$$

## 二、反思应该检查什么

反思不能只说“还可以更完善”，必须围绕验收标准给出可执行反馈：

| 维度 | 典型问题 |
| --- | --- |
| 事实 | 是否存在无来源、过时或互相矛盾的陈述？ |
| 逻辑 | 推论是否成立，步骤之间是否连贯？ |
| 完整性 | 是否遗漏用户约束、边界情况或必要章节？ |
| 效率 | 是否存在不必要步骤、更优算法或更低成本路径？ |
| 安全性 | 是否可能越权、泄露数据或执行危险动作？ |
| 表达 | 输出格式、术语、结构是否满足要求？ |

## 三、适用场景

- 关键业务代码、算法和测试
- 技术方案、研究报告和决策材料
- 有明确评分标准或自动验证器的任务
- 初次答案大体可用，但细节质量需要提升的任务

不适合对延迟极其敏感，或“基本正确即可”的低价值任务。

## 四、最小 Python 实现

```python
from dataclasses import dataclass, field


@dataclass
class Revision:
    output: str
    feedback: str | None = None


@dataclass
class ReflectionState:
    task: str
    revisions: list[Revision] = field(default_factory=list)


class ReflectionAgent:
    def __init__(self, generator, reviewer, max_iterations: int = 3):
        self.generator = generator
        self.reviewer = reviewer
        self.max_iterations = max_iterations

    def run(self, task: str, criteria: list[str]) -> str:
        state = ReflectionState(task=task)
        draft = self.generator.create(task)
        state.revisions.append(Revision(output=draft))

        for _ in range(self.max_iterations):
            review = self.reviewer.review(task, draft, criteria)
            state.revisions[-1].feedback = review.feedback

            if review.passed:
                return draft

            draft = self.generator.refine(
                task=task,
                previous_output=draft,
                feedback=review.feedback,
            )
            state.revisions.append(Revision(output=draft))

        return draft
```

## 五、结构化评审提示词

```text
你是严格的评审员。请依据验收标准审查候选结果。

原始任务：{task}
验收标准：{criteria}
候选结果：{draft}

规则：
1. 逐项检查验收标准，并引用候选结果中的具体位置。
2. 区分致命错误、重要问题和改进建议。
3. 反馈必须具体、可执行。
4. 只有全部硬性标准满足时才能 passed=true。

输出 JSON：
{
  "passed": false,
  "issues": [
    {"severity": "critical|major|minor", "problem": "...", "fix": "..."}
  ],
  "feedback": "供优化器使用的精炼建议"
}
```

让评审器和生成器承担不同角色，通常比让同一提示词笼统地“自我改进”更稳定。高风险任务还可以使用不同模型或外部验证器，减少相同盲点。

## 六、记忆与状态

每轮至少保存：

- 版本号与候选输出
- 评审反馈和严重程度
- 使用的验收标准
- 自动测试或工具验证结果
- 是否通过以及停止原因

不要无限携带所有原文。可保留当前版本、未解决问题以及旧版本摘要；完整轨迹存到外部日志中。

## 七、如何停止

合理的停止条件包括：

- 所有硬性验收标准通过
- 自动测试、编译、事实核查等外部验证通过
- 连续两轮没有实质质量提升
- 达到最大反思次数、Token 预算或超时
- 用户中断

不要只依赖反馈里是否出现“无需改进”几个字；更可靠的方法是解析结构化的 `passed` 字段，并结合外部验证结果。

## 八、成本收益

Reflection 是典型的“以成本换质量”：

- 每轮通常至少增加一次评审和一次优化调用
- 循环是串行的，会明显增加延迟
- 多角色提示与验收标准提高开发复杂度

换来的收益是更高的完整性、鲁棒性和最终质量。可以采用分级策略：普通任务不反思，重要任务反思一轮，高风险任务结合测试与人工审核。

## 九、常见失败

### 反馈空泛

原因通常是没有给验收标准。应要求指出具体位置、影响和修改方式。

### 反复润色但不修核心问题

把问题分严重程度，优先修复 `critical` 和 `major`，并检查这些问题是否在下一版关闭。

### 评审器与生成器共享盲点

引入单元测试、静态分析、搜索证据或另一个模型作为独立信号。

### 无限优化

设置最大轮数，并检测分数或问题列表是否已经收敛。

## 十、落地检查清单

- [ ] 反思前有明确、可测量的验收标准
- [ ] 反馈指出具体问题和可执行修改方式
- [ ] 保存版本、反馈和验证结果
- [ ] 硬性问题修复后才允许通过
- [ ] 尽可能结合测试或工具验证，而非只靠模型自评
- [ ] 设置迭代次数、预算、超时和收敛条件
- [ ] 最终输出使用最新版，而不是最后一条反馈文本

## 参考

- [Hello-Agents：第四章智能体经典范式构建](https://datawhalechina.github.io/hello-agents/#/./chapter4/%E7%AC%AC%E5%9B%9B%E7%AB%A0%20%E6%99%BA%E8%83%BD%E4%BD%93%E7%BB%8F%E5%85%B8%E8%8C%83%E5%BC%8F%E6%9E%84%E5%BB%BA)
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
