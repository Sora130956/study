# FastAPI 核心入门教程

```
- 目标读者：会 Python 语法，做过 FastAPI 但说不清原理（「能跑起来，但不知道为什么」）
- 前置知识：第 3 节的 ASGI vs WSGI、依赖注入、生命周期、async/def 端点差异（你多半已懂，扫术语表即可）
- 学习时长：约 60 分钟 / 可跳过：第 1、2 节是纯背景，急着上手直接进第 4 节；第 6 节坑必读
- 环境：Python 3.12+、FastAPI 0.141.1、pydantic 2.13.5、starlette 1.6.0（本文档结论均在此版本源码中核对过）
```

> 本篇不讲 SSE / 流式响应，那是单独一篇的主题；这里只在中间件一节顺带提一句它为什么和普通响应不同。

---

## 第 1 节：它是什么

**一句话**：FastAPI 是一个 Web 框架，你写一个带类型注解的普通 Python 函数，它就自动帮你把 HTTP 请求里的数据取出来、转成对的类型、校验合法性，再把你的返回值转成 JSON，顺手生成一份在线接口文档。

**生活类比**：把 FastAPI 想成餐厅的**前台点单员**。

- 客人（HTTP 请求）来了，说的话乱七八糟：有的写在纸上（请求体），有的口头报座位号（路径参数），有的补一句"不要辣"（查询参数）。
- 你（后厨 / 端点函数）只想拿到一张**格式统一、已经核对过**的工单：几号桌、菜名、辣度是数字 0-3。
- 点单员的活儿就是：**接客人的话 → 翻译成标准工单 → 客人说的不合规矩就当场退回（422），根本不进后厨** → 后厨做好菜，他再摆盘端出去（序列化成 JSON）。
- 顺带，点单员还在门口挂了块**菜单牌**，客人不用问就知道能点什么——这就是 `/docs`。

这个类比后面每一节都对得上：第 4 节里 `item_id: int` 就是"核对座位号是数字"，`response_model` 就是"摆盘时只摆该露出的东西"，`Depends` 就是"后厨共用的那口锅由谁递给你"。

---

## 第 2 节：它解决什么问题

### 反面推演：不用它会怎样

假设需求很朴素：`GET /items/42?q=book`，要求 `item_id` 必须是整数，`q` 可选。用裸 WSGI 手写大概是这样：

```python
# 朴素做法：不用框架的参数解析，全靠自己
def application(environ, start_response):
    path = environ["PATH_INFO"]                    # 拿到 "/items/42"，是个字符串
    if not path.startswith("/items/"):             # 路由靠字符串前缀判断
        start_response("404 Not Found", [("Content-Type", "application/json")])
        return [b'{"detail":"not found"}']

    raw_id = path[len("/items/"):]                 # 手动切出 "42"
    try:
        item_id = int(raw_id)                      # 手动转类型，URL 里永远是字符串
    except ValueError:                             # 转不动要自己造错误响应
        start_response("422 Unprocessable Entity", [("Content-Type", "application/json")])
        return [b'{"detail":"item_id must be an integer"}']

    from urllib.parse import parse_qs
    query = parse_qs(environ.get("QUERY_STRING", ""))   # 手动解析 ?q=book
    q = query.get("q", [None])[0]                       # parse_qs 返回列表，还要取第 0 个

    import json
    body = json.dumps({"item_id": item_id, "q": q})     # 手动序列化
    start_response("200 OK", [("Content-Type", "application/json")])
    return [body.encode()]
```

可感知的痛点：

1. **4 行业务变 20 行样板**：真正的业务只有最后拼 dict 那一行，其余全是取值、转类型、校验、造错误响应。
2. **每个接口重复一遍**：下一个接口还要再写一遍 `int()` + try/except，20 个接口就是 20 份复制粘贴。
3. **校验规则散落在函数体里**：想知道这个接口收什么参数，只能逐行读代码，没有一处声明。
4. **文档靠手维护**：接口文档写在 Wiki 里，改了代码忘了改文档，前端照着过期文档对接。
5. **错误响应格式不统一**：这个接口返回 `detail`，那个返回 `message`，前端要写两套处理。
6. **同步阻塞**：WSGI 一个请求占一个线程，接口里等一个 3 秒的 LLM 响应，这个线程就干等 3 秒。

**根源一句话**：参数的**取值、类型转换、校验、错误响应、文档**这五件事，本质上都能从「函数签名的类型注解」里推导出来，但裸写法迫使你在每个调用点手工重复。

### 经典应用场景

- **LLM / AI 服务后端**（本文主线场景）：接收 prompt、转发给模型 API、流式或整段返回，天生 IO 密集，`async` 优势最大
- 前后端分离的 JSON API：自动生成的 `/docs` 直接当接口契约给前端
- 内部微服务之间的 RPC 式 HTTP 接口
- 给已有 Python 模型 / 脚本包一层 HTTP 壳，让别的服务能调

---

## 第 3 节：读下去之前，先搞懂这些概念

> 你做过 FastAPI，本节概念多半已经懂个七八成。可以只扫结尾的**术语速查表**，有陌生的再回来看对应小节，然后直接进第 4 节。唯一强烈建议细看的是 3.8（`async def` vs `def`），它是第 6 节几个坑的共同根源。

### 3.1 ASGI vs WSGI

- **定义**：两者都是"Web 服务器"和"Python 应用"之间的接口规范。WSGI 是同步的（一次调用处理一个请求，处理完才返回），ASGI 是异步的（用 `async`，一个线程能同时挂着很多请求）。
- **大白话**：WSGI 像单窗口柜台，前一个人办完才叫下一个；ASGI 像叫号大厅，你在等材料的时候柜员先去服务别人。
- **最小示例**：

```python
async def app(scope, receive, send):   # ASGI 应用就是这么一个 async 函数（三个参数）
    ...                                # scope=本次连接信息, receive/send=收发消息的通道
```

- **为什么你要在意**：接口里"等 LLM 返回"就是纯等待。ASGI 下这个等待不占线程，一台机器能同时挂几千个等待中的请求；WSGI 下每个等待都占一个线程。

### 3.2 路由与路径操作

- **定义**：路由 = URL 和处理函数的对应关系。FastAPI 把「一个 HTTP 方法 + 一个路径 + 一个处理函数」这个组合叫**路径操作**（path operation），处理函数叫**路径操作函数**。
- **大白话**：一张"什么门牌号找谁办事"的表。
- **最小示例**：

