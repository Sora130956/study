# httpx 异步客户端入门（AsyncClient 并发请求、连接池复用、超时配置、与同步写法耗时对比）

- 目标读者：熟悉 `requests`（会用 `requests.get` / `post` / `.json()` / headers），但没用过 httpx，也刚接触 async
- 前置知识：见第 3 节。会把「同步阻塞 vs 异步非阻塞、`async with`、连接池、Keep-Alive、超时四阶段、`Response`、`raise_for_status`」讲清楚；asyncio 本身的细节（事件循环、`gather`、`Task`）另见同目录 `asyncio-入门.md`
- 学习时长：约 60~80 分钟（含动手跑代码） / 可跳过：第 2 节是反面推演，如果你已经被同步 for 循环慢过，扫一眼直接去第 3 节

环境：Python 3.12+，httpx 0.28.1（本文所有默认值、异常名都对着这个版本的源码核对过）

---

## 1. 它是什么（一句话 + 一句类比）

**一句话**：httpx 就是「支持 async 的 requests」——它的 API 是**刻意**做得跟 requests 几乎一样的，所以你已经会的那套写法基本能直接搬过来，额外多了一个异步的用法。

这不是我的比喻，是 httpx 自己的设计目标：它把 requests 的接口当成事实标准来抄。所以下面这两行，你不看 import 根本分不清是谁：

```python
r = requests.get("https://example.com")   # requests
r = httpx.get("https://example.com")      # httpx，一模一样
```

**类比：从「一条固定电话」升级成「一台带多条线路的总机」**

- `requests` 像你桌上一台固定电话：一次只能打一通，打的时候你必须**握着听筒等对方接**。要打 10 个电话，就得一通一通打完。
- `httpx.AsyncClient` 像一台总机：你可以**一口气把 10 通电话全拨出去**，然后谁先接通你就先处理谁。而且这台总机还会**把已经接通的线路留着不挂断**，下次打给同一个人直接复用，不用重新拨号。

这个类比后面会真实对应到代码上：

| 类比里的东西 | 代码里的东西 | 在第几节 |
| --- | --- | --- |
| 一次只能打一通、必须握着听筒 | `requests.get` 同步阻塞 | 第 2 节 |
| 一口气把 10 通全拨出去 | `asyncio.gather` + `await client.get` | 第 4 节示例 2 |
| 线路留着不挂断、下次复用 | 连接池 + Keep-Alive | 第 3 节、第 4 节示例 3 |
| 总机最多能有几条线 | `httpx.Limits(max_connections=...)` | 第 4 节示例 3 |
| 「拨号 10 秒不通就放弃」 | `httpx.Timeout(connect=...)` | 第 3 节、第 4 节示例 3 |
| 一台总机全公司共用，不是每次打电话搬一台新的 | 全局长生命周期 `AsyncClient` | 第 5 节 |

最后一条是本文最容易被忽略、但收益最大的一点：**总机要复用**。每次请求都新建一个 `AsyncClient`，等于每打一通电话就搬来一台新总机再扔掉，连接池的好处一点都拿不到。第 4 节会单独展开讲为什么。

---

## 2. 它解决什么问题

### 2.1 先看不用它会怎样

假设你在做一个 AI 应用，要一次性请求 10 个 URL（可能是 10 个 LLM 接口调用，也可能是抓 10 个网页）。用你已经熟悉的 requests 写法：

```python
import time
import requests

urls = [f"https://httpbin.org/delay/1?n={i}" for i in range(10)]  # 每个 URL 服务端故意等 1 秒才返回

start = time.perf_counter()           # 记下开始时刻，用来算总耗时
results = []
for url in urls:                      # 挨个来：第 2 个请求必须等第 1 个整个回来才发出
    resp = requests.get(url)          # 这一行会卡住约 1 秒，期间程序什么都做不了
    results.append(resp.status_code)
print(f"同步 for 循环总耗时 {time.perf_counter() - start:.1f} 秒")
# 实测输出：同步 for 循环总耗时 10.3 秒
```

### 2.2 可感知的痛点

- **10 个请求各 1 秒 = 10 秒，而这活儿本该 1 秒多就干完。** 10 个请求彼此毫无依赖，完全可以同时在飞，却被排成了一队。数量翻倍时间就翻倍，100 个 URL 就是 100 秒。
- **每次 `requests.get(url)` 都新建一条 TCP 连接，还要重做一次 TLS 握手。** 这里的 `requests.get` 是模块级函数，它内部每次都临时造一个 session 用完扔掉。HTTPS 的握手要几个来回，往往几十到几百毫秒——10 个请求就白白多花了这么多次。
- **在 `async def` 里调 `requests.get` 会卡死整个服务。** 这条最要命。FastAPI 的事件循环是单线程的，`requests.get` 是同步阻塞的，它一卡住，**整个进程连别人的请求都没法处理**。1 个用户的 10 秒查询，能让所有人一起等 10 秒。
- 这 10 秒里 CPU 基本是 0%，加机器、换更快的 CPU 都没用，因为瓶颈不在计算。

### 2.3 问题根源，一句话

**慢的不是请求本身，是「排队等」和「每次重新建连接」；而同步代码在等的时候把整个线程占着不放。**

