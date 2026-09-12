# Python asyncio 核心入门（事件循环、async/await、gather、create_task、超时）

- 目标读者：会 Python 语法（函数、类、import、装饰器基础），但对并发/异步完全陌生
- 前置知识：无。第 3 节会把「阻塞、IO 密集、并发、协程、事件循环、awaitable、Task」这几个词从零讲清楚
- 学习时长：约 60~90 分钟（含动手跑代码） / 可跳过：第 2 节的「反面推演」如果你已经被同步循环慢过，可以扫一眼直接跳到第 3 节

环境：Python 3.12+（本文所有代码在 3.12 上验证语法；用到 3.11 才引入的 API 会单独标注，并给旧写法对照）

---

## 1. 它是什么（一句话 + 一句类比）

**一句话**：asyncio 是 Python 自带的一个工具箱，它让你的程序在「等外面回话」的时候，不要傻站着，而是先去干别的事，等回话到了再回来接着处理。

这里的「等外面回话」，指的就是发一个网络请求出去，然后等对方服务器返回结果——比如你调一次 LLM 的接口，可能要等好几秒。这几秒里你的程序其实什么都没做，只是在等。

**类比：一个人在餐厅里点菜**

假设你是服务员，有 10 桌客人要点菜。

- 笨办法：走到第 1 桌，记下菜单，跑去厨房，**站在灶台前盯着厨师做完这道菜**，端上桌，再去第 2 桌。10 桌就要串行做完 10 次。
- asyncio 的办法：走到第 1 桌记下菜单，交给厨房，**立刻**去第 2 桌记菜单，再交给厨房……10 桌的菜单很快全都进了厨房，然后哪桌的菜先做好，你就先端哪桌。

这个类比后面会真实对应到代码上：

| 类比里的角色 | 代码里的东西 |
| --- | --- |
| 服务员（只有一个人） | 事件循环（单线程） |
| 记菜单、端菜 | 你自己的 Python 代码 |
| 厨房做菜（你只能等） | 网络请求 / 等 LLM 返回 |
| 「交给厨房，先去下一桌」 | `await` 让出控制权 |
| 一次把 10 桌菜单都下出去 | `asyncio.gather` / `create_task` |
| 「这桌菜 30 秒还没上就撤单」 | `asyncio.timeout` |

注意一个关键点：服务员**始终只有一个人**。asyncio 不是请了 10 个服务员（那是多线程/多进程），它是让一个人不再干等。这一点第 3 节还会再强调。

---

## 2. 它解决什么问题

### 2.1 先看不用它会怎样

假设你要调 LLM 接口，对同一批 10 个问题各生成一个答案。用最朴素的同步写法：

```python
import time
import httpx  # 常用的 HTTP 客户端库，同步和异步都支持

questions = [f"第 {i} 个问题" for i in range(10)]  # 10 个待处理的问题

def ask_sync(client: httpx.Client, question: str) -> str:
    # client.post 发出请求后，这一行会一直卡住，直到服务器返回
    resp = client.post("https://api.example.com/v1/chat", json={"q": question})
    return resp.json()["answer"]

start = time.perf_counter()          # 记录开始时间，用来算总耗时
with httpx.Client() as client:       # 复用连接
    answers = []
    for q in questions:              # 挨个来：第 2 个请求必须等第 1 个回来才发出
        answers.append(ask_sync(client, q))
print(f"总耗时 {time.perf_counter() - start:.1f} 秒")
```

### 2.2 可感知的痛点

假设每次调用 LLM 要 1 秒：

- **10 个请求 × 1 秒 = 10 秒**。100 个问题就是 100 秒，1000 个就是 16 分钟——时间和数量成正比，线性增长。
- 这 10 秒里，你的 CPU 使用率几乎是 **0%**。它不是在算东西，是在等网卡。机器明明闲着。
- 如果这段代码在一个 Web 服务里（比如 FastAPI 的一个接口），那么处理这个请求的过程中，**你的服务无法响应别人**。1 个用户就能把服务拖住 10 秒。
- 加机器、换更快的 CPU，一点用都没有，因为瓶颈不在计算。

### 2.3 问题根源，一句话

**慢的不是计算，是等待；而同步代码在等待时把整个线程占着不放。**

### 2.4 经典应用场景

asyncio 适合「一次要等很多个外部结果」的活儿：