```python
@app.get("/ping")          # 装饰器把下面这个函数登记到 GET /ping 这条路由上
def ping(): return "pong"
```

### 3.3 路径参数 vs 查询参数 vs 请求体

三者的区别只有一句话：**数据长在 HTTP 请求的哪个位置**。

| | 长在哪 | 例子 | FastAPI 怎么认出来 |
|---|---|---|---|
| 路径参数 | URL 路径里 | `/items/42` 的 `42` | 函数参数名和路径里 `{}` 中的名字一致 |
| 查询参数 | URL 问号后面 | `/items?q=book` 的 `book` | 函数参数是简单类型（int/str/bool…）且名字没出现在路径里 |
| 请求体 | HTTP 报文的 body | POST 发的那段 JSON | 函数参数的类型是 pydantic 模型 |

- **大白话**：路径参数是"几号桌"（定位到哪个资源），查询参数是"不要辣"（怎么筛选/微调），请求体是"点菜的整张单子"（一坨结构化数据）。
- **最小示例**：

```python
@app.post("/items/{item_id}")                       # item_id 在路径里 → 路径参数
def f(item_id: int, q: str | None = None, body: Item = ...):  # q 是简单类型 → 查询参数；body 是模型 → 请求体
    ...
```

**这条规则要记住**：FastAPI 靠**类型注解**来分辨，不是靠你写在第几个位置。简单类型默认当查询参数，pydantic 模型默认当请求体。

### 3.4 响应模型

- **定义**：<mark style="background: #BBFABBA6;">在装饰器上用 `response_model=` 声明"这个接口返回的数据长什么样"，FastAPI 会拿它来**校验并过滤**你的返回值，同时写进文档。</mark>
- **大白话**：出菜前的**摆盘规范**——规定盘子里该有什么，多出来的东西一律不上桌。
- **最小示例**：

```python
@app.get("/me", response_model=PublicUser)   # 声明出口格式：只有 PublicUser 里声明的字段会被返回
def me(): return db_user                     # 即使 db_user 带着 password 字段，也不会出现在响应里
```

- **注意**：这个"过滤"是它最有用也最反直觉的行为，第 6 节专门讲。

### 3.5 依赖注入（DI）

- **定义**：端点函数不自己去创建/查找它需要的东西（数据库连接、LLM 客户端、当前登录用户），而是在参数里声明"我需要这个"，由框架负责准备好递进来。
- **大白话**：你不自己去仓库领工具，写张需求单交上去，工具自动摆到你桌上。好处是**换工具不用改你的代码**——测试时递个假的进来就行。
- **最小示例**：

```python
def get_client(): return real_client              # 一个"怎么造这个东西"的函数
@app.get("/x")
def x(c = Depends(get_client)): ...               # 声明"我要 get_client 的产物"，FastAPI 调它并把结果传进来
```

### 3.6 生命周期（lifespan）

- **定义**：<mark style="background: #BBFABBA6;">应用**启动时跑一次**、**关闭前跑一次**的钩子，用来准备和释放全局资源。</mark>
- **大白话**：开店前开灯烧水，关店后熄火锁门。不是每来一个客人做一次，是一天做一次。
- **最小示例**：

```python
@asynccontextmanager
async def lifespan(app):
    yield                    # yield 之前 = 启动时执行；yield 之后 = 关闭时执行
```

### 3.7 中间件

- **定义**：<mark style="background: #BBFABBA6;">包在所有路由外面的一层，每个请求进来先过它，响应出去再过它一次。</mark>
- **大白话**：商场大门的安检 + 出门小票检查，不管你去哪个店都得过这道门。
- **最小示例**：

```python
@app.middleware("http")
async def m(request, call_next):
    response = await call_next(request)   # call_next 往里走，走完拿到响应
    return response                       # 这行前后就是"请求前"和"响应后"的插入点
```

### 3.8 `async def` 端点 vs `def` 端点（重点）

这是**很多性能问题和玄学卡顿的共同根源**，务必搞清。

- <mark style="background: #BBFABBA6;">`async def` 端点：直接跑在**事件循环**里。事件循环是单线程的，靠"遇到 `await` 就切走去干别的"来实现并发。</mark>
- <mark style="background: #BBFABBA6;">`def`（普通同步）端点：FastAPI 发现它不是协程，会把它丢进**线程池**跑（源码里就是 `run_in_threadpool`，在 `fastapi/routing.py`），这样它就算阻塞也只阻塞那个线程，不影响事件循环。</mark>

**推论（第 6 节的坑就在这）**：

- <mark style="background: #BBFABBA6;">在 `async def` 里写阻塞代码（`requests.get`、`time.sleep`、重 CPU 计算）→ **整个事件循环被卡住**，所有其他请求一起等。</mark>
- <mark style="background: #BBFABBA6;">在 `def` 里写阻塞代码 → 没事，它在线程池里，但线程池大小有限，扛不了很高并发。</mark>

```python
async def bad():  time.sleep(3)          # 灾难：3 秒内全服务无响应
async def good(): await asyncio.sleep(3) # 正确：await 期间事件循环去处理别的请求
def also_ok():    time.sleep(3)          # 可接受：普通 def 被放进线程池，只卡自己那个线程
```

一句话原则：**要么全程 `async` + 用异步库（`httpx.AsyncClient`），要么老老实实写 `def` 让框架送你去线程池。最怕的是 `async def` 里塞同步阻塞库。**

### 术语速查表

| 术语 | 大白话 | 正文哪节 |
|---|---|---|
| ASGI | 异步版的"服务器怎么调你的应用"规范 | 3.1 |
| 路径操作 | 一条路由（方法 + 路径 + 函数） | 3.2、4.1 |
| 路径参数 | URL 路径里的值，如 `/items/42` | 3.3、4.2 |
| 查询参数 | URL 问号后的值，如 `?q=book` | 3.3、4.2 |
| 请求体 | POST/PUT 发的那坨 JSON，用 pydantic 模型接 | 3.3、4.2 |
| 响应模型 | 出口格式声明，会过滤多余字段 | 3.4、4.2、6.2 |
| `HTTPException` | 主动抛出、变成对应状态码响应的异常 | 4.2 |
| 异常处理器 | 把某类异常统一翻译成 HTTP 响应 | 4.3 |
| 依赖注入 / `Depends` | 声明"我需要什么"，框架准备好递进来 | 3.5、4.3 |
| 依赖缓存 | 同一请求内同一依赖只算一次 | 4.3 展开一 |
| `yield` 依赖 | 能做善后清理的依赖（用完自动关） | 4.3 展开二 |
| lifespan | 启动/关闭各跑一次的钩子 | 3.6、4.3 展开三 |
| 中间件 | 包在所有路由外的一层拦截 | 3.7、4.3 展开四 |
| `dependency_overrides` | 测试里把真依赖换成假的开关 | 5.2 |
| 事件循环 | 单线程调度器，`async` 并发的底座 | 3.8、6.1 |