### 2.4 经典应用场景

httpx 的异步客户端适合「一次要等很多个外部 HTTP 结果」的活儿：

- 并发调多个 LLM API（本文主线场景），比如把一批问题一次性发出去
- 在 FastAPI 的一个端点里，同时调 2~3 个下游服务，再把结果拼起来返回
- 抓取大量网页
- 需要长期复用连接、稳定压着一个第三方 API 打的后台任务
- 接 LLM 的流式返回（SSE 逐字输出），httpx 的 `client.stream()` 原生支持

**不适合**：纯计算任务（图像处理、大矩阵运算）用它没有任何收益，那是 CPU 密集，得用多进程。

---

## 3. 读下去之前，先搞懂这些概念

这一节的词后面每一节都在用。每个概念都是「一行定义 + 大白话 + 最小示例」。

### 3.1 同步阻塞 vs 异步非阻塞

**定义**：同步阻塞 = 一个操作没做完，代码就停在那一行不往下走；异步非阻塞 = 等待期间把控制权交出去，让别的任务先跑。

**大白话**：同步是「握着听筒等对方接」，异步是「拨完号先去干别的，通了再回来」。

```python
resp = requests.get(url)        # 同步：这一行卡住，整个线程停摆
resp = await client.get(url)    # 异步：等的时候让出去，别的任务能跑
```

`await` 这个词的完整机制（事件循环、协程怎么被调度）不在本文展开，见同目录 `asyncio-入门.md`。本文你只需要记住一件事：**`await` 出现的地方，就是「我要等了，你们先上」的地方**。

### 3.2 `async with` 上下文管理器

**定义**：`with` 的异步版本，进入时自动做异步初始化，离开时自动做异步清理（这里就是关闭所有连接）。

**大白话**：跟 `with open(...)` 一样是「用完自动关」，只是关的动作本身也需要 `await`，所以前面多个 `async`。

```python
async with httpx.AsyncClient() as client:   # 进入：准备好连接池
    r = await client.get("https://example.com")
# 离开这个缩进块：连接池自动关掉，所有 TCP 连接被释放
```

注意最后那句注释——**出了缩进块 client 就废了**，再用会报错。这是第 6 节的坑之一。

### 3.3 连接池（connection pool）

**定义**：客户端内部维护的一组「已经建立好、可以重复使用」的 TCP 连接的集合。

**大白话**：不是每次打电话都重新拨号，而是拨通过的线路先留着，下次直接用。

```python
client = httpx.AsyncClient()   # 这个 client 内部就带着一个连接池
# 用同一个 client 发 100 个请求 → 复用池子里的连接，不用握手 100 次
```

池子有上限，用 `httpx.Limits` 配。httpx 0.28.1 的默认值是 `Limits(max_connections=100, max_keepalive_connections=20)`，另有 `keepalive_expiry=5.0`（空闲连接 5 秒后回收）。

### 3.4 Keep-Alive

**定义**：HTTP 的一个机制，一次请求响应结束后不立刻关闭 TCP 连接，留着给下一个请求用。

**大白话**：挂电话前先问一句「你还有事吗」，有就别挂了，接着说。

```python
# 同一个 client 连发两次，第二次省掉了 TCP 三次握手 + TLS 握手
r1 = await client.get("https://example.com/a")
r2 = await client.get("https://example.com/b")   # 复用 r1 那条连接
```

Keep-Alive 是连接池能生效的**前提**：正是因为服务端愿意不挂断，池子里才有连接可留。这也解释了为什么**必须复用同一个 client**——池子长在 client 身上，client 一换，池子就跟着没了。

### 3.5 超时的四个阶段

**定义**：httpx 把「等」拆成四段，每段可以单独设上限。

**大白话**：一通电话可以卡在四个不同的地方，httpx 让你分别规定每处最多等多久。

| 阶段 | 管什么 | 大白话 |
| --- | --- | --- |
| `connect` | 建立 TCP 连接（含 TLS 握手）最多等多久 | 拨号到对方接通 |
| `read` | 等服务器返回数据，两次收到数据之间最多等多久 | 对方接了，但半天不说话 |
| `write` | 把请求数据发出去最多等多久 | 你要说的话太长，发不出去 |
| `pool` | 从连接池里拿一条空闲连接最多等多久 | 总机线路全占满了，排队等一条 |

```python
timeout = httpx.Timeout(10.0, connect=5.0)   # 其余三项都 10 秒，只把 connect 单独设成 5 秒
client = httpx.AsyncClient(timeout=timeout)
```

**这里有个和直觉相反、必须记住的点**：httpx **默认所有操作都有 5 秒超时**（源码里是 `DEFAULT_TIMEOUT_CONFIG = Timeout(timeout=5.0)`），而 requests **默认无限等**。所以从 requests 迁过来时，同一段代码在 httpx 下可能突然开始抛超时——不是 httpx 有问题，是它默认更严格。调 LLM 这种慢接口，5 秒往往不够，一定要显式放大。第 6 节还会再说一次。

### 3.6 `Response` 对象

**定义**：一次请求的返回结果，带状态码、响应头、响应体。

**大白话**：和 requests 的 `Response` 几乎一样，属性名都对得上。

