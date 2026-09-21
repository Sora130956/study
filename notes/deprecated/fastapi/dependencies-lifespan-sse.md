# FastAPI 依赖注入、生命周期与 SSE 流式响应 教程

> **来源:** [FastAPI 官方文档](https://fastapi.tiangolo.com/)（Lifespan Events / Dependencies with yield / Handling Errors / Custom Response / Stream Data / Server-Sent Events 六章，2026-09-07 实际访问校准）
> **对应计划周次:** 第 1 周 · 周三 上+下午 · 计划原文「依赖注入（`Depends`）、生命周期（`lifespan`）、异常处理器 / SSE 流式响应：`StreamingResponse` + 生成器，浏览器验证逐字输出」
> **关键版本:** FastAPI 0.141.x（PyPI 当前最新 0.141.1，2026-07-29 发布）· Python 3.10+ · pydantic v2。内置 SSE 支持（`fastapi.sse`）需 **0.135.0+**；`response_class=StreamingResponse` 直接 yield 写法需 **0.134.0+**
> **生成日期:** 2026-09-07

---

## 为什么学这个

先看一个场景：你接到第一个 AI 单子，要做一个「调 LLM 生成回复」的接口。写着写着会发现四个绕不开的问题——

1. **LLM 客户端放哪？** 每个请求都 `httpx.AsyncClient()` 新建一个，等于每次都重新建 TCP 连接、重新 TLS 握手，并发一上来又慢又容易被限流。但它又不能写在函数里——这就是**生命周期（lifespan）**管的事：整个应用启动时创建一次、共享给所有请求、关闭时释放。
2. **接口函数怎么拿到它？** 全局变量当然可以，但测试时没法替换、没法按配置切换（W3 多模型适配就死在这）。FastAPI 的答案是**依赖注入（Depends）**：声明「我需要什么」，框架负责给。
3. **上游 LLM 超时、429 限流了怎么办？** 不能把 httpx 的原始报错直接甩给客户。**异常处理器**把这些统一翻译成干净的 JSON 错误响应。
4. **LLM 生成一段回复要 20 秒，用户盯着转圈？** 把回复拆成 token 逐个推送——**SSE 流式响应**。ChatGPT 那种逐字输出效果，后端就是它。

这四样东西合起来，恰好就是计划里 `ai-service-template` 骨架的完整形态：**全局资源（lifespan）→ 注入使用（Depends）→ 出错兜底（异常处理器）→ 流式吐出（SSE）**。W2 的项目一（流式对话 API）只需要在这个骨架上把「假 LLM」换成真 SDK。

衔接已有知识：你已经知道 `async def` / `await` / `async for`（见 [asyncio 教程](../python/asyncio-tutorial.md) 第 6 章——`async for` 正是本章流式输出的语法基础），也用过 pydantic v2 模型。本章不再解释这些。

---

## 重点标注（星级导航）

> 星级标准：★★★ 必须精通（学完当天能默写关键代码）｜★★☆ 需要理解（能讲清楚原理，细节可查）｜★☆☆ 了解即可（用到再回来看）

| 章节 | 星级 | 为什么（对照学习目的/计划周次） |
| ---- | ---- | ---- |
| 第 1 章 依赖注入 `Depends` | ★★★ | W1 验证点原文直接考「`Depends` 的对象什么时候被复用」；项目一/二/三的每个接口都靠它注入 LLM 客户端；W3「多模型适配靠配置切换」就是换一个依赖函数的事 |
| 第 2 章 生命周期 `lifespan` | ★★★ | W1 交付物骨架的核心：全局 `httpx.AsyncClient` 连接池在这里创建，是 W2 调 LLM API 的正确姿势；不知道它你就只能用全局变量，测试和部署都会翻车 |
| 第 3 章 异常处理器 | ★★☆ | W2「错误处理：超时、429、退避」之后的收尾动作——统一错误格式；交付级 API 必备，但模式固定，理解一次即可照抄 |
| 第 4 章 SSE 流式响应 | ★★★ | W1 周三下午作业原文「浏览器验证逐字输出」；W2 把 SDK 的 `stream=True` 接到这里（项目一）；W6 RAG 查询管线流式返回、W7 前端流式展示——整个 P1/P2 都踩在它上面 |
| 综合示例 | ★★★ | 当天结束时这个文件就是 `ai-service-template` 的雏形，建议逐行默写一遍 |

**一日学习路线（对应 W1 周三一天）**：

- 上午：第 1 章 → 第 2 章（这两章互相咬合，一起学）
- 下午：第 4 章为主 → 综合示例在浏览器里跑通 → 第 3 章收尾
- 延后：第 4 章中「心跳与断线重连」「Last-Event-ID」细节到 W2 对接真实 LLM 再回头看

---

## 第 1 章：依赖注入 `Depends` ★★★

### 1.1 概念解释

**问题引入**：写一个调 LLM 的接口，你需要 API key、base_url、超时配置、HTTP 客户端、token 计价表……这些「外围依赖」如果全靠手动传参或全局变量，会越长越乱：

```python
# 手动传参：每个接口都要重复这一坨，换配置要改 N 处
@app.post("/chat")
async def chat(req: ChatRequest):
    client = httpx.AsyncClient(headers={"Authorization": f"Bearer {API_KEY}"})
    ...
```

**依赖注入（Dependency Injection，DI）** 的思路反过来：你不去找依赖，而是**声明**依赖，框架在请求进来时把它准备好递给你。FastAPI 里就是 `Depends`：

- **依赖（dependency）**：一个普通函数（或类），返回值就是「被注入的东西」。
- **注入点**：接口函数（FastAPI 术语叫 *path operation function*，路径操作函数）的参数上写 `Depends(那个函数)`。
- FastAPI 在**每次请求**处理前先执行依赖函数，把返回值塞进你的参数。

一个关键区分（W1 验证点原文）：

> **`Depends` 默认带请求内缓存**——同一个依赖在**同一个请求**里被多个参数/接口引用时，函数只执行一次，对象被复用；但**跨请求**每次都重新执行。想禁用缓存用 `Depends(dep, use_cache=False)`。

这套机制和 Spring 的 `@Autowired` 是同一个思想（你写过 Java，可以这么对应：依赖函数 ≈ `@Bean` 方法，`Depends` 参数 ≈ 构造器注入），只是 FastAPI 不需要容器配置，函数即依赖。

### 1.2 代码示例

推荐用 `Annotated` 写法（官方推荐，类型提示对编辑器友好）：

```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

# ---- 依赖 1：从配置构造 LLM 请求头 ----
def llm_headers() -> dict[str, str]:
    return {"Authorization": "Bearer sk-xxx"}

# ---- 依赖 2：依赖可以嵌套（子依赖）----
def llm_client(headers: Annotated[dict, Depends(llm_headers)]) -> dict:
    # 实际项目里这里返回配置好的客户端对象，第 2 章会改成从 app.state 取
    return {"base_url": "https://api.deepseek.com", "headers": headers}

@app.post("/chat")
async def chat(client: Annotated[dict, Depends(llm_client)]):
    # 参数 client 已被框架填好，函数体里直接用
    return {"using": client["base_url"]}

# 同一请求内多个依赖引用 llm_headers，它只执行一次（缓存）
@app.post("/summarize")
async def summarize(
    h1: Annotated[dict, Depends(llm_headers)],
    h2: Annotated[dict, Depends(llm_headers)],
):
    return {"same_object": h1 is h2}  # True
```

**带 `yield` 的依赖（本章重点）**：普通依赖只能「造东西」，`yield` 依赖还能「收尾」——`yield` 前是准备代码，`yield` 出去的值被注入，接口返回**之后**再执行 `yield` 后的清理代码。请求级资源（会话、锁、per-request 统计）的标准写法：

```python
async def usage_tracker():
    usage = {"prompt_tokens": 0, "cost_usd": 0.0}
    try:
        yield usage          # 注入给接口用
    finally:
        # 响应发送完毕后执行：W2 的「成本与 token 统计接口」就从这里落库
        print(f"request cost: {usage['cost_usd']:.6f} USD")

@app.post("/chat")
async def chat(usage: Annotated[dict, Depends(usage_tracker)]):
    usage["prompt_tokens"] += 120
    usage["cost_usd"] += 0.0002
    return {"ok": True}
```

补充一个校准到的新参数（知道即可）：`Depends(dep, scope="function")` 可以让清理代码提前到**响应发送之前**执行（默认 `"request"` 是发送之后）。AI 应用里日志统计用默认即可。

### 1.3 ⚠️ 常见坑

1. **`yield` 依赖里 `except` 后必须 re-raise**。接口里抛的异常会传进依赖的 `yield` 处，你 `except` 住却不 `raise`，客户端照样收到 500，但服务端**没有任何日志**——错误被吞了。规则：`except` 里要么 `raise HTTPException(...)` 转译，要么裸 `raise` 重新抛出。
2. **一个依赖只 `yield` 一次**，写两个 `yield` 行为未定义。
3. 依赖函数用 `async def` 还是 `def` 都行——内部有 `await` 就必须 `async def`（如 `await client.get(...)`）。
4. 别把「每次请求执行一次」误解为「全局单例」。想跨请求复用（如 HTTP 连接池），看第 2 章。

**坑 1 的最小复现**（可直接跑，亲眼看日志差异）：

```python
from typing import Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

class LLMError(Exception):
    """调上游 LLM 时的内部错误"""

# ❌ 吞异常版：except 住但没 raise
def usage_tracker_swallows():
    usage = {"tokens": 0}
    try:
        yield usage
    except LLMError:
        print("[cleanup] 出错了，但我把异常吞了")   # 注意：没有 raise！

@app.get("/chat-bad")
async def chat_bad(usage: Annotated[dict, Depends(usage_tracker_swallows)]):
    raise LLMError("上游 LLM 返回 500")   # 接口里抛的异常，会被送进依赖的 yield 处

# ✅ 正确版：裸 raise，把异常放回传递链
def usage_tracker_raises():
    usage = {"tokens": 0}
    try:
        yield usage
    except LLMError:
        print("[cleanup] 记录中……然后把异常放回去")
        raise

@app.get("/chat-good")
async def chat_good(usage: Annotated[dict, Depends(usage_tracker_raises)]):
    raise LLMError("上游 LLM 返回 500")
```

跑起来后分别 `curl -i http://127.0.0.1:8000/chat-bad` 和 `/chat-good`，对比现象：

| | `/chat-bad`（吞掉） | `/chat-good`（re-raise） |
| ---- | ---- | ---- |
| 客户端收到 | 500 `Internal Server Error` | 一样是 500 |
| 服务端终端 | 只有 `[cleanup] 出错了…` 一行，**没有任何 Traceback** | `[cleanup] …` + 完整 `ERROR: Exception in ASGI application` 堆栈，直接指到抛出那行 |

为什么客户端都是 500、差别只在日志？因为接口函数里抛的异常不是凭空消失，而是被 FastAPI 沿原路「送回」每个 yield 依赖的 `yield` 处（Python 生成器机制：异常被 throw 进生成器）。你的 `except` 是这条传递链的最后一站——**接住不还回去，链就断在这，堆栈证据随之蒸发**，线上排障等于盲飞；还回去（裸 `raise`），uvicorn 才有机会打印完整堆栈。如果想顺便给客户一个能看懂的错误，就转译：`raise HTTPException(status_code=503, detail="LLM 上游不可用") from e`——这时客户端收到的就是 503 + 干净 JSON，而不是 500。

---

## 第 2 章：生命周期 `lifespan` ★★★

### 2.1 概念解释

**问题引入**：第 1 章说过，`Depends` 每个请求都重新执行。但 HTTP 客户端这种重资源（内部是连接池，建连接 + TLS 握手成本高，LLM API 调用又全是它）应该**整个应用只建一次**。三种放法对比：

| 放法 | 问题 |
| ---- | ---- |
| 模块顶层 `client = httpx.AsyncClient()` | import 即创建；跑个不碰网络的单元测试也要背着它；无法优雅关闭 |
| 每个请求里 `async with httpx.AsyncClient()` | 每次重建连接，高并发下慢且浪费，官方明确不推荐 |
| **`lifespan` 里创建，注入时取用** | 启动建一次，请求共享，关闭时 `aclose()` |

**`lifespan`**（生命周期）就是 FastAPI 给「应用级启动/关闭逻辑」的官方入口：用一个 `@asynccontextmanager` 装饰的异步函数，`yield` **之前**的代码在**收到第一个请求之前**执行一次（startup），`yield` **之后**的代码在**应用关闭时**执行一次（shutdown）——和第 1 章 `yield` 依赖是同一个模式，只是作用域从「一个请求」放大到「整个应用」。

**版本注意（校准结果）**：老教程常见的 `@app.on_event("startup")` / `@app.on_event("shutdown")` **已弃用**（deprecated），官方现在只推荐 `lifespan` 写法。看到旧代码要认识，新代码不要写。

### 2.2 代码示例

标准姿势：资源挂到 `app.state` 上，再用第 1 章的 `Depends` 取出来——两章在此合流：

```python
from contextlib import asynccontextmanager
import httpx
from fastapi import FastAPI, Request, Depends
from typing import Annotated

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ---- startup：应用启动、接客之前，执行一次 ----
    app.state.http = httpx.AsyncClient(timeout=30.0)  # W2 起 LLM 调用全走它
    try:
        yield  # 应用在这里「运行」，期间处理所有请求
    finally:
        # ---- shutdown：应用关闭，执行一次 ----
        await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)

# 依赖：从 app.state 取全局客户端 —— 接口拿到的就是它
def get_http_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.http

@app.get("/health")
async def health(client: Annotated[httpx.AsyncClient, Depends(get_http_client)]):
    # health 里顺便确认全局客户端已就绪（演示注入；实际项目 health 通常不依赖它）
    return {"status": "ok", "client_ready": client is not None}
```

这个 `lifespan + app.state + Depends` 三件套是 `ai-service-template` 的骨架，后面 W2 的 LLM SDK 客户端、W4+ 的 pgvector 连接池，全都放同一个位置。

### 2.3 ⚠️ 常见坑

1. **忘了 `await client.aclose()`**：shutdown 时连接没关，本地没感觉，部署平台日志里会出现 «Unclosed connection» 警告，客户看到会怀疑你的工程质量。用 `try/finally` 包住 `yield` 保证异常退出也清理。
2. **`lifespan` 只对主应用生效**，挂载（mount）的子应用不会执行——目前用不到，知道即可。
3. `app.state.http` 是动态属性，编辑器没有类型提示。介意的话可以定义 `request.app.state.http` 的访问函数（就是上面的 `get_http_client`），类型提示集中在依赖里。
4. startup 阶段抛异常（比如数据库连不上），应用会启动失败——这是特性不是 bug：坏配置宁可起不来，别带病接客。

---

## 第 3 章：异常处理器 ★★☆

### 3.1 概念解释

**问题引入**：接口里调上游 LLM，会撞上超时（`httpx.TimeoutException`）、限流（429）、服务端错误（5xx）。这些异常如果不管，客户端收到的是 FastAPI 默认的 `Internal Server Error`——客户既不知道该重试还是该报错，你也定位不了问题。

FastAPI 的错误处理分两层：

- **`HTTPException`（HTTP 异常）**：业务代码里**主动抛出**的标准错误。注意是 `raise` 不是 `return`——它是异常，抛出后当前请求立即中断，FastAPI 自动转成 `{"detail": ...}` 的 JSON 响应，状态码自选。
- **异常处理器（exception handler）**：对**某一类异常**注册的全局翻译函数，`@app.exception_handler(SomeException)`。任何地方（包括你没 catch 的深层调用）抛出这类异常，都会被拦下，翻译成统一格式的响应。

类比你熟悉的 Spring：`HTTPException` ≈ `ResponseStatusException`，异常处理器 ≈ `@ControllerAdvice` + `@ExceptionHandler`。

### 3.2 代码示例

AI 应用的典型做法：定义一个业务异常表示「上游 LLM 出问题」，全局统一翻译成 503 + 固定格式：

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import httpx

app = FastAPI()

class LLMProviderError(Exception):
    """上游 LLM 不可用：超时 / 429 / 5xx"""

@app.exception_handler(LLMProviderError)
async def llm_error_handler(request: Request, exc: LLMProviderError):
    # 统一错误格式：客户的前端只需要处理一种结构
    return JSONResponse(
        status_code=503,
        content={"error": "llm_unavailable", "detail": str(exc), "retry_after": 5},
    )

@app.post("/chat")
async def chat():
    try:
        ...  # 这里调 LLM
    except httpx.TimeoutException as e:
        raise LLMProviderError(f"llm timeout: {e}") from e
```

再补一个兜底的 `Exception` 处理器，接住所有不认识的异常（bug、第三方库报错）：

```python
import logging

logger = logging.getLogger("app")

@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception):
    # 必须自己记 traceback：异常在你这层被"处理"后就不再冒泡，服务器默认的堆栈日志就没有了
    logger.exception("unhandled error on %s %s", request.method, request.url.path)
    return JSONResponse(
        status_code=500,
        content={"error": "internal_error", "detail": "服务器内部错误，请稍后再试"},
        # detail 不放 str(exc)：未知异常的内容可能含 SQL、路径等内部信息
    )