---

## 第 4 节：从最简单的代码示例开始

### 4.1 示例一：最小可运行版本

```python
# main.py
from fastapi import FastAPI          # FastAPI 是应用类，一个进程通常只建一个实例

app = FastAPI()                      # 建应用实例。它本身就是个 ASGI 应用，uvicorn 待会儿要的就是它

@app.get("/ping")                    # 装饰器：把下面的函数登记为 GET /ping 的处理函数
async def ping():                    # 用 async def，因为真实项目里这里迟早要 await 调外部服务
    return {"message": "pong"}       # 返回 dict，FastAPI 自动转成 JSON 并加 Content-Type 头
```

起服务：

```bash
uvicorn main:app --reload    # main = 文件名(不含.py)，app = 上面那个变量名；--reload 改代码自动重启（仅开发用）
```

打开 <http://127.0.0.1:8000/docs>，你会看到一个能点着试的交互式文档，`/ping` 已经在里面了。

**逐段解读**

- `app = FastAPI()`：这一个对象承担三件事——存路由表、存全局状态（`app.state`）、当 ASGI 入口被 uvicorn 调用。
- `@app.get("/ping")`：装饰器只做"登记"，不改变函数本身。等价于 `app.get("/ping")(ping)`。
- <mark style="background: #BBFABBA6;">返回 `dict`：FastAPI 会走一遍 `jsonable_encoder` 把返回值转成 JSON 能表示的结构，再包成 `JSONResponse`。你不用手写 `json.dumps`。</mark>
- `/docs` 从哪来：FastAPI 扫描所有路由的函数签名和类型注解，生成一份 OpenAPI 描述（在 `/openapi.json`），`/docs` 只是渲染它的页面。**文档不是你写的，是从代码推导的**——这就解决了第 2 节"改代码忘改文档"的痛点。

**这一步你得到了什么**：一个真能跑的 HTTP 服务，外加一份零成本、永不过期的接口文档。

### 4.2 示例二：加一个真实小需求

需求：查一个提示词模板 `GET /prompts/{prompt_id}`，可选参数 `lang` 控制语言；新建模板 `POST /prompts`，校验入参；查不到的 id 返回 404。

```python
# main.py
from fastapi import FastAPI, HTTPException      # HTTPException：抛它就直接变成一个 HTTP 错误响应
from pydantic import BaseModel, Field           # pydantic v2：BaseModel 定义数据结构，Field 加约束

app = FastAPI()

# ---------- 数据模型 ----------
class PromptIn(BaseModel):                      # 入口模型：描述"客户端该发什么"
    name: str = Field(min_length=1, max_length=50)   # 约束写在类型旁边，违反了框架自动返回 422
    template: str                                     # 必填，缺了就 422
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)  # 有默认值=可选；ge/le 限定取值范围

class PromptOut(BaseModel):                     # 出口模型：描述"我们对外返回什么"
    id: int
    name: str
    template: str                               # 注意这里故意不含 temperature 和 internal_note

# ---------- 假数据 ----------
FAKE_DB: dict[int, dict] = {
    1: {"id": 1, "name": "翻译", "template": "把下面内容翻译成{lang}：{text}",
        "temperature": 0.3, "internal_note": "内部备注，不该外泄"},
}

# ---------- 路径操作 ----------
@app.get("/prompts/{prompt_id}", response_model=PromptOut)   # {prompt_id} 声明了一个路径参数
async def get_prompt(
    prompt_id: int,                             # 名字和路径里的 {prompt_id} 一致 → 路径参数，且自动转 int
    lang: str = "zh",                           # 名字没出现在路径里、类型简单 → 查询参数，默认 "zh"
):
    item = FAKE_DB.get(prompt_id)               # 正常业务查询
    if item is None:                            # 查不到就主动抛异常
        raise HTTPException(status_code=404, detail=f"prompt {prompt_id} 不存在")
        # 抛出而不是 return：异常会被 FastAPI 捕获并转成 {"detail": "..."} 的 404 响应
    return {**item, "template": item["template"].replace("{lang}", lang)}
    # 这里返回的 dict 带着 temperature 和 internal_note，但 response_model 会把它们滤掉

@app.post("/prompts", response_model=PromptOut, status_code=201)  # 201 = Created，语义比默认 200 准确
async def create_prompt(payload: PromptIn):     # 参数类型是 pydantic 模型 → 从请求体读 JSON
    new_id = max(FAKE_DB) + 1                   # 到这一行时 payload 一定是合法的，不用再校验
    FAKE_DB[new_id] = {"id": new_id, **payload.model_dump()}  # v2 用 model_dump()，不是 v1 的 dict()
    return FAKE_DB[new_id]
```

**逐段解读**

- **两个模型分开写**（`PromptIn` / `PromptOut`）：入口和出口的字段本来就不一样——入口要 `temperature` 但没有 `id`，出口有 `id` 但不该暴露 `internal_note`。这是最常用的一组约定。
- <mark style="background: #BBFABBA6;">**`prompt_id: int` 做了三件事**：从 URL 取字符串 → 转成 int → 转不动就返回 422</mark>。第 2 节手写的 try/except 全被这一个注解替代。
- **关键执行流程**（请求 `GET /prompts/1?lang=en`）：
  1. 匹配路由 → 2. 从路径取 `"1"`、从 query 取 `"en"` → 3. 按注解校验转换 → 4. 校验失败直接 422，**你的函数根本不会被调用** → 5. 调你的函数 → 6. 返回值过一遍 `PromptOut` 校验+过滤 → 7. 序列化成 JSON。
- **`raise` 而不是 `return`**：`HTTPException` 被框架的异常处理器接住。好处是可以在任意深度的调用栈里抛（哪怕在一个被 `Depends` 注入的函数里），不用层层 return 错误码。

**原理展开：`response_model` 到底做了什么**

它不是"注释"，是一道**真实的数据出口关卡**。上面 `get_prompt` 返回的 dict 里有 `internal_note`，但客户端只会收到 `id / name / template` 三个字段。等价的普通写法是：

