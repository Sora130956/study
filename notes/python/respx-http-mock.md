# respx 入门：在 HTTP 层拦住 httpx 请求，让测试不打真实网络

- 目标读者：会 Python 语法、刚学会 pytest 基础（`test_` 函数、fixture、参数化），用过 httpx / requests，但没做过任何 mock
- 前置知识：见第 3 节（mock/stub/fake、测试边界、httpx transport 层、route 与 matcher、`side_effect` vs `return_value`、两个 assert 开关、pytest fixture）
- 学习时长：约 50 分钟 / 可跳过：第 2 节末尾的"经典应用场景"纯背景，赶时间可跳
- 实测环境：Python 3.12+、respx 0.23.1、httpx 0.28.1、pytest 9.1.1（本文所有代码在此版本组合下跑过）

---

## 第 1 节：它是什么

**一句话**：respx 是一个测试工具，它能在你的测试里把 httpx 发出的网络请求半路截下来，换成你事先写好的假响应，所以测试不会真的连上外网。

**类比**：像在你家小区门口装了个**假邮局**。

你还是照常写信、贴邮票、投进邮筒（你的代码一行没改，照样调 `httpx.post(...)`）。但信到了小区门口就被假邮局收下了，根本没出小区（没有真的联网）。假邮局按你事先写好的剧本，立刻塞给你一封回信（假响应）。

这个类比后面会一一对上：

- "照常寄信" = 被测代码里那句 `httpx.post("https://api.openai.com/v1/chat/completions", ...)` 原封不动
- "小区门口" = httpx 的 transport 层，respx 就挂在这里（第 4 节有示意图）
- "事先写好的回信" = `respx_mock.post(...).respond(200, json={...})`
- "剧本" = 匹配规则（什么 URL、什么请求体才给这封回信）

---

## 第 2 节：它解决什么问题

### 反面推演一：测试里直接调真 API

假设你在学 AI 应用开发，写了个函数问 LLM 一句话：

```python
# llm_client.py
import os
import httpx

def ask_llm(prompt: str) -> str:
    """把 prompt 发给 LLM，返回回答文本"""
    resp = httpx.post(
        "https://api.openai.com/v1/chat/completions",
        json={"model": "gpt-4o-mini", "messages": [{"role": "user", "content": prompt}]},
        headers={"Authorization": f"Bearer {os.environ['LLM_API_KEY']}"},
    )
    resp.raise_for_status()
    return resp.json()["choices"][0]["message"]["content"].strip()
```

最朴素的测法：测试里直接调它，让它真的打出去。

```python
# test_llm.py —— 朴素做法，别学
def test_ask_llm():
    answer = ask_llm("1+1 等于几？")
    assert "2" in answer          # 真的联网、真的花钱、真的看运气
```

跑起来可能是绿的，但你很快会撞上这些痛点：

1. **每跑一次测试花真钱**。测试是要反复跑的——改一行代码跑一次，CI 上每次提交跑一次。一天下来几百次调用，账单实打实。
2. **网断了、key 过期了、对方限流了，测试就变红，但你的代码一点问题没有**。测试变红本该意味着"代码坏了"，现在它变成了"今天网不好"，你就再也不敢相信红灯了。
3. **模型返回不确定，断言根本没法写**。同一个 prompt 问两次，回答的字都不一样。你只能写 `assert "2" in answer` 这种极弱的断言，或者干脆 `assert answer` —— 等于没测。
4. **慢**。一次 LLM 调用一两秒，二十个测试就是半分钟往上；CI 里本来 10 秒的套件变成 10 分钟。
5. **测不了异常分支**。你想验证"对方返回 429 限流时我的重试逻辑对不对"，可你没法命令 OpenAI 现在给你返回一个 429。

### 反面推演二：直接 patch 掉自己写的函数

于是有人想到用 Python 自带的 `unittest.mock.patch`，把 `ask_llm` 整个换成一个假函数：

```python
# 朴素做法二 —— 看着聪明，实际什么都没测
from unittest.mock import patch

def test_ask_llm_bad():
    with patch("llm_client.ask_llm", return_value="2"):   # 把被测函数本身换成假的
        assert ask_llm("1+1 等于几？") == "2"              # 断言的是自己刚填进去的 "2"
```

这个测试永远绿、飞快、不花钱。它的问题是**致命**的：

- `ask_llm` 里那些真正需要被验证的逻辑——URL 拼对了没有、`Authorization` 头带对了没有、`resp.json()["choices"][0]...` 这串取值路径写对了没有、`.strip()` 有没有生效——**一行都没执行**。它们被整段跳过了。
- 断言 `== "2"` 验证的是 `return_value="2"` 这个你自己刚设进去的值。你在测 `unittest.mock` 库能不能正常工作，不是在测你的代码。
- 最坏的情况：你把 URL 写错成 `chat/completion`（少个 s），这个测试**照样是绿的**。它给了你虚假的安全感，比没有测试更危险。

这也正是工作区规则里那条"**禁止 mock 被测对象本身**"要挡的东西。