```

简单解释：不加这个处理器框架也会兜底（Starlette 最外层返回 500 纯文本），加它的目的是**统一 JSON 错误格式 + 主动记日志 + 不泄露内部细节**。匹配时按继承链找最具体的处理器——`LLMProviderError` 走自己的 503 处理器，不会被 `Exception` 兜底抢先，两者放心共存。分层原则：**认识的异常 detail 可以给客户端看，不认识的只说「内部错误」、堆栈进日志**。

两个校准到的细节：

- 注册 `HTTPException` 的自定义处理器时，要用 **Starlette 的** `HTTPException`（FastAPI 的那个继承自它），这样 Starlette 内部抛的也能接住：`from starlette.exceptions import HTTPException as StarletteHTTPException`。
- `HTTPException` 的 `detail` 可以是 dict/list，不只字符串——错误响应想带结构化信息直接塞。

### 3.3 ⚠️ 常见坑

1. **SSE 流式响应开始后，异常处理器救不了你**——状态码 200 已经发出去了，中途想改回 500 不可能。正确做法是把错误当成一个 SSE 事件发出去（见 4.2）。这是新手做流式接口最常见的认知错位。
2. 不要用异常处理器掩盖 bug：`except Exception` 一把梭再返回 200，错误全被埋掉。只翻译你**认识**的异常。

---

## 第 4 章：SSE 流式响应 ★★★

### 4.1 概念解释

**问题引入**：LLM 生成 300 字回复要 15 秒。普通接口的语义是「算完 → 一次性返回」，用户前 15 秒只能看转圈。但想想真实的 LLM 服务：模型其实是一个 token 一个 token 生成的——服务器**手里早就有一半答案了**，只是普通响应没机会先吐出来。

HTTP 本身支持「响应体慢慢发」：服务器把响应切成小块（chunk）逐个发送。**SSE（Server-Sent Events，服务器发送事件）** 就是在这上面定的一层薄格式。不抽象描述了，直接看一份完整的 SSE 响应——假设你问「你好」，服务器逐字回答：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: 你

data: 好

data: [DONE]

```

