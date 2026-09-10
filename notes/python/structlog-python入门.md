# structlog 入门：把日志从"流水账"变成"表格"

- 目标读者：纯新手（刚学 Python，对日志也陌生）
- 前置知识：见第 3 节（日志、结构化数据、关键字参数、上下文、绑定）
- 学习时长：约 40 分钟 / 可跳过：第 2 节后半段"应用场景"纯属背景，赶时间可跳

---

## 第 1 节：它是什么

**一句话**：structlog 是一个 Python 库，帮你把程序运行过程记成"一张张填好字段的表格"，而不是"一段段文字"。

**类比**：想象你开了一家店，要记录每天发生的事。

- 普通日志（`print` / 标准 `logging`）像**写日记**：`"今天下午 3 点，用户张三登录了，IP 是 1.2.3.4"`。人读得懂，但你想"把所有张三的记录挑出来"时，得逐句翻、逐句找"张三"两个字。
- structlog 像**填表格**：一行一条记录，每列一个字段——`时间 | 事件 | 用户 | IP`。想查"张三"？直接筛"用户"这一列就行，机器一秒搞定。

后面代码里你会看到，structlog 的每一行日志，本质就是"一个事件名 + 一堆 `字段=值`"。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 反面推演：不用它会怎样

假设你在写一个网站后端，用户登录时要记一条日志。最朴素的做法：

```python
# 朴素做法：直接 print 一段文字
print(f"用户 {user_id} 在 {time} 登录了，IP 是 {ip}")
```

程序跑起来没问题，但很快你会撞上这些**可感知的痛点**：

1. **想查数据查不动**。老板问"昨天有多少个 `user_id=123` 的登录？"你只能 `grep "123"` 那段文字，结果把"IP 是 123.45.6.7"也一起捞出来了——因为字段边界丢了，`123` 到底是用户还是 IP 的一部分，机器分不清。
2. **改格式要改十处**。今天想给每条日志加个"服务器名"，你得把项目里**每一处** `print` 都翻出来改一遍。
3. **日志进不了监控系统**。现在流行把日志灌进 ELK、Grafana 这类工具做统计图表。这些工具只认"字段=值"的结构化数据，你那段文字它只能当一坨字符串存着，没法按字段画图。

### 问题根源（一句话）

**信息被"揉成一段文字"，字段和字段之间的边界丢了。** 机器要的是"用户是 123、IP 是 1.2.3.4"这种分好格子的数据，而不是一句人话。

### 经典应用场景

- **Web 后端**（FastAPI / Django / Flask）：给每个请求记日志，带上 `user_id`、`request_id`、`path`、`耗时`。
- **微服务**：多个服务之间靠日志串起一次完整调用链，字段统一才好关联。
- **数据管道 / 定时任务**：跑批处理时记下"处理到第几条、成功几条、失败几条"。

---

## 第 3 节：读下去之前，先搞懂这些概念

这一节是"看得懂"的关键。正文代码里出现的每个机制，这里都先讲清。

### 3.1 什么是"日志"（log）

- **定义**：程序运行过程中，把"发生了什么"记下来，方便事后排查。
- **大白话**：程序不会说话，日志就是它写给你的"运行记录"，出 bug 时靠它还原现场。
- **最小示例**：`print("程序启动了")` —— 这就是最朴素的日志。

### 3.2 什么是"结构化数据"（structured data）

- **定义**：把信息拆成一个个有名字的字段，而不是一段连续文字。
- **大白话**：表格 vs 日记。`{"用户": "张三", "IP": "1.2.3.4"}` 是结构化的；`"张三从 1.2.3.4 登录"` 不是。
- **最小示例**：`{"user_id": 123, "ip": "1.2.3.4"}` —— 一个字典，两个字段。

### 3.3 什么是"关键字参数"（keyword argument）