```python
# 没有 response_model 时，你得手写这一段
async def get_prompt(...):
    item = ...
    return {"id": item["id"], "name": item["name"], "template": item["template"]}  # 手工挑字段，容易漏
```

`response_model` 的额外好处是**顺手写进 OpenAPI 文档**，手工挑字段没这个效果。它反直觉的一面（静默丢字段）见 6.2。

**这一步你得到了什么**：参数校验、类型转换、错误响应、出口过滤、接口文档，全部由类型声明驱动，函数体里只剩业务逻辑。

### 4.3 示例三：贴近真实项目（LLM 服务的骨架）

<mark style="background: #BBFABBA6;">现在把四个机制一起用上：`lifespan` 建全局 `httpx.AsyncClient`、`Depends` 注入它、自定义异常处理器、耗时中间件。</mark>

```python
# main.py
import time
import logging
from contextlib import asynccontextmanager      # 造异步上下文管理器，lifespan 要求这个形态

import httpx
from fastapi import FastAPI, Depends, Request, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel

logger = logging.getLogger(__name__)

# ---------- 1. 自定义业务异常 ----------
class UpstreamError(Exception):                 # 纯业务异常，不带 HTTP 语义
    def __init__(self, provider: str, detail: str):
        self.provider = provider                # 存上下文，供异常处理器组装响应
        self.detail = detail

# ---------- 2. lifespan：全局资源的生与死 ----------
@asynccontextmanager                            # 必须是异步上下文管理器，FastAPI 会 async with 它
async def lifespan(app: FastAPI):
    # === yield 之前：启动阶段，整个进程只跑一次 ===
    client = httpx.AsyncClient(                 # 建连接池。复用连接省掉每次请求的 TCP+TLS 握手
        base_url="https://api.example-llm.com",
        timeout=httpx.Timeout(30.0, connect=5.0),   # LLM 慢，读超时给足；连接超时要短，快速失败
    )
    app.state.llm_client = client               # 挂到 app.state 上，端点通过 request.app.state 能拿到
    logger.info("LLM 客户端已就绪")
    yield                                       # ← 服务在这里对外提供能力，直到收到关闭信号
    # === yield 之后：关闭阶段，进程退出前跑一次 ===
    await client.aclose()                       # 必须关：否则连接池泄漏，日志里会警告未关闭的连接
    logger.info("LLM 客户端已关闭")

app = FastAPI(lifespan=lifespan)                # 把 lifespan 交给应用（不要用旧的 @app.on_event）

# ---------- 3. 依赖：把全局客户端"递"给端点 ----------
async def get_llm_client(request: Request) -> httpx.AsyncClient:
    # 依赖函数可以声明 Request 参数，框架会把本次请求对象传进来
    client = getattr(request.app.state, "llm_client", None)
    if client is None:                          # 防御：lifespan 没跑（比如误用 TestClient 不触发时）
        raise RuntimeError("LLM 客户端未初始化，检查 lifespan 是否生效")
    return client

# ---------- 4. 模型 ----------
class ChatIn(BaseModel):
    prompt: str
    max_tokens: int = 512

class ChatOut(BaseModel):
    reply: str

# ---------- 5. 端点 ----------
@app.post("/chat", response_model=ChatOut)
async def chat(
    payload: ChatIn,                                        # 请求体
    client: httpx.AsyncClient = Depends(get_llm_client),    # 注入依赖，注意 Depends(fn) 不加括号调用 fn
):
    try:
        resp = await client.post("/v1/chat", json=payload.model_dump())  # await：等待期间事件循环去干别的
        resp.raise_for_status()                             # 4xx/5xx 抛 httpx.HTTPStatusError
    except httpx.HTTPError as exc:                          # 网络层/状态码错误统一转成业务异常
        raise UpstreamError(provider="example-llm", detail=str(exc)) from exc
    return {"reply": resp.json()["choices"][0]["message"]["content"]}

# ---------- 6. 异常处理器：把业务异常翻译成 HTTP 响应 ----------
@app.exception_handler(UpstreamError)           # 注册：凡是端点里漏出来的 UpstreamError 都走这里
async def upstream_error_handler(request: Request, exc: UpstreamError):
    logger.warning("上游失败 provider=%s detail=%s", exc.provider, exc.detail)
    return JSONResponse(                        # 必须返回一个 Response 对象，不能 return dict
        status_code=502,                         # 502 Bad Gateway：语义是"我没错，我依赖的上游错了"
        content={"error": "upstream_unavailable", "provider": exc.provider},
    )                                            # 注意不把 exc.detail 透给客户端，避免泄漏内部信息

# ---------- 7. 中间件：记录每个请求的耗时 ----------
@app.middleware("http")
async def add_timing_header(request: Request, call_next):
    start = time.perf_counter()                 # perf_counter 是单调时钟，适合测时间差
    response = await call_next(request)          # 往内层走：过其余中间件 → 依赖 → 端点，拿回响应
    elapsed_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Process-Time-Ms"] = f"{elapsed_ms:.1f}"   # 只能改头，body 已经定型
    logger.info("%s %s -> %s %.1fms", request.method, request.url.path, response.status_code, elapsed_ms)
    return response                              # 必须把响应返回，否则客户端收不到任何东西
```

**逐段解读**

- **`lifespan`（第 2 段）**：`yield` 把函数劈成两半——上半启动、下半关闭。客户端在这里建**一次**，被所有请求共享。
- **`app.state`**：一个随便挂属性的容器（starlette 的 `State` 类）。它和请求无关，是应用级的。
- **`get_llm_client`（第 3 段）**：这个依赖不"创建"东西，只是**取出**已有的客户端。这点很关键——如果在依赖里 `httpx.AsyncClient()`，就变成每个请求建一个连接池，白白浪费握手开销。
- **`Depends(get_llm_client)`（第 5 段）**：传的是**函数对象**，不是调用结果。FastAPI 在处理请求时才调它。
- **异常处理器（第 6 段）**：端点只管抛"上游挂了"这个**业务事实**，"该返回什么状态码、藏哪些细节"由处理器统一决定。业务代码里不出现 HTTP 细节。
- **中间件（第 7 段）**：`await call_next(request)` 之前是"请求进来时"，之后是"响应出去前"。

#### 原理展开一：`Depends` 的缓存语义（重点，这是你的验证目标）

**结论先说**：

> **同一个请求内**，同一个依赖函数默认**只执行一次**，后续声明它的地方复用第一次的结果。
> **跨请求绝不复用**——下一个请求会从头再算一遍。

看例子：

