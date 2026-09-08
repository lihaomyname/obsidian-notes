# Python 装饰器与 `property`：Java 开发者入门

> `@property`、`@classmethod` 等写法是 Python 装饰器，不是 Java 意义上的注解；装饰器会在定义阶段转换函数或类。

## 先纠正一个术语

Java 开发者看到下面的 `@`，很容易把它叫作“注解”：

```python
@property
def name(self) -> str:
    return self._name
```

但 Python 中应该区分两个概念：

| 写法          | Python 名称            | 主要作用      |
| ----------- | -------------------- | --------- |
| `@property` | 装饰器 Decorator        | 转换或增强函数、类 |
| `name: str` | 类型注解 Type Annotation | 描述变量类型    |
| `-> int`    | 返回值类型注解              | 描述函数返回值类型 |

Java Annotation 通常提供元数据，由编译器、反射或框架解释；Python 装饰器则会直接接收并替换被装饰的对象。

## 一、装饰器的本质

下面两种写法基本等价。

使用 `@`：

```python
@decorator
def hello():
    print("hello")
```

展开后：

```python
def hello():
    print("hello")


hello = decorator(hello)
```

可以把装饰器理解成：

```text
原函数 → 交给装饰器 → 得到新对象 → 重新绑定到原函数名
```

这与 Spring AOP 有一些用途上的相似之处，例如记录日志、重试和权限检查，但实现机制不同。Python 装饰器只是语言层面的可调用对象转换，不等于代理机制或 Spring AOP。

## 二、最简单的函数装饰器

```python
from functools import wraps


def log_call(func):
    """接收一个函数，返回包装后的函数。"""

    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"开始调用：{func.__name__}")
        result = func(*args, **kwargs)
        print(f"调用结束：{func.__name__}")
        return result

    return wrapper


@log_call
def add(left: int, right: int) -> int:
    return left + right


print(add(2, 3))
```

输出：

```text
开始调用：add
调用结束：add
5
```

### `*args` 和 `**kwargs`

- `*args` 收集位置参数，例如 `add(2, 3)` 中的 `2`、`3`；
- `**kwargs` 收集关键字参数，例如 `add(left=2, right=3)`；
- 包装函数使用它们，可以适配不同参数结构的函数。

### 为什么使用 `@wraps(func)`

如果不使用 `wraps`，包装后函数的名称和文档可能变成 `wrapper` 的信息。`@wraps(func)` 会尽量保留原函数的名称、文档和类型标注。

自己编写函数装饰器时，通常应该使用它。

## 三、`@property`：把方法变成属性接口

Java 常见写法：

```java
String fullName = user.getFullName();
```

Python 使用 `property` 后：

```python
full_name = user.full_name
```

完整示例：

```python
class User:
    def __init__(self, first_name: str, last_name: str):
        self.first_name = first_name
        self.last_name = last_name

    @property
    def full_name(self) -> str:
        """根据两个现有字段动态计算完整姓名。"""
        return f"{self.first_name} {self.last_name}"


user = User("Li", "Hao")

# 不写括号。外部看起来像读取字段，内部实际调用了方法。
print(user.full_name)
```

输出：

```text
Li Hao
```

修改 `first_name` 后，`full_name` 会在下次访问时重新计算：

```python
user.first_name = "Wang"
print(user.full_name)  # Wang Hao
```

## 四、只读 property

如果只定义 getter，没有定义 setter，这个属性不能直接赋值：

```python
class Circle:
    def __init__(self, radius: float):
        self.radius = radius

    @property
    def area(self) -> float:
        return 3.14 * self.radius ** 2


circle = Circle(2)
print(circle.area)  # 12.56
```

下面的赋值会报错：

```python
circle.area = 100
```

因为面积是根据半径计算出来的派生属性，不应该由外部直接修改。

## 五、使用 setter 校验赋值

```python
class User:
    def __init__(self, age: int):
        # 这里也会调用下面的 setter，因此构造时同样会校验。
        self.age = age

    @property
    def age(self) -> int:
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        if value < 0:
            raise ValueError("年龄不能为负数")

        # _age 是内部真正保存数据的字段。
        self._age = value


user = User(18)
print(user.age)  # 调用 getter

user.age = 20    # 调用 setter
print(user.age)
```