```python
r.status_code   # 200
r.json()        # 解析 JSON → dict；注意这是同步方法，不用 await
r.text          # 响应体的字符串形式
```

`r.json()` 不需要 `await` —— 这点和 aiohttp 不同，是个常见混淆，第 6 节单独讲。

### 3.7 `raise_for_status`

**定义**：检查状态码，如果是 4xx / 5xx 就抛异常，否则什么都不做。

**大白话**：把「悄悄返回一个错误页」变成「大声报错」，免得你把错误页当成正常数据往下处理。

```python
r = await client.get(url)
r.raise_for_status()      # 状态码是 404 / 500 时抛 httpx.HTTPStatusError
```

### 3.8 术语速查表

| 术语 | 大白话 | 正文哪节 |
| --- | --- | --- |
| 同步阻塞 | 卡在那一行不动，整个线程停摆 | 2.1 / 3.1 |
| 异步非阻塞 | 等的时候让别人先跑 | 3.1 / 4.2 |
| `await` | 「我要等了，你们先上」 | 3.1 / 4.2 |
| `async with` | 用完自动关，关的动作本身也要等 | 3.2 / 4.2 |
| 连接池 | 拨通过的线路留着复用 | 3.3 / 4.3 |
| Keep-Alive | 挂电话前问「还有事吗」，有就别挂 | 3.4 / 4.3 |
| `Limits` | 总机最多能有几条线 | 3.3 / 4.3 |
| `Timeout` 四阶段 | 一通电话能卡在四个地方，分别限时 | 3.5 / 4.3 |
| `raise_for_status` | 把悄悄的错误变成大声报错 | 3.7 / 4.3 |
| `asyncio.gather` | 一口气把 10 通电话全拨出去 | 4.2 |
| `Semaphore` | 只放 10 个人同时进门，其余排队 | 第 5 节 / 第 7 节 |

### 3.9 requests → httpx 迁移对照表

这张表是本文的核心工具，卡住的时候回来查它。

| 你要做的事 | requests 写法 | httpx 同步写法 | httpx 异步写法 |
| --- | --- | --- | --- |
| 发个 GET | `requests.get(url)` | `httpx.get(url)` | `await client.get(url)` |
| 发个 POST（JSON） | `requests.post(url, json=d)` | `httpx.post(url, json=d)` | `await client.post(url, json=d)` |
| 复用连接的会话 | `requests.Session()` | `httpx.Client()` | `httpx.AsyncClient()` |
| 会话上下文管理 | `with requests.Session() as s:` | `with httpx.Client() as c:` | `async with httpx.AsyncClient() as c:` |
| 取 JSON | `r.json()` | `r.json()` | `r.json()`（**不用 await**） |
| 状态码 | `r.status_code` | `r.status_code` | `r.status_code` |
| 出错就抛 | `r.raise_for_status()` | `r.raise_for_status()` | `r.raise_for_status()` |
| 查询参数 | `params={"a": 1}` | `params={"a": 1}` | `params={"a": 1}` |
| 设超时 | `timeout=10`（默认无限） | `timeout=10`（**默认 5 秒**） | `timeout=10`（**默认 5 秒**） |
| 统一前缀 | 无，手动拼字符串 | `httpx.Client(base_url=...)` | `httpx.AsyncClient(base_url=...)` |
| 超时异常 | `requests.exceptions.Timeout` | `httpx.TimeoutException` | `httpx.TimeoutException` |
| 状态码异常 | `requests.exceptions.HTTPError` | `httpx.HTTPStatusError` | `httpx.HTTPStatusError` |
| 连接池上限 | 靠 adapter，较绕 | `httpx.Limits(...)` | `httpx.Limits(...)` |

**关键差异只有三个，其余都能照搬**：① 异步写法要 `await` 和 `async with`；② 默认超时从「无限」变成「5 秒」；③ 异常类名换了一套（而且 httpx 的更好记，都挂在 `httpx.` 下）。

---

## 4. 从最简单的代码示例开始

先装上（工作区用 uv）：

```bash
uv add httpx
```

### 4.1 示例 1：最小可运行版本（先建立「无痛切换」的信心）

```python
# demo1_sync.py
import httpx  # 唯一的新 import，用法和 requests 高度一致

# httpx.get 是模块级便捷函数，和 requests.get 对应，签名几乎相同
resp = httpx.get("https://httpbin.org/get", params={"hello": "world"})

resp.raise_for_status()      # 4xx/5xx 直接抛异常，避免把错误页当正常数据用
print(resp.status_code)      # 200，属性名和 requests 一样
print(resp.json()["args"])   # {'hello': 'world'}，json() 也是同步方法
```

**逐段解读**

- `import httpx`：这一行是唯一的改动。把 `requests` 换成 `httpx`，上面这段代码剩下的部分**一个字都不用改**就能跑。
- `httpx.get(...)`：模块级便捷函数。`params` 会被自动拼成 `?hello=world`，和 requests 行为一致。
- `resp.raise_for_status()`：状态码正常就什么都不做，异常就抛。这个方法名和 requests 完全相同。
- `resp.json()`：把响应体按 JSON 解析成 dict。注意这里**没有 await**，因为这是同步的 `httpx.get`。

