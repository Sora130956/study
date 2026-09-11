# asyncio 异步编程入门教程

> 适合：会 Python 基础语法（函数、类、模块），但完全没接触过异步编程的人
> Python 版本：建议 3.10+（涉及 3.11+ 新特性的地方会单独标注）
> 学习方式：边看边敲，每个代码块都可以直接运行（涉及网络的需先 `pip install httpx`）
>
> **学习原则**：先理解"异步到底解决什么问题"，再学 API。asyncio 的 API 不少，但核心思想只有一句话：**等待 IO 时不要干等着，切去干别的**。

---

## 重点标注（AI 应用方向 · 星级）

> 星级标准：★★★ 必须精通（学完当天就要能默写关键代码）｜★★☆ 需要理解（能讲清楚原理，细节可查）｜★☆☆ 了解即可（用到再回来看）

| 章节                           | 星级  | 为什么（对照 AI 应用学习计划）                                                         |
| ---------------------------- | --- | ------------------------------------------------------------------------- |
| 第 0 章 为什么需要 asyncio          | ★★★ | 建立"LLM 调用 = 高延迟 IO"的认知，这是整个计划的立论基础                                        |
| 第 1 章 第一个异步程序                | ★★★ | FastAPI 每个接口都是 `async def`；"协程调用不执行"是第一大坑                                 |
| 第 2 章 事件循环                   | ★★☆ | 只需精读 2.2（只在 await 处切换）；调度器内部机制了解即可                                        |
| 第 3 章 并发执行多个协程               | ★★★ | 批量调 LLM（W3）、并行工具调用（W8）全靠 `gather`/`create_task`                           |
| 第 4 章 超时与取消                  | ★★★ | LLM 响应动辄 10s+，`wait_for` 是保命符，4.1 必会；4.3 取消原理理解即可                         |
| 第 5 章 同步原语                   | ★★☆ | **其中 5.2 Semaphore 单独 ★★★**——W3"批量处理 1000 条不被限流"就是它；Lock/Event/Queue 了解即可 |
| 第 6 章 async with/for + httpx | ★★★ | W1 周二下午作业原文；`async for` 是流式输出 / SSE 的语法基础                                 |
| 第 7 章 阻塞调用与 to_thread        | ★★★ | 计划 W1 验证点直接考："为什么 `async def` 里不能调阻塞的 `requests.get`"                     |
| 第 8 章 常见错误清单                 | ★★★ | 学完全部内容后回来扫一遍，当自查表用                                                        |
| 第 9 章 调试技巧                   | ★☆☆ | 用到再查                                                                      |
| 附录 A 核心 API 速查表              | ★★★ | 写代码时随手翻                                                                   |
| 附录 B 选型决策树                   | ★★☆ | RAG 阶段（W4+）会用到 asyncpg；先记住同步/异步库的映射                                       |
| 附录 C 一页纸爬虫模板                 | ★☆☆ | 内容本身爬虫向，但"限流+超时+失败容忍"的模式可直接迁移到批量调 LLM                                     |

**一日速通路线（对应 W1 周二一天）**：

- 上午：第 0 章 → 第 1 章 → 第 2 章（重点 2.2）→ 第 3 章
- 下午：第 4 章（4.1 为主）→ 第 6 章实战 → 第 7 章 → 完成计划作业：用 httpx.AsyncClient 并发请求 10 个 URL，对比同步耗时
- 延后：5.2 Semaphore 在 W3（并发与批处理）之前补上；第 9 章、附录 B 用到再查

---

## 第 0 章：为什么需要 asyncio ★★★

### 0.1 一个熟悉的场景

假设你要下载 100 个网页，同步代码这样写：

```python
import requests

urls = ["https://example.com"] * 100

for url in urls:
    resp = requests.get(url)   # 每次等待网络响应约 0.5 秒
    print(resp.status_code)

# 总耗时 ≈ 100 × 0.5 = 50 秒
```

问题在哪？`requests.get` 发出请求后，你的程序**什么都不干，纯等着**。等网络响应的这段时间里，CPU 是空闲的——这是巨大的浪费。

asyncio 的思路：**既然你在等 A 的响应，不如先去发 B 的请求，等谁的响应先到就先处理谁**。

### 0.2 四种写法对比

| 方案           | 适用场景                     | 典型例子                 | 上手难度 |
| -------------- | ---------------------------- | ------------------------ | -------- |
| 同步顺序执行   | 任务少、单次耗时可忽略       | 读 3 个本地文件          | 低       |
| **asyncio**    | **IO 密集 + 高并发**         | 爬虫、批量调 API、代理池 | 中       |
| threading      | IO 密集，但依赖的库不支持异步 | 用不支持异步的老库       | 中       |
| multiprocessing | CPU 密集型计算               | 图像处理、大数据运算     | 高       |

