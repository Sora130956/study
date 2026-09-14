# SSE 流式响应：心跳、断线与关闭

> **目标读者**：会写 FastAPI 普通接口（`@app.get` 返回 dict），但没做过"一个字一个字往外吐"的流式接口。
> **前置知识**：Python 函数、`async def` / `await` 的基本概念、FastAPI 路由、浏览器打开过网页。
> **学习时长**：约 2.5 小时（第 1~4 节 1.5 小时，第 5~7 节 1 小时）。第 3 节若你已熟悉异步生成器可跳过；第 6 节**不要跳**，线上事故全在那。

> **教学锚点**：本文示例最终落到工作区骨架 `templates/ai-service-template-project/ai-service-template/app/main.py`。该骨架目前只有 `/health` 和 `/chat`（一次性返回），**缺 `/stream`** —— 第 5 节就是把它补上。

---

## 1. 它是什么

一句话：**SSE（Server-Sent Events）是"服务器把一个 HTTP 响应的 body 拖着不关，分很多次往里写数据"的约定。**

生活化类比：**打印机的连续出纸**。

普通接口像**复印机**：你按下按钮，机器在里面咔咔响 30 秒，然后"啪"一下把整叠纸一次性推出来。这 30 秒里你盯着出纸口，什么都没有。

SSE 像**针式打印机**：按下按钮后，纸一行一行往外爬。第 1 秒你就能看到第一行字，虽然整页还要 30 秒才打完。纸一直连在机器上没被撕断——**连接没关**。

| 类比里的东西             | 代码里的东西                               |
| ------------------ | ------------------------------------ |
| 一卷连续的纸             | 一个没有 `Content-Length`、一直不结束的 HTTP 响应 |
| 打印机一行一行吐字          | 异步生成器每次 `yield` 一段数据                 |
| 每行末尾的换行            | 每个事件末尾的空行 `\n\n`                     |
| 机器空转时的"咔哒"声（证明没死机） | 心跳注释行 `: ping\n\n`                   |
| 你中途拔了打印机电源         | 客户端关闭连接，服务端需要停止生成                    |
| 最后一行打印"—— 完 ——"    | 结束哨兵 `data: [DONE]`                  |

这张表后面每一行都会出现在真实代码里，你可以随时回来对照。

---

## 2. 它解决什么问题

### 先看不用 SSE 会怎样

你写了一个调用大模型的接口：

```python
@app.post("/chat")
async def chat(req: ChatRequest):
    resp = await llm_client.chat.completions.create(...)  # 模型要生成 400 字
    return {"reply": resp.choices[0].message.content}
```

模型生成 400 个字要 25 秒。于是：

**痛点 1：用户看到的是一片空白。** 前端 `fetch` 发出请求后，`await response.json()` 这行会卡整整 25 秒。用户看到的是一个转圈圈的 loading，或者干脆什么都没有。30 秒后用户已经刷新了三次页面——每刷一次你就多烧一次 token。

**痛点 2：<mark style="background: #BBFABBA6;">网关会直接把你掐了。** Render、Fly.io、Nginx 默认有响应超时（常见 30s / 60s）。一次性返回意味着"整整 25 秒这个连接上一个字节都没传输过"，某些代理判定为僵死连接直接断开，用户收到 502，而你的模型其实算完了。</mark>

**痛点 3：你不知道它卡在哪。** 一次性返回只有两种状态：没返回、返回了。模型是在第 2 秒卡住还是第 24 秒卡住，你从日志里看不出来。

### 问题根源

<mark style="background: #BBFABBA6;">一句话：**HTTP 默认假设"响应是一个完整的、长度已知的东西"，而 LLM 的输出是"随时间逐渐长出来的东西"，两者的时间模型对不上。**</mark>

<mark style="background: #BBFABBA6;">SSE 的做法是：不告诉浏览器 body 有多长（不发 `Content-Length`），改用分块传输，写一段算一段。</mark>

### 经典应用场景

- LLM 对话的逐字输出（ChatGPT 网页版就是 SSE）
- 长任务进度条：`data: {"step": 3, "total": 10}`
- 服务端推送通知、股价、日志尾随（`tail -f` 的网页版）

**什么时候不用 SSE**：需要客户端也频繁往服务端发消息（比如在线协作光标位置）——那是 WebSocket 的活。SSE 是**单向**的，只有服务器往下推。

---

## 3. 前置知识