**这一步你得到了什么**：确认了从 requests 迁过来的心理成本几乎是零。但要注意，这段代码**还是同步的**，没有任何并发收益——`httpx.get` 和 `requests.get` 一样会卡住整个线程。真正的收益在下一个示例。

另外，模块级 `httpx.get()` 每次调用都会临时建一个 client、用完扔掉，**拿不到连接池好处**，所以它只适合写脚本和临时调试。真实项目一律用 client 对象。

### 4.2 示例 2：并发请求 10 个 URL，和同步耗时并排对比（本文核心）

这是本文最重要的一段代码。同一件事写两遍，直接看数字差距。

```python
# demo2_compare.py
import asyncio
import time
import httpx

# 10 个 URL，每个都让服务端故意延迟 1 秒返回，方便观察并发效果
URLS = [f"https://httpbin.org/delay/1?n={i}" for i in range(10)]


def fetch_all_sync() -> list[int]:
    """同步版：一个接一个,前一个不回来后一个不发。"""
    with httpx.Client() as client:          # 用 Client 而非 httpx.get,已经享受连接池复用
        results = []
        for url in URLS:                    # 关键:for 循环天然串行
            resp = client.get(url)          # 卡住约 1 秒
            results.append(resp.status_code)
        return results


async def fetch_one(client: httpx.AsyncClient, url: str) -> int:
    """异步版的单个请求。注意 client 是传进来的,不是在这里新建的。"""
    resp = await client.get(url)            # await:等的时候让出控制权,别的请求趁机发出去
    return resp.status_code


async def fetch_all_async() -> list[int]:
    """异步版:10 个请求同时在飞。"""
    async with httpx.AsyncClient() as client:      # 整批请求共用一个 client(一个连接池)
        tasks = [fetch_one(client, url) for url in URLS]  # 只是造出 10 个协程对象,还没开始跑
        return await asyncio.gather(*tasks)        # 这一行才真正把 10 个请求一起发出去并等齐


if __name__ == "__main__":
    start = time.perf_counter()                    # perf_counter 比 time.time 更适合测耗时
    fetch_all_sync()
    print(f"同步  : {time.perf_counter() - start:.2f} 秒")

    start = time.perf_counter()
    asyncio.run(fetch_all_async())                 # asyncio.run 是异步世界的入口
    print(f"异步  : {time.perf_counter() - start:.2f} 秒")
```

实测输出的量级（具体数字受网络波动影响，但差距的量级是稳定的）：

```
同步  : 10.42 秒
异步  : 1.18 秒
```

**约 9 倍差距**。而且这个差距会随 URL 数量继续放大：同步是 `N × 1 秒`，异步基本还是 `1 秒多`（直到撞上连接池上限或对方限流）。

**逐段解读**

- `fetch_all_sync`：`with httpx.Client()` 建了一个同步 client。它已经有连接池了，所以省掉了重复握手——但**请求之间仍然是排队的**，因为 `for` 循环里 `client.get` 每次都要卡到返回。这说明「连接池」和「并发」是两个独立的收益，这个函数只拿到了前者。
- `fetch_one`：把「发一个请求」抽成独立协程。**关键在于 `client` 是参数传进来的**，不是函数内部新建的——这样 10 个请求才能共用同一个连接池。为什么这点重要，下面单独展开。
- `fetch_all_async` 里的 `tasks = [...]`：这行**没有发出任何请求**。调用一个 `async def` 函数只是得到一个协程对象，像一张写好但还没交出去的菜单。
- `await asyncio.gather(*tasks)`：这一行是并发真正发生的地方。它把 10 个协程一起交给事件循环，10 个请求几乎同时发出，然后等它们全部完成，按**传入顺序**返回结果列表（不是按完成顺序，这点很实用）。
- `asyncio.run(...)`：同步代码进入异步世界的入口，负责创建事件循环、跑完、清理。

#### 展开 1：为什么每个请求新建 `AsyncClient` 会丧失连接池收益

这是新手最常犯的错。对比两种写法：

```python
# 错误示范:每个请求自己建一个 client
async def fetch_one_bad(url: str) -> int:
    async with httpx.AsyncClient() as client:   # 每次都新建 → 新连接池
        resp = await client.get(url)
        return resp.status_code
    # 出了这个块,连接池连同里面的 TCP 连接立刻被销毁
```

```python
# 正确写法:client 从外面传进来,大家共用
async def fetch_one_good(client: httpx.AsyncClient, url: str) -> int:
    resp = await client.get(url)
    return resp.status_code
```

**原理**：连接池是**长在 client 实例身上**的属性（回看 3.3）。client 一销毁，池子里所有 Keep-Alive 连接跟着关闭。所以错误写法下，10 个请求 = 建 10 个池子 = 做 10 次完整的 TCP 三次握手 + 10 次 TLS 握手，一次都没复用上。

代价有多大：HTTPS 握手通常几十到几百毫秒。10 个请求就白扔掉这么多次；如果是一个每秒几百请求的服务，这笔浪费会直接变成延迟和 CPU 占用。更糟的是连接数会失控——每个新 client 有自己的 `max_connections`，全局上限形同虚设。