两个关键认知，先记下来，后面反复验证：

1. **asyncio 只对 IO 密集型任务有用**（网络、文件、数据库）。它不能让 CPU 计算变快。
2. **asyncio 不是多线程，是单线程**。它靠"协作式调度"（谁遇到等待就让出 CPU）实现并发，绕开了多线程的锁、竞态等大部分麻烦。

> **GIL 补充**：CPython 有全局解释器锁，多线程也无法让 Python 代码在多个 CPU 核上并行执行。所以 IO 并发首选 asyncio，CPU 并行首选 multiprocessing。（Python 3.13 有实验性 free-threading 模式，日常先不用管。）

---

## 第 1 章：第一个异步程序

### 1.1 async def 与 await

两个新关键字，先看最小可运行示例：

```python
import asyncio


async def hello():                     # async def 定义"协程函数"
    print("start")
    await asyncio.sleep(1)             # await：等待一个耗时操作完成
    print("end")


asyncio.run(hello())                   # 启动事件循环并运行协程
# 输出：
# start
# （停 1 秒）
# end
```

语法规则：

- `async def` 定义的函数叫**协程函数**，调用它得到的不是结果，而是一个**协程对象**（coroutine）
- `await` 只能写在 `async def` 函数内部，意思是"这里可能要等，等待期间可以让出控制权"
- 协程对象必须交给事件循环运行，入口是 `asyncio.run()`

### 1.2 第一个坑：调用协程函数不会执行它

这是新手 100% 会遇到的问题：

```python
import asyncio


async def hello():
    print("hello")


hello()          # 什么都没打印！只是创建了协程对象
print("after")   # after

# 运行时会收到警告：
# RuntimeWarning: coroutine 'hello' was never awaited
```

**协程函数被调用时只是"准备好了一个待执行的任务"，不会运行任何函数体**。要执行它，必须 `await` 它，或交给 `asyncio.run()` / `asyncio.create_task()`。

```python
asyncio.run(hello())   # ✓ 这样才会执行

# 在另一个协程里：
async def main():
    await hello()      # ✓ 这样也会执行
```

记忆方式：协程对象像一张"待办清单"，`await` 才是"开始办这件事"。

### 1.3 await 到底在等什么

`await` 后面只能跟"可等待对象"（awaitable）：

| 可等待对象             | 含义                     | 例子                          |
| ---------------------- | ------------------------ | ----------------------------- |
| 协程对象               | 另一个异步函数的调用     | `await hello()`               |
| Task                   | 被"包起来"的协程         | `await some_task`             |
| Future                 | 底层占位结果（较少直接用） | `asyncio.gather()` 的返回中间态 |

日常 99% 的场景就是 `await` 一个协程。记住一句话即可：**`await X` = 等 X 的结果，等待期间控制权交还给事件循环**。

### 1.4 asyncio.sleep vs time.sleep

理解 asyncio 的关键实验，两个版本对比：

```python
import asyncio
import time


async def bad():
    time.sleep(1)          # ✗ 同步阻塞：整个程序卡住 1 秒，什么都干不了


async def good():
    await asyncio.sleep(1) # ✓ 异步等待：这 1 秒里事件循环可以去跑别的协程
```

- `time.sleep(1)`：线程原地暂停，事件循环被"冻住"，其他协程全部无法运行
- `await asyncio.sleep(1)`：告诉事件循环"我 1 秒内不需要 CPU，先去忙别的"

**这个区别是整个 asyncio 的灵魂**。后面所有 API 都建立在"等待时主动让出"这个机制上。

---

## 第 2 章：事件循环——asyncio 的心脏

### 2.1 概念模型

事件循环（event loop）就是一个**单线程的任务调度器**，它维护着一个待办任务列表，循环执行：

```
┌─────────────────────────────────────────┐
│              事件循环（单线程）             │
│                                         │
│  1. 取出一个就绪的协程，开始执行            │
│  2. 协程遇到 await（需要等待）→ 挂起，      │
│     让出控制权                            │
│  3. 事件循环转去执行下一个就绪的协程         │
│  4. 等待的操作完成后，把挂起的协程           │
│     重新放回就绪队列                       │
│  5. 回到第 1 步                           │
└─────────────────────────────────────────┘
```

因为只有一个线程，**任何时刻只有一个协程在真正运行**。并发感来自"等待时的切换"。

### 2.2 协作式调度：只在 await 处切换

重要推论：**两个协程之间的切换只发生在 `await` 处**。看这个实验：