- 并发调多个 LLM / 第三方 API（本文主线场景）
- 一个请求里要同时查数据库、查缓存、调下游服务，然后把结果拼起来
- 爬取大量网页
- Web 服务器同时服务成百上千个连接（FastAPI、aiohttp 就是靠这个）
- 处理流式返回（LLM 的 SSE 逐字输出）

**不适合**的场景也要记住：如果任务是纯计算（图像处理、大矩阵运算、暴力破解），asyncio 帮不上忙，那要用多进程。原因在第 3 节讲「IO 密集 vs CPU 密集」时说清楚。

---

## 3. 读下去之前，先搞懂这些概念

这一节每个概念都是：一行定义 + 大白话 + 最小示例。别跳，第 4 节的代码全靠这些词。

### 3.1 阻塞 / 非阻塞

**定义**：一个函数调用如果在结果拿到之前不把控制权交还给你，就叫阻塞；如果它立刻返回、让你先去干别的，就叫非阻塞。

**大白话**：阻塞 = 你打电话给对方，对方说「你别挂，我去查一下」，你就得举着手机干等。非阻塞 = 对方说「查到了我回你」，你可以先挂了去做别的事。

```python
time.sleep(1)          # 阻塞：这一行期间整个线程什么都做不了
await asyncio.sleep(1) # 非阻塞：这 1 秒里事件循环去跑别的任务
```

这两行的区别是整篇文章最重要的一件事。

### 3.2 IO 密集 vs CPU 密集

**定义**：程序的时间主要花在等外部输入输出（网络、磁盘、数据库）上，叫 IO 密集；主要花在 CPU 计算上，叫 CPU 密集。

**大白话**：IO 密集 = 大部分时间在等别人；CPU 密集 = 大部分时间在自己算。

```python
resp = httpx.get(url)              # IO 密集：99% 时间在等网络
result = sum(i * i for i in range(10_000_000))  # CPU 密集：全程在算
```

**为什么要分清**：asyncio 只能优化「等」的部分。等的时候让出去干别的，这叫赚到了。但如果是在算，没有「空闲的等待」可以利用，asyncio 一点忙帮不上——反而因为单线程，一个长计算会把所有任务都堵死（第 6 节有这个坑）。调 LLM 是典型的 IO 密集，所以 asyncio 特别适合。

### 3.3 并发 vs 并行

**定义**：并发（concurrency）是多个任务交替推进、宏观上一起在跑；并行（parallelism）是多个任务在多个 CPU 核上真的同时执行。

**大白话**：并发 = 一个服务员招呼 10 桌（快速轮换）；并行 = 10 个服务员各招呼 1 桌。

```python
await asyncio.gather(task_a(), task_b())  # 并发：单线程交替执行
# 并行需要 multiprocessing / ProcessPoolExecutor，本文不涉及
```

**asyncio 提供的是并发，不是并行。** 它全程跑在**一个线程**里。10 个网络请求「同时」进行，是因为它们大部分时间都只是在等网卡，等的动作不占 CPU，所以一个线程完全招呼得过来。

### 3.4 协程（coroutine）

**定义**：用 `async def` 定义的函数叫协程函数；调用它得到的对象叫协程对象——一个「可以暂停、之后能从暂停处继续」的函数执行过程。

**大白话**：普通函数一旦开始跑，就一口气跑到 return。协程可以跑到一半举手说「我要等网络了，你们先忙」，之后再从那一行继续。

```python
async def fetch():        # async def 定义的就是协程函数
    return 42
coro = fetch()            # 注意：这一步函数体一行都没执行，只拿到一个协程对象
```

**最容易踩的点**：`fetch()` 不会执行任何代码。协程必须被 `await`（或包成 Task）才会真正跑起来。第 6 节专门讲这个坑。

### 3.5 事件循环（event loop）

**定义**：asyncio 里的调度器。它维护一堆待办任务，挑一个来跑，遇到 `await` 就把它挂起、换下一个能跑的来，某个任务等的东西到位了就恢复它。

**大白话**：就是那个唯一的服务员。所有任务都排在它手上，它不停地在任务之间来回切。

```python
asyncio.run(main())  # 创建事件循环、跑 main() 直到结束、关闭循环
```

你几乎不需要手动操作事件循环，`asyncio.run` 一行就够。知道它存在、知道它只有一个、知道它会被阻塞代码卡死就行。