**记住这条规则**：`AsyncClient` 要**尽可能长命**。脚本里是「整批请求共用一个」，Web 服务里是「整个进程共用一个」（第 5 节）。

#### 展开 2：`gather` 如何让请求真正重叠（时间轴示意）

同步 for 循环，10 个请求各 1 秒：

```
时间 →  0s      1s      2s      3s              ...        10s
请求1  [====]
请求2          [====]
请求3                  [====]
请求4                          [====]
...
请求10                                                [====]
总耗时 ≈ 10 秒（1 秒 × 10）
```

`asyncio.gather`，同样 10 个请求：

```
时间 →  0s      1s
请求1  [====]
请求2  [====]     ← 全都在同一段时间里等
请求3  [====]
请求4  [====]
...
请求10 [====]
总耗时 ≈ 1 秒（≈ 最慢的那一个）
```

**为什么能重叠**：每个请求的 1 秒里，99.9% 的时间是「等网卡把服务器的回包送来」，程序其实没事干。`await client.get(url)` 一执行到等待点，就把控制权交还给事件循环，事件循环立刻去启动下一个请求。10 个请求全部发出后，事件循环就守着这 10 条连接，谁的数据到了就唤醒谁。

所以并发的上限不是「CPU 有几核」，而是「你愿意同时挂多少条连接」——单线程也能轻松同时等几百个请求。这也解释了 `gather` 的耗时约等于**最慢那一个**，而不是所有请求之和。

等价的普通写法对照（不用 `gather`，手动创建 Task）：

```python
# gather 的等价展开,便于理解它到底做了什么
tasks = [asyncio.create_task(fetch_one(client, u)) for u in URLS]  # create_task 立刻排进事件循环
results = [await t for t in tasks]        # 逐个收结果;因为都已在跑,所以总耗时仍是 ≈1 秒
```

注意这里的微妙之处：`create_task` 是**立刻开始跑**，所以即便后面用普通 for 循环 `await`，请求也早就重叠了。而示例 2 里 `[fetch_one(...) for ...]` 造出的是**还没启动**的协程，必须靠 `gather` 才会跑起来。这是 `create_task` 和裸协程的核心区别。

**这一步你得到了什么**：一个能直接量化的结论——同样 10 个请求，10.4 秒变成 1.2 秒；并且知道了两条铁律：client 要复用、并发靠 `gather`。

### 4.3 示例 3：贴近项目的样子（长生命周期 client + 完整配置 + 错误分类）

真实项目里的 client 不会是裸的 `httpx.AsyncClient()`，而是把 base_url、headers、超时、连接池上限都配好，并且对错误分类处理。

```python
# demo3_llm_client.py
import asyncio
import os
import httpx

# 超时按四个阶段分开配。调 LLM 时 read 必须放大:模型生成本身就慢
LLM_TIMEOUT = httpx.Timeout(
    connect=5.0,   # 建连接 5 秒还不通,基本是网络或域名问题,早失败早重试
    read=60.0,     # 等模型返回给到 60 秒(默认 5 秒对 LLM 远远不够)
    write=10.0,    # 把 prompt 发出去,一般很快,给 10 秒足够
    pool=5.0,      # 池子满了最多排队 5 秒,再等不到就报错而不是无限堆积
)

# 连接池上限。默认是 max_connections=100 / max_keepalive_connections=20
LLM_LIMITS = httpx.Limits(
    max_connections=20,            # 最多同时 20 条连接,避免把对方或自己打爆
    max_keepalive_connections=10,  # 其中 10 条空闲时也留着复用
)


def build_llm_client() -> httpx.AsyncClient:
    """造一个配置好的 client。注意这里只是造出来,不负责关闭。"""
    return httpx.AsyncClient(
        base_url="https://api.openai.com/v1",   # 之后调用只写路径,不用反复拼域名
        headers={                                # 默认请求头,每个请求自动带上
            "Authorization": f"Bearer {os.environ['LLM_API_KEY']}",  # 密钥从环境变量读,绝不写进代码
            "Content-Type": "application/json",
        },
        timeout=LLM_TIMEOUT,
        limits=LLM_LIMITS,
    )


async def ask_llm(client: httpx.AsyncClient, question: str) -> str:
    """调一次 LLM,并把各类失败翻译成人能看懂的结果。"""
    try:
        resp = await client.post(
            "/chat/completions",                 # 相对路径,自动接在 base_url 后面
            json={
                "model": "gpt-4o-mini",
                "messages": [{"role": "user", "content": question}],
            },
        )
        resp.raise_for_status()                  # 4xx/5xx 转成 HTTPStatusError,下面统一捕获
        return resp.json()["choices"][0]["message"]["content"]

    except httpx.TimeoutException as exc:
        # 四个阶段的超时都是它的子类,所以这一句能同时兜住 Connect/Read/Write/PoolTimeout
        return f"[超时] {type(exc).__name__}"

    except httpx.HTTPStatusError as exc:
        # 服务端明确返回了错误状态码。exc.response 还在手上,可以读出错误详情
        return f"[HTTP {exc.response.status_code}] {exc.response.text[:100]}"

    except httpx.RequestError as exc:
        # 请求根本没发出去或连接断了(DNS 失败、拒绝连接等),兜底用
        return f"[请求失败] {type(exc).__name__}: {exc}"


async def main() -> None:
    questions = [f"用一句话解释概念 {i}" for i in range(10)]

    client = build_llm_client()
    try:
        # 整批问题共用这一个 client,连接池被充分复用
        answers = await asyncio.gather(*(ask_llm(client, q) for q in questions))
        for q, a in zip(questions, answers):
            print(f"{q} -> {a[:60]}")
    finally:
        await client.aclose()      # 手动建的 client 必须手动关,否则连接泄漏


if __name__ == "__main__":
    asyncio.run(main())
```