```python
import asyncio


async def count():
    for i in range(3):
        print(i)
        await asyncio.sleep(0)   # sleep(0)：主动让出一次控制权


async def main():
    await asyncio.gather(count(), count())   # 同时跑两个

asyncio.run(main())
# 输出（两个 count 交替执行）：
# 0
# 0
# 1
# 1
# 2
# 2
```

如果把 `await asyncio.sleep(0)` 去掉，第一个 `count` 会一次性打印完 `0 1 2`，第二个才开始——**没有 await，就没有让出控制权的机会**。

这带来一个实践铁律：**协程里绝对不能有长时间阻塞调用**（详见第 7 章）。

### 2.3 asyncio.run 的规矩

```python
asyncio.run(main())
```

- 程序的异步入口，创建事件循环、运行 `main()` 直到完成、关闭循环
- 一个程序通常只调用一次 `asyncio.run()`
- `asyncio.run()` 内部不能嵌套调用（已经是运行中的循环里再调会报 `RuntimeError`）
- 在 Jupyter Notebook 里不能直接用 `asyncio.run()`（notebook 自身已运行事件循环），直接在单元格里写 `await main()` 即可

---

## 第 3 章：并发执行多个协程

这是 asyncio 的核心价值所在。前面学的 `await` 只是"换了个写法的顺序执行"，这一章才体现"快"。

### 3.1 顺序 await 不是并发

```python
import asyncio
import time


async def fetch(name, delay):
    await asyncio.sleep(delay)     # 模拟网络请求
    print(f"{name} 完成")
    return name


async def main():
    start = time.perf_counter()
    await fetch("A", 1)            # 等 A 做完
    await fetch("B", 1)            # 再做 B
    print(f"耗时 {time.perf_counter() - start:.2f} 秒")

asyncio.run(main())
# A 完成
# B 完成
# 耗时 2.00 秒
```

`await fetch("A", 1)` 的意思是"等 A 完全做完再往下走"，和同步代码没有区别。**要并发，必须显式创建 Task**。

### 3.2 create_task：把协程变成后台任务

`asyncio.create_task(coro)` 把协程"打包"成 Task 并**立即排队执行**，不等待：

```python
import asyncio
import time


async def fetch(name, delay):
    await asyncio.sleep(delay)
    print(f"{name} 完成")
    return name


async def main():
    start = time.perf_counter()

    task_a = asyncio.create_task(fetch("A", 1))   # 创建后立即开始跑
    task_b = asyncio.create_task(fetch("B", 1))   # 同上，A、B 并发

    result_a = await task_a    # 此时通常已经在跑了，await 只是收结果
    result_b = await task_b

    print(result_a, result_b)
    print(f"耗时 {time.perf_counter() - start:.2f} 秒")

asyncio.run(main())
# A 完成
# B 完成
# A B
# 耗时 1.00 秒   ← 并发后从 2 秒变 1 秒
```

**⚠️ 经典坑：Task 必须保存引用**。`asyncio.create_task(fetch(...))` 如果不把返回值存到变量，Task 可能被垃圾回收器中途回收，任务莫名消失。规则：**创建 Task 必赋值，或直接配合 gather/TaskGroup 使用**。

### 3.3 gather：并发 + 统一收结果（最常用）

`asyncio.gather()` 接收多个协程，并发执行，按**传入顺序**返回结果列表：

```python
import asyncio


async def fetch(name, delay):
    await asyncio.sleep(delay)
    print(f"{name} 完成")
    return f"{name} 的结果"


async def main():
    # 三个任务并发执行，谁耗时久不影响别人
    results = await asyncio.gather(
        fetch("A", 2),
        fetch("B", 1),
        fetch("C", 3),
    )
    print(results)
    # ['A 的结果', 'B 的结果', 'C 的结果']
    # 顺序与传入顺序一致，与完成顺序无关

asyncio.run(main())
```

耗时 = 最慢的那个（本例 3 秒），而不是各任务之和（6 秒）。

**批量写法**（爬虫标配）：

```python
tasks = [fetch(f"任务{i}", i % 3 + 1) for i in range(10)]
results = await asyncio.gather(*tasks)
```

**异常处理**：默认情况下任一任务抛异常，gather 立刻把该异常抛给 await 处；加 `return_exceptions=True` 则异常不中断其他任务，异常对象会出现在结果列表里：

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        print("有个任务失败了:", r)
    else:
        print("成功:", r)
```

### 3.4 as_completed：谁先完成先处理谁

`gather` 要等全部完成。如果你希望"完成一个处理一个"（比如流式写入结果），用 `asyncio.as_completed`：

```python
import asyncio


async def fetch(name, delay):
    await asyncio.sleep(delay)
    return name


async def main():
    coros = [fetch("A", 3), fetch("B", 1), fetch("C", 2)]

    for finished in asyncio.as_completed(coros):
        result = await finished       # 每次拿到的是"当前最早完成"的结果
        print("先到先处理:", result)

