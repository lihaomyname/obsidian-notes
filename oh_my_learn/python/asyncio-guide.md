# Python 异步编程：`asyncio` 从入门到工程实践

> Python 异步编程通过事件循环在任务等待 I/O 时切换到其他任务，提高大量网络、数据库和工具调用的整体吞吐量。

## 这篇笔记解决什么问题

异步编程经常出现在：

- HTTP 请求和网络连接；
- 数据库查询；
- 消息队列和 WebSocket；
- 流式大模型输出；
- Agent 并发调用多个工具；
- 同时等待多个外部服务。

作为 Java 开发者，可以暂时把 Python 的 `asyncio + async/await` 类比为事件循环结合 `CompletableFuture`，但两者的调度和运行机制并不完全相同。

最重要的结论是：

> 异步通常不会让单个任务更快，而是在一个任务等待 I/O 时，让其他任务继续执行。

## 一、同步等待的问题

```python
import time


def download(name: str) -> str:
    print(f"开始下载 {name}")
    time.sleep(2)
    print(f"下载完成 {name}")
    return name


download("A")
download("B")
```

执行过程：

```text
A 等待 2 秒
    ↓
B 等待 2 秒
    ↓
总计约 4 秒
```

程序执行 A 时，即使 A 只是在等待，B 也不能开始。异步编程允许 A 在等待时暂时让出执行权：

```text
A 开始等待 ───────┐
B 开始等待 ───────┤
                  ↓
         等待完成后分别继续
```

两个独立任务的总耗时可能接近其中最慢的任务，而不是所有等待时间之和。

## 二、第一个异步程序

```python
import asyncio


async def main() -> None:
    print("Hello")
    await asyncio.sleep(1)
    print("World")


asyncio.run(main())
```

### `async def`

```python
async def main():
    ...
```

表示定义协程函数。调用协程函数时不会直接得到最终返回值，而是先得到协程对象。

### `await`

```python
await asyncio.sleep(1)
```

表示等待当前异步操作完成。在等待期间，当前协程暂停，事件循环可以调度其他任务。

`await` 通常只能写在 `async def` 内部。

### `asyncio.run()`

```python
asyncio.run(main())
```

它负责创建事件循环、运行入口协程、等待其结束并清理事件循环。普通脚本通常只在最外层调用一次。

## 三、协程函数、协程对象和返回值

普通函数调用后直接得到结果：

```python
def get_name() -> str:
    return "Li Hao"


result = get_name()
print(result)
```

异步函数调用后先得到协程对象：

```python
async def get_name() -> str:
    return "Li Hao"


coroutine = get_name()
print(coroutine)
```

如果协程一直没有被等待，Python 可能发出警告：

```text
RuntimeWarning: coroutine 'get_name' was never awaited
```

正确写法：

```python
import asyncio


async def get_name() -> str:
    return "Li Hao"


async def main() -> None:
    result = await get_name()
    print(result)


asyncio.run(main())
```

记忆关系：

```text
调用普通函数       → 返回最终结果
调用 async 函数    → 返回协程对象
await 协程对象      → 得到最终结果
```

## 四、事件循环如何工作

事件循环可以看作不断检查任务状态的调度员：

```text
检查 Task A
  ↓
A 正在等待网络，暂停 A
  ↓
检查 Task B
  ↓
B 可以继续，执行一段
  ↓
B 进入数据库等待
  ↓
检查哪些等待已经完成
  ↓
恢复对应任务
```

Python `asyncio` 通常使用协作式调度：协程运行到 `await` 等位置时主动让出控制权，事件循环不会随意在任意一行打断它。

因此，如果一个协程长时间执行 CPU 计算且没有有效的 `await`，它仍会阻塞整个事件循环。

## 五、连续 `await` 仍然可能是串行

```python
import asyncio


async def download(name: str) -> str:
    print(f"开始下载 {name}")
    await asyncio.sleep(2)
    print(f"完成下载 {name}")
    return name


async def main() -> None:
    result_a = await download("A")
    result_b = await download("B")
    print(result_a, result_b)


asyncio.run(main())
```

这里必须等 A 完成才会执行下一行，所以 B 仍然晚于 A 开始，总耗时约 4 秒。

判断是否并发时，不要只看有没有 `async` 和 `await`，而要看多个任务是否在同一时间段内都已经被调度。