**逐段解读**

- `LLM_TIMEOUT`：把四个阶段分开配，而不是一个 `timeout=60`。这样做的好处是**失败得更快也更准**：连不上就 5 秒放弃（重试有意义），而模型慢慢生成就允许等 60 秒（重试只是浪费钱）。用一个数字统配就没法区分这两种情况。
- `LLM_LIMITS`：给连接池设上限，防止并发量一大就开出上百条连接。`max_keepalive_connections` 比 `max_connections` 小是正常的：允许峰值开 20 条，但平时只留 10 条空闲的。
- `build_llm_client`：**只负责造，不负责关**。这个职责划分很重要，第 5 节里「谁来关」会交给 FastAPI 的 `lifespan`。
- `base_url` + `headers`：配一次，之后每个调用点只写 `/chat/completions`，密钥也不用反复传。密钥走 `os.environ`，不进代码、不进日志。
- `ask_llm` 的三层 `except`：顺序是**从具体到宽泛**，这是关键。`TimeoutException` 和 `HTTPStatusError` 先捕获，最后用 `RequestError` 兜底。
- `try / finally` + `await client.aclose()`：因为这里的 client 是手动 `build` 出来的（不是 `async with`），所以必须自己关。`aclose` 的 `a` 就是 async。

#### 展开 3：`Timeout` 四个阶段各管什么（配合真实故障场景）

回看 3.5 的表格，这里补上「每种超时对应什么真实故障」，这决定了你该怎么设值：

| 阶段 | 触发的真实场景 | 抛出的异常 | 该重试吗 |
| --- | --- | --- | --- |
| `connect` | 域名解析不了、对方端口不通、网络断了 | `httpx.ConnectTimeout` | 该，通常是瞬时网络问题 |
| `read` | 连上了，但服务器迟迟不返回数据（LLM 正在生成、对方过载） | `httpx.ReadTimeout` | 谨慎，LLM 场景重试很贵 |
| `write` | 请求体很大（长 prompt、上传文件），发不出去 | `httpx.WriteTimeout` | 该，但先怀疑网络上行 |
| `pool` | 你自己的连接池被占满，新请求排不进去 | `httpx.PoolTimeout` | **不该**，这是你自己并发开太大，该限流 |

最后一行值得单独说：`PoolTimeout` 是**自己的问题**，不是对方的问题。看到它就说明并发量超过了 `max_connections`，正确的反应是加 `Semaphore` 限流（第 5 节）或调大池子，而不是重试——重试只会让排队更长。

四个阶段的异常都继承自 `httpx.TimeoutException`，继承链是 `HTTPError → RequestError → TransportError → TimeoutException → 四个具体类`。所以想统一处理就捕获 `TimeoutException`，想分开处理就捕获具体的子类。

等价的普通写法对照——如果你只想「所有阶段都 30 秒」，不必写四个参数：

```python
httpx.Timeout(30.0)                  # 四个阶段全设 30 秒
httpx.Timeout(30.0, connect=5.0)     # connect 单独 5 秒,其余三个 30 秒
httpx.Timeout(None)                  # 完全不限时(等同 requests 的默认行为,不推荐)
client = httpx.AsyncClient(timeout=30.0)   # 直接传数字,httpx 内部自动包成 Timeout(30.0)
```

**注意**：`httpx.Timeout()` 不给任何参数会直接报 `ValueError`，它要求你「至少给一个默认值，或者把四个参数全写明」。这是刻意的设计，防止你以为自己配了超时其实没配。

**这一步你得到了什么**：一个可以照着改的真实 client 配置，知道了超时该怎么按阶段拆、错误该怎么按类型分、以及 client 的「造」和「关」应该分开由谁负责。

---

## 5. 最小可用的实际用法（生产场景模板）

这是你在 FastAPI 项目里真正要写的样子：**整个进程只有一个 `AsyncClient`**，随应用启动而生，随应用关闭而亡，通过 `Depends` 注入到端点里。

FastAPI 本身的细节（`lifespan` 的机制、`Depends` 的解析规则）这里从简，另有专门文档；本节只关注「client 该放在哪」。