### 3.6 可等待对象（awaitable）

**定义**：能放在 `await` 后面的东西。有三类：协程对象、Task、Future。

**大白话**：`await x` 的意思是「我要 x 的结果，没好之前我先让出去，好了叫我」。x 必须是这三类之一。

```python
await fetch()                      # 协程对象，可以 await
await asyncio.create_task(fetch()) # Task，可以 await
```

`await 5`、`await requests.get(url)` 都会直接报 `TypeError: object int can't be used in 'await' expression`，因为普通值和同步函数的返回值不是 awaitable。

### 3.7 任务（Task）

**定义**：把协程交给事件循环、让它**立刻进入调度队列开始跑**的包装对象。用 `asyncio.create_task(coro)` 创建。

**大白话**：`await coro` 是「我现在就要这个结果，等到为止」；`create_task(coro)` 是「你先跑起来，我待会儿再来收结果」。这个「先跑起来」是并发的关键。

```python
task = asyncio.create_task(fetch())  # 立即开始调度，不阻塞当前这行的下一行
result = await task                  # 之后再取结果
```

### 3.8 术语速查表

| 术语 | 大白话 | 对应正文 |
| --- | --- | --- |
| 阻塞 | 干等着，谁也别动 | 3.1 / 6.1 |
| 非阻塞 | 我先让开，好了叫我 | 3.1 / 4.1 |
| IO 密集 | 时间都在等外部 | 3.2 / 2.3 |
| CPU 密集 | 时间都在自己算 | 3.2 / 6.1 |
| 并发 | 一个人轮换招呼多件事 | 3.3 / 4.2 |
| 并行 | 多个人真的同时干 | 3.3（本文不涉及） |
| 协程 coroutine | 能中途暂停再续上的函数 | 3.4 / 4.1 |
| 事件循环 event loop | 唯一的调度员/服务员 | 3.5 / 6.3 |
| awaitable | 能放在 await 后面的东西 | 3.6 |
| Task | 「先跑起来，我待会收结果」 | 3.7 / 4.3 |
| `asyncio.run` | 启动整个异步世界的入口 | 4.1 |
| `gather` | 一次性并发跑一批 | 4.2 |
| `asyncio.timeout` | 超时就撤单（3.11+） | 4.3 |
| `Semaphore` | 限制同时最多几个在跑 | 4.3 / 5 |

---

## 4. 从最简单的代码示例开始

### 4.1 示例一：最小可运行版本

只保留让异步代码「活起来」的最少东西：`async def` + `await` + `asyncio.run`。

```python
import asyncio  # 标准库自带，不用安装

async def ask_llm(question: str) -> str:
    # async def：声明这是协程函数，函数体里才允许出现 await
    print(f"发出请求：{question}")
    # 这里用 sleep 模拟「等 LLM 服务器返回」的那 1 秒网络等待。
    # 真实代码里这一行会是 await client.post(...)，作用相同：让出控制权去等
    await asyncio.sleep(1)
    print(f"收到回答：{question}")
    return f"{question} 的答案"

async def main() -> None:
    # 在协程里调另一个协程，必须 await，否则函数体不会执行
    answer = await ask_llm("什么是 asyncio")
    print(answer)

# asyncio.run 是同步世界进入异步世界的唯一入口：
# 它建好事件循环、跑完 main()、再关掉循环。整个程序只调一次
asyncio.run(main())
```

输出：

```
发出请求：什么是 asyncio
收到回答：什么是 asyncio
什么是 asyncio 的答案
```

**逐段解读**

- `async def ask_llm(...)`：这段负责「一次请求」。它是协程函数，所以里面能写 `await`。单独写 `ask_llm("x")` 只会得到一个协程对象，函数体一行都不跑。
- `await asyncio.sleep(1)`：这一段是全文的核心动作。执行到这里，`ask_llm` 暂停，把控制权还给事件循环；1 秒后事件循环把它恢复，从这一行的下一行继续。
- `async def main()`：约定俗成的总入口。异步代码只能被异步代码 `await`，所以你需要一个 `main` 当「异步世界的第一层」。
- `asyncio.run(main())`：这一行是同步的。它把 `main()` 这个协程对象交给一个新建的事件循环跑到结束。关键变量就是它的返回值——`main` 的返回值会被 `asyncio.run` 原样返回。
- 执行流程：`asyncio.run` 建循环 → 跑 `main` → `main` 遇到 `await ask_llm(...)` 挂起 → 跑 `ask_llm` → 遇到 `await asyncio.sleep(1)` 挂起，此刻循环手上没有别的任务，只能空转等 1 秒 → 恢复 → 逐层返回 → 循环关闭。