```python
call_count = 0
def get_token():                                # 一个有副作用的依赖，方便观察它被调了几次
    global call_count
    call_count += 1
    return f"token-{call_count}"

def dep_a(t: str = Depends(get_token)): return t   # A 依赖 get_token
def dep_b(t: str = Depends(get_token)): return t   # B 也依赖 get_token

@app.get("/demo")
async def demo(
    a: str = Depends(dep_a),
    b: str = Depends(dep_b),
    t: str = Depends(get_token),                # 第三次声明同一个依赖
):
    return {"a": a, "b": b, "t": t}             # 三个值完全相同，get_token 本次请求只跑了 1 次
```

一次请求返回 `{"a": "token-1", "b": "token-1", "t": "token-1"}`。再请求一次，全变成 `token-2`。

**为什么**（源码级机制，在 `fastapi/dependencies/utils.py` 的 `solve_dependencies`）：

1. 处理每个请求时，框架建一个**空的 `dependency_cache` 字典**——注意是每个请求新建，这就是"跨请求不复用"的原因。
2. 解析依赖树时，每个依赖算一个 **cache key**。在 0.141.1 里这个 key 是 `(依赖函数对象, OAuth scopes, scope)`（见 `dependencies/models.py` 的 `_get_cache_key`）。
3. 命中 key 就直接取缓存值，不再调用函数；没命中就调用，然后把结果写进缓存。

**关键推论**：缓存的判定依据是**函数对象本身**。所以：

```python
# 同一个函数对象 → 共享缓存
Depends(get_token)        # 和别处的 Depends(get_token) 命中同一 key

# 不同函数对象 → 各算一次，哪怕代码一模一样
Depends(lambda: uuid4())  # 每个 lambda 都是新对象，永不互相命中
```

**关掉缓存**：

```python
async def demo(
    t1: str = Depends(get_token, use_cache=False),   # 显式要求每次都重新执行
    t2: str = Depends(get_token, use_cache=False),
): ...                                                # t1 != t2，get_token 被调了两次
```

什么时候要 `use_cache=False`？依赖每次调用应当产出不同值时（如生成唯一请求 ID、取当前精确时间）。绝大多数情况保持默认。

**一句话记住**：`Depends` 的缓存是**请求级**的，键是**函数对象**。它不是性能优化的全局缓存，而是保证"同一请求内同一份数据的一致性"——所以不要指望它跨请求缓存昂贵计算（那要用 `lru_cache` 或外部缓存）。

> 小注：本版本的 `Depends` 还有一个 `scope` 参数（`"function"` / `"request"`），影响 `yield` 依赖的清理时机。默认行为就是上面描述的，入门阶段不用动它。

#### 原理展开二：`yield` 依赖的清理时机

<mark style="background: #BBFABBA6;">依赖函数可以用 `yield` 代替 `return`，这样就能写"用完之后的善后"：</mark>

```python
async def get_session():
    session = Session()                 # yield 之前：准备阶段，每个请求执行
    try:
        yield session                   # ← 把它交给端点使用，函数在这里挂起
    finally:
        await session.close()           # yield 之后：善后阶段，请求处理完才执行
```

**清理什么时候跑？** 顺序是：

1. <mark style="background: #BBFABBA6;">依赖的 `yield` 之前部分执行 → 2. 端点函数执行 → 3. 响应被生成 → 4. **`yield` 之后的清理代码执行** → 5. 响应发给客户端。</mark>

关键点：**清理跑在端点结束之后**，由框架用一个 `AsyncExitStack` 统一管理（`fastapi/routing.py` 里能看到 `async with AsyncExitStack()`）。多个 `yield` 依赖按**后进先出**（栈序）清理，和嵌套 `with` 一致。

<mark style="background: #BBFABBA6;">**端点抛异常怎么办？** 清理照样执行——所以 `finally` 里的资源释放是可靠的。想在依赖里感知异常，就写 `except`：</mark>

```python
async def get_session():
    session = Session()
    try:
        yield session
    except Exception:                   # 端点抛出的异常会在这里被重新抛出一次
        await session.rollback()        # 所以能做回滚这种"出错才做"的善后
        raise                           # 必须 raise，否则异常被吞掉，客户端拿到诡异的 500
    finally:
        await session.close()
```

**等价的普通写法**（不用 `yield` 依赖，你得自己在每个端点里写 `async with`）：

```python
@app.post("/x")
async def x():
    async with Session() as session:     # 每个端点都要重复这一行
        ...                              # 而且要用它的地方全都得缩进一层
```

`yield` 依赖的价值就是把这个 `async with` 提到函数签名里，端点体保持扁平。

**和 `lifespan` 的分工**：`lifespan` 管**进程级**、活整个应用生命周期的资源（`AsyncClient` 连接池）；`yield` 依赖管**请求级**、用完就该释放的资源（数据库会话、事务）。别把 `AsyncClient` 放进 `yield` 依赖——那等于每个请求建一个连接池。

#### 原理展开三：`lifespan` 为什么取代 `@app.on_event`

旧写法（**已废弃，不要在新代码里用**）：

```python
@app.on_event("startup")            # 废弃
async def startup():
    app.state.client = httpx.AsyncClient()

@app.on_event("shutdown")           # 废弃
async def shutdown():
    await app.state.client.aclose()
```

被取代的原因，按重要性排：

1. **启动和关闭逻辑被拆到两个函数里**，共享资源只能靠全局变量或 `app.state` 传递。`lifespan` 用一个函数的局部变量就串起来了——`client` 在 `yield` 前后是同一个局部变量，不用往外存。
2. **没有异常安全保证**。`on_event` 的两个回调是独立的：启动成功了但后面某步失败，shutdown 该不该跑、跑到哪一步，语义模糊。`lifespan` 是标准上下文管理器，`yield` 外面套 `try/finally` 就能拿到 Python 原生的异常保证。
3. **配对关系是语言层面强制的**。上下文管理器天然"进就必出"，不会出现"忘了写对应的 shutdown"。
4. **对齐 ASGI 规范**。lifespan 本来就是 ASGI 协议里的一个事件类型，starlette 的实现直接 `async with self.lifespan_context(app)`（见 `starlette/routing.py`），`on_event` 只是早期在它之上包的一层便利 API。

顺带一个版本细节：starlette 现在只接受 `@asynccontextmanager` 装饰的形态，直接传裸的 async 生成器函数会报废弃警告。老老实实加装饰器。

#### 原理展开四：中间件与依赖的执行顺序