```python
# app/main.py —— 可直接复制进真实项目
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Annotated

import httpx
from fastapi import Depends, FastAPI, HTTPException

# ---- 必须 ----
# 超时和连接池配置。数值可调,但这两项必须显式配置,别用默认的 5 秒
TIMEOUT = httpx.Timeout(connect=5.0, read=60.0, write=10.0, pool=5.0)
LIMITS = httpx.Limits(max_connections=20, max_keepalive_connections=10)


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """应用启动时建 client,关闭时销毁。整个进程就这一个。"""
    # ---- 必须 ---- yield 之前的代码在应用启动时执行一次
    async with httpx.AsyncClient(
        base_url="https://api.openai.com/v1",   # 可选:只调一个上游时才有意义
        timeout=TIMEOUT,                         # 必须
        limits=LIMITS,                           # 必须
    ) as client:
        app.state.http_client = client   # 必须:挂到 app.state 上,端点才拿得到
        yield                            # 必须:应用在这里开始服务请求
    # 出了 async with,client 自动关闭,所有连接被释放(必须,否则连接泄漏)


app = FastAPI(lifespan=lifespan)   # 必须:把 lifespan 挂上去,否则上面的代码根本不执行


# ---- 必须 ---- 依赖函数:把 client 从 app.state 取出来交给端点
async def get_http_client() -> httpx.AsyncClient:
    return app.state.http_client


# 可选:起个别名,让端点签名短一点。删掉它就把下面写成 Depends(get_http_client)
HttpClient = Annotated[httpx.AsyncClient, Depends(get_http_client)]


@app.post("/ask")
async def ask(question: str, client: HttpClient) -> dict[str, str]:
    """端点里直接用注入进来的 client,不要在这里新建。"""
    try:
        resp = await client.post(                # 必须:await
            "/chat/completions",
            json={"model": "gpt-4o-mini",
                  "messages": [{"role": "user", "content": question}]},
        )
        resp.raise_for_status()                  # 必须:否则错误响应会被当成正常数据
    except httpx.TimeoutException:
        # 可选但强烈建议:把上游超时翻译成 504,而不是让它变成 500
        raise HTTPException(status_code=504, detail="上游模型超时")
    except httpx.HTTPStatusError as exc:
        raise HTTPException(status_code=502, detail=f"上游返回 {exc.response.status_code}")

    return {"answer": resp.json()["choices"][0]["message"]["content"]}
```

**哪几行是必须的**：`lifespan` 里建 client 并 `yield`、`app = FastAPI(lifespan=lifespan)`、依赖函数、端点里的 `await` 和 `raise_for_status`、以及显式的 `timeout` / `limits`。

**哪几行可以删**：`base_url`（要调多个不同上游时就别配）、`HttpClient` 类型别名（纯粹为了好看）、两个 `except` 转 `HTTPException`（删掉的话上游错误会变成 500，能跑但对调用方不友好）。

**为什么非要放在 `lifespan` 里**：如果在每个端点里 `async with httpx.AsyncClient()`，那就是「每个 HTTP 请求都新建一个连接池又扔掉」——正是 4.2 展开 1 讲的那个错误，只是换了个地方犯。放在 `lifespan` 里，client 的生命周期和应用一致，所有请求共享一个连接池，连接复用率最高。

### 实际项目通常还会配什么（进阶，可先跳过）

这几项不影响你现在跑通，等真的上量了再加：

- **重试 + 指数退避**：网络抖动、502、503 这类瞬时错误值得重试，间隔按 1s → 2s → 4s 递增（加点随机抖动，避免所有客户端同时重试）。常用 `tenacity` 库，或用 httpx 的 `HTTPTransport(retries=...)`（注意它只重试连接层错误，不重试状态码）。
- **429 处理**：几乎所有 LLM API 都限流。收到 429 时优先读响应头里的 `Retry-After`，按它说的等，别自己瞎猜间隔。
- **`asyncio.Semaphore` 控并发**：`gather` 一口气发 1000 个请求会同时打爆对方限流和自己的连接池。用 `Semaphore(10)` 把同时在飞的数量压到 10 个，其余自动排队。这是第 7 节第 3 题要你实现的东西。
- **日志**：把每个请求的 URL、状态码、耗时记下来（`resp.elapsed` 直接给你耗时）。注意**绝对不要把 `Authorization` 头写进日志**。
- **重要**：这些都要加在**同一个复用的 client 外面**，别因为加重试就退回到「每次新建 client」。

---

## 6. 常见坑与易错点

### 坑 1：`async with` 块外继续用 client

**报错原文**

```
RuntimeError: Cannot send a request, as the client has been closed.
```

**现象**：在函数里用 `async with` 建了 client 并 return 出去，外面再用就炸；或者 `gather` 写在 `async with` 块外面。

**原因**：`async with` 一退出就调用了 `aclose()`，连接池已销毁。client 是**一次性**的，关掉之后不能重开（重开会报 `Cannot reopen a client instance, once it has been closed.`）。

**解决**：让 `async with` 的作用域覆盖所有用到 client 的代码。

```python
# 错误:gather 在块外,此时 client 已关闭
async with httpx.AsyncClient() as client:
    tasks = [fetch(client, u) for u in urls]
results = await asyncio.gather(*tasks)      # RuntimeError

# 正确:把 gather 挪进块内
async with httpx.AsyncClient() as client:
    tasks = [fetch(client, u) for u in urls]
    results = await asyncio.gather(*tasks)
```

### 坑 2：httpx 默认有 5 秒超时，而 requests 默认无限等

**现象**：同一段调 LLM 的代码，requests 版本跑得好好的（慢但能出结果），换成 httpx 后稳定抛 `httpx.ReadTimeout`。