对照着看，规则只有三条：

- **`Content-Type: text/event-stream`**：一个普通的响应头取值，作用类似 `application/json` 声明「body 是 JSON」——它声明「body 是一串事件，连接别关，后面还有」。浏览器看到这个值，就不会等响应「结束」才处理，而是来一块处理一块。
- **响应体是纯文本，每条事件以空行结尾**。上面 body 里其实有 3 条事件，空行就是事件之间的「句号」。每条事件由「字段: 值」一行一个组成：最常用 `data:`（载荷）；可选 `event:`（事件类型）、`id:`（断线重传用）、`retry:`（重连间隔毫秒），先只用 `data:` 就够。
- **浏览器按空行把文本切开，每块就是一条完整事件**。

`EventSource` 是**浏览器内置的 JS API**（W3C 标准，所有现代浏览器都有，不用装任何库），它替你做了「保持连接 + 按空行切事件 + 解析字段」三件事。「统一 API」的意思就是：后端不管用什么语言什么框架，只要按上面的格式发文本，前端代码永远是同一份：

```javascript
const es = new EventSource('/chat?prompt=你好');  // 发起 GET 并保持连接
es.onmessage = (e) => console.log(e.data);        // 每收到一条完整事件触发一次
```

服务器发的原始文本和 JS 收到的数据一一对应：