执行链路：

```text
user.age = 20
      ↓
调用 @age.setter
      ↓
检查 value
      ↓
保存到 self._age
```

### 为什么使用 `_age`

Python 没有与 Java `private` 完全相同的访问控制。单下划线表达的是约定：

```text
self.age     对外公开的 property
self._age    内部实现字段，外部不应该直接使用
```

## 六、setter 中最常见的递归错误

错误写法：

```python
class User:
    @property
    def age(self):
        return self.age

    @age.setter
    def age(self, value):
        self.age = value
```

getter 中读取 `self.age` 会再次进入 getter；setter 中设置 `self.age` 会再次进入 setter，最终导致无限递归。

正确做法是使用单独的内部字段：

```python
class User:
    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        self._age = value
```

## 七、`property` 的等价写法

常用语法：

```python
class User:
    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        self._age = value
```

大致等价于：

```python
class User:
    def get_age(self):
        return self._age

    def set_age(self, value):
        self._age = value

    age = property(get_age, set_age)
```

所以 `property` 会在类上创建一个特殊属性，负责把读取和赋值分别转发给 getter、setter。

## 八、什么时候适合使用 property

适合：

- 属性需要动态计算；
- 赋值时需要校验或转换；
- 希望属性只读；
- 原本是公开字段，后来需要增加逻辑，但不想修改外部调用方式。

不适合：

- getter 会执行数据库查询或网络请求；
- 读取会造成明显副作用；
- 操作非常耗时，但名称看起来只是普通字段。

下面的接口容易误导使用者：

```python
orders = user.orders  # 看起来是简单读取，实际却可能访问网络
```

此时更适合显式方法：

```python
orders = user.fetch_orders()
```

方法调用的括号可以提醒使用者：这里正在执行一个动作。

## 九、`@staticmethod`

静态方法不接收 `self`，也不接收当前类：

```python
class MathUtils:
    @staticmethod
    def add(left: int, right: int) -> int:
        return left + right


print(MathUtils.add(2, 3))
```

它类似 Java 的静态方法，适用于逻辑上属于这个类，但不需要访问对象状态或类状态的函数。

如果这个函数放在模块顶层同样清晰，也不必为了使用 `staticmethod` 强行创建工具类。Python 中模块本身就可以承担 Java 静态工具类的一部分职责。

## 十、`@classmethod`

类方法的第一个参数通常写成 `cls`，表示当前类：

```python
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

    @classmethod
    def from_dict(cls, data: dict) -> "User":
        """根据字典创建对象，是一个替代构造方法。"""
        return cls(
            name=data["name"],
            age=data["age"],
        )


user = User.from_dict({"name": "Li Hao", "age": 18})
print(user.name)
```

为什么使用 `cls(...)`，而不是写死 `User(...)`？因为子类调用这个方法时，`cls` 会指向子类，可以正确创建子类对象。

三种方法对照：

| 方法类型 | 第一个参数 | 可以访问什么 | 常见用途 |
| --- | --- | --- | --- |
| 实例方法 | `self` | 当前对象和类 | 普通业务行为 |
| `@classmethod` | `cls` | 当前类 | 替代构造方法、类级工厂 |
| `@staticmethod` | 无特殊参数 | 不自动获得对象或类 | 与类相关的纯工具逻辑 |

## 十一、`@abstractmethod`

`@abstractmethod` 声明具体子类必须实现的方法：

```python
from abc import ABC, abstractmethod


class Tool(ABC):
    @abstractmethod
    def run(self) -> str:
        pass
```

子类没有实现 `run()` 时，不能创建对象。详细说明参见[抽象基类与 abstractmethod](abstract-base-class.md)。

它也可以和 `property` 组合，规定每个具体子类都必须提供某个属性：

```python
from abc import ABC, abstractmethod


class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        pass


class SearchTool(Tool):
    @property
    def name(self) -> str:
        return "search"
```

多个装饰器的顺序具有意义。这里先由靠近函数的 `@abstractmethod` 标记抽象方法，再由外层 `@property` 创建属性。

## 十二、`@dataclass`