### 3.1 <mark style="background: #BBFABBA6;">异步生成器（async generator）</mark>

**定义**：<mark style="background: #BBFABBA6;">用 `async def` 定义、函数体内含 `yield` 的函数，调用它得到一个可以 `async for` 迭代的对象。</mark>

**大白话**：普通函数是"算完一次性还给你"，生成器是"算一点还一点，还完了在原地停住等你要下一个"。加个 async 就是"停住的期间还能让出 CPU 给别人干活"。

```python
async def counter():
    for i in range(3):
        yield i            # 吐出一个值，然后就地暂停
        await asyncio.sleep(1)   # 暂停期间事件循环去干别的
```

关键点：**函数体不是一次执行完的**。每次外部要下一个值，它才从上次 `yield` 的地方接着往下跑。

### 3.2 <mark style="background: #BBFABBA6;">StreamingResponse</mark>

**定义**：FastAPI/Starlette 提供的响应类，接收一个可迭代对象，每迭代出一段就往连接上写一段。

**大白话**：普通 `return {"a": 1}` 是"把纸交给 FastAPI，它帮你封装好寄出去"；`StreamingResponse` 是"把出纸口交给 FastAPI，它守在那儿，你吐一行它寄一行"。

```python
from fastapi.responses import StreamingResponse
return StreamingResponse(counter(), media_type="text/event-stream")
```

### 3.3 SSE 的报文格式

**定义**：SSE 不是二进制协议，就是纯文本，靠**空行**分隔事件。

**大白话**：<mark style="background: #BBFABBA6;">你往连接里写的每一段，必须长成 `data: 内容` 加两个换行。少写一个换行，浏览器会一直等下一半，你就看不到任何东西。</mark>

```
data: 你好\n\n
```

<mark style="background: #BBFABBA6;">一个事件可以有多个字段：</mark>

| 字段      | 写法                  | 作用                    |
| ------- | ------------------- | --------------------- |
| `data`  | `data: hello\n\n`   | 事件内容，必填               |
| `event` | `event: progress\n` | 事件类型，前端按类型分开监听        |
| `id`    | `id: 42\n`          | 事件编号，断线重连时浏览器会带回来     |
| `retry` | `retry: 3000\n\n`   | 告诉浏览器断线后隔多久重连（毫秒）     |
| 注释      | `: 任意内容\n\n`        | 冒号开头，前端**忽略**，专门用来做心跳 |

### 3.4 术语速查表

| 术语 | 一句话 |
| --- | --- |
| SSE | 服务器单向持续推送数据的 HTTP 约定 |
| `text/event-stream` | SSE 专用的 Content-Type，写错浏览器不按流处理 |
| 事件（event） | 一段以空行结尾的文本，前端触发一次回调 |
| 心跳 / keep-alive | 没数据时定期发的空注释，防止连接被判定为僵死 |
| 哨兵（sentinel） | 约定的结束标记，如 `[DONE]`，告诉前端可以关了 |
| 半包 | 一个事件被网络切成两半到达，前端需要缓冲拼接 |
| 缓冲（buffering） | 代理攒够一批才转发，导致"流式失效" |
| 对端（peer） | 连接的另一头，即浏览器 / curl / 中间的代理 |
| Task | asyncio 里正在跑的一个协程的句柄，可以被外部取消 |
| `CancelledError` | 取消一个 Task 时，asyncio 往它挂起处扔的异常 |

---

## 4. 三个递进示例

### 示例 1：<mark style="background: #BBFABBA6;">最小可运行的流</mark>

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()


async def tick():
    # 异步生成器：每次 yield 一段完整的 SSE 事件文本
    for i in range(5):
        # 关键：data: 开头，\n\n 结尾。少一个 \n 前端就收不到这个事件
        yield f"data: 第 {i} 条\n\n"
        # 模拟"生成要花时间"；sleep 期间事件循环可以服务其他请求
        await asyncio.sleep(1)


@app.get("/stream")
async def stream():
    return StreamingResponse(
        tick(),
        # 必须是 text/event-stream，否则浏览器把它当普通文本，EventSource 不工作
        media_type="text/event-stream",
    )