## 六、使用 `asyncio.gather()` 并发收集结果

```python
import asyncio


async def download(name: str) -> str:
    print(f"开始下载 {name}")
    await asyncio.sleep(2)
    print(f"完成下载 {name}")
    return name


async def main() -> None:
    results = await asyncio.gather(
        download("A"),
        download("B"),
        download("C"),
    )

    print(results)


asyncio.run(main())
```

总耗时约 2 秒。`gather()` 返回的结果顺序与传入协程的顺序一致，不取决于完成顺序。

它可以粗略类比 Java 的 `CompletableFuture.allOf()`，但 `gather()` 会直接组织各协程的结果。

## 七、Coroutine、Task 和 Future

### Coroutine

调用 `async def` 函数后得到的协程对象，描述一段可以暂停和恢复的计算。

### Task

Task 是已经交给事件循环调度的协程：

```python
task = asyncio.create_task(download("A"))
```

创建 Task 后，它可以与当前协程并发推进。

### Future

Future 表示一个未来才会完成的结果。Task 是 Future 的一种更高层形式，负责驱动协程并保存最终结果或异常。

初学阶段可以先记：

```text
Coroutine：异步逻辑本身
Task：已经开始被事件循环调度的 Coroutine
Future：未来结果的低层表示
```

## 八、使用 `create_task()`

```python
import asyncio


async def download(name: str) -> str:
    await asyncio.sleep(2)
    return f"{name} 下载完成"


async def main() -> None:
    task_a = asyncio.create_task(download("A"))
    task_b = asyncio.create_task(download("B"))

    print("任务已经启动")

    result_a = await task_a
    result_b = await task_b

    print(result_a)
    print(result_b)


asyncio.run(main())
```

Task 创建后要保留引用，并在合适的位置等待、取消或处理异常。不要随意创建“无人管理”的后台任务，否则程序退出时任务可能尚未完成，异常也可能无人观察。

## 九、使用 `TaskGroup` 管理一组子任务

`TaskGroup` 提供结构化并发：一组相关任务在明确的代码块中创建，并在离开代码块前完成或统一失败。

```python
import asyncio


async def request(name: str) -> str:
    await asyncio.sleep(1)
    return f"{name} 完成"


async def main() -> None:
    async with asyncio.TaskGroup() as group:
        task_a = group.create_task(request("A"))
        task_b = group.create_task(request("B"))

    # 离开 TaskGroup 时，这一组任务已经结束。
    print(task_a.result())
    print(task_b.result())


asyncio.run(main())
```

如果其中一个子任务失败，`TaskGroup` 会协调取消其他尚未完成的兄弟任务，并使用异常组报告失败。

### 三种并发方式怎么选

| 写法 | 适用场景 |
| --- | --- |
| 连续 `await` | 后一个操作依赖前一个结果 |
| `asyncio.gather()` | 简单并发一批协程并收集结果 |
| `create_task()` | 需要提前启动、稍后等待或单独取消 |
| `TaskGroup` | 一组生命周期相关的子任务，需要结构化管理 |

## 十、异步适合 I/O，不直接解决 CPU 密集计算

适合异步的任务：

- HTTP 和数据库请求；
- Socket、消息队列和 WebSocket；
- 调用大模型或远程工具；
- 等待计时器和外部服务。

下面的函数即使写成 `async def`，也会阻塞事件循环：

```python
async def calculate() -> int:
    total = 0

    for number in range(100_000_000):
        total += number

    return total
```

循环中没有真正让出控制权的异步等待。CPU 密集任务通常考虑多进程、原生扩展、独立计算服务或 `ProcessPoolExecutor`。

## 十一、不要在异步函数里阻塞事件循环

错误示例：

```python
import time


async def task() -> None:
    print("开始")
    time.sleep(5)  # 阻塞事件循环所在的线程
    print("结束")
```

正确写法：

```python
import asyncio


async def task() -> None:
    print("开始")
    await asyncio.sleep(5)
    print("结束")
```

区别：

```text
time.sleep(5)          整个事件循环线程无法继续
await asyncio.sleep(5) 只暂停当前协程，其他任务可以运行
```

同理，异步程序通常应选择异步 HTTP 客户端和异步数据库驱动。

## 十二、兼容同步阻塞函数：`to_thread()`

如果必须调用现有同步 I/O 函数，可以把它放到线程中：