### 问题根源（一句话）

真实网络是"不可控 + 花钱 + 慢"的外部世界，但我们要验证的是**自己写的那几十行代码**；所以正确的做法是把"替身"放在**自己代码的最外边界**（发出 HTTP 请求的那一刻），而不是放在自己代码的**里面**（把自己的函数换掉）。

respx 就挂在那个最外边界上：**被测代码的每一行照常执行**，只有"信送出小区"这一步被换成了假邮局。

### 经典应用场景

- **测调 LLM 的代码**：拦 chat completions，构造 200 / 429 / 500 / 超时，验证解析逻辑和重试逻辑
- **测依赖第三方 API 的服务**：你的 FastAPI 服务里要调支付、地图、天气接口，测自己的服务时把上游全拦掉
- **测重试、退避、熔断**：这类逻辑只有在"对方出错"时才跑得到，真 API 不配合你，假响应随叫随到
- **测错误处理**：对方返回畸形 JSON、缺字段、空数组时，你的代码是抛清晰异常还是 `KeyError` 崩掉

---

## 第 3 节：读下去之前，先搞懂这些概念

正文代码里出现的每个机制，这里先讲清。没做过 mock 的话，3.1 到 3.3 一定要看。

### 3.1 mock / stub / fake 到底啥区别

这三个词经常被混着用，你只要抓住区别就行。它们统称"**测试替身**"（test double），意思是：测试的时候，把真东西换成一个假的替代品，像拍电影用替身演员。

- **stub（桩）**：只负责"被问到时给个固定答案"，不检查你怎么问它。
  一行定义：预设的固定返回值。
  大白话：一个只会说一句话的复读机。
  ```python
  respx_mock.get("https://a.test/x").respond(200, json={"ok": True})  # 不管谁来问，都返回这个
  ```

- **mock（模拟对象）**：除了给答案，还**记录你怎么调用它**，好让你事后断言"你确实调了 2 次、第二次带的是这个请求体"。
  一行定义：带调用记录的替身。
  大白话：复读机 + 通话录音。
  ```python
  assert route.calls.call_count == 2  # 事后查录音：一共被调了 2 次
  ```

- **fake（假实现）**：一个能真跑的简化版实现，有自己的逻辑，只是不碰真资源。
  一行定义：轻量真实现。
  大白话：不是复读机，是个小玩具版的真机器（比如用内存字典代替数据库）。
  ```python
  class FakeCache(dict): ...  # 用字典冒充 Redis，get/set 真的能存能取
  ```

respx 提供的是 **stub + mock**：`respond(...)` 是 stub 的部分，`route.calls` 是 mock 的部分。

### 3.2 什么叫"测试边界"

- **定义**：被测代码和外部世界的交界线。替身应该放在这条线上，线以内的代码全部真跑。
- **大白话**：你要检查厨师的手艺，就该在"食材进厨房"这里换成假食材，而不是把厨师换成假厨师。
- **最小示例**：`ask_llm` 这个函数，边界就是它发出 HTTP 请求那一刻。放在这里 → 函数体全跑；放在函数名上（`patch("llm_client.ask_llm")`）→ 函数体一行不跑。

### 3.3 httpx 的 transport 层：respx 挂在哪一层

httpx 内部分两层：**你调的 API 层**（`httpx.get` / `client.post` 这些好用的方法）和底下的 **transport 层**（真正建 TCP 连接、发字节、收字节的那个东西，默认是 `httpx.HTTPTransport`）。

- **定义**：transport 是 httpx 里"负责真的上网"的那个可替换零件。
- **大白话**：API 层是你用的遥控器，transport 是电视机后面的插头。respx 做的事是**把插头拔下来，插到一个假电源上**。
- **最小示例**：respx 生效期间，httpx 内部的 transport 被换掉了，所以 `httpx.get(...)` 这行代码本身完全不用改。

### 3.4 路由（route）与匹配器（matcher）

- **route（路由）**：一条规则，说"符合这些条件的请求，返回这个响应"。
- **matcher（匹配器）**：route 里用来判断"符不符合"的条件，比如方法、URL、query 参数、json 请求体。
- **大白话**：route 是假邮局墙上贴的一条规定；matcher 是这条规定里"寄给谁、信里写什么"的部分。
- **最小示例**：
  ```python
  respx_mock.post("https://api.test/v1/chat", json__model="gpt-4o-mini")  # 方法+URL+请求体字段 三重匹配
  ```
  `json__model="gpt-4o-mini"` 这个双下划线写法的意思是"请求体 JSON 里的 `model` 字段等于 gpt-4o-mini"。

### 3.5 `return_value` vs `side_effect`

- **`return_value`**：固定返回同一个响应，调 100 次都一样。
- **`side_effect`**：可以传一个**列表**，第一次调用返回第 1 个、第二次返回第 2 个……也可以传一个函数，让响应根据请求内容动态算出来；还能传异常类来模拟超时。
- **大白话**：`return_value` 是复读机；`side_effect` 是有剧本的演员，第几次出场说什么台词都排好了。
- **最小示例**：
  ```python
  route.mock(side_effect=[httpx.Response(429), httpx.Response(200, json={})])  # 先限流，再成功
  ```