```

**逐段解读**

- `async def tick()` + `yield`：这是个异步生成器。FastAPI 拿到它之后，每要一个值才让函数体往前跑一步，所以数据是"边算边发"而不是"算完再发"。
- <mark style="background: #BBFABBA6;">`f"data: 第 {i} 条\n\n"`：SSE 的最小事件单位。第一个 `\n` 结束 `data` 这一行，第二个 `\n` 是空行，表示"这个事件完了"。**这两个换行是全文第一个坑**。</mark>
- `await asyncio.sleep(1)`：真实场景里这里是 `await` 模型的下一个 token。用 sleep 模拟是为了让你能肉眼看到流式效果。
- <mark style="background: #BBFABBA6;">`media_type="text/event-stream"`：告诉浏览器"这是 SSE"。写成默认的 `text/plain` 的话，`EventSource` 会直接报错。</mark>

**验证**：<mark style="background: #BBFABBA6;">启动后在终端跑 `curl -N http://127.0.0.1:8000/stream`。`-N` 表示不缓冲。你应该看到 5 行**每隔一秒**冒出来一行，而不是 5 秒后一次性出现 5 行。</mark>

> 如果一次性出现 5 行，说明 curl 或中间某层在缓冲。先确认加了 `-N`。

**这一步你得到了什么**：一个真正在流式输出的接口。你已经能用它替代"转圈圈 25 秒"的体验了。但它有个致命问题——**如果模型思考 40 秒没吐第一个字，这条连接会被网关掐掉**。下一步解决它。

---

### 示例 2：<mark style="background: #BBFABBA6;">加心跳 + 断线检测</mark>

真实需求：<mark style="background: #BBFABBA6;">模型可能思考很久才吐第一个字；用户可能中途关掉页面。</mark>

```python
import asyncio
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()


async def slow_source():
    """模拟一个"先沉默很久、再慢慢出字"的模型。"""
    await asyncio.sleep(8)          # 模型在"思考"，这 8 秒一个字都没有
    for word in ["流式", "输出", "的", "好处", "是", "体感快"]:
        yield word
        await asyncio.sleep(0.5)


async def event_stream(request: Request):
    source = slow_source()
    # 用一个任务去取下一段数据，这样我们能同时做超时判断
    while True:
        # 每次最多等 3 秒。等到了就发数据，等不到就发心跳。
        # 心跳的作用：让连接上始终有字节流动，代理才不会判定它僵死
        try:
            chunk = await asyncio.wait_for(source.__anext__(), timeout=3.0)
        except StopAsyncIteration:
            # 生成器正常跑完了 —— 发结束哨兵，让前端知道可以关连接了
            yield "data: [DONE]\n\n"
            return
        except asyncio.TimeoutError:
            # 3 秒没等到新内容：发一条注释行。冒号开头，前端会忽略它，
            # 但它是真实字节，足以让网关认为连接活着
            yield ": ping\n\n"
            continue

        # 每次准备发数据前检查：用户是不是已经关掉页面了？
        # 是的话直接 return，生成器被回收，上游的模型调用也会被取消，省钱
        if await request.is_disconnected():
            return

        yield f"data: {chunk}\n\n"


@app.get("/stream")
async def stream(request: Request):
    return StreamingResponse(event_stream(request), media_type="text/event-stream")
```

**逐段解读**

- <mark style="background: #BBFABBA6;">`asyncio.wait_for(source.__anext__(), timeout=3.0)`：这是整段的核心。它的意思是"给上游 3 秒时间交出下一段内容，超时我就自己处理"。没有它，那 8 秒的沉默期里连接上一个字节都没有。</mark>
- <mark style="background: #BBFABBA6;">`except StopAsyncIteration`：异步生成器耗尽时抛这个异常。注意**不能**让它冒泡出去，否则前端收不到结束信号，会一直等到超时。</mark>
- <mark style="background: #BBFABBA6;">`yield "data: [DONE]\n\n"`：结束哨兵。SSE 协议本身**没有**"流结束"的标准信号，所以业界统一靠一个约定字符串（OpenAI 用的就是 `[DONE]`）。前端看到它就主动 `close()`。</mark>
- <mark style="background: #BBFABBA6;">`yield ": ping\n\n"`：心跳。冒号开头在 SSE 规范里是注释，前端的 `onmessage` **不会**被触发，用户无感知，但 TCP 上确实传输了 8 个字节。</mark>
- <mark style="background: #BBFABBA6;">`await request.is_disconnected()`：FastAPI 提供的断线检查。用户关页面后这里返回 `True`，我们直接 `return` 停止生成。**不加这行，用户关了页面你还在给 OpenAI 付钱。**</mark>

