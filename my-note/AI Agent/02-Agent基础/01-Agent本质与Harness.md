# Agent 本质与 Harness

> Agent 的关键不是更会聊天，而是围绕目标自主选择下一步，并在工具、状态、权限和审批边界内持续推进任务。

## Chatbot 与 Agent

课程资料把差异归纳为主动性、决策流和副作用控制。链接资料进一步说明：

| Chatbot | Agent |
| --- | --- |
| 一问一答，主要生成文本 | 接受目标后可连续推进多步任务 |
| 通常不改变外部状态 | 可通过工具读取或改变外部系统 |
| 用户显式驱动每一步 | 模型根据观察自主选择下一步 |
| 回答是主要产物 | 执行轨迹、状态变化和最终结果都是产物 |

Agent 涉及删除、发送、付款、发布等副作用时，必须有权限、确认点和审计记录。自主性越高，并不意味着越少约束。

## Agent 的组成

Wayland 的章节使用四部分帮助建立直觉：

- LLM：理解目标并产生下一步决策。
- Tools：搜索、文件、API、代码执行等行动接口。
- Memory：保存当前任务和跨会话所需信息。
- Autonomy：在边界内自主决定何时做什么。

生产系统还需要 Guardrails，包括预算、权限、审批、沙箱和审计。护栏决定 Agent 的能力能否安全上线。

## Runtime、Framework、Harness

课程资料提出 Runtime → Framework → Harness 三层视角。Deep Agents 链接的重点是：Framework 提供模型、工具和图等开发抽象；Harness 则把完成长任务所需的工程能力组装起来，例如：

- 上下文管理和压缩
- 状态与检查点
- 任务规划和子 Agent
- 文件系统与工具执行
- 中断、审批和恢复
- 权限、安全与可观测性

因此，“能调用一次工具”还不等于具备生产级 Agent Harness。

## AI Native 与固定流程

课程资料把 AI Native Agent 与 Dify/Coze 等流程型系统进行区分。基于文档可以安全得出的结论是：两者主要差别在决策权放在哪里。

- 固定流程：开发者预先定义大部分节点和路径，LLM 处理节点内数据。
- AI Native Agent：模型根据当前状态选择工具和下一步，实际路径可能动态变化。

实际系统可以混合两种方式：关键业务流程固定，局部探索任务交给 Agent 自主决策。

## 适用边界

链接资料认为，Agent 更适合目标明确、步骤可拆解、结果可验证、信息可获取的任务。高风险和不可逆操作不应直接全自动执行，应设置人工确认。

## 关联笔记

- [[../01-课程路线/01-阶段一至三学习地图|阶段一至三学习地图]]
- [[../03-智能体范式/01-智能体经典范式|智能体经典范式]]
- [[../04-工具与扩展/01-Tool-Calling与工具系统|Tool Calling 与工具系统]]
- [[../05-上下文记忆/01-上下文工程|上下文工程]]

## 来源与核对范围

- [Hello-Agents：第一章 初识智能体](https://datawhalechina.github.io/hello-agents/#/./chapter1/%E7%AC%AC%E4%B8%80%E7%AB%A0%20%E5%88%9D%E8%AF%86%E6%99%BA%E8%83%BD%E4%BD%93)
- [AI Agent Book：Agent 的本质](https://waylandz.com/ai-agent-book/%E7%AC%AC01%E7%AB%A0-Agent%E7%9A%84%E6%9C%AC%E8%B4%A8/)
- [Deep Agents：从 Agent Framework 到 Agent Harness](https://datawhalechina.github.io/deepagents-in-action/chapters/ch01-agent-harness/)

> “Runtime → Framework → Harness”是课程采用的学习视角，不应被误解为所有 Agent 系统必须采用的唯一标准分层。