| 服务器发出（原始文本） | 浏览器里发生的事 |
| ---- | ---- |
| `data: 你` + 空行 | `onmessage` 触发，`e.data === "你"` |
| `data: 好` + 空行 | `onmessage` 再触发，`e.data === "好"` |
| `data: [DONE]` + 空行 | `onmessage` 又触发，你的代码认出结束哨兵，调 `es.close()` |

另外 `EventSource` 自带断线自动重连（这就是 `id:` 字段存在的意义，W2 之后用到再看）。

**对比 WebSocket**：SSE 是**单向**（服务器 → 客户端）、纯 HTTP、浏览器自动重连；WebSocket 是双向、独立协议。对话场景里用户发消息走普通 POST，回复流回来走 SSE——单向够用且简单得多，ChatGPT 早期网页版就是这么做的。**AI 应用 95% 的「流式」需求 = SSE**。

**生成器（generator）**是服务端的另一半：带 `yield` 的函数是「生产一块、被消费一块」，`StreamingResponse` 负责把生成器吐出的每一块立即写给客户端，而不是攒齐。你在 asyncio 教程第 6 章学的 `async for` 在这里直接用上。

> **`AsyncIterator` 为什么必须配 `async for`**：普通 `for` 每次迭代调的是同步的 `__next__()`——它不能 `await`，取值这一刻必须立刻有值。而异步生成器的下一个值可能还在网络上（上游 LLM 的下一个 token 还没到），「取下一个值」这个动作本身就是一次 IO，必须能挂起等待。`async for` 每次迭代等价于 `await it.__anext__()`，这个 `await` 就是挂起点：**等下一个 token 的几十毫秒里，事件循环被让出来处理别的请求**——这正是流式接口能高并发的根本原因：每条连接大部分时间在「等下一个 token」，等待期间不占执行资源。如果对异步生成器用普通 `for`，会直接报 `TypeError: 'async_generator' object is not iterable`。