**单独展开：`async` / `await` 到底在做什么**

- `async def f(): ...` 定义的不是普通函数。调用 `f()` 得到的是一个**协程对象**，可以理解成「一份写好但还没开始执行的执行计划」。
- `await x` 做两件事：① 要求 x 的结果；② **在等结果的这段时间里，把控制权交回事件循环**，让循环去跑其他任务。

等价的普通写法对照——同步版本长这样：

```python
def ask_llm_sync(question: str) -> str:
    time.sleep(1)                 # 干等，整个线程冻住
    return f"{question} 的答案"
answer = ask_llm_sync("什么是 asyncio")  # 直接拿到结果
```

两者在**单个请求**时耗时完全一样（都是 1 秒），代码还更长。所以：**只有一个请求时，asyncio 没有任何好处。** 它的价值从下一个示例开始体现。

**这一步你得到了什么**：你会写、会跑最小的异步程序，并且知道 `await` 的语义是「让出控制权去等」，而不是「变快」。

### 4.2 示例二：加一个真实需求——并发跑 10 个请求

需求：一次问 10 个问题，总耗时要接近 1 秒而不是 10 秒。

```python
import asyncio
import time

async def ask_llm(question: str) -> str:
    await asyncio.sleep(1)              # 模拟 1 秒的网络往返（等 LLM 返回）
    return f"{question} 的答案"

async def main() -> None:
    questions = [f"问题{i}" for i in range(10)]

    # 顺序版：写法上仍然是 async，但一个 await 完才发下一个，等于白用异步
    start = time.perf_counter()
    for q in questions:
        await ask_llm(q)                # 每次都等它彻底完成再继续下一轮
    print(f"顺序：{time.perf_counter() - start:.2f} 秒")

    # 并发版：先把 10 个协程对象放进一个列表（此时都还没开始跑）
    coros = [ask_llm(q) for q in questions]
    start = time.perf_counter()
    # gather 把这一批一起交给事件循环，它们同时进入「等待网络」状态
    # 返回值是一个列表，顺序和传入顺序一致（不是谁先完成谁在前）
    answers = await asyncio.gather(*coros)
    print(f"并发：{time.perf_counter() - start:.2f} 秒，拿到 {len(answers)} 个答案")

asyncio.run(main())
```

输出：

```
顺序：10.03 秒
并发：1.00 秒，拿到 10 个答案
```

**逐段解读**

- 顺序版这段是**反面教材**，专门用来证明：写了 `async` 不等于并发。并发的前提是「多个任务同时处于等待状态」，而 `for` 里的 `await` 保证了任何时刻只有一个在等。
- <mark style="background: #BBFABBA6;">`coros = [ask_llm(q) for q in questions]`：关键变量。这行只是造了 10 个协程对象，一次请求都还没发出去。</mark>
- <mark style="background: #BBFABBA6;">`await asyncio.gather(*coros)`：`*` 是解包，因为 `gather` 接受的是多个位置参数而不是一个列表。执行到这里，10 个协程被同时启动，几乎同时打到 `await asyncio.sleep(1)` 然后一起挂起；1 秒后一起被唤醒。所以总耗时 ≈ 最慢的那一个，不是所有耗时之和。</mark>
- 执行流程一句话：**10 个「等 1 秒」重叠在了一起。**

**单独展开：`gather` 的 `return_exceptions`**

<mark style="background: #BBFABBA6;">默认 `return_exceptions=False`。此时如果某个协程抛异常，`gather` 会立刻把这个异常抛给你——但**其余协程不会被自动取消，仍在后台继续跑**，你也拿不到它们的结果。</mark>

```python
results = await asyncio.gather(*coros, return_exceptions=True)
# 改成 True 后：不抛异常，出错的那一项在结果列表里就是异常对象本身
for q, r in zip(questions, results):
    if isinstance(r, Exception):        # 必须自己判断，否则会把异常当答案用
        print(f"{q} 失败：{r!r}")
    else:
        print(f"{q} 成功：{r}")
```