```python
import asyncio
import time


def blocking_operation(name: str) -> str:
    time.sleep(2)
    return f"{name} 完成"


async def main() -> None:
    result = await asyncio.to_thread(
        blocking_operation,
        "任务 A",
    )

    print(result)


asyncio.run(main())
```

并发调用多个同步函数：

```python
async def main() -> None:
    results = await asyncio.gather(
        asyncio.to_thread(blocking_operation, "A"),
        asyncio.to_thread(blocking_operation, "B"),
    )

    print(results)
```

`to_thread()` 适合兼容阻塞 I/O，不代表纯 Python CPU 密集代码会因此获得理想的多核并行效果。

## 十三、异常处理

```python
import asyncio


async def request(name: str) -> str:
    if name == "B":
        raise RuntimeError("B 请求失败")

    await asyncio.sleep(1)
    return f"{name} 成功"


async def main() -> None:
    try:
        results = await asyncio.gather(
            request("A"),
            request("B"),
        )
        print(results)
    except RuntimeError as error:
        print(f"执行失败：{error}")


asyncio.run(main())
```

如果使用：

```python
results = await asyncio.gather(
    request("A"),
    request("B"),
    return_exceptions=True,
)
```

异常会作为结果返回，调用者必须逐项判断：

```python
for result in results:
    if isinstance(result, Exception):
        print(f"失败：{result}")
    else:
        print(f"成功：{result}")
```

不要为了“程序不报错”而无条件使用 `return_exceptions=True`，否则容易把失败当成普通业务结果。

## 十四、超时控制

```python
import asyncio


async def slow_request() -> str:
    await asyncio.sleep(10)
    return "完成"


async def main() -> None:
    try:
        async with asyncio.timeout(2):
            result = await slow_request()
            print(result)
    except TimeoutError:
        print("请求超时")


asyncio.run(main())
```

也可以针对单个 awaitable 使用：

```python
result = await asyncio.wait_for(
    slow_request(),
    timeout=2,
)
```

网络、数据库和 Agent 工具调用都应该有超时，避免任务永久占用资源。

## 十五、任务取消

```python
import asyncio


async def worker() -> None:
    try:
        while True:
            print("执行任务")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("收到取消请求，清理资源")

        # 清理连接或临时状态后，通常继续传播取消。
        raise


async def main() -> None:
    task = asyncio.create_task(worker())
    await asyncio.sleep(3)

    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("任务已取消")


asyncio.run(main())
```

取消通常在协程到达下一个可取消的等待点时生效。捕获 `CancelledError` 后通常应完成必要清理并继续 `raise`，不要无意吞掉取消信号。

## 十六、限制并发数量

异步不是无限并发。一次创建大量请求可能耗尽连接池、文件描述符、数据库连接或第三方 API 配额。

```python
import asyncio


semaphore = asyncio.Semaphore(3)


async def request(number: int) -> str:
    async with semaphore:
        print(f"开始请求 {number}")
        await asyncio.sleep(1)
        return f"请求 {number} 完成"


async def main() -> None:
    results = await asyncio.gather(
        *(request(number) for number in range(10))
    )

    print(results)


asyncio.run(main())
```

虽然创建了 10 个协程，但最多只有 3 个同时进入受保护区域。

## 十七、常用异步同步原语

| 原语 | 用途 |
| --- | --- |
| `asyncio.Lock` | 同一时刻只允许一个协程修改共享资源 |
| `asyncio.Semaphore` | 限制同时进入某区域的协程数量 |
| `asyncio.Event` | 一个协程通知其他协程某个事件已经发生 |
| `asyncio.Condition` | 等待共享状态满足特定条件 |
| `asyncio.Queue` | 在生产者和消费者之间安全传递任务 |

异步锁只协调同一个事件循环中的协程，不等于跨线程、跨进程或分布式锁。

## 十八、生产者—消费者与背压

```python
import asyncio


async def producer(queue: asyncio.Queue[int | None]) -> None:
    for number in range(5):
        await queue.put(number)
        print(f"生产：{number}")

    # None 是结束信号。
    await queue.put(None)


async def consumer(queue: asyncio.Queue[int | None]) -> None:
    while True:
        item = await queue.get()

        try:
            if item is None:
                return

            await asyncio.sleep(0.5)
            print(f"消费：{item}")
        finally:
            queue.task_done()


async def main() -> None:
    # 有界队列可以限制未消费任务数量，形成基础背压。
    queue: asyncio.Queue[int | None] = asyncio.Queue(maxsize=2)

    async with asyncio.TaskGroup() as group:
        group.create_task(producer(queue))
        group.create_task(consumer(queue))


asyncio.run(main())
```