### 4.2 代码示例

**写法一（计划指定，经典通用）：`StreamingResponse` + 异步生成器**

直接上真实版——调一个 OpenAI 兼容的 LLM API（DeepSeek / 通义均可），把它的流式输出转发给浏览器：

```python
import json
from collections.abc import AsyncIterator

import httpx
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

LLM_API_KEY = "sk-xxx"                      # 实际项目从 .env 读（W1 周五的 pydantic-settings）
LLM_BASE_URL = "https://api.deepseek.com"   # DeepSeek / 通义均兼容 OpenAI 格式

async def llm_stream(prompt: str) -> AsyncIterator[str]:
    """调真实 LLM，逐 token 产出。"""
    async with httpx.AsyncClient(base_url=LLM_BASE_URL, timeout=60.0) as client:
        async with client.stream(               # stream()：边收边读，不等 body 收完
            "POST",
            "/v1/chat/completions",
            headers={"Authorization": f"Bearer {LLM_API_KEY}"},
            json={
                "model": "deepseek-chat",
                "messages": [{"role": "user", "content": prompt}],
                "stream": True,                 # 打开上游的流式开关
            },
        ) as resp:
            resp.raise_for_status()
            async for line in resp.aiter_lines():   # 上游 body 一行一行到达
                if not line.startswith("data: "):
                    continue                        # 跳过事件之间的空行
                data = line.removeprefix("data: ")
                if data == "[DONE]":                # 上游自己的结束哨兵
                    break
                # 每行 data 后面是一条 JSON，这个 token 藏在 choices[0].delta.content
                delta = json.loads(data)["choices"][0]["delta"]
                if content := delta.get("content"):  # 首个块可能只有 role 没有 content
                    yield content

@app.get("/chat")
async def chat(prompt: str) -> StreamingResponse:
    async def sse() -> AsyncIterator[str]:
        async for token in llm_stream(prompt):
            yield f"data: {token}\n\n"          # 把 token 重新包成 SSE 发给浏览器
        yield "data: [DONE]\n\n"
    return StreamingResponse(sse(), media_type="text/event-stream")
```

