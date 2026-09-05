# Python 抽象基类与 `@abstractmethod`

> `@abstractmethod` 用来声明“子类必须实现的方法”，让一组不同的类遵守统一接口。

## 这个知识解决什么问题

假设一个 Agent 系统支持搜索、计算和天气查询等工具。虽然每种工具的执行逻辑不同，但系统希望它们都提供相同的方法：

```python
tool.run(parameters)
tool.get_parameters()
```

如果只把这个约定写在文档里，开发者可能忘记实现其中一个方法。抽象基类可以让 Python 在创建对象时检查这个约定。

可以把它理解为一份合同：

```text
Tool 抽象基类：规定工具必须具备哪些能力
        │
        ├── SearchTool：实现搜索逻辑
        ├── CalculatorTool：实现计算逻辑
        └── WeatherTool：实现天气查询逻辑
```

## 最重要的一句话

`@abstractmethod` 标记的是抽象方法：基类只规定方法的名称和形状，具体功能由子类实现。

## `ABC` 和 `abstractmethod` 从哪里来

它们位于 Python 标准库的 `abc` 模块中：

```python
from abc import ABC, abstractmethod
```

- `ABC` 是 Abstract Base Class 的缩写，即“抽象基类”。
- `abstractmethod` 是一个装饰器，用来标记抽象方法。
- 装饰器写在函数上方，以 `@` 开头，可以为函数增加额外规则。

## 逐行理解原始代码

```python
from abc import ABC, abstractmethod
from typing import Any, Dict, List


class Tool(ABC):
    """所有工具共同遵守的抽象基类。"""

    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    @abstractmethod
    def run(self, parameters: Dict[str, Any]) -> str:
        """执行工具。具体执行方式由子类实现。"""
        pass

    @abstractmethod
    def get_parameters(self) -> List["ToolParameter"]:
        """返回工具的参数定义。"""
        pass
```

### `class Tool(ABC)`

`Tool` 继承 `ABC`，表示它是一个抽象基类。抽象基类主要用于被其他类继承，通常不直接创建对象。

### `@abstractmethod`

它告诉 Python：

```text
任何具体的 Tool 子类，都必须实现下面这个方法。
```

这里规定子类必须实现：

- `run()`；
- `get_parameters()`。

### `pass`

`pass` 表示“这里暂时不执行任何操作”，主要用于保持代码语法完整。

真正阻止对象创建的不是 `pass`，而是 `@abstractmethod`。

## 为什么不能直接创建 `Tool`

```python
tool = Tool("search", "搜索资料")
```

运行后会得到类似错误：

```text
TypeError: Can't instantiate abstract class Tool
with abstract methods get_parameters, run
```

`instantiate` 的意思是“实例化”，也就是根据一个类创建对象。

Python 拒绝创建对象，是因为 `Tool` 中还有没有具体实现的抽象方法。

## 最小可运行示例

下面的代码可以保存为 `abstract_tool.py` 并直接运行：

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any, Dict, List


@dataclass
class ToolParameter:
    """描述一个工具参数。"""

    name: str
    description: str


class Tool(ABC):
    """工具抽象基类：定义所有工具必须遵守的接口。"""

    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    @abstractmethod
    def run(self, parameters: Dict[str, Any]) -> str:
        """执行工具并返回字符串结果。"""
        pass

    @abstractmethod
    def get_parameters(self) -> List[ToolParameter]:
        """返回工具支持的参数列表。"""
        pass


class SearchTool(Tool):
    """一个具体的搜索工具。"""

    def run(self, parameters: Dict[str, Any]) -> str:
        # 从字典中取得调用者传入的 keyword。
        keyword = parameters["keyword"]
        return f"正在搜索：{keyword}"

    def get_parameters(self) -> List[ToolParameter]:
        # 返回一个参数定义，而不是真正执行搜索。
        return [
            ToolParameter(
                name="keyword",
                description="需要搜索的关键词",
            )
        ]


# SearchTool 已实现全部抽象方法，因此可以实例化。
search_tool = SearchTool(
    name="search",
    description="根据关键词搜索资料",
)

print(search_tool.name)
print(search_tool.get_parameters())
print(search_tool.run({"keyword": "Python abstractmethod"}))
```

预期输出类似：

```text
search
[ToolParameter(name='keyword', description='需要搜索的关键词')]
正在搜索：Python abstractmethod
```

## 如果子类少实现一个方法

```python
class BrokenTool(Tool):
    def run(self, parameters: Dict[str, Any]) -> str:
        return "执行完成"

    # 忘记实现 get_parameters()


