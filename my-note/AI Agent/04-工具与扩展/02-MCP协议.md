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

```text
建立 Transport
    ↓
Client 发送 initialize（协议版本、Client 能力）
    ↓
Server 返回版本、Server 能力和基本信息
    ↓
Client 通知 initialized
    ↓
tools/list、resources/list、prompts/list 等能力发现
    ↓
tools/call 或读取 Resource
    ↓
关闭会话与 Transport
```

初始化阶段的价值在于“先确认共同能力，再使用可选功能”。协议版本、能力字段和 SDK 调用方式会演进，工程代码应以当前规范为准。

## Tool 从 Server 到模型的适配

MCP Client 发现的 Tool 还不能直接等于应用最终暴露给模型的 Tool。Host 通常要经过一层适配：

```python
def import_mcp_tools(mcp_client, policy):
    imported = []

    for remote_tool in mcp_client.list_tools():
        if not policy.allow_server_tool(remote_tool.name):
            continue  # Host 仍保留最终授权权力

        imported.append({
            "name": f"docs__{remote_tool.name}",  # 加命名空间，避免重名
            "description": remote_tool.description,
            "parameters": remote_tool.input_schema,
            "execute": lambda args, name=remote_tool.name:
                mcp_client.call_tool(name, args),
        })

    return imported
```

这是教学伪代码，表达的是适配责任：命名空间、Schema 转换、权限过滤、超时和结果规范化都属于 Host/Client 层，而不是交给模型自行处理。

## Tools、Resources、Prompts 如何选择

| 需求 | 更合适的原语 | 原因 |
| --- | --- | --- |
| 查询订单、创建工单 | Tool | 有明确参数和一次操作结果 |
| 读取项目说明、日志片段 | Resource | 适合按 URI 定位和读取上下文 |
| 提供代码审查入口 | Prompt | 复用一组参数化提示与工作流开场 |

边界不是“Tool 一定写、Resource 一定读”。判断关键是由谁控制：Tool 通常供模型主动选择，Resource 通常由应用或用户选择后加入上下文，Prompt 通常是用户选择的模板入口。

## Transport 的工程差异

| stdio | Streamable HTTP |
| --- | --- |
| 常用于本机子进程 | 常用于远程服务 |
| 标准输入输出承载协议消息 | HTTP 承载请求与流式响应 |
| 进程生命周期通常由 Host 管理 | 需要额外认证、会话和网络防护 |
| 不应把日志写进协议输出流 | 需要处理代理、超时和断线恢复 |

远程 MCP Server 应按外部服务看待：校验来源、限定可访问域、隔离凭据、限制返回体，并对有副作用的调用再次做业务授权。

## 一个端到端例子

```text
用户：“查项目说明里的发布窗口”
  → Host 连接文档 MCP Server
  → Client 初始化并发现 Resources / Tools
  → 应用读取项目说明 Resource
  → 相关片段进入模型上下文
  → 模型基于片段回答并附资源位置
```

若用户接着要求“创建发布任务”，模型可能再调用 Tool；读资料与执行动作因此可以使用不同原语和权限策略。

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

- [AI Agent Book：MCP 协议详解](https://waylandz.com/ai-agent-book/%E7%AC%AC04%E7%AB%A0-MCP%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3/)

> MCP 规范持续演进；本文只保留链接资料中较稳定的架构概念，没有抄录下载量、Server 数量等时效数据。