看懂这段代码的一个关键发现：**上游 LLM 的流式接口本身就是 SSE**——`stream: True` 之后，它返回的也是 `Content-Type: text/event-stream`，也是 `data: {...}\n\n` 一条一条发，结束时也发 `data: [DONE]`。所以你的服务器实际在做「SSE 翻译」：解析上游的 SSE（`aiter_lines` + `json.loads`），取出 token，再包成自己的 SSE 发给浏览器。`[DONE]` 这个「约定俗成的哨兵」就是从 OpenAI 系 API 来的。

两个实现细节：

- **`client.stream(...)` 是必须的**：如果用普通的 `client.post()`，httpx 会把整个 body 收完才返回——上游流式白开了，你这边又变回「转圈 20 秒」。
- 为突出主线，这里每请求新建 client、裸抛错误；交付版应换成第 2 章的 lifespan 全局 client + 第 1 章 Depends 注入 + 第 3 章 `LLMProviderError` 转译（综合示例演示了怎么拼起来）。

W2 实际更可能用官方 SDK，等价的生成器只有几行（SDK 替你做了 SSE 解析）：

```python
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=LLM_API_KEY, base_url=LLM_BASE_URL)

async def llm_stream_sdk(prompt: str) -> AsyncIterator[str]:
    stream = await client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": prompt}],
        stream=True,
    )
    async for chunk in stream:
        if content := chunk.choices[0].delta.content:
            yield content
```

本地没有 API key 也能验证流式效果：把 `llm_stream` 换成一个「`await asyncio.sleep(0.05)` + 逐字 `yield`」的假生成器即可（综合示例里就是可跑的假 LLM 版），接口层代码一行不用改。

**写法二（FastAPI 0.135.0+ 内置，新项目推荐）**：`fastapi.sse` 模块自带 `EventSourceResponse`，`yield` 什么它自动按 SSE 格式编码，不用手拼 `\n\n`：

```python
from collections.abc import AsyncIterable
from fastapi import FastAPI
from fastapi.sse import EventSourceResponse, ServerSentEvent

app = FastAPI()

@app.get("/chat2", response_class=EventSourceResponse)
async def chat2(prompt: str) -> AsyncIterable[ServerSentEvent]:
    async for token in llm_stream(prompt):      # 复用写法一的 llm_stream
        yield ServerSentEvent(raw_data=token)   # raw_data：原文发送，不 JSON 编码
    yield ServerSentEvent(raw_data="[DONE]")
```