`@dataclass` 是类装饰器，可以自动生成构造方法、字符串表示和比较逻辑等样板代码：

```python
from dataclasses import dataclass


@dataclass
class User:
    name: str
    age: int


user = User(name="Li Hao", age=18)
print(user)
```

输出类似：

```text
User(name='Li Hao', age=18)
```

它可以类比 Java `record` 或 Lombok 减少样板代码的用途，但具体生成内容和可变性规则并不完全相同。

## 十三、带参数的装饰器

下面的装饰器带有括号：

```python
@repeat(times=3)
def say_hello(name: str) -> None:
    print(f"Hello, {name}")
```

执行关系近似为：

```python
decorator = repeat(times=3)
say_hello = decorator(say_hello)
```

完整示例：

```python
from functools import wraps


def repeat(times: int):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = None

            for _ in range(times):
                result = func(*args, **kwargs)

            return result

        return wrapper

    return decorator


@repeat(times=3)
def say_hello(name: str) -> None:
    print(f"Hello, {name}")


say_hello("Li Hao")
```

## 十四、多个装饰器的顺序

```python
@decorator_a
@decorator_b
def hello():
    pass
```

定义阶段相当于：

```python
hello = decorator_a(decorator_b(hello))
```

靠近函数的 `decorator_b` 先包装，再由外层 `decorator_a` 包装。

调用时可以想象为：

```text
进入 decorator_a
  → 进入 decorator_b
    → 执行 hello
  ← 离开 decorator_b
← 离开 decorator_a
```

与多个 Java 注解简单并列不同，Python 装饰顺序可能直接改变运行行为。

## 十五、类型注解不会默认强制校验

```python
def add(left: int, right: int) -> int:
    return left + right
```

`left: int` 和 `-> int` 主要服务于：

- IDE 自动补全；
- mypy、Pyright 等静态检查器；
- 框架读取类型并生成 Schema；
- 开发者理解接口。

Python 默认不会只凭类型注解进行 Java 编译器式的强校验。例如下面的赋值不一定立即报错：

```python
age: int = "十八"
```

要尽早发现问题，需要配合 IDE 或静态类型检查工具。

## Java 与 Python 对照表

| Java 常见写法 | Python 常用表达 |
| --- | --- |
| `private` 字段 + getter | `_field` + `@property` |
| setter 中校验 | `@field.setter` |
| `static` 方法 | `@staticmethod` |
| 静态工厂方法 | `@classmethod` |
| 接口或抽象方法 | `ABC` + `@abstractmethod` |
| Lombok `@Data` / Java record | `@dataclass`，但不完全等价 |
| Java Annotation | 装饰器外观相似，机制不同 |
| 静态类型声明 | Python 类型注解，默认不做运行时强校验 |

## 推荐学习顺序

1. 普通实例方法和 `self`；
2. `@property`、getter 和 setter；
3. `@staticmethod`；
4. `@classmethod`；
5. `ABC` 与 `@abstractmethod`；
6. `@dataclass`；
7. 自定义函数装饰器；
8. 带参数装饰器与多个装饰器的顺序。

## 小练习

实现一个 `Product` 类：

1. 使用 `_price` 保存真实价格；
2. 使用 `@property` 提供只读接口；
3. 使用 `@price.setter` 拒绝负数；
4. 使用 `@classmethod from_dict()` 根据字典创建商品；
5. 使用 `@staticmethod is_valid_name()` 判断名称是否为空；
6. 编写一个装饰器，在修改价格时打印调用日志。

思考：如果读取价格需要访问远程接口，是否还应该使用 `product.price`？为什么？

## 总结

```text
@property          把方法包装成属性访问
@field.setter      定义属性赋值逻辑
@staticmethod      定义不依赖对象和类的方法
@classmethod       定义接收当前类的方法
@abstractmethod    强制具体子类实现方法
@dataclass         自动生成常见数据类样板代码
name: str          类型注解，不是装饰器
```

对于 Java 开发者，最重要的转换是：不要把所有 `@xxx` 都直接理解成 Java Annotation。Python 装饰器会真正参与对象创建或函数包装，顺序和返回值都可能改变程序行为。