asyncio.run(main())
# 先到先处理: B   （1 秒时）
# 先到先处理: C   （2 秒时）
# 先到先处理: A   （3 秒时）
```

典型用途：并发查询多个数据源，谁先返回先用谁；实时打印每个爬取结果。

### 3.5 TaskGroup：结构化并发（Python 3.11+，推荐）

`asyncio.TaskGroup` 是 3.11 引入的现代写法，优势：**任何子任务失败，自动取消其余所有子任务并抛出异常组**，不会留下失控的后台任务：

```python
import asyncio


async def fetch(name, delay):
    await asyncio.sleep(delay)
    if name == "B":
        raise ValueError(f"{name} 出错了")
    return name


async def main():
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch("A", 1))
        t2 = tg.create_task(fetch("B", 1))
        t3 = tg.create_task(fetch("C", 5))   # B 失败时 C 会被自动取消
        # async with 块结束时自动等待全部任务完成

    print(t1.result(), t3.result())

asyncio.run(main())
# 抛出 ExceptionGroup，包含 ValueError("B 出错了")
# C 虽然要 5 秒，但不会真的等 5 秒——被自动取消
```

选择建议：

- 任务互相独立、允许个别失败 → `gather(return_exceptions=True)`
- 任务是一个整体，一损俱损 → `TaskGroup`（3.11+）

---

## 第 4 章：超时与取消

### 4.1 单个任务加超时：wait_for

网络请求必须设超时。`asyncio.wait_for(aw, timeout)` 超时后取消任务并抛 `TimeoutError`：

```python
import asyncio


async def slow():
    await asyncio.sleep(10)
    return "结果"


async def main():
    try:
        result = await asyncio.wait_for(slow(), timeout=3)
    except TimeoutError:                 # 3.11+ 直接捕获内置 TimeoutError
        print("超时了，放弃这个任务")

asyncio.run(main())
# 超时了，放弃这个任务（3 秒后）
```

### 4.2 一段代码加超时：asyncio.timeout（3.11+）

给整个代码块设置总超时，比逐个任务设更简洁：

```python
import asyncio


async def main():
    try:
        async with asyncio.timeout(5):       # 块内总耗时超过 5 秒就中断
            await asyncio.sleep(3)           # 任务 1
            await asyncio.sleep(3)           # 任务 2 —— 累计 6 秒 > 5 秒
    except TimeoutError:
        print("整体超时")

asyncio.run(main())
```

### 4.3 取消的原理

Task 被取消时，会在它**当前 await 的位置**抛出 `asyncio.CancelledError`（注意：它是 `BaseException` 的子类，`except Exception` 捕获不到它）：

```python
import asyncio


async def worker():
    try:
        await asyncio.sleep(100)
    except asyncio.CancelledError:
        print("被取消，做点清理工作（关连接、写日志等）")
        raise        # 惯例：清理完后重新抛出，让取消流程正常完成


async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(1)
    task.cancel()                 # 请求取消
    try:
        await task
    except asyncio.CancelledError:
        print("任务已取消")

asyncio.run(main())
```

`task.cancel()` 是"请求取消"，任务会在下一个 await 点真正退出。

---

## 第 5 章：同步原语——协调多个协程 ★★☆

单线程为什么还需要锁？因为**协程之间依然会交错执行**（每个 await 都是潜在的切换点），"检查再修改"这种非原子操作照样可能出问题。

### 5.1 Lock：保护共享状态

```python
import asyncio

counter = 0


async def unsafe_add(n):
    global counter
    for _ in range(n):
        tmp = counter            # 读
        await asyncio.sleep(0)   # ← 切换点！其他协程可能在这里改了 counter
        counter = tmp + 1        # 写回旧值 + 1，更新丢失


async def safe_add(lock, n):
    global counter
    for _ in range(n):
        async with lock:         # 和 threading.Lock 用法一致，但是 await 版
            tmp = counter
            await asyncio.sleep(0)
            counter = tmp + 1


async def main():
    global counter
    lock = asyncio.Lock()

    counter = 0
    await asyncio.gather(*(unsafe_add(100) for _ in range(3)))
    print("不加锁:", counter)     # 几乎必然 < 300，更新丢失

    counter = 0
    await asyncio.gather(*(safe_add(lock, 100) for _ in range(3)))
    print("加锁:  ", counter)     # 300

asyncio.run(main())
```

### 5.2 Semaphore：限制并发数（爬虫必备）

高并发爬虫如果不限流，几百个请求同时打过去，要么被 ban，要么打垮目标服务器。`Semaphore(n)` 同时最多放行 n 个：

```python
import asyncio