- **定义**：调用函数时，用 `名字=值` 的方式传参。
- **大白话**：不靠位置猜，而是"点名"传值，谁是谁一目了然。
- **最小示例**：`log.info("登录", user_id=123)` —— `user_id=123` 就是关键字参数。

### 3.4 什么是"上下文"（context）

- **定义**：一段代码运行时所处的"背景信息"，比如"当前是哪个用户在操作"。
- **大白话**：同一件事，在不同背景下结果不同。日志里"谁触发的"就是上下文。
- **最小示例**：处理请求时，`user_id` 就是这段代码的上下文。

### 3.5 什么是"绑定"（bind）

- **定义**：把某个字段"钉"在日志对象上，之后每次记日志都自动带上它。
- **大白话**：像在表格表头先写死一列"用户=张三"，之后每填一行，这列自动填好，不用每行重写。
- **最小示例**：`log = log.bind(user_id=123)`，之后 `log.info("下单")` 会自动带上 `user_id=123`。

### 术语速查表

| 术语 | 大白话解释 | 出现在正文哪节 |
|------|-----------|--------------|
| 日志 log | 程序写的运行记录 | 第 4 节所有示例 |
| 结构化数据 | 拆成字段的数据，不是一段文字 | 第 4 节示例 1 |
| 关键字参数 | `名字=值` 传参 | 第 4 节示例 1 |
| 上下文 context | 当前"谁在操作"这类背景 | 第 4 节示例 2 |
| 绑定 bind | 把字段钉在日志对象上，自动带上 | 第 4 节示例 2 |
| 处理器 processor | 日志输出前，依次经过的一串"加工步骤" | 第 4 节示例 3 |
| 渲染器 renderer | 决定日志最终长什么样（文字 or JSON） | 第 4 节示例 3 |

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本

先让 structlog"活着"——记一条带字段的日志。

```python
# 第 1 步：导入 structlog 库
import structlog

# 第 2 步：拿到一个"日志对象"，后面都用它来记日志
# get_logger() 是 structlog 的入口，返回一个能记日志的对象
log = structlog.get_logger()

# 第 3 步：记一条日志
# "user_logged_in" 是"事件名"，说明发生了什么
# 后面 user_id=123、ip="1.2.3.4" 是这条日志的"字段"，用关键字参数传
log.info("user_logged_in", user_id=123, ip="1.2.3.4")
```

运行后，终端会输出类似：

```
2024-01-01 12:00:00 [info     ] user_logged_in              user_id=123 ip=1.2.3.4
```

**逐段解读**：

- **第 1 步**：`import structlog` 把库引进来，和 `import math` 一个道理。
- **第 2 步**：`get_logger()` 返回一个日志对象，存进变量 `log`。这个对象就是你的"表格本"，之后所有记录都往它上面写。
- **第 3 步**：`log.info(...)` 是"记一条 info 级别的日志"。第一个参数 `"user_logged_in"` 是**事件名**（这条日志在说什么事），后面 `user_id=123`、`ip="1.2.3.4"` 是**字段**（这件事的细节）。

**关键点展开——为什么字段要写成 `名字=值`？**

对比一下两种写法：

```python
# 写法 A：揉成一段文字（普通 logging 的常见做法）
log.info(f"用户 {user_id} 登录了，IP 是 {ip}")

# 写法 B：拆成字段（structlog 的做法）
log.info("user_logged_in", user_id=user_id, ip=ip)
```

写法 A 里，`user_id` 和 `ip` 被拼进了一句话，机器想单独拿"IP 是多少"得去拆字符串。写法 B 里，`user_id` 和 `ip` 是两个独立字段，机器直接 `记录["ip"]` 就能拿到。**这就是 structlog 的核心价值：字段不揉进文字，而是独立存在。**

**这一步你得到了什么**：你学会了 structlog 最核心的写法——`log.info("事件名", 字段=值, ...)`。字段是独立的，机器能直接按字段查。

---

### 示例 2：加一个真实小需求（绑定上下文）

