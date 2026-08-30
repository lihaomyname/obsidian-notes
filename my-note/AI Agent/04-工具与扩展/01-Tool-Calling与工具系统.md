# Tool Calling 与工具系统

> Tool Calling 让模型输出结构化的工具请求，由宿主程序验证并执行，再把结果作为新的观察交回模型。

## 完整链路

```text
用户请求
  ↓
模型判断是否需要工具
  ↓
输出工具名与结构化参数
  ↓
宿主校验名称、类型、权限和预算
  ↓
执行工具并返回结构化结果
  ↓
模型依据结果继续调用或生成答案
```

Function Calling 本身不会执行函数；模型只产生调用意图，真实执行发生在 Agent Runtime 或应用代码中。

## 工具定义的三个部分

### 元数据

至少应说明工具名称、用途和类别。生产实现还可能记录认证要求、超时、限流、成本、沙箱和危险等级。

### 参数 Schema

JSON Schema 或等价的强类型定义应明确：

- 参数名称与类型
- 是否必填和默认值
- 枚举、范围、格式约束
- 清晰的用途说明和示例

链接资料反复强调 description 的重要性：模型依靠描述判断何时调用工具以及如何填写参数。

### 结构化结果

工具不应只返回不可解析文本，至少要区分成功、输出和错误。耗时、重试建议、成本等可放在 metadata 中。

## 失败与降级

- 对超时、网络失败、非法参数返回结构化错误。
- 不默认重复执行有副作用的工具。
- 允许 Agent 根据错误选择重试、换工具、澄清或终止。
- 参数容错必须在明确规则内完成，不能静默改变业务语义。

## 多工具管理

工具注册表负责发现和分发。给模型暴露的工具应按任务、角色、权限和成本筛选；功能重叠的工具越多，选择歧义越大。

工具网关或中间件可集中处理认证、权限、限流、日志和结果截断，但参考资料中的“统一工具网关”属于架构方案，不是 Function Calling 协议本身的必选组件。

## 一份可执行的工具契约

下面是教学版数据结构。重点不是类名，而是让“成功、失败、是否可重试”成为机器可判断的字段。

```python
from dataclasses import dataclass, field
from typing import Any, Callable

@dataclass
class ToolSpec:
    name: str
    description: str
    parameters: dict[str, Any]       # JSON Schema
    handler: Callable[..., Any]      # 真实执行函数
    timeout_seconds: float = 10
    risk: str = "read"               # read / write / destructive

@dataclass
class ToolResult:
    ok: bool
    content: Any = None
    error_code: str | None = None
    error_message: str | None = None
    retryable: bool = False
    metadata: dict[str, Any] = field(default_factory=dict)
```

一个搜索工具的参数 Schema 可以写成：

```python
SEARCH_SCHEMA = {
    "type": "object",
    "properties": {
        "query": {
            "type": "string",
            "minLength": 1,
            "description": "要搜索的关键词，不要放完整自然语言指令",
        },
        "limit": {
            "type": "integer",
            "minimum": 1,
            "maximum": 20,
            "default": 5,
        },
    },
    "required": ["query"],
    "additionalProperties": False,
}
```

## 注册表与执行器

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, ToolSpec] = {}

    def register(self, spec: ToolSpec) -> None:
        if spec.name in self._tools:
            raise ValueError(f"工具重复注册: {spec.name}")
        self._tools[spec.name] = spec

    def visible_tools(self, allowed_risks: set[str]) -> list[ToolSpec]:
        # 在送给模型前先过滤，避免模型看到无权使用的工具
        return [t for t in self._tools.values() if t.risk in allowed_risks]

    def execute(self, name: str, arguments: dict) -> ToolResult:
        spec = self._tools.get(name)
        if spec is None:
            return ToolResult(False, error_code="NOT_FOUND",
                              error_message="工具不存在")

        try:
            validate_json_schema(arguments, spec.parameters)  # 教学占位函数
            output = run_with_timeout(
                spec.handler, arguments, spec.timeout_seconds
            )                                                  # 教学占位函数
            return ToolResult(True, content=output)
        except SchemaError as exc:
            # 参数错误通常应由模型修正，而不是原样重试
            return ToolResult(False, error_code="BAD_ARGUMENTS",
                              error_message=str(exc), retryable=False)
        except TimeoutError:
            return ToolResult(False, error_code="TIMEOUT",
                              error_message="执行超时", retryable=True)
```

真实实现还需要把权限校验、幂等键、审计和输出截断放进执行链；上例中的校验器与超时器是教学占位，不是 Python 内置函数。

## Runtime 的决策循环

```python
for step in range(MAX_STEPS):
    response = model.chat(messages, tools=registry.visible_tools({"read"}))

    if not response.tool_calls:
        return response.text               # 模型决定直接回答

    for call in response.tool_calls:
        result = registry.execute(call.name, call.arguments)
        messages.append({
            "role": "tool",
            "tool_call_id": call.id,       # 将结果绑定到原调用
            "content": serialize(result),  # 把结构化结果交还模型
        })

raise RuntimeError("超过最大工具调用步数")
```

这里必须限制步数；写操作还应增加幂等键，防止超时后“不确定是否成功”造成重复发送或重复扣款。

## 错误应如何驱动下一步

| 错误 | Runtime / Agent 的合理动作 |
| --- | --- |
| `BAD_ARGUMENTS` | 根据 Schema 修正参数，有限次数重试 |
| `NOT_FOUND` | 重新发现工具或换用其他工具 |
| `TIMEOUT` | 只在操作具备幂等性时自动重试 |
| `PERMISSION_DENIED` | 请求授权或终止，不能换写法绕过 |
| `RATE_LIMITED` | 遵守等待提示，或降级为缓存结果 |
| `OUTPUT_TOO_LARGE` | 保存到文件，只把摘要和路径放回上下文 |

## 安全边界

- 禁止直接 `eval` 用户或模型生成的表达式。
- Shell、文件和代码执行需要沙箱与路径限制。
- 删除、发送、部署等操作应进入人工审批。
- 工具返回仍是不可信输入，不能直接提升为系统指令。

## 关联笔记

- [[02-MCP协议]]
- [[03-Skills可复用技能体系]]
- [[04-沙箱与执行安全]]
- [[../03-智能体范式/02-ReAct循环|ReAct 循环]]

## 来源与核对范围

- [AI Agent Book：工具调用基础](https://waylandz.com/ai-agent-book/%E7%AC%AC03%E7%AB%A0-%E5%B7%A5%E5%85%B7%E8%B0%83%E7%94%A8%E5%9F%BA%E7%A1%80/)
- [Hello-Agents：构建 Agent 框架中的工具系统](https://datawhalechina.github.io/hello-agents/#/./chapter7/%E7%AC%AC%E4%B8%83%E7%AB%A0%20%E6%9E%84%E5%BB%BA%E4%BD%A0%E7%9A%84Agent%E6%A1%86%E6%9E%B6?id=_75-%e5%b7%a5%e5%85%b7%e7%b3%bb%e7%bb%9f)

> 未沿用链接文章中的具体工具数量或准确率主张，因为这些数字与模型、任务和评测设置有关。
