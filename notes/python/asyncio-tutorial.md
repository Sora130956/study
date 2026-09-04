# asyncio 零基础教程（FastAPI 视角）

> **来源:** https://fastapi.tiangolo.com/async/ （官方「Concurrency and async / await」精读）+ https://docs.python.org/3/library/asyncio-task.html
> **对应计划周次:** 第 1 周 · 周二 asyncio 核心（事件循环、`async/await`、`gather`、`create_task`、超时）

## 核心理解

asyncio 做的事一句话概括：**让一个线程在等待 IO（网络、磁盘、数据库）的间隙去干别的活，而不是干等**。

Web 服务器的大部分时间都在「等」：等客户端把请求发过来、等数据库返回、等 LLM API 吐字。同步写法下，一个请求在等 LLM 响应的 3 秒里，整个服务器什么都干不了；异步写法下，这 3 秒可以处理几百个其他请求。**这就是 FastAPI 性能对标 NodeJS / Go 的原因，也是 AI 应用必须异步的原因——LLM 调用全是高延迟 IO**。

心智模型记住三个词：

- **事件循环（event loop）**：一个单线程的调度器，轮流推进所有任务。谁 `await` 了，谁就让出控制权，循环去推进别人；等 IO 好了再回来继续。
- **协程（coroutine）**：`async def` 定义的函数。调用它**不会执行**，只是造出一个协程对象；交给事件循环（用 `await` 或 `create_task`）才真正跑。
- **await = 让出点**：代码里每个 `await` 都是「这里要等 IO，先让别人跑」的标记。**没有 `await` 的长代码段会独占事件循环**，这就是「`async def` 里不能调阻塞的 `requests.get`」的答案（详见第 7 节）。

并发（concurrency）≠ 并行（parallelism）。asyncio 是并发：一个线程来回切换，适合**大量等待**的 IO 密集场景。并行是多核同时算，适合 CPU 密集场景（那是 `multiprocessing` 的事，不属于本周）。

## 关键点

### 1. async def 与协程：调了不等于跑了

```python
import asyncio

async def main():
    print('hello')
    await asyncio.sleep(1)   # 等 1 秒，期间事件循环可以去干别的
    print('world')

asyncio.run(main())   # 程序入口：创建事件循环，跑 main()，跑完关闭
```

三个必记的事实：

1. **直接调用 `async def` 函数只会得到一个协程对象，不会执行**：

```python
main()   # <coroutine object main at 0x...> —— 什么都没打印
         # 还会有 RuntimeWarning: coroutine 'main' was never awaited
```

2. **`await` 只能写在 `async def` 里面**。普通 `def` 里写 `await` 直接语法错误。

3. **`asyncio.run()` 是顶层入口，一个程序只调一次**。它内部做的事：创建事件循环 → 把 `main()` 包成 Task 跑 → 结束后关闭循环。在 FastAPI 里不用你调，uvicorn 替你做了。

### 2. 事件循环的心智模型：串行 await ≠ 并发

官方文档这个例子是理解一切的钥匙。两个 `say_after` 顺序 await：

```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    print(f"started at {time.strftime('%X')}")
    await say_after(1, 'hello')   # 等这 1 秒过完，才走下一行
    await say_after(2, 'world')   # 再等 2 秒
    print(f"finished at {time.strftime('%X')}")

asyncio.run(main())
# started at 17:13:52
# hello
# world
# finished at 17:13:55   ← 总共 3 秒，串行
```

**直接 await 是串行的**：一个等完才等下一个。想让两个等待同时进行，必须用 `create_task` 或 `gather` 把它们「登记」到事件循环上（下两节）。

⚠️ `await asyncio.sleep(1)` 和 `time.sleep(1)` 天差地别：前者让出控制权（别人能跑），后者把**整个事件循环**冻住 1 秒（所有人都卡住）。写异步代码时，标准库/第三方库必须选 async 版本：`asyncio.sleep`、`httpx.AsyncClient`、`asyncpg`——用错成同步版，并发能力直接归零。

### 3. create_task：把协程登记成并发任务

`asyncio.create_task()` 把协程包成 **Task** 并**立即调度**（不等 await 就开始跑），返回值是个 Task 对象：

```python
async def main():
    task1 = asyncio.create_task(say_after(1, 'hello'))   # 登记完立刻开跑
    task2 = asyncio.create_task(say_after(2, 'world'))   # 这个也同时开跑

    print(f"started at {time.strftime('%X')}")
    await task1   # 等 task1 结束（此时 task2 已经在跑了）
    await task2
    print(f"finished at {time.strftime('%X')}")

asyncio.run(main())
# finished at ... 比串行版快 1 秒 —— 总耗时 ≈ 最长的那个任务（2 秒）
```

⚠️ 两个坑：

1. **`create_task` 必须在「正在运行的事件循环」里调**（也就是在 `async def` 里）。模块顶层直接调会抛 `RuntimeError: no running event loop`。
2. **事件循环对 Task 只持弱引用**。`asyncio.create_task(coro())` 不存变量的话，任务可能被垃圾回收、中途消失。要么存住并 `await`，要么官方推荐的「发射后不管」写法：

