# MCP 协议

> MCP 用统一协议连接 Agent 应用与外部工具、资源和提示模板，重点解决发现、调用、能力协商和传输的一致性。

## MCP 解决什么

没有统一协议时，每个 Agent 都要分别集成 GitHub、数据库、消息系统等服务，并重复处理认证、格式适配和生命周期。MCP 通过 Client—Server 模型提供统一接口，使工具服务能被不同宿主复用。

MCP 不保证工具质量，也不代替 Agent 的规划、业务权限和结果验收。

## 核心角色

| 角色 | 职责 |
| --- | --- |
| Host / 应用 | 承载用户体验和 Agent 运行环境 |
| MCP Client | 与某个 Server 建立连接、协商能力并发起请求 |
| MCP Server | 暴露 Tools、Resources、Prompts 等能力 |
| Transport | 承载消息，例如本地 stdio 或远程 Streamable HTTP |

## 三类主要 Server 能力

- **Tools**：模型可选择调用的类型化操作。
- **Resources**：由 URI 标识、供应用读取和组织的上下文资源。
- **Prompts**：用户可选择的复用提示模板或工作流入口。

Tools 不等于“写”，Resources 也不等于“静态只读”；区别主要在控制方和交互方式。

## 生命周期与能力协商

链接资料强调，完整 MCP 连接不仅是一次 HTTP 请求，还包括初始化、双方能力声明、发现、调用和关闭等过程。可选能力没有在初始化阶段协商，就不应假设对方支持。

## MCP 与 Tool Calling 的关系

```text
Tool Calling：模型如何表达“我要调用某工具”
MCP：应用如何标准化地发现并连接外部工具服务
```

Agent 可以直接注册本地函数而不用 MCP，也可以把 MCP Server 提供的工具转换为模型可理解的 Tool Schema。两者是互补层次。

## 工程关注点

- 本地与远程传输的安全边界不同。
- 凭据不应暴露给模型或写入普通提示词。
- Server 返回内容仍需大小限制、超时、校验和审计。
- 远程访问应限制允许域名并防范 SSRF。
- 有副作用的 Tool 仍应单独授权或审批。

## 关联笔记

- [[01-Tool-Calling与工具系统]]
- [[03-Skills可复用技能体系]]
- [[04-沙箱与执行安全]]

## 来源与核对范围

- [POPO：Agent开发学习课表](https://docs.popo.netease.com/team/pc/EHRZHAOPIN/pageDetail/5b5253077e2a43f7ac89deedfc6db29a)
- [AI Agent Book：MCP 协议详解](https://waylandz.com/ai-agent-book/%E7%AC%AC04%E7%AB%A0-MCP%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3/)

> MCP 规范持续演进；本文只保留链接资料中较稳定的架构概念，没有抄录下载量、Server 数量等时效数据。