### 3.6 两个保险开关：`assert_all_mocked` 与 `assert_all_called`

- **`assert_all_mocked`（默认 True）**：测试期间如果冒出一个**没有任何 route 匹配**的请求，直接抛错，绝不让它出去。
  大白话：假邮局遇到一封"墙上规定里没写过"的信，不会好心帮你寄出去，而是当场拉警报。
  **这是本文最重要的一条，不要关掉它**（第 4.2 和第 6.1 详细讲为什么这个报错是好事）。
- **`assert_all_called`**：退出时检查"定义了的 route 是不是都被调用过"，有没被调用的就抛错。
  大白话：检查你贴的规定有没有白贴。
  注意实测值：`respx_mock` fixture 默认 `assert_all_called=False`、`assert_all_mocked=True`；而 `with respx.mock(...)`（带括号）默认两个都是 True。第 6.3 讲这个坑。

### 3.7 pytest fixture（一句话）

- **定义**：pytest 里给测试准备"材料 + 事后清理"的函数，测试函数把它的名字写进参数就能拿到。
- **大白话**：开演前搭好布景、演完拆掉。`respx_mock` 就是 respx 提供的一个现成 fixture：进测试自动挂上假邮局，出测试自动拆掉。
- 详细用法另有 pytest 专篇，这里知道"写进参数就能用"即可。

### 术语速查表

| 术语 | 大白话 | 正文哪节 |
| --- | --- | --- |
| 测试替身 / mock | 测试时用的假货 | 3.1 |
| stub | 只会给固定答案的复读机 | 3.1 |
| fake | 能跑的玩具版真实现 | 3.1 |
| 测试边界 | 自己代码和外部世界的交界线 | 3.2、第 2 节 |
| transport | httpx 里真正上网的那个插头 | 3.3、4.2 |
| route | 假邮局墙上的一条规定 | 3.4、4.1 |
| matcher | 规定里"寄给谁、写什么"的判定条件 | 3.4、4.2 |
| `respond()` | 直接写一封固定回信 | 4.1 |
| `return_value` | 每次都回同一封信 | 3.5 |
| `side_effect` | 有剧本：第几次回哪封信 | 3.5、4.3 |
| `route.calls` | 通话录音，事后查调了几次、带了什么 | 3.1、4.3 |
| `assert_all_mocked` | 漏网的请求当场拉警报（别关） | 3.6、6.1 |
| `assert_all_called` | 检查规定有没有白贴 | 3.6、6.3 |

---

## 第 4 节：从最简单的代码示例开始

先装好（工作区用 uv）：

```bash
uv add --dev respx pytest pytest-asyncio     # respx 只在测试时用，装到 dev 依赖
```

### 4.1 示例一：最小可运行版本

业务代码，一个用 `httpx.get` 取数据的函数：

```python
# weather.py
import httpx

def get_city_temp(city: str) -> float:
    """查某个城市的当前气温，返回摄氏度数字"""
    resp = httpx.get("https://api.weather.test/current", params={"city": city})  # 发真请求（测试里会被拦）
    resp.raise_for_status()          # 非 2xx 直接抛异常，别让脏数据往下流
    return resp.json()["temp_c"]     # 从响应里取出气温字段，这行取值路径是本函数最容易写错的地方
```

测试：

```python
# test_weather.py
from weather import get_city_temp

def test_get_city_temp(respx_mock):                      # respx_mock 是 respx 自带的 fixture，写进参数就自动生效
    respx_mock.get("https://api.weather.test/current").respond(   # 贴一条规定：GET 这个 URL
        200,                                             # 假响应的状态码
        json={"temp_c": 23.5, "city": "hangzhou"},       # 假响应的 JSON body，字段名要和真 API 一致
    )

    temp = get_city_temp("hangzhou")                     # 照常调业务函数，函数体每一行都真的执行了

    assert temp == 23.5                                  # 断言业务结果：函数是否正确地从响应里取出了 temp_c
```

跑：

```bash
uv run pytest -q test_weather.py
# 1 passed
```

**逐段解读**

- `def test_get_city_temp(respx_mock)`：参数里的 `respx_mock` 就是"进门装假邮局、出门拆掉"。装了 respx 库之后这个 fixture 自动可用，不需要任何 import，也不需要写 conftest。
- `respx_mock.get(URL)`：注册一条 route（第 3.4 节）。它返回一个 `Route` 对象，可以接着链式调用。
- `.respond(200, json={...})`：给这条 route 配一封固定回信。`respond` 是快捷写法，内部帮你造了一个 `httpx.Response`。
- `get_city_temp("hangzhou")`：**关键点** —— 这里调的是真的业务函数，`httpx.get`、`params` 拼接、`raise_for_status()`、`.json()["temp_c"]` 全部真跑了一遍。只有"真的上网"这一步被换掉。
- `assert temp == 23.5`：断言的是**业务结果**（函数返回的气温数字），而不是"我刚设的 json 是不是我刚设的那个"。