async def fetch(sem, url_id):
    async with sem:                    # 第 6 个开始在这里排队等待
        print(f"抓取 {url_id}")
        await asyncio.sleep(1)         # 模拟请求
        return url_id


async def main():
    sem = asyncio.Semaphore(5)         # 最多 5 个并发
    results = await asyncio.gather(
        *(fetch(sem, i) for i in range(20))
    )
    print(f"完成 {len(results)} 个")

asyncio.run(main())
```

这比"分批抓"（每批 gather）更好：先完成的任务立刻释放名额，不用等整批结束。

### 5.3 Event：通知"可以开始了"

一个协程等待某个条件，另一个协程触发它：

```python
import asyncio


async def waiter(event, name):
    print(f"{name} 就绪，等待信号")
    await event.wait()                 # 异步等待，直到 event 被 set
    print(f"{name} 收到信号，开始工作")


async def signal_later(event):
    await asyncio.sleep(2)
    print("发出信号！")
    event.set()                        # 所有等待者同时放行


async def main():
    event = asyncio.Event()
    await asyncio.gather(
        waiter(event, "worker-1"),
        waiter(event, "worker-2"),
        signal_later(event),
    )

asyncio.run(main())
# worker-1 就绪，等待信号
# worker-2 就绪，等待信号
# （2 秒后）发出信号！
# worker-1 收到信号，开始工作
# worker-2 收到信号，开始工作
```

`event.set()` 可以在任何协程里调用；`event.clear()` 可以重置信号，复用同一事件。

### 5.4 Queue：生产者-消费者

`asyncio.Queue` 是协程间传数据的标准方式，本身线程不安全（只给协程用）：

```python
import asyncio


async def producer(q):
    for i in range(10):
        await q.put(i)                 # 队列满（maxsize）时自动等待
        print(f"生产 {i}")
    # 不再生产，通知消费者退出
    await q.join()                     # 等所有数据被处理完
    for _ in range(3):                 # 3 个消费者，每人塞一个结束标记
        q.put_nowait(None)


async def consumer(q, name):
    while True:
        item = await q.get()           # 队列空时自动等待
        if item is None:
            break
        await asyncio.sleep(0.5)       # 模拟处理
        print(f"{name} 处理完 {item}")
        q.task_done()                  # 告诉队列：这条处理完了


async def main():
    q = asyncio.Queue(maxsize=5)       # 限制积压量
    await asyncio.gather(
        producer(q),
        consumer(q, "消费者A"),
        consumer(q, "消费者B"),
        consumer(q, "消费者C"),
    )
    print("全部完成")

asyncio.run(main())
```

用途：爬虫的 URL 调度（生产 URL → 多个 worker 消费）、日志缓冲写入等。

---

## 第 6 章：async with 与 async for

### 6.1 async with：需要"异步清理"的上下文管理器

HTTP 客户端的连接建立/释放是 IO 操作，所以要用 `async with`：

```python
# 伪代码示意
async with client as c:        # __aenter__：异步地建立连接
    await c.get(...)           # 使用
                               # 离开块时 __aexit__：异步地关闭连接
```

你在 `asyncio.timeout`、`asyncio.Lock`、`TaskGroup` 的例子里其实已经用过它了。

#### 为什么需要它

普通 `with` 的 `__enter__`/`__exit__` 是**同步函数**，里面不能 `await`。但很多资源的打开/关闭本身就是 IO（TCP/TLS 握手、关闭连接池等），用普通 `with` 会阻塞整个事件循环。所以 Python 3.5 引入异步版协议：

| 普通 with | async with |
|---|---|
| `__enter__()` | `__aenter__()`（async 方法） |
| `__exit__(exc_type, exc, tb)` | `__aexit__(exc_type, exc, tb)`（async 方法） |

#### 等价展开（脱糖）

上面的伪代码大致等价于：

```python
c = await client.__aenter__()   # 异步建立连接
try:
    await c.get(...)
finally:                          # 无论正常结束还是抛异常都会执行
    await client.__aexit__(None, None, None)   # 异步关闭连接
```

要点：
- `__aenter__` 的返回值赋给 `as` 后面的变量；
- 块内抛异常时，异常信息 `(exc_type, exc_value, traceback)` 会传给 `__aexit__`；
- 清理逻辑在任何路径下（包括异常）都保证执行——这就是"上下文管理"的核心价值。

#### 自己写一个

方式一：类实现 `__aenter__` / `__aexit__`：

```python
import asyncio

class AsyncConnection:
    async def __aenter__(self):
        await asyncio.sleep(0.5)   # 异步地做准备（模拟建连）
        return self                # as 后面拿到的就是它

    async def __aexit__(self, exc_type, exc, tb):
        await asyncio.sleep(0.5)   # 异步地清理（模拟关连接）
        return False               # 不吞异常

    async def get(self, url):
        await asyncio.sleep(0.1)
        return f"响应: {url}"