<mark style="background: #BBFABBA6;">**这一步你得到了什么**：一个能扛住"长时间沉默"和"用户中途跑路"的流。心跳解决网关误杀，断线检测解决无谓烧钱。</mark>

---

### 示例 3：贴近项目 —— 接真实 LLM + 出错也要好好收尾

真实场景：数据是 JSON、模型可能中途报错、无论怎样都要释放资源。

```python
import asyncio
import json
import logging

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

log = logging.getLogger(__name__)
router = APIRouter()


def sse(data: dict, event: str | None = None) -> str:
    """把 dict 打包成一个合法 SSE 事件。

    单独抽出来，是因为 `\\n\\n` 忘写是最高频的 bug —— 只在一个地方写，就只可能错一次。
    ensure_ascii=False 让中文在日志里可读；但要保证 JSON 内部没有裸换行，
    否则会被 SSE 当成"事件结束"，把一个事件劈成两半。
    """
    payload = json.dumps(data, ensure_ascii=False)
    prefix = f"event: {event}\n" if event else ""
    return f"{prefix}data: {payload}\n\n"


async def llm_stream(client, prompt: str, request: Request):
    stream = None
    try:
        # stream=True 让 SDK 返回一个可异步迭代的对象，而不是最终结果
        stream = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            stream=True,
        )

        async for chunk in stream:
            # 用户关页面了就立刻停，别再往下烧 token
            if await request.is_disconnected():
                log.info("client disconnected, aborting stream")
                return

            delta = chunk.choices[0].delta.content
            # 首个 chunk 的 content 常常是 None（那一包只带 role 信息），必须过滤
            if delta:
                yield sse({"delta": delta})

        yield sse({"done": True}, event="done")

    except asyncio.CancelledError:
        # 连接被对端切断时 asyncio 会往生成器里扔这个异常。
        # 记一笔就重新抛出 —— 不能吞掉，否则任务取消机制会失效
        log.info("stream cancelled")
        raise

    except Exception as exc:
        # 流已经开始了，HTTP 状态码早就发出去了（200），
        # 这时候再 raise HTTPException 前端只会看到连接莫名断开。
        # 正确做法：把错误也当成一个事件发下去，前端能显示"生成失败"
        log.exception("stream failed")
        yield sse({"message": "生成失败，请重试"}, event="error")

    finally:
        # 无论正常结束、报错还是被取消，都要关掉上游流，释放连接
        if stream is not None:
            await stream.close()


@router.get("/stream")
async def stream(request: Request, prompt: str):
    client = request.app.state.llm_client
    return StreamingResponse(
        llm_stream(client, prompt, request),
        media_type="text/event-stream",
        headers={
            # 禁止浏览器/CDN 缓存流式响应，否则第二次请求可能直接读缓存
            "Cache-Control": "no-cache",
            # 关键：告诉 Nginx 别缓冲这个响应。不加这行，
            # Nginx 会攒满 4KB 才转发一次，你的"流式"在线上会变回"一次性"
            "X-Accel-Buffering": "no",
        },
    )
```

**逐段解读**

- `def sse(...)`：把格式化收拢到一个函数。这是第 6 节坑 1 的**结构性防御**——你只在这一处写 `\n\n`。
- `if delta:`：OpenAI 流的第一个 chunk 通常 `content` 为 `None`，直接 `yield` 会把字符串 `"None"` 发给前端。
- `except asyncio.CancelledError` 单独一支：连接被对端切断时会走到这里，只记日志然后**原样抛出**。为什么必须这么写，见下面的「延伸」。
- `except Exception` 里 `yield` 而不是 `raise`：这是流式接口和普通接口**最大的行为差异**。状态码在第一个字节发出时就定死了，之后再抛异常改不了状态码，前端只会看到连接断开。所以要把错误降级成一个 `event: error` 事件。
- `log.exception(...)` 而不是 `log.error(...)`：两者只差一件事——`log.exception` **自动附上当前异常的完整堆栈**（等价于 `log.error(..., exc_info=True)`），异常类型、报错信息、抛出位置全在日志里，只能用在 `except` 块内。`log.error("stream failed")` 只会打出这五个字，事后查日志根本不知道是超时还是鉴权失败。
- `finally: await stream.close()`：生成器被提前销毁时 `finally` 依然会执行，这是唯一可靠的资源释放点。
- `X-Accel-Buffering: no`：本地一切正常、一上 Render/Nginx 就变成"等 30 秒一次性出全部"，99% 是这行没加。