当队列达到 `maxsize` 时，`await queue.put()` 会暂停生产者，直到消费者腾出空间。这可以防止生产速度长期超过消费能力。

生产系统还可能结合批处理、限流、超时、拒绝策略和监控实现背压。

## 十九、`async with`：异步资源管理

普通资源管理器：

```python
with open("data.txt") as file:
    content = file.read()
```

异步资源使用：

```python
async with resource:
    ...
```

常见场景：

```python
async with semaphore:
    await request()
```

异步 HTTP 客户端和数据库会话也常使用 `async with`，确保正常完成、异常或取消时都能释放连接。

普通 `with` 对应 `__enter__()`、`__exit__()`；异步版本对应 `__aenter__()`、`__aexit__()`。

## 二十、`async for`：异步迭代

```python
import asyncio
from collections.abc import AsyncIterator


async def generate_numbers() -> AsyncIterator[int]:
    for number in range(3):
        await asyncio.sleep(1)
        yield number


async def main() -> None:
    async for number in generate_numbers():
        print(number)


asyncio.run(main())
```

异步迭代器适合：

- 流式大模型输出；
- 消息队列消费；
- 分页接口；
- Socket 或 WebSocket 数据流。

## 二十一、Agent 并发调用工具示例

```python
import asyncio


async def search_web(query: str) -> str:
    await asyncio.sleep(1)
    return f"网页搜索结果：{query}"


async def search_database(query: str) -> str:
    await asyncio.sleep(1.5)
    return f"数据库结果：{query}"


async def build_context(query: str) -> list[str]:
    results = await asyncio.gather(
        search_web(query),
        search_database(query),
    )

    return list(results)


async def main() -> None:
    context = await build_context("Python asyncio")

    for item in context:
        print(item)


asyncio.run(main())
```

执行结构：

```text
用户问题
   │
   ├── search_web ───────┐
   │                     ├── 合并上下文
   └── search_database ──┘
```

独立读取通常适合并发，有冲突的写操作需要额外控制：

| 操作 | 是否适合直接并发 |
| --- | --- |
| 读取多个独立数据源 | 通常适合 |
| 写同一份文件 | 需要锁或串行化 |
| 修改同一数据库记录 | 需要事务和并发控制 |
| 发送不可重复消息 | 需要幂等设计 |

## 二十二、并发不等于并行

```text
并发 Concurrency：多个任务在同一时间段内交替推进
并行 Parallelism：多个任务在同一时刻真正同时执行
```

`asyncio` 通常在一个线程中运行，通过任务在 `await` 处让出控制权实现并发，并不代表多段 Python CPU 代码在多个核心上同时运行。

| 模型 | 适用场景 |
| --- | --- |
| 同步单线程 | 简单、顺序明确的流程 |
| `asyncio` | 大量 I/O 等待 |
| 多线程 | 阻塞 I/O、兼容同步库 |
| 多进程 | CPU 密集计算 |
| 分布式任务 | 大规模、跨机器任务 |

## 二十三、与 Java 的对照

| Java | Python |
| --- | --- |
| `CompletableFuture<T>` | Coroutine / `asyncio.Task` |
| `.join()`、`.get()` | `await`，但不会以相同方式阻塞事件循环线程 |
| `CompletableFuture.allOf()` | `asyncio.gather()` |
| `ExecutorService` | 线程池或进程池 |
| `Semaphore` | `asyncio.Semaphore` |
| `try-with-resources` | `with` / `async with` |
| Reactive Stream | 异步迭代器及相关流式库 |
| Structured Concurrency | `asyncio.TaskGroup` |

不要把 Python `await` 简单理解成阻塞式 `Future.get()`：`await` 暂停的是当前协程，事件循环线程可以继续运行其他就绪任务。

## 二十四、常见错误

### 调用异步函数却没有等待

```python
result = async_function()
```

此时得到的是协程对象，不是最终结果。

### 在 `async def` 中使用阻塞调用