async def main():
    async with AsyncConnection() as conn:
        print(await conn.get("/users"))

asyncio.run(main())
```

方式二：装饰器写法（更常用）：

```python
from contextlib import asynccontextmanager

@asynccontextmanager            # 注意：同步用 @contextmanager
async def connection():
    await asyncio.sleep(0.5)    # 进入前：建连
    try:
        yield "conn对象"        # yield 的值就是 as 拿到的值
    finally:
        await asyncio.sleep(0.5)   # 退出时：关连接
```

#### 内置的三个典型例子

- `asyncio.timeout(n)`：进入时启动计时，退出时清理并按需取消内部任务；
- `asyncio.Lock`：进入时 `await acquire()`（可能等别的协程放锁），退出时自动 `release()`；
- `asyncio.TaskGroup`：进入时创建组，退出时等待所有子任务完成（任一失败抛 `ExceptionGroup`）。

共同点：**进入和/或退出时需要 await**，所以必须用 `async with`。

一句话总结：`async with` = 普通 `with` 的异步版，把进入/退出钩子换成可 `await` 的版本，专门管理"打开和关闭本身都是 IO"的资源，并保证清理逻辑在任何路径下都执行。

### 6.2 async for：迭代异步生成器

生成器也能是异步的——每次 `yield` 前可以 await：

```python
import asyncio


async def numbers(n):
    for i in range(n):
        await asyncio.sleep(0.5)   # 模拟每条数据要等 IO
        yield i                    # 注意：async def + yield = 异步生成器


async def main():
    async for num in numbers(5):   # async for 迭代
        print(num)                 # 每 0.5 秒打印一个

asyncio.run(main())
```

典型场景：分页 API 逐页拉取、逐行读取大文件/流式响应。

### 6.3 实战：httpx 并发抓取

`requests` 是同步库，在协程里用它会把事件循环卡死。异步 HTTP 的主流选择是 **httpx**（API 和 requests 几乎一样，另有 aiohttp）：

```python
# pip install httpx
import asyncio
import httpx
import time

URLS = [f"https://httpbin.org/delay/{i}" for i in [1, 1, 1, 1, 1]]


async def fetch(client, url):
    resp = await client.get(url, timeout=10)
    return url, resp.status_code


async def main():
    start = time.perf_counter()
    async with httpx.AsyncClient() as client:     # 复用连接池，必须 async with
        results = await asyncio.gather(
            *(fetch(client, url) for url in URLS)
        )
    for url, status in results:
        print(status, url)
    print(f"总耗时 {time.perf_counter() - start:.2f} 秒")   # ≈ 1 秒多

asyncio.run(main())
```

对比：同步 requests 逐个抓要 5 秒+，asyncio 版本几乎只要最慢那一个的时间。

**加限流完整版**（Semaphore + 失败容忍，可直接改造用于接单）：

```python
import asyncio
import httpx


async def fetch(client, sem, url):
    async with sem:
        try:
            resp = await client.get(url, timeout=10)
            return url, resp.status_code, None
        except Exception as e:
            return url, None, str(e)


async def main():
    urls = [f"https://httpbin.org/delay/1" for _ in range(30)]
    sem = asyncio.Semaphore(10)               # 最多 10 并发

    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(
            *(fetch(client, sem, u) for u in urls),
            return_exceptions=True,
        )

    ok = [r for r in results if r[1] == 200]
    failed = [r for r in results if r[1] != 200]
    print(f"成功 {len(ok)}，失败 {len(failed)}")

asyncio.run(main())
```

---

## 第 7 章：异步代码里调用同步代码

### 7.1 问题：阻塞调用会冻结整个事件循环

```python
import asyncio
import time


async def heartbeat():
    for _ in range(5):
        print(f"[心跳] {time.strftime('%H:%M:%S')}")
        await asyncio.sleep(1)


async def heavy_sync_task():
    time.sleep(3)            # ✗ 阻塞调用！心跳会停 3 秒


async def main():
    await asyncio.gather(heartbeat(), heavy_sync_task())

asyncio.run(main())
# 心跳打印到一半卡住 3 秒 —— 因为整个循环被 time.sleep 冻住了
```

常见的"阻塞元凶"：`time.sleep`、`requests`、`open()` 读大文件、pandas 重计算、`psycopg2` 查询、任何 CPU 重循环。

### 7.2 解决：asyncio.to_thread（Python 3.9+）

把阻塞函数扔到线程池里跑，事件循环继续自由调度：

```python
import asyncio
import time


def heavy_sync():            # 注意：这是普通同步函数
    time.sleep(3)
    return "同步任务完成"