**原因**：**这是和直觉相反的关键差异**。httpx 的默认值是 `Timeout(timeout=5.0)`，四个阶段全是 5 秒；requests 的 `timeout` 默认是 `None`，也就是无限等。所以不是 httpx 更脆弱，是它默认更严格——反倒是 requests 的「永远不超时」才是危险的默认值。

**解决**：调慢接口时显式放大 `read`。

```python
# LLM 生成一段长文本很容易超过 5 秒
client = httpx.AsyncClient(timeout=httpx.Timeout(connect=5.0, read=60.0, write=10.0, pool=5.0))
```

### 坑 3：`AsyncClient` 跨事件循环复用

**报错原文**（视情况不同，可能是下面几种之一）

```
RuntimeError: Event loop is closed
RuntimeError: <asyncio.locks.Lock object> is bound to a different event loop
```

**现象**：把 client 建成模块级全局变量，然后被多次 `asyncio.run(...)` 用到；或者在测试里每个用例各起一个事件循环，却共用一个模块级 client。

**原因**：`asyncio.run` 每次调用都会**新建一个事件循环，跑完就关闭**。client 内部的连接和锁是绑在创建它的那个事件循环上的，循环一关，这些资源就失效了。

**解决**：client 的生命周期必须**套在事件循环内部**，不能跨越它。

```python
# 错误:模块级建 client,和事件循环生命周期脱钩
client = httpx.AsyncClient()          # 此时可能还没有事件循环
asyncio.run(main())                   # 循环关闭
asyncio.run(main())                   # 第二次:client 绑的循环已经没了

# 正确:在 async 函数内部建,或交给 FastAPI 的 lifespan 管
async def main():
    async with httpx.AsyncClient() as client:
        ...
asyncio.run(main())
```

在测试里同理：client fixture 要和事件循环同 scope（工作区规则要求共享 fixture 用 `function` scope，正好避开这个坑）。

### 坑 4：忘记 `await`，拿到一个 coroutine

**报错原文**

```
AttributeError: 'coroutine' object has no attribute 'status_code'
RuntimeWarning: coroutine 'AsyncClient.get' was never awaited
```

**现象**：`resp.status_code` 报 AttributeError，或者程序不报错但打印出 `<coroutine object ...>`。

**原因**：`client.get(...)` 返回的是协程对象，不是 `Response`。少了 `await`，请求根本没有发出去。

**解决**：`AsyncClient` 的所有请求方法前面都要有 `await`。

```python
resp = client.get(url)          # 错:resp 是 coroutine,请求没发出
resp = await client.get(url)    # 对:resp 是 Response
```

那句 `RuntimeWarning: coroutine ... was never awaited` 尤其要留意——它意味着**这段代码什么都没做却没报错**。工作区的测试规则专门点了这一条：测试里出现这个警告，说明是假通过。

### 坑 5：`response.json()` 在 httpx 里是同步方法

**现象**

```
TypeError: object dict can't be used in 'await' expression
```

**原因**：从 aiohttp 迁过来的人最容易踩。aiohttp 里是 `await resp.json()`，因为它把「读响应体」延后了；httpx 在 `await client.get()` 返回时**响应体已经读完在内存里了**，所以 `.json()` 就是个纯粹的同步解析，不用 await。

**解决**：去掉 `await`。这一点上 httpx 和 requests 完全一致。

```python
data = await resp.json()   # 错:httpx 不是这样
data = resp.json()         # 对:和 requests 一模一样
```

**例外**：只有用流式请求 `client.stream()` 时，响应体才没有预先读取，那时需要 `await resp.aread()` 或 `async for chunk in resp.aiter_bytes()`。普通请求不涉及。

---

## 7. 验证你学会了（自测题）

不看正文做，卡住了再回去翻对应小节。

**第 1 题（复述）**
用自己的话回答三个问题：① httpx 和 requests 是什么关系，为什么它的 API 长得几乎一样？② 在 `async def` 里调 `requests.get` 会发生什么坏事？③ 「连接池复用」和「并发请求」是同一个收益吗，各解决什么问题？
> 答案在第 1 节、第 2 节，第三问在 4.2 的逐段解读

**第 2 题（默写）**
不看正文，写出一段能跑的代码：并发请求 10 个 URL 并打印总耗时。要求 ① 只用一个 `AsyncClient` ② 用 `asyncio.gather` ③ 写出 `asyncio.run` 入口。写完检查两件事：`client.get` 前面有 `await` 吗？`gather` 在 `async with` 块**里面**吗？
> 答案在 4.2，两个检查点分别对应坑 4 和坑 1

**第 3 题（组合应用）**
改造第 2 题：并发请求 **100 个** URL，但**最多只允许 10 个同时在飞**。提示：需要 `asyncio.Semaphore(10)`，在单个请求的协程里用 `async with sem:` 把请求包起来。做完再想两个问题：① 如果不限流，直接 `gather` 100 个，会先撞到什么错误（提示：和 `Limits` 有关）？② 这时候 `httpx.Limits(max_connections=...)` 该设多少，和 `Semaphore` 的 10 是什么关系？
> `Semaphore` 见第 5 节进阶清单；撞到的错误见 4.3 展开 3 的表格最后一行；`Limits` 见 3.3 和 4.3