**这一步你得到了什么**：一个生产可用的流式端点——能处理 `None`、能上报错误、能被取消、不会被代理缓冲、不会泄漏连接。

---

### 延伸：`CancelledError` 到底是怎么回事

这是示例 3 里信息密度最高的五行，单独拆开。

#### 「对端」是谁

一次 HTTP 请求有两端：**你这端**是跑 FastAPI 的服务器进程，**对端（peer）**是发起请求的一方——浏览器、`curl`，或者中间的 Nginx / 云平台负载均衡。

「被对端切断」= **不是你主动结束的，是对面把 TCP 连接关了**。

| 场景 | 谁关的 |
| --- | --- |
| 用户关掉浏览器标签页 | 浏览器 |
| 用户点了页面上的「停止」按钮，JS 调 `es.close()` | 浏览器 |
| 用户刷新页面（旧连接被丢弃） | 浏览器 |
| 请求超过平台最大时长（Render 常见 300s） | 负载均衡 |
| 用户手机切到后台 / 断网 | 操作系统或网络 |

普通接口 100ms 就返回了，对端几乎没机会中途跑掉。**但 SSE 连接要开几十秒甚至几分钟，中途被切断是常态而非异常**——这就是流式接口必须处理它的原因。

#### <mark style="background: #BBFABBA6;">从「对端切断」到「生成器收到异常」的链路</mark>

```
用户关标签页
  → 浏览器关闭 TCP 连接
  → uvicorn 检测到连接已断（写数据失败 / 收到 FIN）
  → uvicorn 取消正在处理这个请求的 asyncio Task
  → 该 Task 里正 await 着的那个点抛出 asyncio.CancelledError
  → 你的生成器正卡在 `async for chunk in stream` 的 await 上
  → CancelledError 就从那一行抛出来
```

关键理解：**你的整个请求处理逻辑，在 asyncio 眼里就是一个 Task**。uvicorn 发现连接没了，没必要继续干活，于是对这个 Task 调 `task.cancel()`。

![[Pasted image 20260914103539.png]]

#### 什么是「任务取消机制」

asyncio 提供的「从外部让一个协程停下来」的机制。它的实现方式很特别：**不是暴力杀死，而是往协程当前挂起的地方扔一个异常。**

```python
import asyncio


async def worker():
    print("开始")
    await asyncio.sleep(10)    # 协程挂起在这一行
    print("这行永远不会执行")


async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(1)
    task.cancel()              # 往 worker 当前挂起的位置扔 CancelledError
    try:
        await task
    except asyncio.CancelledError:
        print("任务已取消")
```

`task.cancel()` 做的事就是：**在 `await asyncio.sleep(10)` 那一行抛出 `CancelledError`**。协程从那里开始沿着正常的异常传播路径往外走——会触发 `except`、会执行 `finally`，和普通异常完全一样。

这个设计的好处：你有机会做清理（关文件、关连接、回滚）。坏处：**如果你把这个异常吞掉，取消就失败了**。

#### 为什么「吞掉就会让取消机制失效」

反面例子：

```python
async def bad_worker():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        log.info("被取消了")
        # 少了 raise —— 异常在这里被"消化"了
    print("我还在跑！")        # 居然会执行
```

发生的事：

1. 外部调 `task.cancel()`，asyncio 认为"我已经发出取消信号了，等它结束"
2. `except` 捕获后没重新抛出，协程**继续往下执行**，甚至可能正常 `return`
3. 外部 `await task` 拿到的是一个**正常完成**的结果，而不是 `CancelledError`

外部就无从判断这个任务到底是"被成功取消"还是"自己跑完了"。更实际的问题：

```python
# 上层想在 5 秒内拿到结果，超时就放弃
await asyncio.wait_for(bad_worker(), timeout=5)
```

`wait_for` 的实现就是「超时 → `task.cancel()` → 等待 `CancelledError` → 抛 `TimeoutError`」。如果 `bad_worker` 吞掉了异常，`wait_for` 会一直等它跑完——**你的超时保护彻底失效**。同理，`asyncio.TaskGroup`、示例 2 里的 `asyncio.wait_for` 都依赖"取消信号能传出去"。

所以规矩是：**`CancelledError` 可以捕获（为了清理），但必须 `raise` 出去。**

#### 回到示例 3 的三个分支