真实项目里，一个请求从头到尾会记很多条日志，每条都要带上"是哪个用户"。如果每条都手写 `user_id=123`，既啰嗦又容易漏。用 `bind` 解决：

```python
import structlog

# 拿到日志对象
log = structlog.get_logger()

# 关键：把 user_id 这个字段"钉"在日志对象上
# bind() 会返回一个"新"的日志对象，它记住了 user_id=123
# 之后用这个新对象记日志，每条都会自动带上 user_id=123
log = log.bind(user_id=123)

# 下面两条日志，都不用再手写 user_id 了，它会自动出现
log.info("user_logged_in", ip="1.2.3.4")   # 自动带 user_id=123
log.info("order_created", order_id=999)    # 自动带 user_id=123
```

运行后输出类似：

```
2024-01-01 12:00:00 [info     ] user_logged_in              user_id=123 ip=1.2.3.4
2024-01-01 12:00:00 [info     ] order_created               user_id=123 order_id=999
```

**逐段解读**：

- `log = log.bind(user_id=123)` 这行是重点。`bind` 的意思是"绑定"——把 `user_id=123` 这个字段**钉**进日志对象。它返回一个**新对象**（原来的 `log` 不变），所以要用 `log =` 重新接住。
- 之后两条 `log.info(...)` 里，你**没写** `user_id`，但输出里它自动出现了。这就是"绑定"的效果：钉一次，处处生效。

**关键点展开——`bind` 为什么返回新对象，而不是改原对象？**

这是 structlog 一个容易懵的设计。直觉上你可能以为 `log.bind(...)` 会"就地修改" `log`。实际上它**不改原对象，而是返回一个带新字段的新对象**。所以必须写 `log = log.bind(...)`，漏掉 `log =` 就白绑了。

为什么这么设计？因为"不可变"更安全：同一个基础 `log` 可以派生出多个不同上下文的子对象，互不干扰。比如：

```python
base = structlog.get_logger()
log_zhang = base.bind(user_id=123)   # 张三的日志对象
log_li    = base.bind(user_id=456)   # 李四的日志对象
# 两个对象各带各的 user_id，互不影响
```

**这一步你得到了什么**：你学会了 `bind`——把"每条日志都要带的公共字段"钉一次，之后自动生效，不用每条重复写。这是真实项目里最常用的功能。

---

### 示例 3：贴近项目的样子（配置 + 输出 JSON）

真实项目里，日志通常要输出成 **JSON**（方便灌进监控系统），并且要统一配置"日志长什么样"。structlog 用 `configure()` 一次性配好：

```python
import structlog

# 一次性配置：告诉 structlog 日志要经过哪些"加工步骤"，最后输出成什么
structlog.configure(
    processors=[
        # 步骤 1：给日志加上"时间戳"字段
        structlog.processors.TimeStamper(fmt="iso"),
        # 步骤 2：给日志加上"级别"字段（info / error 等）
        structlog.processors.add_log_level,
        # 步骤 3：最后一步，把日志渲染成 JSON 字符串输出
        structlog.processors.JSONRenderer(),
    ],
)

# 配置完之后，照常用 get_logger() 拿对象
log = structlog.get_logger()

# 记一条日志，字段照旧
log.info("user_logged_in", user_id=123, ip="1.2.3.4")
```

运行后输出（一行 JSON）：

```json
{"event": "user_logged_in", "user_id": 123, "ip": "1.2.3.4", "timestamp": "2024-01-01T12:00:00", "level": "info"}
```

**逐段解读**：

- `structlog.configure(processors=[...])` 是"总开关"，一次性定义日志的加工流水线。
- `processors` 是一个**列表**，日志会**按顺序**经过列表里的每个"处理器"（processor）：
  1. `TimeStamper` 给日志加 `timestamp` 字段；
  2. `add_log_level` 加 `level` 字段；
  3. `JSONRenderer` 把整条日志变成 JSON 字符串输出。