<mark style="background: #BBFABBA6;">批量调 LLM 时几乎总是要 `return_exceptions=True`：10 个问题里 1 个超时，不该让另外 9 个的结果全丢掉。</mark>

**这一步你得到了什么**：你能用 `gather` 把 N 个网络请求的总耗时从「相加」变成「取最大」，并且知道单个失败时怎么保住其余结果。

### 4.3 示例三：贴近真实项目的样子

加上真实项目一定会有的四件事：`create_task`、超时、异常处理、并发上限。

```python
import asyncio
import time

MAX_CONCURRENCY = 3      # 同时最多几个请求在飞。LLM 接口通常有速率限制，必须自己压住
REQUEST_TIMEOUT = 2.0    # 单个请求最多等几秒，超了就放弃这一个

async def call_llm(question: str, delay: float) -> str:
    # delay 参数只为演示用：模拟不同问题的响应快慢不同
    await asyncio.sleep(delay)                 # 模拟网络往返
    if "坏" in question:
        raise ValueError("上游返回了 500")      # 模拟服务端报错
    return f"{question} 的答案"

async def ask_one(question: str, delay: float, sem: asyncio.Semaphore) -> str:
    # async with sem：拿到一个「通行证」才继续，拿不到就在这里等着。
    # 这样同时进入下面代码块的最多只有 MAX_CONCURRENCY 个
    async with sem:
        # asyncio.timeout 需要 Python 3.11+；超时会抛 TimeoutError
        async with asyncio.timeout(REQUEST_TIMEOUT):
            return await call_llm(question, delay)

async def main() -> None:
    questions = [("问题A", 0.5), ("问题B", 1.0), ("坏问题", 0.3), ("慢问题", 5.0)]
    sem = asyncio.Semaphore(MAX_CONCURRENCY)   # 计数器，初值就是允许的并发数

    # create_task 会立刻把协程排进事件循环开始跑，不等它完成就返回 Task 对象。
    # 这个列表推导跑完的瞬间，4 个任务已经在并发推进了
    tasks = [asyncio.create_task(ask_one(q, d, sem), name=q) for q, d in questions]

    start = time.perf_counter()
    # return_exceptions=True：让失败的任务把异常「作为结果」交回来，不中断其他任务
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for (q, _), r in zip(questions, results):
        if isinstance(r, TimeoutError):        # 超时单独处理，通常是重试的候选
            print(f"[超时] {q}")
        elif isinstance(r, Exception):         # 其余业务/网络错误
            print(f"[失败] {q}: {r!r}")
        else:
            print(f"[成功] {q}: {r}")
    print(f"总耗时 {time.perf_counter() - start:.2f} 秒")

asyncio.run(main())
```

输出（顺序与传入顺序一致）：

```
[成功] 问题A: 问题A 的答案
[成功] 问题B: 问题B 的答案
[失败] 坏问题: ValueError('上游返回了 500')
[超时] 慢问题
总耗时 2.00 秒
```

**逐段解读**

- 两个常量放在最上面，是真实项目的习惯：并发数和超时是最常调的两个旋钮，别埋进函数体里。
- `ask_one` 这段负责「一个请求的完整生命周期」：先排队拿并发许可，再套超时，最后真正调用。**顺序很重要**：`sem` 在外、`timeout` 在内，意味着超时只计算「真正发请求」的时间，不把排队等许可的时间算进去。如果反过来，一个任务可能因为排队太久就被判超时，而它其实还没发出去。
- `sem = asyncio.Semaphore(3)`：关键变量。它必须在 `main` 里创建、传给每个任务共用；如果在 `ask_one` 内部创建，每个任务各有一个，限流就失效了。
- `tasks = [asyncio.create_task(...)]`：这行之后 4 个任务全都在跑了。
- `gather(*tasks, return_exceptions=True)` + `isinstance` 分类：这是批量调用的标准收尾。先判 `TimeoutError` 再判 `Exception`，因为前者是后者的子类，顺序颠倒的话超时会被归进「失败」分支。
- 为什么总耗时是 2 秒：`慢问题` 要 5 秒，但 2 秒超时把它砍掉了，所以上限就是超时值。

**单独展开一：`create_task` 的调度时机**

<mark style="background: #BBFABBA6;">`create_task(coro)` 立刻把协程登记进事件循环，但**并不会马上执行它的第一行代码**——真正开始执行要等当前这段代码遇到下一个 `await`、把控制权交回循环之后。</mark>