校准到的要点：`ServerSentEvent` 的 `data=` 会把值 JSON 编码后再发（适合结构化增量），`raw_data=` 原样发送（适合纯文本 token 和 `[DONE]` 哨兵），两者互斥；`event=`/`id=`/`retry=` 可选。此外它还支持读浏览器重连时带回的 `Last-Event-ID` 请求头做断点续传（W2 之后有需要再看官方 SSE 章）。

**浏览器验证（计划原文要求「浏览器验证逐字输出」）**：`EventSource` 是浏览器原生 API，五行 JS 就能验证。给服务加个测试页：

```python
from fastapi.responses import HTMLResponse

@app.get("/", response_class=HTMLResponse)
async def index():
    return """
    <html><body><div id="out"></div>
    <script>
      const out = document.getElementById('out');
      const es = new EventSource('/chat?prompt=RAG');
      es.onmessage = e => {
        if (e.data === '[DONE]') { es.close(); return; }
        out.append(e.data);          // 看到文字一点点长出来 = 验证通过
      };
    </script></body></html>
    """
```

`uvicorn main:app` 起服务，浏览器开 `http://127.0.0.1:8000/`，应该看到文字**逐个蹦出来**而不是一次性出现。也可以直接 `curl -N "http://127.0.0.1:8000/chat?prompt=hi"` 看原始事件流。

### 4.3 ⚠️ 常见坑

1. **忘写 `\n\n`（写法一）**：SSE 以空行分割事件。只 `yield f"data: {token}"` 不带 `\n\n`，浏览器 `onmessage` 一个都不会触发——事件永远「没结束」。排查时用 `curl -N` 看原始输出。
2. **`media_type` 不对**：不设 `text/event-stream`，响应被当普通文本，`EventSource` 直接报错。
3. **`EventSource` 只支持 GET**：浏览器的原生限制。对话类接口要 POST（带消息体）时，前端改用 `fetch` + `ReadableStream` 手动解析（W7 项目二前端会做），后端两种写法都不用改。
4. **反向代理缓冲**：Nginx 等代理默认会攒一块再转发，把流式效果吃掉。部署时加响应头 `X-Accel-Buffering: no`（Render/Fly.io 一般不用管）。
5. **长连接被掐**：中间网关常在 30–60s 强制断开空闲连接。对话一般几十秒内结束问题不大；更长的流（agent 中间步骤）W9 再补心跳（周期性发 `: ping\n\n` 注释行）。
6. **生成器里的异常**：状态码 200 已发出（见 3.3），只能 `yield "data: {\"error\": ...}\n\n"` 把错误作为事件发出，前端 `onmessage` 里识别并处理。

---

## 综合示例

一个文件串起全部四章：lifespan 管全局客户端 → Depends 注入 → 异常处理器兜底 → SSE 流式输出 → 浏览器验证页。这就是 `ai-service-template` 的雏形：

```python
# app.py —— 运行：uvicorn app:app --reload，然后浏览器开 http://127.0.0.1:8000/
import asyncio
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Annotated

import httpx
from fastapi import Depends, FastAPI, HTTPException, Request
from fastapi.responses import HTMLResponse, JSONResponse, StreamingResponse

# ---------- 第 2 章：生命周期 ----------
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(timeout=30.0)   # startup：全局连接池
    try:
        yield
    finally:
        await app.state.http.aclose()                  # shutdown：释放

app = FastAPI(lifespan=lifespan)

# ---------- 第 1 章：依赖注入 ----------
def get_http_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.http

async def usage_tracker():
    """yield 依赖：请求结束后统计成本（W2 接真实 API 时在此落库）"""
    usage = {"tokens": 0}
    try:
        yield usage
    finally:
        print(f"[usage] tokens={usage['tokens']}")

# ---------- 第 3 章：异常处理器 ----------
class LLMProviderError(Exception):
    """上游 LLM 不可用"""

@app.exception_handler(LLMProviderError)
async def llm_error_handler(request: Request, exc: LLMProviderError):
    return JSONResponse(status_code=503,
                        content={"error": "llm_unavailable", "detail": str(exc)})

# ---------- 第 4 章：SSE 流式 ----------
async def fake_llm_stream(prompt: str) -> AsyncIterator[str]:
    """假 LLM：逐字输出。W2 换成真实 SDK 后接口零改动。"""
    if not prompt.strip():
        raise HTTPException(status_code=422, detail="prompt 不能为空")
    reply = f"你问的是「{prompt}」。这是一段模拟的流式回复——能看到它逐字出现，说明 SSE 通了。"
    for token in reply:
        await asyncio.sleep(0.04)
        yield token

@app.get("/chat")
async def chat(
    prompt: str,
    client: Annotated[httpx.AsyncClient, Depends(get_http_client)],  # 演示注入
    usage: Annotated[dict, Depends(usage_tracker)],
) -> StreamingResponse:
    async def sse() -> AsyncIterator[str]:
        try:
            async for token in fake_llm_stream(prompt):
                usage["tokens"] += 1
                yield f"data: {token}\n\n"
            yield "data: [DONE]\n\n"
        except Exception as e:                     # 流中错误只能当事件发（3.3 坑 1）
            yield f'data: {{"error": "{type(e).__name__}"}}\n\n'
    return StreamingResponse(sse(), media_type="text/event-stream",
                             headers={"X-Accel-Buffering": "no"})

@app.get("/", response_class=HTMLResponse)
async def index():
    return """
    <html><head><meta charset="utf-8"><title>SSE demo</title></head>
    <body><div id="out" style="font-size:18px"></div>
    <script>
      const out = document.getElementById('out');
      const es = new EventSource('/chat?prompt=什么是RAG');
      es.onmessage = e => {
        if (e.data === '[DONE]') { es.close(); out.append(' ✔'); return; }
        out.append(e.data);
      };
    </script></body></html>
    """
```