async def heartbeat():
    for _ in range(5):
        print(f"[心跳] {time.strftime('%H:%M:%S')}")
        await asyncio.sleep(1)


async def main():
    await asyncio.gather(
        heartbeat(),
        asyncio.to_thread(heavy_sync),   # ✓ 阻塞函数在线程里跑，不冻结循环
    )

asyncio.run(main())
# 心跳全程正常，每秒一跳
```

经验法则：

- **能找到异步库就用异步库**（httpx 替代 requests、`asyncio.sleep` 替代 `time.sleep`）
- **找不到异步替代品 → `asyncio.to_thread` 包一层**（比如必须用某个同步 SDK）
- **CPU 密集计算 → to_thread 也没用**（GIL），用 `ProcessPoolExecutor`：

```python
from concurrent.futures import ProcessPoolExecutor

executor = ProcessPoolExecutor()

async def main():
    result = await asyncio.get_running_loop().run_in_executor(
        executor, cpu_heavy_function, arg1
    )
```

---

## 第 8 章：常见错误清单（新手自查表）

### 错误 1：调用协程函数但忘记 await

```python
async def get_data():
    return 42

async def main():
    data = get_data()      # ✗ data 是协程对象，不是 42
    data = await get_data()  # ✓
```

症状：打印出 `<coroutine object get_data at 0x...>`，或警告 `coroutine ... was never awaited`。

### 错误 2：在协程里用阻塞函数

```python
async def bad():
    time.sleep(1)                 # ✗ 冻结整个事件循环
    requests.get(url)             # ✗ 同上

async def good():
    await asyncio.sleep(1)        # ✓
    await client.get(url)         # ✓ 异步客户端
    await asyncio.to_thread(blocking_fn)  # ✓ 实在没法异步时
```

症状：明明并发了 50 个任务，总耗时却是所有任务之和；或 `Executing <Handle...> took 2.00 seconds` 警告。

### 错误 3：以为 asyncio 能加速 CPU 计算

```python
async def crunch():
    result = sum(i * i for i in range(10_000_000))   # 纯 CPU，无 await
```

协程从头跑到尾没有任何切换点，和同步一样慢，甚至更慢。CPU 密集 → multiprocessing。

### 错误 4：create_task 不保存引用

```python
async def main():
    asyncio.create_task(background_job())   # ✗ 可能被 GC 回收，任务蒸发
    task = asyncio.create_task(background_job())  # ✓
    ...
    await task                              # ✓ 顺便兜住异常
```

### 错误 5：在普通函数里写 await

```python
def main():
    await something()     # ✗ SyntaxError，await 只能出现在 async def 里
```

如果入口本身是同步的，用 `asyncio.run()` 包一层。

### 错误 6：并发场景下依然逐个 await

```python
# ✗ 顺序执行，没有并发，白写了 async
for coro in coros:
    await coro

# ✓ 三种正确姿势
results = await asyncio.gather(*coros)               # 全部完成后统一拿
for fut in asyncio.as_completed(coros):              # 完成一个处理一个
    r = await fut
async with asyncio.TaskGroup() as tg:                # 3.11+，一损俱损
    tasks = [tg.create_task(c) for c in coros]
```

### 错误 7：用 except Exception 吞掉 CancelledError

`CancelledError` 继承自 `BaseException`，`except Exception` 捕不到它（好事）。但如果写了 `except BaseException` 或裸 `except`，记得重新抛出：

```python
try:
    await something()
except asyncio.CancelledError:
    cleanup()
    raise        # ✓ 必须重新抛出，否则取消流程被打断，任务"拒绝被取消"
```

---

## 第 9 章：调试技巧 ★☆☆

### 9.1 开启 debug 模式，自动报告慢协程和未 await

```python
asyncio.run(main(), debug=True)      # 或环境变量 PYTHONASYNCIODEBUG=1
```

debug 模式会：

- 检测回调/协程运行超过 100ms 并告警（帮你定位阻塞调用）
- `asyncio.sleep(0)` 之类细节更严格
- 未消费的协程对象警告更详细

命令行等价：`python -X dev main.py`。

### 9.2 打印当前所有任务

```python
async def main():
    ...
    for task in asyncio.all_tasks():
        print(task.get_name(), task.get_coro(), task.done())
```

配合 `task.set_name("fetch-page-3")` 给任务命名，日志一目了然。

### 9.3 看懂异常组（3.11+ TaskGroup）

TaskGroup 抛出的 `ExceptionGroup` 内含多个子异常，遍历打印：

```python
try:
    async with asyncio.TaskGroup() as tg:
        ...
except* ValueError as eg:      # except* 专门解包异常组
    for exc in eg.exceptions:
        print("子异常:", exc)