```python
task = asyncio.create_task(call_llm("x", 1))  # 已排队，但函数体还没开始跑
print("这行先打印")                            # 当前代码没让出控制权，任务还在等
await asyncio.sleep(0)                        # 让出一次，任务才真正开始
```

对比 `await`：

```python
await call_llm("x", 1)                    # 现在就跑，跑完为止，期间我不往下走
t = asyncio.create_task(call_llm("x", 1)) # 你先跑，我继续往下走，待会儿 await t 收结果
```

**并发的本质就是这句话**：先把所有任务都 `create_task` 出去，再统一 `await`。`gather` 内部帮你做了同样的事——传给 `gather` 的协程会被自动包装成 Task，所以示例二里不写 `create_task` 也能并发。

另外注意：`create_task` 返回的 Task 你**必须自己拿变量存住**（这里存进了 `tasks` 列表）。如果不存，它可能在跑完之前被垃圾回收掉，任务莫名消失。

**单独展开二：超时的两种写法**

<mark style="background: #BBFABBA6;">`asyncio.timeout` 是 **Python 3.11+** 才有的上下文管理器。3.10 及更早要用 `asyncio.wait_for`：</mark>

```python
# 3.11+ 推荐：上下文管理器，可以给「一整段代码」加超时，包含多次调用
async with asyncio.timeout(2.0):
    a = await call_llm("q1", 0.5)
    b = await call_llm("q2", 0.5)     # 两次调用合起来不超过 2 秒

# 旧写法（所有版本可用）：只能包住单个 awaitable
a = await asyncio.wait_for(call_llm("q1", 0.5), timeout=2.0)
```

两者超时都抛 `TimeoutError`（Python 3.11 起 `asyncio.TimeoutError` 已是内置 `TimeoutError` 的别名，写哪个都行，新代码直接写 `TimeoutError`），并且都会**取消**里面正在跑的任务，不会让它在后台继续偷跑。

顺带一提 `asyncio.TaskGroup`（也是 **3.11+**）：它是 `gather` 的现代替代，好处是任何一个任务失败时会自动取消同组其他任务。批量调 LLM 时我们通常**不**要这个行为（希望其余的照常完成），所以本文主线仍用 `gather` + `return_exceptions=True`。

**这一步你得到了什么**：你有了一个能真实上线的骨架——并发、限流、超时、逐项异常分类，四件事齐了。

---

## 5. 最小可用的实际用法（生产场景模板）

<mark style="background: #BBFABBA6;">把示例三换成真实的 HTTP 调用。这段可以直接复制进项目，改掉 URL 和 payload 就能用。依赖：`pip install httpx`（`httpx.AsyncClient` 是异步版客户端，`requests` 没有异步支持，不能用）。</mark>

```python
import asyncio
import os
from dataclasses import dataclass

import httpx

API_URL = "https://api.example.com/v1/chat/completions"
MAX_CONCURRENCY = 5          # 必须：按上游速率限制调整
REQUEST_TIMEOUT = 30.0       # 必须：LLM 响应慢，30 秒是常见起点


@dataclass
class AskResult:              # 可选：用 dataclass 让调用方好取值，直接返回元组也行
    question: str
    answer: str | None
    error: str | None


async def _ask_one(
    client: httpx.AsyncClient,
    sem: asyncio.Semaphore,
    question: str,
) -> AskResult:
    async with sem:                                   # 必须：压住并发，否则会被限流/封
        try:
            async with asyncio.timeout(REQUEST_TIMEOUT):   # 必须（3.11+）
                resp = await client.post(
                    API_URL,
                    json={"model": "some-model", "messages": [{"role": "user", "content": question}]},
                    headers={"Authorization": f"Bearer {os.environ['LLM_API_KEY']}"},  # 必须：密钥走环境变量，别写进代码
                )
                resp.raise_for_status()               # 必须：4xx/5xx 转成异常，否则会把错误页当答案
                data = resp.json()
                return AskResult(question, data["choices"][0]["message"]["content"], None)
        except TimeoutError:
            return AskResult(question, None, "timeout")        # 可选：也可以让异常抛出去，交给 gather 收
        except httpx.HTTPError as exc:                          # 可选：分类越细，上层重试策略越好写
            return AskResult(question, None, f"http: {exc!r}")


async def ask_many(questions: list[str]) -> list[AskResult]:
    sem = asyncio.Semaphore(MAX_CONCURRENCY)          # 必须：在这里建，所有任务共用一个
    # 必须：AsyncClient 复用连接池，比每次请求新建客户端快很多
    async with httpx.AsyncClient() as client:
        tasks = [asyncio.create_task(_ask_one(client, sem, q)) for q in questions]
        # return_exceptions=True 必须：一个炸了不能拖垮整批
        results = await asyncio.gather(*tasks, return_exceptions=True)

    out: list[AskResult] = []
    for q, r in zip(questions, results):
        if isinstance(r, BaseException):              # 必须：兜住上面没 catch 到的意外异常
            out.append(AskResult(q, None, f"unexpected: {r!r}"))
        else:
            out.append(r)
    return out


if __name__ == "__main__":
    qs = ["用一句话解释事件循环", "gather 和 create_task 的区别"]
    for item in asyncio.run(ask_many(qs)):
        print(item)
```