**单独展开：respx 到底拦在哪一层**

```
你的业务代码                httpx 的 API 层                httpx 的 Transport 层            真实网络
get_city_temp()   ──►   httpx.get(...) / client.post()  ──►   HTTPTransport      ──►    api.weather.test
   ↑                            ↑                                   ↑
 真跑                         真跑                          ← respx 在这里把它换成假的 →  ✗ 根本没走到这
```

理解这张图，就理解了 respx 为什么是"正确的 mock 位置"：切口开在最右边，左边你写的每一行都还在被检验。对比第 2 节的 `patch("llm_client.ask_llm")` —— 那是把切口开在了最左边，整个中间全被跳过。

**这一步你得到了什么**：一个不联网、每次结果都一样、毫秒级跑完的测试，而且它真的在验证 `get_city_temp` 的取值逻辑——你把 `temp_c` 写错成 `temp`，它会立刻变红。

### 4.2 示例二：POST + 匹配请求体 + 看保险生效

真实小需求：给一个"提交订单"的函数写测试，要求验证它**发对了请求体**，并且能读到响应头里的追踪 ID。

```python
# orders.py
import httpx

def create_order(sku: str, qty: int) -> dict:
    """下单，返回 {"order_id": ..., "trace_id": ...}"""
    resp = httpx.post(
        "https://api.shop.test/v1/orders",
        json={"sku": sku, "qty": qty},        # 请求体：这两个字段名写错了，上游就会拒单
    )
    resp.raise_for_status()
    return {
        "order_id": resp.json()["id"],                    # 业务字段从 body 拿
        "trace_id": resp.headers.get("x-trace-id", ""),   # 追踪 ID 从响应头拿，没有就给空串
    }
```

```python
# test_orders.py
import pytest
from orders import create_order

def test_create_order(respx_mock):
    route = respx_mock.post(                      # 把 route 存下来，后面要查它的调用记录
        "https://api.shop.test/v1/orders",
        json={"sku": "A-100", "qty": 2},          # matcher：请求体必须完全等于这个 dict，否则不匹配
    ).respond(
        201,
        json={"id": "ord_888"},                   # 假的响应 body
        headers={"x-trace-id": "trace-abc"},      # 假的响应头
    )

    result = create_order("A-100", 2)             # 真跑业务函数

    assert result == {"order_id": "ord_888", "trace_id": "trace-abc"}  # 断言业务结果：两个来源都拼对了
    assert route.calls.call_count == 1            # 断言只发了 1 次请求（没有重复下单）
```

**逐段解读**

- `json={"sku": "A-100", "qty": 2}` 出现在 `post()` 里时，它是**匹配条件**（进来的请求体必须长这样）；出现在 `respond()` 里时，它是**假响应的 body**。同一个关键字，位置不同意思完全不同，这是新手最容易混的一点。
- 匹配条件写得越具体，测试越有价值。这里等于顺手验证了"业务函数确实把 sku 和 qty 装进了请求体"——如果 `create_order` 里把 `qty` 写成了 `quantity`，这个 route 就匹配不上，测试会以"not mocked"报错变红。
- `route.calls.call_count`：这就是 3.1 说的"通话录音"。断言次数能抓住"循环里多发了一次请求"这类 bug。

**单独展开：漏网的请求会怎样（保险生效的样子）**

故意让业务代码去请求一个没注册过的 URL，比如测试里只贴了 `/v1/orders` 的规定，代码却请求了 `/v2/orders`。运行结果：

```
respx.models.AllMockedAssertionError: RESPX: <Request('POST', 'https://api.shop.test/v2/orders')> not mocked!
```

**这个报错是好事，不是 bug。** 它的含义是："有一个请求我没认领，我不知道该给它什么假响应，所以我拦下来告诉你。"如果没有这道保险，这个请求就会**真的发到外网**——于是测试悄悄开始联网、开始花钱、开始看运气，而你毫不知情。

所以工作区规则那句"**未覆盖的请求让它报错，不要关掉这个保险**"，落到代码上就是：**永远不要写 `assert_all_mocked=False`**。看到这个报错，正确反应是"我漏贴了一条规定"或者"我的业务代码请求了预期外的地址"，两种都值得你去看一眼。

**等价的普通写法对照：`respx_mock` fixture vs `with respx.mock(...)`**

```python
# 写法 A：fixture（推荐，最省事）
def test_a(respx_mock):
    respx_mock.get("https://a.test/x").respond(200, json={})
    ...
# 实测默认：assert_all_called=False, assert_all_mocked=True

# 写法 B：上下文管理器，作用域自己控制
import respx
def test_b():
    with respx.mock(base_url="https://a.test") as router:   # 带括号 = 新建一个独立 router
        router.get("/x").respond(200, json={})              # 有 base_url，这里只写路径
        ...
# 带括号的默认：assert_all_called=True, assert_all_mocked=True

# 写法 C：给 fixture 改配置，用 marker
@pytest.mark.respx(base_url="https://a.test")
def test_c(respx_mock):
    respx_mock.get("/x").respond(200, json={})
    ...
# 注意：走 marker 时 assert_all_called 会变成 True
```