```

---

## 附录 A：核心 API 速查表

| API                                            | 作用                     | 记忆点                          |
| ---------------------------------------------- | ------------------------ | ------------------------------- |
| `asyncio.run(main())`                          | 程序入口，启动事件循环     | 只调一次，不能嵌套               |
| `await coro()`                                 | 等一个协程完成            | 顺序执行，不是并发               |
| `asyncio.sleep(s)`                             | 异步等待                  | 替代 `time.sleep`               |
| `task = asyncio.create_task(c)`                | 立即并发运行协程          | **必须保存引用**                 |
| `await asyncio.gather(*coros)`                 | 并发 + 按序收结果         | 最常用；`return_exceptions=True` |
| `for fut in asyncio.as_completed(coros)`       | 谁先完成先处理            | 流式处理结果                     |
| `async with asyncio.TaskGroup()` (3.11+)       | 结构化并发                | 一损俱损，自动取消兄弟任务        |
| `await asyncio.wait_for(c, timeout=3)`         | 单任务超时                | 超时抛 `TimeoutError`           |
| `async with asyncio.timeout(3)` (3.11+)        | 代码块总超时              | 优先用这个                       |
| `asyncio.Semaphore(n)`                         | 限制并发数                | 爬虫限流标配                     |
| `asyncio.Lock()`                               | 保护共享状态              | `async with lock:`              |
| `asyncio.Queue(maxsize=n)`                     | 生产者/消费者队列          | 协程专用，非线程安全             |
| `asyncio.Event()`                              | 广播信号                  | `await e.wait()` / `e.set()`    |
| `await asyncio.to_thread(fn, *args)` (3.9+)    | 在线程里跑阻塞函数         | 不冻结事件循环                   |
| `task.cancel()`                                | 请求取消任务               | 下一个 await 点生效              |

## 附录 B：技术选型决策树 ★★☆

```
任务是 CPU 密集还是 IO 密集？
├─ CPU 密集（计算、图像处理）→ multiprocessing / ProcessPoolExecutor
└─ IO 密集（网络、文件、数据库）
   ├─ 依赖库有异步版本（httpx、aiofiles、asyncpg...）→ asyncio ✅
   ├─ 只有同步库，且并发量不大（< 几十）→ threading / to_thread 也够
   └─ 只有同步库，但要高并发 → to_thread + Semaphore 限流，或换异步库
```

常用异步生态对应表：

| 领域     | 同步库          | 异步替代                                   |
| -------- | --------------- | ------------------------------------------ |
| HTTP     | requests        | **httpx**、aiohttp                          |
| 文件读写 | open()          | aiofiles                                    |
| PostgreSQL | psycopg2      | asyncpg                                     |
| MySQL    | PyMySQL         | aiomysql                                    |
| Redis    | redis-py        | redis.asyncio                               |
| SQLite   | sqlite3         | aiosqlite                                   |
| Web 框架 | Flask           | FastAPI、Starlette（天然 async）             |

## 附录 C：一页纸完整示例（爬虫模板）★☆☆

```python
"""并发抓取模板：限流 + 超时 + 失败容忍 + 进度输出"""
import asyncio
import time

import httpx


async def fetch(client: httpx.AsyncClient, sem: asyncio.Semaphore, url: str):
    async with sem:
        try:
            resp = await client.get(url, timeout=10)
            return {"url": url, "ok": resp.status_code == 200, "data": resp.text}
        except Exception as e:
            return {"url": url, "ok": False, "data": None, "error": str(e)}


async def main():
    urls = [f"https://httpbin.org/delay/1?page={i}" for i in range(20)]
    sem = asyncio.Semaphore(5)
    start = time.perf_counter()

    async with httpx.AsyncClient() as client:
        results = []
        for fut in asyncio.as_completed(
            [fetch(client, sem, u) for u in urls]
        ):
            r = await fut
            results.append(r)
            print(f"[{len(results)}/{len(urls)}] {'✓' if r['ok'] else '✗'} {r['url']}")

    ok = sum(r["ok"] for r in results)
    print(f"完成 {ok}/{len(urls)}，耗时 {time.perf_counter() - start:.1f}s")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 总结：五句话带走 asyncio

1. **asyncio 解决的是"等待时的浪费"**——IO 等待时切去干别的，单线程实现高并发
2. **`async def` 定义协程，`await` 等待结果**——调用协程函数不执行它，必须 await / run / create_task
3. **并发三件套**：`create_task`（立即跑）、`gather`（并发收结果）、`TaskGroup`（结构化，一损俱损）
4. **协程里绝不阻塞**：`time.sleep`/`requests` 换成异步版，实在不行 `to_thread`
5. **IO 密集用 asyncio，CPU 密集用 multiprocessing**——async 不会让计算变快