<mark style="background: #BBFABBA6;">一个请求的完整旅程（外 → 内 → 外）：</mark>

```
请求进来
  ↓ 中间件 1（call_next 之前的代码）
  ↓ 中间件 2（call_next 之前的代码）
  ↓ 路由匹配
  ↓ 依赖解析（Depends，yield 之前的部分）
  ↓ 【端点函数执行】
  ↑ response_model 校验 + 过滤 + 序列化
  ↑ 依赖清理（yield 之后的部分，后进先出）
  ↑ 中间件 2（call_next 之后的代码）
  ↑ 中间件 1（call_next 之后的代码）
响应出去
```

**必须记住的几条**：

1. **中间件在依赖外面**。所以中间件里**拿不到**依赖的结果——它跑的时候依赖还没解析（进来时）或已经清理完（出去时）。要在中间件里用某个东西，只能从 `request` 上自己取。
2. **注册顺序和执行顺序相反**。多个 `@app.middleware("http")`，**最后注册的最先执行**（像穿衣服，最后穿的在最外层）。
3. **中间件里的 `HTTPException` 不会被异常处理器接住**。异常处理器装在路由层内侧，中间件抛异常会直接变成 500。要在中间件里返回错误，自己 `return JSONResponse(...)`。
4. **中间件拿到的是完整响应对象**，但 body 已经组装好了，改 body 要重新构造响应（流式响应下更麻烦，这也是 SSE 场景要小心中间件的原因，本篇不展开）。

一个常见的错误认知：以为"依赖比中间件先跑，因为依赖写在函数签名上"。反了——**中间件永远在最外层**。

**这一步你得到了什么**：一个结构完整的 LLM 服务骨架——全局连接池只建一次、端点通过声明拿到它、错误被统一翻译、每个请求都有耗时记录。并且你现在能解释清楚每个机制在请求生命周期里的确切位置。

---

## 第 5 节：最小可用的实际用法（生产模板）

### 5.1 可直接复制的服务骨架

注释里标了 **[必须]** 和 **[可选]**，可选的删掉服务照样跑。

```python
# app/main.py
import logging
import os
import time
from contextlib import asynccontextmanager

import httpx
from fastapi import Depends, FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)


# ---------- 业务异常 ----------
class UpstreamError(Exception):                          # [可选] 不做统一错误处理可删
    def __init__(self, detail: str) -> None:
        self.detail = detail


# ---------- 生命周期 ----------
@asynccontextmanager                                     # [必须] lifespan 必须是异步上下文管理器
async def lifespan(app: FastAPI):
    client = httpx.AsyncClient(                          # [必须] 全局复用一个连接池
        base_url=os.environ["LLM_BASE_URL"],             # [必须] 配置从环境变量来，别硬编码
        headers={"Authorization": f"Bearer {os.environ['LLM_API_KEY']}"},  # [必须] 密钥只在这里读一次
        timeout=httpx.Timeout(60.0, connect=5.0),        # [必须] 一定要设超时，否则请求可能永久挂住
        limits=httpx.Limits(max_connections=100),        # [可选] 限制并发连接数，防打爆上游
    )
    app.state.llm_client = client                        # [必须] 存起来供依赖取用
    try:
        yield                                            # [必须] 分界线：上面启动，下面关闭
    finally:
        await client.aclose()                            # [必须] try/finally 保证异常时也会关闭


app = FastAPI(
    title="LLM Gateway",                                 # [可选] 只影响文档标题
    lifespan=lifespan,                                   # [必须]
)


# ---------- 依赖 ----------
async def get_llm_client(request: Request) -> httpx.AsyncClient:   # [必须]
    return request.app.state.llm_client                  # 只取不建，保证共享同一个连接池


# ---------- 模型 ----------
class ChatIn(BaseModel):                                 # [必须] 入口模型
    prompt: str = Field(min_length=1, max_length=8000)   # [可选] 长度上限，防超大请求打爆上游
    max_tokens: int = Field(default=512, ge=1, le=4096)  # [可选] 但强烈建议，防刷爆 token 成本


class ChatOut(BaseModel):                                # [必须] 出口模型
    reply: str
    model: str


# ---------- 端点 ----------
@app.post("/chat", response_model=ChatOut)               # [必须]
async def chat(
    payload: ChatIn,
    client: httpx.AsyncClient = Depends(get_llm_client),
) -> dict:
    try:
        resp = await client.post("/v1/chat/completions", json={
            "model": "gpt-4o-mini",
            "messages": [{"role": "user", "content": payload.prompt}],
            "max_tokens": payload.max_tokens,
        })
        resp.raise_for_status()
    except httpx.HTTPError as exc:
        raise UpstreamError(str(exc)) from exc            # from exc 保留原始堆栈，方便排查
    data = resp.json()
    return {"reply": data["choices"][0]["message"]["content"], "model": data["model"]}


@app.get("/health")                                       # [必须] 容器/K8s 存活探针要用
async def health() -> dict:
    return {"status": "ok"}                               # 保持极简：不要在这里调上游，否则上游抖动会导致自己被重启


# ---------- 全局异常处理 ----------
@app.exception_handler(UpstreamError)                     # [可选] 但生产建议有
async def handle_upstream_error(request: Request, exc: UpstreamError) -> JSONResponse:
    logger.warning("upstream failed: %s", exc.detail)     # 细节只进日志
    return JSONResponse(status_code=502, content={"error": "upstream_unavailable"})  # 不外泄细节


# ---------- 中间件 ----------
@app.middleware("http")                                   # [可选] 观测用
async def timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time-Ms"] = f"{(time.perf_counter() - start) * 1000:.1f}"
    return response
```

跑起来：

```bash
export LLM_BASE_URL=https://api.openai.com LLM_API_KEY=sk-xxx   # 密钥走环境变量，不要进代码库
uvicorn app.main:app --host 0.0.0.0 --port 8000                 # 生产环境去掉 --reload
```

> ⚠️ 这个骨架**没有任何认证**。`/chat` 是公开的，谁都能调，而它背后是你付费的 LLM API。真上线前至少加一层 API Key 校验（一个 `Depends` 就能做）或放在带鉴权的网关后面，否则等于把你的 token 账单开放给互联网。

### 5.2 测试里用 `dependency_overrides` 换 fake（进阶）

`Depends` 最实用的回报在测试环节。FastAPI 在应用上挂了一个 `dependency_overrides` 字典（源码 `fastapi/applications.py`），**键是原依赖函数，值是替身函数**。解析依赖时框架先查这张表，命中就用替身。