三者拦截能力完全一样，区别只在"作用域"和"默认开关"。日常写测试用 A；需要 `base_url` 少写重复 URL 时用 C；需要在一个测试里精确控制"从哪行开始拦、到哪行结束"时用 B。

**这一步你得到了什么**：会用 matcher 顺手验证请求内容，会用 `route.calls` 验证调用次数，并且见过了保险生效时的真实报错——以后看到它不会慌。

### 4.3 示例三：贴近项目 —— 测 LLM 调用的重试逻辑

这是 AI 应用开发里最典型的一个测试：LLM 接口被限流（429）时，代码要退避后重试；成功后要正确解析出回答文本。真 API 不会配合你返回 429，所以这个分支只有 respx 能测。

```python
# llm_client.py
import os
import time
import httpx

LLM_BASE_URL = "https://api.openai.com/v1"

def ask_llm(prompt: str, *, max_retries: int = 3) -> str:
    """问 LLM 一句话，遇到 429 限流会退避重试，返回回答文本"""
    api_key = os.environ["LLM_API_KEY"]                      # 从环境变量读 key，绝不硬编码进代码
    payload = {
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": prompt}],
    }
    with httpx.Client(base_url=LLM_BASE_URL, timeout=10.0) as client:
        for attempt in range(max_retries):                   # 最多试 max_retries 次
            resp = client.post(
                "/chat/completions",
                json=payload,
                headers={"Authorization": f"Bearer {api_key}"},
            )
            if resp.status_code == 429:                       # 被限流：等一会儿再来
                time.sleep(2 ** attempt)                      # 指数退避：1 秒、2 秒、4 秒
                continue
            resp.raise_for_status()                           # 其他错误码（500 等）直接抛
            return resp.json()["choices"][0]["message"]["content"].strip()
    raise RuntimeError("LLM 连续被限流，重试次数已用尽")        # 试满还是 429，明确报错
```

```python
# test_llm_client.py
import json
import httpx
import pytest
from llm_client import ask_llm

@pytest.fixture(autouse=True)
def fake_api_key(monkeypatch):
    """给所有测试塞一个假 key。autouse=True 表示不用手动写进参数，自动生效"""
    monkeypatch.setenv("LLM_API_KEY", "sk-test-not-a-real-key")  # 必须用 monkeypatch，测试结束自动还原

@pytest.fixture(autouse=True)
def no_sleep(monkeypatch):
    """把退避用的 sleep 变成空操作，否则这个测试要白等 1 秒"""
    monkeypatch.setattr("llm_client.time.sleep", lambda seconds: None)

def _ok_response(text: str) -> httpx.Response:
    """造一个成功的 LLM 假响应，结构照抄真 API 的返回格式"""
    return httpx.Response(200, json={"choices": [{"message": {"content": text}}]})

def test_retry_after_429(respx_mock):
    route = respx_mock.post("https://api.openai.com/v1/chat/completions").mock(
        side_effect=[                                  # 剧本：按顺序，第一次给这个，第二次给那个
            httpx.Response(429, json={"error": {"message": "rate limit"}}),  # 第 1 次：限流
            _ok_response("  你好  "),                   # 第 2 次：成功，故意带前后空格测 .strip()
        ]
    )

    answer = ask_llm("hi")                             # 真跑业务函数，它内部会自己重试

    assert answer == "你好"                             # 业务结果一：重试成功了，且空格被 strip 掉了
    assert route.calls.call_count == 2                 # 业务结果二：确实发了 2 次请求（1 次失败 + 1 次重试）
    sent = json.loads(route.calls.last.request.content)              # 取最后一次实际发出的请求体
    assert sent["messages"][0]["content"] == "hi"                    # 业务结果三：prompt 装对了位置
    assert route.calls.last.request.headers["authorization"] == "Bearer sk-test-not-a-real-key"  # 鉴权头拼对了

def test_all_429_raises(respx_mock):
    respx_mock.post("https://api.openai.com/v1/chat/completions").respond(429, json={})  # 永远限流

    with pytest.raises(RuntimeError, match="重试次数已用尽"):     # 断言：试满后抛的是我们自己那个清晰异常
        ask_llm("hi")

def test_server_error_raises(respx_mock):
    respx_mock.post("https://api.openai.com/v1/chat/completions").respond(500)

    with pytest.raises(httpx.HTTPStatusError):                  # 500 不该被当成限流重试，应直接抛
        ask_llm("hi")
```

跑：

```bash
uv run pytest -q test_llm_client.py
# 3 passed
```

**逐段解读**