```python
background_tasks = set()

task = asyncio.create_task(some_coro())
background_tasks.add(task)                              # 强引用防 GC
task.add_done_callback(background_tasks.discard)        # 完成后自动移除
```

### 4. gather：最常用的并发收口

任务一多，逐个 `create_task` + `await` 太啰嗦。`asyncio.gather` 一把梭：**并发跑一堆 awaitable，按传入顺序返回结果列表**：

```python
async def main():
    results = await asyncio.gather(
        say_after(1, 'hello'),
        say_after(2, 'world'),
    )
    # results 是列表，顺序和传入顺序一致（不是完成顺序）
```

要点：

- 传入协程会自动包成 Task，不用手动 `create_task`。
- **返回顺序 = 传入顺序**，与谁先跑完无关。按序对应结果这点在批处理接单场景极其重要（第 1000 条输入对应 results[999]）。
- 默认任何一个任务抛异常，gather 立刻把异常抛给你（其余任务继续在后台跑）。想让「一个挂了不拖累全局」，用 `return_exceptions=True`，异常会作为元素出现在结果列表里：

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        print(f"失败: {r}")   # 记录失败项，成功的照常入库
    else:
        process(r)
```

批量调 LLM / 批量抓 URL 时，`return_exceptions=True` 是常态——1000 条里挂 3 条不该让整批重来。

### 5. 超时：wait_for 与 asyncio.timeout

外部调用必须设超时，否则一个永远不响应的 API 会永久占住一个任务。

Python 3.11+ 推荐 `asyncio.timeout` 异步上下文管理器：

```python
async def main():
    try:
        async with asyncio.timeout(5):        # 5 秒总预算
            await long_running_call()
    except TimeoutError:
        print("超时了，走降级逻辑")
```

老写法（3.11 之前）是 `asyncio.wait_for`，给单个 awaitable 设超时：

```python
try:
    result = await asyncio.wait_for(long_running_call(), timeout=5)
except TimeoutError:          # 3.11+ 统一用内置 TimeoutError（旧版是 asyncio.TimeoutError）
    ...
```

超时的机理是**取消（cancellation）**：时间一到，事件循环往任务里抛 `CancelledError`。所以协程里如果有清理逻辑（关连接、写半成品的标记），用 `try/finally` 包住保证执行；**不要裸 `except` 吞掉 `CancelledError`**，否则超时和 TaskGroup 会行为异常。

### 6. TaskGroup：create_task 的现代替代品（3.11+）

管理一组相关任务时，官方现在推荐 `TaskGroup`——自动等全部完成、一个失败自动取消其余的、异常打包成 `ExceptionGroup` 抛给你：

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(say_after(1, 'hello'))
        task2 = tg.create_task(say_after(2, 'world'))
    # 走出 async with 时，所有任务保证已结束（隐式 await）
    print(task1.result(), task2.result())
```

对比 `gather` 的选择心法：

- 只是「并发跑一把收结果」→ `gather` 够用，写起来最短。
- 需要结构化并发保证（子任务失败要取消兄弟任务、要拿 Task 对象做取消/查状态）→ `TaskGroup`。
- 需要「发射后不管」的单个后台任务（如 FastAPI 的 BackgroundTasks 之外的场景）→ `create_task` + 第 3 节的 set 引用技巧。

### 7. FastAPI 的规则：async def 还是 def

官方文档的决策树（背下来，本周验证点）：

| 你的路径操作里要干什么 | 怎么声明 |
|------------------------|----------|
| 调用支持 `await` 的异步库（httpx.AsyncClient、asyncpg、Motor） | `async def` |
| 调用不支持 await 的同步库（requests、大部分老牌数据库驱动） | 普通 `def` |
| 纯计算，不跟外部通信 | `async def` |
| 拿不准 | 普通 `def` |

```python
@app.get('/')
async def read_results():
    results = await some_async_library()   # 异步库 → async def + await
    return results

@app.get('/legacy')
def read_legacy():
    results = some_sync_library()          # 同步库 → 普通 def
    return results
```

为什么同步库要用普通 `def` 而不是 `async def`：**FastAPI 会把普通 `def` 的路径操作放到外部线程池里跑再 await 回来**，不阻塞事件循环；而 `async def` 是直接在事件循环线程上跑的——里面一旦写了 `requests.get()` 或 `time.sleep()`，**整个事件循环被卡住，服务器上所有请求一起停摆**。

一句话总结验证点：**`async def` 不是「这个函数变快了」，而是「这个函数承诺只在 await 处让出控制权」。在里面调阻塞函数等于违背承诺，惩罚是全服务器卡死。**

`def` 和 `async def` 可以在同一个 app 里随便混用，FastAPI 会各按各的正确方式处理。

### 8. 下午实战：httpx.AsyncClient 并发 vs 同步耗时对比

计划下午的任务，先把骨架码好：