- 最后 `log.info(...)` 记的字段（`event`、`user_id`、`ip`）会**和处理器加的字段合并**，一起输出。

**关键点展开——"处理器链"是什么？**

你可以把 `processors` 想象成一条**流水线**：日志是一块"原料"，从左边进去，依次经过每个工位（处理器），每个工位给它加一点东西或改一点东西，最后从右边出来变成成品（JSON 字符串）。

- 不配 `processors` 时，structlog 用一套**默认**流水线（就是示例 1、2 里那种"文字表格"输出）。
- 配了 `processors`，你就完全掌控了日志的最终形态。想输出 JSON 就放 `JSONRenderer`，想输出彩色文字就放 `ConsoleRenderer`。

**这一步你得到了什么**：你学会了 `configure()`——用一条"处理器链"统一控制日志的最终形态。这是把 structlog 接进真实项目的关键一步。

---

## 第 5 节：最小可用的实际用法（生产场景模板）

下面是一份**可以直接复制进 FastAPI / 普通 Python 项目**的最小配置。它把 structlog 接到 Python 标准 `logging` 上（这样你项目里其他用 `logging` 的库也能被统一接管），并输出 JSON。

```python
import logging
import structlog

# ===== 必须：配置 structlog =====
structlog.configure(
    processors=[
        # 合并"上下文变量"（配合 contextvars，进阶，可先跳过）
        structlog.contextvars.merge_contextvars,
        # 加"级别"字段
        structlog.stdlib.add_log_level,
        # 加"时间戳"字段
        structlog.processors.TimeStamper(fmt="iso"),
        # 把异常堆栈也格式化进日志（进阶，可先跳过）
        structlog.processors.format_exc_info,
        # 最后一步：渲染成 JSON
        structlog.processors.JSONRenderer(),
    ],
    # 必须：让 structlog 复用标准 logging 的底层
    logger_factory=structlog.stdlib.LoggerFactory(),
    # 可选：缓存日志对象，避免重复创建（性能优化）
    cache_logger_on_first_use=True,
)

# ===== 必须：配置标准 logging 的根级别 =====
# 这行决定"低于 INFO 级别的日志（如 debug）不显示"
logging.basicConfig(level=logging.INFO, format="%(message)s")

# ===== 必须：拿日志对象 =====
log = structlog.get_logger()

# ===== 使用：记日志 =====
log.info("app_started", version="1.0.0")          # 记一条启动日志
log.info("user_logged_in", user_id=123, ip="1.2.3.4")  # 记一条业务日志
try:
    1 / 0  # 故意制造一个异常
except ZeroDivisionError:
    # 记一条错误日志，exc_info=True 会把异常堆栈一起记下来
    log.error("division_failed", exc_info=True)
```

**哪几行是必须的、哪几行可删**：

- **必须**：`structlog.configure(...)` 里的 `processors`（至少要有 `add_log_level` + 一个渲染器）、`logger_factory=structlog.stdlib.LoggerFactory()`、`logging.basicConfig(...)`、`get_logger()`。
- **可删**：`merge_contextvars`、`format_exc_info`、`cache_logger_on_first_use` 都属于进阶，删掉不影响基本功能。

**实际项目里通常还会配什么（进阶，可先跳过）**：

- **错误处理**：`log.error(..., exc_info=True)` 把异常堆栈记进日志，排查 bug 时能直接看到出错位置。
- **资源释放 / 日志轮转**：用标准 `logging` 的 `RotatingFileHandler` 把日志写进文件并按大小切分，避免日志文件无限膨胀。
- **配置项**：把日志级别、输出格式（JSON / 文字）做成环境变量，开发环境看彩色文字、生产环境输出 JSON。

### 实战案例：AI 应用里带 userId + 统计 token 消耗