- `fake_api_key` fixture：业务代码里 `os.environ["LLM_API_KEY"]` 没有这个变量就会 `KeyError`。用 `monkeypatch.setenv` 塞一个假的，测试结束自动还原。**不要直接改 `os.environ`**——那会污染后面所有测试（工作区规则第 4 条）。假 key 只是个占位字符串，反正请求根本发不出去。
- `no_sleep` fixture：这里 patch 的是 `time.sleep`，属于"环境噪音"，不是被测逻辑本身，所以 patch 它是合理的。注意它没有削弱断言：重试**次数**仍然由 `route.calls.call_count == 2` 真实验证着。
- `side_effect=[...]`：整个测试的核心。列表按调用顺序消费——第一次请求命中 429，业务代码走进 `continue` 分支去重试，第二次请求命中 200。这就模拟出了"真实世界里偶发限流"的场景。
- 三个断言全都指向**业务结果**：返回文本对不对、重试次数对不对、实际发出的请求内容对不对。对比一个反例——

```python
# 反例：什么都没验证的断言
route = respx_mock.post(URL).respond(200, json={"x": 1})
resp = httpx.post(URL)
assert resp.json() == {"x": 1}      # ← 断言的是自己上一行刚设进去的值
```

这行断言恒成立，它只证明了 respx 能正常工作。它不涉及任何业务代码，把 `ask_llm` 整个删掉它照样绿。工作区规则"**断言必须针对业务结果**"要挡的就是这种写法。

**单独展开：`side_effect` 还能做什么**

```python
# 1) 传异常类，模拟网络超时（测超时处理分支）
respx_mock.post(URL).mock(side_effect=httpx.ConnectTimeout)

# 2) 传函数，让响应根据请求内容动态决定
def dynamic(request):
    body = json.loads(request.content)
    return httpx.Response(200, json={"echo": body["messages"][0]["content"]})
respx_mock.post(URL).mock(side_effect=dynamic)
```

列表版和函数版的区别：列表版管"第几次调用"，函数版管"这次请求长什么样"。测重试用列表版，测"不同输入走不同分支"用函数版。

**这一步你得到了什么**：能测出真 API 根本没法帮你复现的分支（限流、服务器错误、超时），而且断言的每一条都在检验你自己写的代码。

---

## 第 5 节：最小可用的实际用法（生产场景模板）

直接复制这三个文件就能跑。第 4.3 的业务代码 `llm_client.py` 原样复用，这里补上项目该有的配置和 conftest。

**`pyproject.toml`（相关片段）**

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"        # 必须：async 测试才会被真的 await（漏了会假通过，见第 6.5）
testpaths = ["tests"]        # 可选：只在 tests/ 目录里找测试
addopts = "-q"               # 可选：输出简洁点
```

**`tests/conftest.py`**

```python
import httpx
import pytest

@pytest.fixture(autouse=True)
def fake_api_key(monkeypatch):
    """必须：所有测试共用的假 key。放 conftest 里，每个测试文件都不用再写一遍"""
    monkeypatch.setenv("LLM_API_KEY", "sk-test-not-a-real-key")

@pytest.fixture(autouse=True)
def no_sleep(monkeypatch):
    """可选：项目里没有退避重试逻辑就可以删掉"""
    monkeypatch.setattr("llm_client.time.sleep", lambda seconds: None)

@pytest.fixture
def llm_ok():
    """可选但很好用：造成功响应的小工厂，省得每个测试都手抄一遍 LLM 的返回结构"""
    def _make(text: str, *, status_code: int = 200) -> httpx.Response:
        return httpx.Response(status_code, json={"choices": [{"message": {"content": text}}]})
    return _make
```

注意 `fake_api_key` / `no_sleep` / `llm_ok` 都是默认的 **function scope**（每个测试独立创建、独立清理），符合工作区验收第 5 步。不要给它们加 `scope="session"`——共享状态会让测试互相污染。

**`tests/test_llm_client.py`**

```python
import json
import httpx
import pytest
from llm_client import ask_llm

CHAT_URL = "https://api.openai.com/v1/chat/completions"   # 必须：和业务代码实际请求的完整 URL 一致

def test_happy_path(respx_mock, llm_ok):
    """必须有的基本用例：一次成功，验证解析逻辑"""
    route = respx_mock.post(CHAT_URL).mock(return_value=llm_ok("北京是中国的首都"))

    answer = ask_llm("中国的首都是哪里？")

    assert answer == "北京是中国的首都"                       # 业务结果
    assert route.calls.call_count == 1                      # 没有重复调用
    assert json.loads(route.calls.last.request.content)["model"] == "gpt-4o-mini"  # 模型名没写错

def test_retry_then_success(respx_mock, llm_ok):
    """必须有的重试用例：429 后重试成功"""
    route = respx_mock.post(CHAT_URL).mock(
        side_effect=[httpx.Response(429, json={}), llm_ok("ok")]
    )

    assert ask_llm("hi") == "ok"
    assert route.calls.call_count == 2

def test_retries_exhausted(respx_mock):
    """必须有的失败用例：一直限流，要抛我们自己的清晰异常"""
    respx_mock.post(CHAT_URL).respond(429, json={})

    with pytest.raises(RuntimeError, match="重试次数已用尽"):
        ask_llm("hi")