broken_tool = BrokenTool("broken", "不完整的工具")
```

定义 `BrokenTool` 类本身通常不会报错，但创建 `BrokenTool` 对象时会报错：

```text
TypeError: Can't instantiate abstract class BrokenTool
with abstract method get_parameters
```

这让问题在对象创建阶段就暴露出来，而不是等系统调用 `get_parameters()` 时才发现方法不存在。

## 父类中写了代码，还能标记为抽象方法吗

可以。抽象方法不一定只能写 `pass`，也可以包含供子类复用的默认逻辑：

```python
class Tool(ABC):
    @abstractmethod
    def run(self, parameters: Dict[str, Any]) -> str:
        print("开始执行工具")
```

子类仍然必须覆盖 `run()`，但可以通过 `super()` 调用父类逻辑：

```python
class SearchTool(Tool):
    def run(self, parameters: Dict[str, Any]) -> str:
        # 调用父类抽象方法中已有的公共逻辑。
        super().run(parameters)
        return f"搜索：{parameters['keyword']}"
```

因此，`abstractmethod` 的核心含义是“子类必须覆盖”，而不是“父类绝对不能有代码”。

## `pass`、`NotImplementedError` 和 `@abstractmethod` 的区别

### `pass`

```python
def run(self):
    pass
```

方法被调用时什么也不做，并返回 `None`。它不会强迫子类实现该方法。

### `raise NotImplementedError`

```python
def run(self):
    raise NotImplementedError("子类需要实现 run()")
```

对象仍然可以创建，只有调用 `run()` 时才报错。

### `@abstractmethod`

```python
@abstractmethod
def run(self):
    pass
```

只要具体子类没有实现 `run()`，Python 就不允许创建它的对象，错误出现得更早。

| 写法 | 能否创建对象 | 什么时候发现问题 |
| --- | --- | --- |
| 只有 `pass` | 可以 | 可能一直发现不了，只得到 `None` |
| `NotImplementedError` | 可以 | 调用方法时 |
| `@abstractmethod` | 未实现时不可以 | 创建对象时 |

## 抽象基类带来的好处

### 统一调用方式

系统不需要知道工具的具体类型：

```python
def execute_tool(tool: Tool, parameters: Dict[str, Any]) -> str:
    return tool.run(parameters)
```

只要对象是一个完整的 `Tool` 子类，就可以通过相同方式调用。

### 尽早发现漏实现的方法

子类缺少必要方法时，对象无法创建，减少运行到业务中途才失败的情况。

### 让代码意图更清晰

阅读 `Tool` 类就能知道所有工具应具备哪些能力，不必逐个查看搜索、计算和天气工具。

## 常见错误

### 1. 忘记继承 `ABC`

推荐把抽象基类显式写成：

```python
class Tool(ABC):
    ...
```

这样类的用途最清楚。

### 2. 子类方法名字拼错

```python
class SearchTool(Tool):
    def runs(self, parameters):  # 错误：应该叫 run
        return "..."
```

这不算实现了 `run()`，所以对象仍然不能创建。

### 3. 只实现一个抽象方法

子类必须实现继承链中所有尚未实现的抽象方法。

### 4. 把抽象类理解成完全不能写代码

抽象基类可以拥有普通方法、构造方法、属性和已经实现的公共逻辑。只有带 `@abstractmethod` 的部分强制子类覆盖。

### 5. 以为类型标注会自动校验参数

```python
parameters: Dict[str, Any]
```

这是类型提示，主要帮助阅读和静态检查。Python 默认不会仅凭这个标注，在运行时自动验证字典内容。

## 和 Java 接口的简单类比

如果接触过 Java，可以暂时这样理解：

```text
Python ABC + @abstractmethod
≈ Java 的抽象类或接口所规定的方法契约
```

但两种语言的具体规则并不完全相同。初学阶段只需记住，它们都能规定实现类必须提供某些方法。

## 小练习

在前面的代码中新增一个 `CalculatorTool`：

1. `get_parameters()` 返回 `left`、`right` 两个参数定义；
2. `run()` 读取两个数字并返回它们的和；
3. 故意漏掉 `get_parameters()`，观察实例化时的错误；
4. 再补上该方法，确认工具能够正常运行。

## 总结

```text
ABC                表示这是一个抽象基类
@abstractmethod    声明子类必须实现的方法
pass               只是空语句，不是强制机制
具体子类            实现全部抽象方法后才能创建对象
```

在工具系统中使用抽象基类，是为了让搜索工具、计算工具等不同实现都拥有统一的 `run()` 和 `get_parameters()` 接口。