一个典型需求：日志里要带上 `user_id`，并且统计每次对话消耗的 token 数量。这里正好把上一节讲的 `contextvars` 用起来——AI 应用是异步的，`user_id` 用 contextvars 挂一次，整个请求的日志自动带上；token 用量作为字段直接记。

**文件 1：`logging_config.py`（日志配置）**

```python
import logging
import structlog


def setup_logging() -> None:
    """程序启动时调用一次，配置 structlog 输出 JSON 日志。"""
    structlog.configure(
        processors=[
            # 关键：把 contextvars 里挂的字段（如 user_id）自动合并进每条日志
            # 没有这行，后面 bind_contextvars 挂的 user_id 不会出现在日志里
            structlog.contextvars.merge_contextvars,
            # 加"级别"字段（info / error）
            structlog.stdlib.add_log_level,
            # 加"时间戳"字段
            structlog.processors.TimeStamper(fmt="iso"),
            # 把异常堆栈也格式化进日志（记 error 时用）
            structlog.processors.format_exc_info,
            # 最后一步：渲染成 JSON 字符串
            structlog.processors.JSONRenderer(),
        ],
        # 复用标准 logging 的底层，让项目里其他库的日志也被统一接管
        logger_factory=structlog.stdlib.LoggerFactory(),
        # 缓存日志对象，避免重复创建（性能优化）
        cache_logger_on_first_use=True,
    )
    # 设置标准 logging 的根级别：低于 INFO 的日志（如 debug）不显示
    logging.basicConfig(level=logging.INFO, format="%(message)s")
```

**文件 2：`main.py`（FastAPI 请求处理）**

```python
import structlog
from fastapi import FastAPI, Request
from pydantic import BaseModel

from logging_config import setup_logging

# 启动时先配置日志（必须在 get_logger 之前，否则配置不生效）
setup_logging()

app = FastAPI()
log = structlog.get_logger()


class ChatRequest(BaseModel):
    """聊天请求体：只关心 messages 字段。"""
    messages: list[dict]


# 中间件：每个请求进来时，把 user_id 挂到 contextvars
# 这样这个请求里所有日志都会自动带上 user_id，不用层层传参
@app.middleware("http")
async def bind_user_id(request: Request, call_next):
    # 从请求头取 user_id（真实项目里通常从登录 token 里解析出来）
    user_id = request.headers.get("X-User-Id", "anonymous")
    # 挂到 contextvars：相当于"进门刷卡"，之后所有日志自动带这个字段
    structlog.contextvars.bind_contextvars(user_id=user_id)
    try:
        # 继续处理请求（进入下面的 /chat 路由）
        return await call_next(request)
    finally:
        # 请求结束，清理 contextvars，避免污染下一个请求
        structlog.contextvars.clear_contextvars()


def call_llm(messages: list[dict]):
    """调用大模型，返回 (回复文本, token 用量)。

    真实项目里这里是调用 OpenAI / 其他 LLM 的 SDK，例如：
        response = openai_client.chat.completions.create(...)
        return response.choices[0].message.content, response.usage
    这里用假数据占位，方便你跑通流程。
    """
    class Usage:  # 模拟 SDK 返回的 usage 对象
        prompt_tokens = 120      # 输入（提示词）消耗的 token
        completion_tokens = 80   # 输出（回复）消耗的 token
        total_tokens = 200       # 两者之和

    return "你好，我是 AI 助手", Usage()


@app.post("/chat")
async def chat(req: ChatRequest):
    # 调用大模型，拿到回复和 token 用量
    reply, usage = call_llm(req.messages)

    # 记日志：user_id 会自动带上（因为中间件里 bind_contextvars 挂了）
    # token 用量作为字段直接传，方便之后按字段统计
    log.info(
        "chat_completed",
        input_tokens=usage.prompt_tokens,       # 输入 token 数
        output_tokens=usage.completion_tokens,  # 输出 token 数
        total_tokens=usage.total_tokens,        # 总 token 数
    )

    return {"reply": reply}
```

**执行流程（一次请求发生了什么）**：