```

### 交付前的自检步骤（照着做，四步）

这四步对应工作区的 Test review checklist，全过才算能交：

1. **查假通过**
   ```bash
   uv run pytest -q
   ```
   全绿，且输出头部 `plugins:` 那行要能看到 `asyncio-` 和 `respx-`；输出里不能有 `RuntimeWarning: coroutine ... was never awaited`。

2. **查装饰性测试**（最能暴露问题的一步）
   随手把业务代码改坏一行——比如把 `resp.status_code == 429` 改成 `== 428`，或者把 `["choices"][0]` 改成 `["choices"][1]` —— 再跑一次，**必须有测试变红**。全绿说明你的测试是装饰品，重写。改完记得还原。

3. **查真网外呼：断网再跑一次，必须仍全绿**
   ```bash
   # 关掉 Wi-Fi / 拔网线 / 关掉代理，然后：
   uv run pytest -q
   ```
   还是全绿 → 说明所有请求都被 respx 拦住了，一个都没漏出去。
   报 `ConnectError` / `ConnectTimeout` / DNS 失败 → **有测试在打真实网络**，去找那个漏掉的 route 补上。
   这一步不能省。它是唯一能物理证明"测试不联网"的手段，比读代码可靠。

4. **抽查断言**
   随机挑一个测试，只读它的 `assert` 行，问自己一句：这行验证的是**业务结果**（返回值 / 状态变化 / 抛出的异常 / 实际发出的请求），还是我自己上几行刚 set 进去的 mock 值？后者就重写。

### 实际项目里通常还会配什么（进阶，先知道有这回事）

- **把假响应存成 fixture 文件**：LLM 的真实响应 JSON 很长，塞在测试代码里很难读。常见做法是存成 `tests/fixtures/chat_completion_ok.json`，测试里读进来传给 `respond(json=...)`。
- **录制真实响应做回放**：先真调一次 API，把响应存下来，之后测试全部用存下来的数据。这样假响应的结构保证和真 API 一致，不会出现"测试通过但线上炸"。respx 支持 `pass_through()` 放行真实请求，可以配合做录制。
- **和 FastAPI 配合**：测自己的 FastAPI 服务时，内部依赖用 `dependency_overrides` 注入 fake，**对外发出的 HTTP 请求用 respx 拦**。两者管的层次不同，通常同时用。

这几项都属于加分项，入门阶段不配也能写出合格的测试。

---

## 第 6 节：常见坑与易错点

### 6.1 请求没被任何 route 覆盖

**报错原文**（respx 0.23.1 实测）

```
respx.models.AllMockedAssertionError: RESPX: <Request('POST', 'https://api.shop.test/v2/orders')> not mocked!
```

**现象**：测试变红，报错里带着那个"没人认领"的请求。

**原因**：`assert_all_mocked` 默认开启，遇到没有 route 匹配的请求就抛错，不让它出网。

**解决**：看报错里的 URL 和方法，两种情况——
- 你确实漏贴了规定 → 补一条 route
- 业务代码请求的地址跟你预期的不一样 → 恭喜，测试帮你抓到了一个 bug

**千万不要这样"解决"**：

```python
@pytest.mark.respx(assert_all_mocked=False)   # ❌ 绝对不要写
```

关掉之后，未匹配的请求会被放行成真实网络请求，你的测试从此开始偷偷联网。这个报错是保险丝烧了在告诉你哪里短路，不是保险丝有问题。

### 6.2 URL 匹配不上（尾斜杠 / query 参数 / base_url 拼接）

**现象**：明明贴了 route，还是报 `not mocked!`，或者贴了 route 但没被调用。

**原因与解决**，三个高频细节：

- **尾斜杠算不同 URL**。实测：注册 `/v1/models`，请求 `/v1/models/` 会报 `not mocked!`。解决：照抄业务代码里实际写的路径，别自己加减斜杠。
- **query 参数默认不匹配**。`respx_mock.get("https://a.test/s")` 能匹配带任意 query 的请求；但如果你想验证 query，要显式写：
  ```python
  respx_mock.get("https://a.test/s", params={"q": "hi"}).respond(200, json=[])
  ```
- **`base_url` 是拼接，不是替换**。用了 `@pytest.mark.respx(base_url="https://api.test")` 之后，route 里写 `/v1/orders` 就够了；但如果这时还写完整 URL，就会拼成一个错的地址。两种写法别混用。

排查技巧：把 route 存成变量，在断言前打印 `route.called`，就知道是"没匹配上"还是"匹配上了但断言写错了"。

### 6.3 `assert_all_called` 导致"路由定义了但没被调用"

**报错原文**（实测）

```
AssertionError: RESPX: some routes were not called!
```

**现象**：测试逻辑看着没问题，业务断言也过了，偏偏在测试结束时报这个错。

**原因**：你贴了一条 route，但整个测试跑完它一次都没被用到。而 `assert_all_called` 在两种情况下是开启的（实测 respx 0.23.1）：
- `with respx.mock(...)`（**带括号**）
- 用了 `@pytest.mark.respx(...)` marker 的 `respx_mock`

裸用 `respx_mock` fixture 时它是关的，所以你可能一开始碰不到这个错，换了写法突然就撞上了。

**解决**：先别急着关开关，先想"为什么这条 route 没被调用"——
- 多贴了一条用不上的 route → 删掉它
- 想测的分支根本没走到（比如重试代码有 bug，一次都没重试）→ 这是真 bug，去修业务代码
- 确实是"这条 route 可能被调用、也可能不被调用" → 用 `respx_mock.route(...).mock(...)` 时把它标成可选，或改用不带 marker 的裸 fixture

### 6.4 mock 错了层：测试永绿但完全无效（最反直觉的坑）

**现象**：测试全绿、跑得飞快，但线上一上就炸。你把业务代码改坏，测试**照样绿**。

**原因**：mock 的位置放在了被测代码**里面**，而不是它的**边界**上。

```python
# ❌ 错的层：把自己的函数整个换掉
with patch("llm_client.ask_llm", return_value="2"):
    assert ask_llm("1+1") == "2"       # ask_llm 的函数体一行没跑