**哪些行是骨架，哪些可以删**

必须保留：`Semaphore`（含 `async with sem`）、`asyncio.timeout`、`raise_for_status`、`AsyncClient` 复用、`gather(..., return_exceptions=True)`、最后那圈 `isinstance` 兜底、密钥从环境变量取。

可以删或替换：`AskResult` dataclass（换成元组或 dict）、`except` 的细分（合并成一个 `except Exception`）、`name=` 参数、`__main__` 演示块。

**真实项目通常还会配什么**（属进阶，第一版可以先不做）

- **重试**：对 `timeout`、429、5xx 做指数退避重试，常用 `tenacity` 库
- **日志**：每个请求记 question 摘要、耗时、状态码；注意别把 API key 和用户完整输入打进日志
- **指标**：成功率、P95 耗时、被限流次数，接 Prometheus 之类
- **成本控制**：累计 token 数
- 在 FastAPI 里用的话，路由函数写成 `async def`，直接 `await ask_many(...)`，不要用 `asyncio.run`（见 6.3）

---

## 6. 常见坑与易错点

### 6.1 在 `async def` 里调用阻塞函数（最严重、最常见）

**现象**：明明用了 asyncio 和 `gather`，10 个请求还是花了 10 秒，并发完全没生效。没有任何报错。

```python
async def bad(question: str) -> str:
    resp = requests.get(url)   # 错：requests 是同步库，这一行阻塞整个事件循环
    time.sleep(1)              # 错：同理，把唯一的服务员冻住 1 秒
    return resp.text
```

**原因**：事件循环是单线程的。`requests.get` / `time.sleep` 不会让出控制权，事件循环在这期间无法切换到任何其他任务——所有并发都退化成排队。`async def` 只是声明，它不会把内部的同步调用变成异步。

**解决**：

```python
async def good(client: httpx.AsyncClient, question: str) -> str:
    resp = await client.get(url)   # 用异步库：httpx.AsyncClient / aiohttp
    await asyncio.sleep(1)         # 用 asyncio 版的 sleep
    return resp.text

# 如果某个阻塞调用无法替换（老 SDK、CPU 密集计算），丢到线程池里：
result = await asyncio.to_thread(blocking_sdk_call, arg)  # 3.9+
```

记住这条规则：**`async def` 函数体里出现的每个耗时操作，前面都应该有 `await`。** 看到没带 `await` 的耗时调用，就是在阻塞事件循环。

### 6.2 忘记 `await`

**报错原文**：`RuntimeWarning: coroutine 'ask_llm' was never awaited`

**现象**：函数好像根本没执行，打印出来的是 `<coroutine object ask_llm at 0x...>` 而不是结果。

```python
answer = ask_llm("hi")     # 错：只拿到协程对象，函数体一行没跑
print(answer)              # <coroutine object ask_llm at 0x000001...>
answer = await ask_llm("hi")  # 对
```

**原因**：调用协程函数只是创建执行计划，`await`（或 `create_task`）才是启动它。

**解决**：`await` 它。类型检查器（mypy / pyright）能自动揪出这类错误，建议开起来。顺带一提：把 `RuntimeWarning` 当错误处理（`python -W error::RuntimeWarning`）能让这个问题当场暴露——本仓库测试规范里专门要求检查这条 warning，就是防止「协程没 await 的假通过测试」。