```python
except asyncio.CancelledError:
    log.info("stream cancelled")
    raise

except Exception as exc:
    log.exception("stream failed")
    yield sse({"message": "生成失败，请重试"}, event="error")

finally:
    if stream is not None:
        await stream.close()
```

| 分支 | 连接状态 | 该做什么 | 不该做什么 |
| --- | --- | --- | --- |
| `CancelledError` | **已断开** | 记日志 + `raise` | **不能 `yield`**——对端都不在了，往关闭的连接里写只会再报错 |
| `Exception` | 还活着 | `yield` 一个 `event: error` 让前端能显示失败 | 不能 `raise`——状态码早已是 200，改不了 |
| `finally` | 两种都可能 | `await stream.close()` 关掉上游 LLM 流 | — |

`finally` 是**唯一可靠**的资源释放点，省 token 的关键就在这里。

#### 关于「放在 `except Exception` 前面」

Python 3.8 起 `CancelledError` 继承自 `BaseException` 而非 `Exception`，所以：

```python
try:
    ...
except Exception:      # 捕获不到 CancelledError
    ...
```

从纯语法角度，那个 `except asyncio.CancelledError` 分支即使删掉、或者放在后面，`except Exception` 也不会误吞它。显式写出来是为了两件事：**记一条专门的日志**（区分"用户跑了"和"上游挂了"，排查账单异常时很有用），以及**在代码里明确表达"这条路径我考虑过"**。

顺带一个硬性易错点：如果你写的是 `except BaseException:` 或裸 `except:`，那就**真的会吞掉 `CancelledError`**，取消机制就坏了。这也是「永远不要写裸 `except:`」的一条具体理由。

---

## 5. <mark style="background: #BBFABBA6;">最小可用的实际用法</mark>

直接贴进 `templates/ai-service-template-project/ai-service-template/app/main.py`（骨架里已有 `lifespan` 建好的 `app.state.llm_client`）：

```python
import asyncio
import json
import logging

from fastapi import Request
from fastapi.responses import StreamingResponse

log = logging.getLogger(__name__)


def _sse(data: dict, event: str | None = None) -> str:      # 必须
    payload = json.dumps(data, ensure_ascii=False)
    prefix = f"event: {event}\n" if event else ""
    return f"{prefix}data: {payload}\n\n"


async def _gen(client, prompt: str, request: Request):
    stream = None
    try:
        stream = await client.chat.completions.create(       # 必须
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            stream=True,
        )
        async for chunk in stream:
            if await request.is_disconnected():              # 必须（省钱）
                return
            delta = chunk.choices[0].delta.content
            if delta:                                        # 必须（过滤 None）
                yield _sse({"delta": delta})
        yield _sse({"done": True}, event="done")             # 必须（结束哨兵）
    except asyncio.CancelledError:
        raise
    except Exception:
        log.exception("stream failed")
        yield _sse({"message": "生成失败"}, event="error")    # 必须
    finally:
        if stream is not None:
            await stream.close()                             # 必须（释放连接）


@app.get("/stream")
async def stream(request: Request, prompt: str):
    return StreamingResponse(
        _gen(request.app.state.llm_client, prompt, request),
        media_type="text/event-stream",                      # 必须
        headers={
            "Cache-Control": "no-cache",                     # 必须
            "X-Accel-Buffering": "no",                       # 必须（上线后才看得出）
        },
    )
```

**可选 / 可跳过的进阶项**：

- **心跳**（示例 2 的 `: ping`）：只有当模型首字延迟可能超过网关超时（常见 30s）时才需要。用 `gpt-4o-mini` 短 prompt 一般 1~2 秒出字，可先不加，等真遇到 502 再补。
- **`id:` 字段 + `Last-Event-ID` 断点续传**：浏览器断线自动重连时会在请求头带上 `Last-Event-ID`，你可以据此从断点继续。**大多数 LLM 场景不需要**——模型输出无法"从中间续"，直接让用户重发更简单。跳过。
- **`retry:` 字段**：控制浏览器自动重连间隔。默认 3 秒通常够用。跳过。

**怎么测**：参见 [respx-http-mock](./respx-http-mock.md) 与 [pytest-入门](./pytest-入门.md)。要点是用 `httpx.AsyncClient` 的 `client.stream("GET", "/stream")` 拿到响应后 `async for line in resp.aiter_lines()`，断言收到的事件序列里包含 `delta` 且最后一条是 `done`，而不是只断言 `status_code == 200`（那是装饰性测试）。