# ❌ 也是错的层：把自己内部的方法换掉
with patch("llm_client.httpx.Client.post", return_value=fake):
    ...                                 # 绕过了 httpx 的请求构造，URL/headers 拼错也发现不了

# ✅ 对的层：拦在 HTTP transport 上
def test_ok(respx_mock, llm_ok):
    respx_mock.post(CHAT_URL).mock(return_value=llm_ok("2"))
    assert ask_llm("1+1") == "2"        # ask_llm 全跑，只有"上网"这步是假的
```

**解决**：记一条判断标准——**mock 的对象必须是"别人的东西"（外部服务），不能是"自己的东西"（被测代码）**。这也是第 5 节自检第 2 步（改坏业务代码看是否变红）为什么必须做的原因：它是唯一能自动抓出"mock 错层"的手段。

### 6.5 async 测试忘了 await，测试假通过

**现象**：pytest 输出里出现

```
RuntimeWarning: coroutine 'test_xxx' was never awaited
```

测试显示 passed，但其实**一行都没执行**。

**原因**：`async def` 函数被调用只会得到一个协程对象，不真正运行，除非有人 await 它。pytest 需要 `pytest-asyncio` 且配置 `asyncio_mode = "auto"` 才会自动接管这些 async 测试。

**解决**：
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"        # pyproject.toml 里必须有这行
```
并检查 pytest 输出头部 `plugins:` 行里有 `asyncio-`。async 场景下 respx 用法完全一样，只是记得 await：

```python
async def test_async_llm(respx_mock, llm_ok):
    respx_mock.post(CHAT_URL).mock(return_value=llm_ok("hi"))
    async with httpx.AsyncClient() as client:
        resp = await client.post(CHAT_URL, json={})     # 必须 await
    assert resp.json()["choices"][0]["message"]["content"] == "hi"
```

respx 同时支持 `httpx.Client` 和 `httpx.AsyncClient`，不需要额外配置。async 的其他细节见 pytest 专篇。

---

## 第 7 节：验证你学会了

不给答案，标了在哪节，自己回去翻。

**第 1 题（概念复述）**
不看笔记，用自己的话回答三个问题：① respx 是什么，用一句话加一个类比；② 测"调 LLM 的函数"如果不用它，会有哪些具体麻烦（至少说出 3 个）；③ 为什么应该拦在 HTTP 层，而不是用 `patch` 把自己写的 `ask_llm` 换掉？第 3 点要能说清"被测代码有没有真的执行"这个关键差别。
> 答案在：第 1 节、第 2 节、第 4.1 节的分层示意图

**第 2 题（默写最小拦截测试）**
关掉本文，写出一个完整可跑的测试：业务函数用 `httpx.get` 请求 `https://api.demo.test/users/1`，返回响应 JSON 里的 `name` 字段；测试用 `respx_mock` 拦住它，断言函数返回了正确的名字。要求：不查文档写出 fixture 名、route 注册、`respond()` 的参数。写完自问一句——如果我把业务代码里的 `["name"]` 改成 `["username"]`，这个测试会不会变红？
> 答案在：第 4.1 节

**第 3 题（组合应用）**
给一个"调外部天气 API 带重试"的函数写测试，要求覆盖：① 第一次返回 429、退避后第二次返回 200 的完整重试路径；② 断言最终业务结果正确；③ 断言**总共只发了 2 次请求**；④ 断言第二次请求实际带的 query 参数是对的；⑤ 用 `monkeypatch` 处理掉 API key 和 `sleep` 等待。写完跑第 5 节的自检四步，特别是**断网再跑一遍必须仍全绿**，以及**改坏一行业务代码必须有测试变红**。
> 答案在：第 4.3 节（`side_effect` 列表、`route.calls`）、第 5 节（conftest 与自检步骤）、第 6.2 节（query 参数匹配）