```python
import asyncio
import time

import httpx
import requests

URLS = ["https://httpbin.org/delay/1"] * 10   # 每个请求固定延迟 1 秒

def fetch_all_sync():
    for url in URLS:
        requests.get(url)                     # 一个一个来

async def fetch_one(client, url):
    r = await client.get(url)
    return r.status_code

async def fetch_all_async():
    async with httpx.AsyncClient() as client:
        tasks = [fetch_one(client, url) for url in URLS]
        return await asyncio.gather(*tasks)

if __name__ == "__main__":
    t0 = time.perf_counter()
    fetch_all_sync()
    print(f"同步: {time.perf_counter() - t0:.1f}s")     # ≈ 10 秒

    t0 = time.perf_counter()
    asyncio.run(fetch_all_async())
    print(f"异步: {time.perf_counter() - t0:.1f}s")     # ≈ 1 秒
```

预期结论：**同步总耗时 ≈ 各请求之和，异步总耗时 ≈ 最慢的那个请求**。10 个 1 秒的请求，同步 10 秒，异步 1 秒出头。这个数量级差异就是接 AI 单时批量处理文本（W3 的 `asyncio.Semaphore` 控并发）的地基。

⚠️ `AsyncClient` 要复用（连接池），用 `async with` 包一层或存成长期对象，别每个请求 new 一个。

## 和 FastAPI / AI 接单的关系

- **W1 周三 SSE 流式**：`StreamingResponse` 里就是一个 async 生成器，逐 chunk `yield`——本质是「每次 yield 都让出控制权」，事件循环才有空把数据推给客户端。
- **W2 流式调 LLM**：SDK 的 `stream=True` 在 async 模式下是 async iterator，逐个 token await 出来。
- **W3 批量处理 1000 条文本**：`gather` + `asyncio.Semaphore(20)` 限并发，就是「高并发但不被限流」的标准解法。
- **W2 周二重试与超时**：LLM API 的 429/超时处理，全靠本节 `asyncio.timeout` + 指数退避循环。
- 已有资产 `freelancer-job-analyzer` 的 `RateLimiter` 和 TTL 缓存就是建立在 asyncio 上的，本周学完回去重读一遍它的代码，会有全新理解。

## 动手练习（建议全做）

1. 跑第 2 节的串行版和第 3 节的并发版，亲眼确认 3 秒 vs 2 秒的差别；把其中一个 `asyncio.sleep` 换成 `time.sleep`，观察并发版退化成串行——理解「阻塞调用杀死并发」。
2. 用 `gather` 并发跑 5 个不同延迟的 `say_after`，打印结果列表，确认顺序与传入顺序一致。
3. 故意让 gather 里一个任务抛异常，分别用默认模式和 `return_exceptions=True` 跑，对比行为。
4. 给一个永远睡 60 秒的协程套 `asyncio.timeout(2)`，确认 2 秒后抛 `TimeoutError`。
5. 完成第 8 节的 10 URL 对比实验，记录两组耗时数字（这是周五复盘的素材）。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| event loop | n. | 事件循环，单线程调度器，轮流推进所有异步任务 |
| coroutine | n. | 协程；`async def` 定义的函数（或调用它得到的协程对象） |
| await | v. | 挂起当前协程让出控制权，等结果回来后继续 |
| awaitable | n. | 可以放在 `await` 后面的对象（协程 / Task / Future） |
| Task | n. | 被 `create_task` 包装后调度运行的协程，可取消、可查状态 |
| Future | n. | 低层 awaitable，代表「未来某时刻的结果」，应用层一般不用直接碰 |
| asyncio.run() | v. | 异步程序入口：建事件循环、跑顶层协程、关闭循环 |
| create_task() | v. | 把协程包成 Task 并立即并发调度 |
| gather() | v. | 并发跑一组 awaitable，按传入顺序返回结果列表 |
| return_exceptions | n. | gather 参数：True 时异常作为结果元素返回而不抛出 |
| TaskGroup | n. | 3.11+ 结构化并发：一组任务同生共死，失败自动取消兄弟任务 |
| asyncio.timeout() | n. | 3.11+ 超时上下文管理器，超时抛 `TimeoutError` |
| wait_for() | v. | 给单个 awaitable 设超时的老写法 |
| CancelledError | n. | 任务被取消/超时时抛进协程的异常，不该被裸 except 吞掉 |
| cancellation | n. | 取消：事件循环通过向任务抛 `CancelledError` 来终止它 |
| I/O bound | adj. | IO 密集型：耗时主要在等网络/磁盘/数据库，异步的主场 |
| CPU bound | adj. | CPU 密集型：耗时主要在计算，异步帮不上忙，要并行 |
| concurrency | n. | 并发：单线程交替推进多任务，靠等待间隙干活 |
| parallelism | n. | 并行：多核真正同时执行，Python 里靠多进程 |
| blocking call | n. | 阻塞调用（如 `requests.get`、`time.sleep`），冻住整个事件循环 |
| threadpool | n. | 线程池；FastAPI 把普通 `def` 路径操作放这里跑以保护事件循环 |
| AsyncClient | n. | httpx 的异步客户端，async 代码里的 HTTP 入口 |