1. 请求进来 → 中间件 `bind_user_id` 先跑，把 `user_id` 挂进 contextvars。
2. 进入 `/chat` 路由 → 调 `call_llm` 拿到回复和 token 用量。
3. `log.info("chat_completed", input_tokens=..., ...)` 记日志时，`merge_contextvars` 处理器把 contextvars 里的 `user_id` 自动合并进来。
4. 请求结束 → `finally` 里 `clear_contextvars()` 清掉，下一个请求不受影响。

**最终日志长这样（一行 JSON）**：

```json
{"event": "chat_completed", "user_id": "123", "input_tokens": 120, "output_tokens": 80, "total_tokens": 200, "level": "info", "timestamp": "2024-01-01T12:00:00"}
```

注意 `user_id` 你**没在 `log.info` 里写**，是中间件挂上去自动带出来的；而 `input_tokens` 等是你在 `log.info` 里**显式传的字段**。两者最终合并成一条。

**之后怎么统计 token**：因为 token 是独立字段，灌进监控系统（ELK / Grafana）后，直接按字段聚合——每个用户总消耗 `sum(total_tokens) group by user_id`，平均每次对话消耗 `avg(total_tokens)`。这就是结构化日志的价值：字段独立，机器能直接算。

---

## 第 6 节：常见坑与易错点

### 坑 1：`bind` 忘了重新赋值

- **现象**：绑定了字段，但日志里没出现。
- **原因**：`bind` 返回新对象，不修改原对象。
- **解决**：写成 `log = log.bind(user_id=123)`，别漏掉 `log =`。

### 坑 2：`configure` 之后才 `get_logger`，顺序反了

- **现象**：配置没生效，日志还是默认格式。
- **原因**：`get_logger()` 拿到的对象会"记住"当时的配置。先拿对象、后 `configure`，对象还是旧配置。
- **解决**：**先 `configure`，再 `get_logger`**。把 `configure` 放在程序最开头（比如 `main` 函数第一行）。

### 坑 3：字段名和事件名搞混

- **现象**：日志里 `event` 字段的值不对，或字段名乱。
- **原因**：`log.info("事件名", 字段=值)` 里，**第一个位置参数是事件名**，后面才是字段。写成 `log.info(user_id=123)` 会报错（缺事件名）。
- **解决**：永远先写事件名，再写字段。事件名用字符串，字段用 `名字=值`。

### 坑 4：JSON 输出里中文变成 `\uXXXX`

- **现象**：输出 JSON 里中文显示成 `登录` 这种转义。
- **原因**：`JSONRenderer` 默认对非 ASCII 字符做转义。
- **解决**：`JSONRenderer(ensure_ascii=False)` 关掉转义，中文就能正常显示。

### 和我直觉相反的一点

**`bind` 不会"就地修改"对象，而是返回新对象。** 大多数新手（包括很多老手）第一次都会写成 `log.bind(user_id=123)` 然后纳闷"怎么没生效"。记住：structlog 的日志对象是"不可变"的，任何"加字段"的操作都返回新对象，必须重新赋值。

---

## 第 7 节：验证你学会了（自测题）

**第 1 题（复述）**：用一句话说清 structlog 是什么、解决什么问题。它和 `print` 记日志的本质区别是什么？

> 答案位置：第 1 节 + 第 2 节"问题根源"。

**第 2 题（默写）**：不看文档，默写出"记一条带 `user_id` 和 `ip` 字段的日志"的最小代码（3 行以内）。

> 答案位置：第 4 节示例 1。

**第 3 题（组合应用）**：现在要写一个函数，处理某个用户的订单。要求：① 日志输出成 JSON；② 每条日志都自动带上 `user_id`；③ 记一条"订单创建成功"的日志，带 `order_id` 字段。写出完整代码。

> 答案位置：第 4 节示例 2（bind）+ 示例 3（configure 输出 JSON）组合。