```python
# tests/conftest.py
import httpx
import pytest
from fastapi.testclient import TestClient

from app.main import app, get_llm_client


@pytest.fixture                                  # 默认 function scope：每个测试独立，不共享状态
def client(monkeypatch: pytest.MonkeyPatch):
    monkeypatch.setenv("LLM_BASE_URL", "https://fake-llm.test")   # 环境变量一律用 monkeypatch
    monkeypatch.setenv("LLM_API_KEY", "test-key")                 # 禁止直接改 os.environ

    fake_client = httpx.AsyncClient(base_url="https://fake-llm.test")
    app.dependency_overrides[get_llm_client] = lambda: fake_client  # 键 = 原依赖函数对象
    with TestClient(app) as c:                   # with 会触发 lifespan（不进 with 则 lifespan 不跑）
        yield c
    app.dependency_overrides.clear()             # 必须清理，否则污染后续测试
```

配合 `respx` 在 HTTP 层拦截上游（符合工作区 `CLAUDE.md` 的 mock 边界纪律：测业务逻辑用依赖注入 fake，测上游调用用 respx）：

```python
# tests/test_chat.py
import respx
from httpx import Response


@respx.mock(assert_all_mocked=True)              # 未被路由覆盖的请求直接报错，保证不打真实网络
async def test_chat_returns_reply(respx_mock, client):
    respx_mock.post("https://fake-llm.test/v1/chat/completions").mock(
        return_value=Response(200, json={
            "model": "gpt-4o-mini",
            "choices": [{"message": {"content": "你好"}}],
        })
    )

    resp = client.post("/chat", json={"prompt": "hi"})

    assert resp.status_code == 200
    assert resp.json() == {"reply": "你好", "model": "gpt-4o-mini"}   # 断言业务结果，不是 mock 值本身
```

两个要点：

- **键必须是那个函数对象本身**。写 `app.dependency_overrides["get_llm_client"]`（字符串）不会生效——和 4.3 缓存 key 用函数对象是同一个道理。
- **`TestClient` 要用 `with`**，否则 lifespan 不触发，`app.state.llm_client` 不存在。这是"明明本地能跑，测试里报 AttributeError"的常见原因。

**这一步你得到了什么**：一个能进真实项目的最小服务，以及把真依赖换成 fake 的测试入口。

---

## 第 6 节：常见坑与易错点

### 6.1 在 `async def` 端点里调阻塞代码

**现象**：单个请求自己测没问题；一有并发，**所有**接口一起变慢，连 `/health` 都要等好几秒才响应。没有报错，只是全体卡住。

**触发代码**：

```python
import requests, time

@app.post("/chat")
async def chat(payload: ChatIn):
    r = requests.post(url, json=...)   # ❌ requests 是同步库，会阻塞整个事件循环
    time.sleep(1)                       # ❌ 同理，卡死事件循环
    return r.json()
```

**原因**：`async def` 端点直接跑在事件循环里，而事件循环是**单线程**的。它靠"遇到 `await` 就切走处理别的请求"实现并发。`requests.post` 和 `time.sleep` 都不是 `await` 点——它们在 C 层面把线程整个占住，事件循环没有任何机会切走。于是这 1 秒内，**所有**等待中的请求都停摆。LLM 服务尤其致命，因为一次调用动辄几秒到几十秒。

**解决**（按优先级）：

```python
# 方案 1（首选）：换异步库
@app.post("/chat")
async def chat(payload: ChatIn, client: httpx.AsyncClient = Depends(get_llm_client)):
    r = await client.post(url, json=...)      # ✅ await 期间事件循环去处理别的请求
    await asyncio.sleep(1)                     # ✅ 异步版 sleep

# 方案 2：确实只有同步库可用，就把端点写成普通 def
@app.post("/chat")
def chat(payload: ChatIn):                     # ✅ 去掉 async，框架自动送进线程池
    return requests.post(url, json=...).json() # 只阻塞线程池里的一个线程

# 方案 3：async 端点里必须调同步函数（如重 CPU 计算）
from starlette.concurrency import run_in_threadpool

@app.post("/chat")
async def chat(payload: ChatIn):
    result = await run_in_threadpool(heavy_sync_function, payload.prompt)  # ✅ 显式丢进线程池
    return result
```

**自查方法**：在 `async def` 端点里搜有没有 `requests.`、`time.sleep`、同步数据库驱动（`psycopg2`、`pymysql`）、`open()` 读大文件。有就是隐患。

### 6.2 `response_model` 静默过滤掉没声明的字段

**现象**：没有报错，但客户端收到的 JSON **少了字段**。你在端点里 `print` 返回值明明是全的，接口出去就少了。

**触发代码**：

```python
class ChatOut(BaseModel):
    reply: str                                  # 只声明了 reply

@app.post("/chat", response_model=ChatOut)
async def chat(...):
    return {"reply": "你好", "usage": {"total_tokens": 42}}  # usage 会被静默丢掉
```

客户端实际收到 `{"reply": "你好"}`，`usage` 消失且**没有任何警告**。

**原因**：`response_model` 不只做校验，它会用这个模型**重新构造**响应数据——只保留模型中声明的字段。这是设计意图（防止误泄漏 `password`、`internal_note` 这类字段），但当你只是忘了声明新字段时，它的静默行为就很坑。

**解决**：把要返回的字段补进模型。

```python
class Usage(BaseModel):
    total_tokens: int

class ChatOut(BaseModel):
    reply: str
    usage: Usage                                # ✅ 声明了才会出现在响应里
```

**顺带一个实用配置**：想让"值为 None 的可选字段"不出现在响应里，用 `response_model_exclude_none=True`：

```python
@app.post("/chat", response_model=ChatOut, response_model_exclude_none=True)
```

### 6.3 `Depends()` 忘了写、或写成了调用

这是三个相近的错，症状完全不同。

```python
# 错法 A：漏了 Depends
@app.post("/chat")
async def chat(client: httpx.AsyncClient):      # ❌ 没有 Depends
    ...
```

**报错**：`fastapi.exceptions.FastAPIError: Invalid args for response field! Hint: check that <class 'httpx.AsyncClient'> is a valid Pydantic field type`
**原因**：没有 `Depends`，FastAPI 就把它当成普通请求参数，试图用 pydantic 解析 `AsyncClient` 类型 —— 解析不了。

```python
# 错法 B：写成了调用
async def chat(client = Depends(get_llm_client())):   # ❌ 多了一对括号
    ...
```