---

## 6. 常见坑与易错点

### 坑 1：忘了第二个换行，前端一个字都收不到

**现象**：`curl -N` 能看到数据在流，浏览器 `EventSource` 的 `onmessage` 从不触发。

**原因**：你写的是 `yield f"data: {x}\n"`。SSE 靠**空行**判定事件结束，只有一个 `\n` 意味着"这个事件还没写完"，浏览器会一直缓着等下文。

**解决**：永远走统一的 `_sse()` 函数，不要在业务代码里手拼字符串。

---

### 坑 2：JSON 里有换行，一个事件被劈成两半

**现象**：前端 `JSON.parse` 报 `Unexpected end of JSON input`。

**原因**：如果你 `json.dumps(..., indent=2)` 或者数据里带了裸 `\n`（比如模型输出的代码块），这些换行会被 SSE 解析器当作事件分隔。

**解决**：`json.dumps` 默认不带 `indent`，产出的是单行 JSON，且会把内容里的换行转义成 `\\n` —— 所以**别加 `indent`**。如果你要发纯文本而非 JSON，务必先把 `\n` 替换掉或改用多行 `data:` 语法。

---

### 坑 3：本地流式正常，上线后变成"等半天一次性出全部"

**报错**：没有报错，就是不流式了。

**原因**：Nginx / CDN / 某些 PaaS 的反向代理默认开启响应缓冲，攒满一定字节才往下游转发。

**解决**：响应头加 `"X-Accel-Buffering": "no"`；若自管 Nginx，在 location 里加 `proxy_buffering off;`。这是第 5 节把它标成"必须"的原因。

---

### 坑 4：流已经开始了才报错，前端只看到"连接断开"

**现象**：后端日志有 500 异常栈，前端只触发 `onerror`，拿不到任何错误信息，而且 `EventSource` 还会**自动重连**，于是错误请求被无限重放。

**原因**：HTTP 状态码在第一个字节写出时就固定为 200 了，之后 `raise HTTPException` 改不了它。

**解决**：流中途的错误必须**降级成事件**发出去（`event: error`），前端收到后主动 `close()` 阻止重连。

---

### 坑 5：用户关了页面，你还在烧 token

**现象**：账单比预期高，日志里有大量"生成完成"但用户侧没有对应行为。

**原因**：只写了 `async for chunk in stream: yield ...`，没有任何断线检查。TCP 断开后，服务端的写操作不一定立刻报错，循环可能跑完整个生成。

**解决**：每次 `yield` 前 `if await request.is_disconnected(): return`，并在 `finally` 里 `await stream.close()`。

---

### 和直觉相反的一点

**`return` 一个生成器，不等于代码已经开始跑。**

```python
@app.get("/stream")
async def stream():
    return StreamingResponse(gen(), media_type="text/event-stream")
```

`gen()` 这一步**什么都没执行**——它只是创建了生成器对象。函数体第一行代码要等到 FastAPI 开始迭代它、也就是响应头已经发出去之后才跑。

推论：**在生成器里做参数校验是没用的**。如果 `prompt` 为空你想返回 400，必须在进入 `StreamingResponse` **之前**校验：

```python
@app.get("/stream")
async def stream(request: Request, prompt: str):
    if not prompt.strip():
        raise HTTPException(400, "prompt 不能为空")   # 这里 raise 才有效
    return StreamingResponse(_gen(...), ...)          # 这之后 raise 就没用了
```

---

## 7. 自测题

1. 服务端 `curl -N` 看数据在一条条往外流，但浏览器的 `EventSource.onmessage` 一次都没触发。可能的原因有哪两个？分别怎么验证？
   （提示：看第 6 节坑 1，以及第 4 节示例 1 对 `media_type` 的解读。）

2. 你的 `/stream` 在本地跑得好好的，部署到 Render 后用户反馈"要等 20 秒才一次性看到全部内容"。你会先检查哪一项配置？为什么本地测不出来？
   （提示：第 6 节坑 3，以及第 4 节示例 3 的 `headers` 部分。）

3. 模型生成到第 50 个字时上游抛了异常。如果你在生成器里写 `raise HTTPException(500, "失败")`，前端会观察到什么现象？为什么这个写法是错的？正确做法是什么？
   （提示：第 6 节坑 4 说明了现象与原因，第 4 节示例 3 的 `except Exception` 分支是正确写法，第 6 节最后一段解释了状态码何时被定死。）