### 6.3 在已经运行的事件循环里再调 `asyncio.run`

**报错原文**：`RuntimeError: asyncio.run() cannot be called from a running event loop`

**现象**：在 FastAPI 的 `async def` 路由、Jupyter Notebook 单元格里调 `asyncio.run(...)` 直接报错。

```python
@app.get("/ask")
async def ask_route():
    return asyncio.run(ask_many(qs))   # 错：这里已经在事件循环里了
    return await ask_many(qs)          # 对：直接 await
```

**原因**：`asyncio.run` 的职责是「新建一个事件循环并跑到结束」，一个线程里同时只能有一个运行中的循环。FastAPI/Jupyter 已经替你开好了循环。

**解决**：只在程序最外层（`if __name__ == "__main__":` 或脚本入口）调一次 `asyncio.run`；已经在异步上下文里就直接 `await`。

### 6.4 Semaphore 建错位置，限流失效

**现象**：设了 `Semaphore(5)`，上游还是返回 `429 Too Many Requests`。

```python
async def ask_one(q):
    sem = asyncio.Semaphore(5)   # 错：每个任务各建一个，每个都有 5 个额度
    async with sem:
        ...
```

**原因**：限流靠的是**共用同一个计数器**。各自新建等于没限。

**解决**：在 `main` / 批量函数里建一次，作为参数传给每个任务（见第 5 节）。

### 6.5 和直觉相反的一条

**`gather` 的返回顺序和完成顺序无关。** 就算第 10 个请求最先返回，`results[9]` 依然是第 10 个的结果——`gather` 按你传入的顺序对齐输出。所以用 `zip(questions, results)` 配对是安全的，不需要在结果里塞 id 来对号。

反过来，如果你**想要**「谁先完成先处理谁」（比如流式往前端推），`gather` 就不合适，要用 `asyncio.as_completed`：

```python
for coro in asyncio.as_completed(tasks):   # 按完成顺序逐个 yield
    result = await coro
    print("先到先处理：", result)
```

另一条反直觉：`asyncio` 不会让**单个**请求变快一毫秒。它优化的只是「多个等待重叠起来」。单请求场景用同步代码更简单。

---

## 7. 验证你学会了（自测题）

### 第 1 题：概念复述

不看文档，用自己的话回答：

1. asyncio 是什么？用一句不含「协程、事件循环、异步」这些术语的话说清楚。
2. 它解决的核心问题是什么？为什么加 CPU 或换更快的机器解决不了？
3. 并发和并行的区别是什么？asyncio 提供的是哪一个？
4. 为什么调 LLM 接口适合用 asyncio，而给一万张图片做缩放不适合？

> 答案在第 1 节、第 2 节、第 3.2 / 3.3 节。

### 第 2 题：默写最小示例

关掉本文，写出一个能跑的程序：定义一个协程 `fetch(n)`，内部用 `asyncio.sleep(0.5)` 模拟网络等待并返回 `n * 2`；在 `main` 里并发跑 `n = 0..4`，打印结果列表和总耗时。要求总耗时约 0.5 秒而不是 2.5 秒。

自查三点：入口是不是只有一个 `asyncio.run`？有没有漏掉 `await`？传给 `gather` 时有没有忘记 `*` 解包？

> 答案在第 4.1、4.2 节。

### 第 3 题：组合应用

在第 2 题基础上改造成一个真实场景：20 个问题要调 LLM，要求

- 同时最多 4 个请求在飞
- 单个请求超过 3 秒就放弃它，但不影响其他请求
- 最后要能分别打印出「成功了几个 / 超时了几个 / 其他错误几个」
- 其中第 7 个问题会抛 `ValueError`，程序不能因此崩溃

想清楚这几个问题再动手：`Semaphore` 该建在哪一层？`timeout` 放在 `async with sem` 里面还是外面，为什么？`return_exceptions` 该设什么？判断异常类型时 `TimeoutError` 和 `Exception` 谁先判？

> 答案在第 4.3 节（结构与顺序）、第 5 节（生产模板）、第 6.4 节（Semaphore 位置）。

**做完第 3 题，再回到第 6.1 节**：把你代码里的 `await asyncio.sleep(...)` 改成 `time.sleep(...)`，跑一遍，看总耗时怎么变。这个对比会让「阻塞事件循环」这件事彻底刻进记忆里。