**原因**：`get_llm_client()` 在**导入模块时**就被执行了，`Depends` 收到的是它的返回值（或直接报错，因为此时 `request` 参数缺失）。`Depends` 要的是**函数对象**，由框架在处理请求时调用。

```python
# 错法 C：默认值写成裸的 Depends 类
async def chat(client = Depends):                     # ❌ 没有括号，传的是类本身
    ...
```

**解决**：牢记形态是 `参数名: 类型 = Depends(函数名)` —— **`Depends` 有括号，里面的函数名没括号**。

用 `Annotated` 写法可以彻底避免这类混淆，也是现在官方推荐的形态：

```python
from typing import Annotated

LLMClient = Annotated[httpx.AsyncClient, Depends(get_llm_client)]   # 定义一次，到处复用

@app.post("/chat")
async def chat(payload: ChatIn, client: LLMClient):   # ✅ 干净，且类型检查器能正确推断
    ...
```

### 6.4 在 `lifespan` 外创建全局 `AsyncClient`

**现象**：本地起服务时能用；但在某些环境下（多 worker、测试套件、`asyncio.run` 调用）报
`RuntimeError: Event loop is closed` 或 `Future attached to a different loop`，也可能表现为连接莫名挂死。

**触发代码**：

```python
# main.py 模块顶层
client = httpx.AsyncClient(base_url="...")      # ❌ 在导入时创建

@app.post("/chat")
async def chat(...):
    r = await client.post(...)                   # 可能绑在一个已经关闭的事件循环上
```

**原因**：`AsyncClient` 内部的连接池、锁等对象会**绑定到创建它时所在的事件循环**。模块顶层代码在**导入时**执行，那时 uvicorn 的事件循环还没建立（或是另一个循环）。等真正处理请求时，用的是另一个循环，两边对不上。而且这个客户端**永远不会被关闭**，进程退出时会留下未关闭连接的警告。

**解决**：所有需要事件循环的资源，都在 `lifespan` 里创建和释放。

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    client = httpx.AsyncClient(base_url="...")   # ✅ 此时事件循环已就绪且是正确的那个
    app.state.llm_client = client
    try:
        yield
    finally:
        await client.aclose()                    # ✅ 同一个循环里正常关闭
```

**判断规则**：一个对象如果内部有 `asyncio` 的锁、队列、连接池，就不能在模块顶层建。`httpx.AsyncClient`、异步数据库连接池、`redis.asyncio` 客户端全都属于这一类。

### 6.5 中间件里抛 `HTTPException` 得不到预期响应

**现象**：中间件里 `raise HTTPException(status_code=401, ...)`，客户端收到的是 **500 Internal Server Error**，不是 401。

**原因**：`HTTPException` 靠异常处理器翻译成响应，而异常处理器装在**路由层内侧**（见 4.3 展开四的顺序图）。中间件在更外层，它抛的异常已经越过了处理器的作用范围，只能由服务器兜底成 500。

**解决**：中间件里直接返回响应对象，不要抛异常。

```python
@app.middleware("http")
async def require_api_key(request: Request, call_next):
    if request.url.path != "/health" and request.headers.get("X-API-Key") != EXPECTED:
        return JSONResponse(status_code=401, content={"error": "unauthorized"})  # ✅ 直接返回
    return await call_next(request)
```

更好的做法：认证这类需求用 `Depends` 实现，而不是中间件——依赖跑在路由内侧，可以正常抛 `HTTPException`，还能自动写进 OpenAPI 文档。

---

## 第 7 节：验证你学会了

不看正文作答。每题标了答案在哪一节。

**第 1 题（复述）**
用你自己的话回答，不许用"高性能""现代化"这类空词：

1. FastAPI 到底替你做了哪几件具体的事？至少说出 5 件。
2. 第 2 节那段裸 WSGI 代码里，哪些行在 FastAPI 版本里彻底消失了？为什么能消失？
3. `def` 端点和 `async def` 端点，框架分别怎么执行它们？

> 答案在第 1、2 节和 3.8

**第 2 题（默写）**
不看文档，从空文件写出一个服务，要求：

1. `lifespan` 里创建一个 `httpx.AsyncClient` 并在关闭时释放
2. 一个依赖函数把这个客户端取出来
3. 一个 `POST /chat` 端点，用 pydantic 模型接请求体、用 `response_model` 声明出口、通过 `Depends` 拿到客户端
4. 一个 `/health`
5. 写完后回答：`@asynccontextmanager` 能不能省？`Depends(get_client)` 里的 `get_client` 能不能加括号？

> 答案在 4.3、第 5 节、6.3

**第 3 题（组合应用，两个必答的原理题）**

**(a) `Depends` 的对象什么时候被复用？** 下面这段代码，回答三个问题：

```python
counter = 0
def get_request_id():
    global counter
    counter += 1
    return counter

def dep_a(rid: int = Depends(get_request_id)): return rid
def dep_b(rid: int = Depends(get_request_id)): return rid

@app.get("/demo")
async def demo(
    a: int = Depends(dep_a),
    b: int = Depends(dep_b),
    c: int = Depends(get_request_id),
    d: int = Depends(get_request_id, use_cache=False),
):
    return {"a": a, "b": b, "c": c, "d": d}
```

1. 第一次请求 `/demo`，返回的四个值分别是什么？`get_request_id` 一共被调用了几次？
2. 紧接着第二次请求，四个值是什么？为什么和第一次不同？
3. 如果把 `dep_b` 改成 `def dep_b(rid: int = Depends(lambda: get_request_id())): return rid`，答案会变吗？为什么？
4. 用一句话说出缓存的**键**是什么、缓存的**生存期**是多长。

> 答案在 4.3 原理展开一

**(b) 为什么 `async def` 里不能调 `requests.get`？**

1. 解释具体机制：事件循环在这一刻发生了什么？为什么会影响到**其他**请求？
2. 如果这个接口 QPS 只有 1，还有问题吗？为什么并发一上来才暴露？
3. 给出三种改法，说明各自适用什么场景。
4. 下面哪几个也有同样的问题？`time.sleep(1)` / `await asyncio.sleep(1)` / `psycopg2` 查询 / `json.loads(小字符串)` / 一个跑 3 秒的 for 循环。

> 答案在 3.8 和 6.1

**进阶自查（选做）**
你的测试里，`app.dependency_overrides` 的键为什么必须是函数对象而不能是字符串？这和第 3 题 (a) 的缓存机制是同一个道理吗？

> 答案在 5.2 和 4.3 原理展开一