**验证清单**：① 浏览器逐字输出；② `curl -N "http://127.0.0.1:8000/chat?prompt="` 返回 422 JSON（`HTTPException` 路径）；③ Ctrl+C 停服务，终端先打出 `[usage] tokens=...` 再退出（yield 依赖收尾 + shutdown 清理）。三条都过，W1 周三的内容就闭环了。

---

## 附录 A：API 速查表

| API / 写法 | 作用 | 关键参数 / 注意点 |
| ---- | ---- | ---- |
| `Annotated[T, Depends(dep)]` | 声明依赖注入（官方推荐写法） | 同一请求内默认缓存复用；`use_cache=False` 关闭 |
| `def dep(): yield x` / `async def dep(): yield x` | yield 依赖：请求级资源 + 收尾 | 只 yield 一次；`except` 后必须 re-raise；清理代码在响应发送后执行（`scope="function"` 可提前） |
| `@asynccontextmanager` + `async def lifespan(app)` | 应用级启动/关闭逻辑 | `yield` 前 = startup，后 = shutdown；传给 `FastAPI(lifespan=...)`；`on_event` 已弃用 |
| `app.state.xxx` | 挂全局资源 | lifespan 里赋值，依赖里 `request.app.state.xxx` 取 |
| `raise HTTPException(status_code, detail)` | 主动抛 HTTP 错误 | `detail` 可以是 dict/list |
| `@app.exception_handler(Exc)` | 全局异常 → 统一响应 | 注册 `HTTPException` 处理器用 Starlette 版本；救不了已开始的流式响应 |
| `StreamingResponse(gen, media_type="text/event-stream")` | 流式响应（经典写法） | `gen` 为（异步）生成器；每块自己拼 `data: ...\n\n` |
| `response_class=EventSourceResponse`（0.135+） | 内置 SSE（新写法） | 直接 `yield` 数据；自动 SSE 编码 |
| `ServerSentEvent(data=..., raw_data=..., event=..., id=..., retry=...)` | 一条 SSE 事件 | `data` JSON 编码、`raw_data` 原文，互斥 |
| `new EventSource(url)` + `es.onmessage` | 浏览器消费 SSE | 仅 GET；自动重连；结束记得 `es.close()` 防重连 |

## 附录 B：官方文档精确链接

以下均为本次校准（2026-09-07）实际访问过的页面：

- Lifespan Events：<https://fastapi.tiangolo.com/advanced/events/>
- Dependencies with yield：<https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/>
- Handling Errors：<https://fastapi.tiangolo.com/tutorial/handling-errors/>
- Custom Response（含 `StreamingResponse`）：<https://fastapi.tiangolo.com/advanced/custom-response/>
- Stream Data（0.134+ 直接 yield 写法）：<https://fastapi.tiangolo.com/advanced/stream-data/>
- Server-Sent Events（0.135+ 内置 SSE）：<https://fastapi.tiangolo.com/tutorial/server-sent-events/>
- 版本确认：PyPI <https://pypi.org/project/fastapi/>（0.141.1）· Release Notes <https://fastapi.tiangolo.com/release-notes/>