`time.sleep()`、同步 HTTP SDK 或长时间 CPU 计算都会阻塞事件循环。

### 认为连续两个 `await` 会并发

```python
await first()
await second()
```

默认是串行。需要并发时使用 `gather()`、Task 或 `TaskGroup`。

### 认为 `async def` 自动创建新线程

不会。协程默认仍由事件循环所在的线程执行。

### 无限创建 Task

并发任务会消耗内存、连接和外部服务额度，需要信号量、有界队列或连接池限制。

### 创建 Task 后不管理

未等待的任务可能在程序退出前未完成，其异常也可能无人处理。

### 吞掉取消异常

清理资源后通常应该重新抛出 `CancelledError`，否则上层可能误以为任务正常完成。

## 二十五、从“会用”到生产级还需要什么

前面的内容足以用于：

- 看懂常见异步代码；
- 编写基础异步程序；
- 并发调用 HTTP、数据库或 Agent 工具；
- 处理基本异常、超时、取消和限流。

生产级异步系统还需要继续掌握：

### 协程协议与底层对象

- `Awaitable`、Coroutine、Future、Task 的完整关系；
- `__await__()` 协议；
- 事件循环恢复协程的过程。

### 结构化并发的失败传播

- `ExceptionGroup` 和 `except*`；
- `TaskGroup` 中一个任务失败后其他任务如何取消；
- 取消传播、屏蔽和清理边界。

### 真实网络和数据访问

- 异步 HTTP 客户端与连接池；
- WebSocket 和流式响应；
- 异步数据库驱动与事务；
- 重试、限流、熔断和降级。

### 生命周期管理

- 应用启动和关闭；
- 后台任务归属；
- 数据库、HTTP Client 和线程池关闭；
- 收到退出信号后的优雅停机。

### 调试与测试

- 检测阻塞事件循环的同步代码；
- 查找 Task 泄漏和未处理异常；
- 异步单元测试；
- 模拟超时、取消和部分失败。

### 业务正确性

- 幂等性；
- 超时预算如何逐层传递；
- 并发写冲突；
- 任务部分成功；
- 日志、指标和链路追踪。

## 学习路线

建议按照以下顺序实际编写代码：

```text
async def 与协程对象
        ↓
await 与事件循环
        ↓
串行 await 和 gather
        ↓
create_task 与 TaskGroup
        ↓
异常、超时、取消
        ↓
Semaphore、Lock、Queue
        ↓
async with 与 async for
        ↓
同步阻塞代码兼容
        ↓
生产者—消费者与背压
        ↓
真实 HTTP / 数据库项目
        ↓
优雅关闭、测试和可观测性
```

## 练习

### 练习一：验证串行和并发耗时

分别使用连续 `await` 和 `gather()` 执行三个 `asyncio.sleep(1)`，记录总耗时并解释差异。

### 练习二：限制工具并发

模拟 20 个 Agent 工具调用，使用 `Semaphore(4)` 保证最多同时运行 4 个。

### 练习三：生产者—消费者

创建一个容量为 3 的 Queue，一个生产者生成 10 个任务，两个消费者处理任务，并观察生产者何时因队列已满而等待。

### 练习四：超时和取消清理

创建一个执行 10 秒的协程，设置 2 秒超时，并在取消处理里打印资源清理日志。

### 练习五：阻塞事件循环

分别在协程中使用 `time.sleep(2)` 和 `await asyncio.sleep(2)`，同时启动另一个每 0.5 秒打印一次的协程，观察两者区别。

## 总结

```text
async def       定义协程函数
await           暂停当前协程并等待结果
Task            已交给事件循环调度的协程
gather          并发等待多个协程并收集结果
TaskGroup       结构化管理一组相关任务
Semaphore       限制并发数量
Queue           解耦生产者和消费者并形成背压
to_thread       兼容同步阻塞 I/O
```

判断是否应该使用异步，可以先问两个问题：

1. 任务的大部分时间是否在等待外部 I/O？
2. 等待期间是否存在其他独立任务可以推进？

如果答案都是“是”，异步通常能改善整体吞吐量；如果主要工作是 CPU 计算，则应考虑多进程等其他并行方案。

> `TaskGroup`、`asyncio.timeout()` 等 API 与 Python 版本有关。实际项目应先确认运行环境版本，再决定是否使用相应写法。
