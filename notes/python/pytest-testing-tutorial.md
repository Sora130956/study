# pytest 测试实战教程：fixture、参数化、异步测试与 HTTP mock 教程

> **来源:** [pytest 官方 how-to（fixtures / parametrize / monkeypatch）](https://docs.pytest.org/en/stable/how-to/fixtures.html)、[pytest-asyncio 文档](https://pytest-asyncio.readthedocs.io/en/latest/)、[RESPX 文档](https://lundberg.github.io/respx/)、[FastAPI Testing](https://fastapi.tiangolo.com/tutorial/testing/)、[unittest.mock 标准库](https://docs.python.org/3/library/unittest.mock.html)
> **对应计划周次:** 第 1 周 · 周四全天 · 计划原文「pytest fixture / conftest / 参数化 / pytest-asyncio；mock 外部 HTTP：respx 或 unittest.mock，测试不打真实网络」
> **关键版本:** pytest 8.x/9.x · pytest-asyncio ≥ 0.24（本文写法兼容 1.x） · respx 0.2x（需 httpx ≥ 0.25） · httpx 0.28 · FastAPI 0.11x —— 2026-09 按官方文档校准
> **生成日期:** 2026-09-08

---

## 为什么学这个

看一个你下周就会遇到的场景：W1 的交付物 `ai-service-template` 要求「pytest 全绿」，里面有一个调 LLM API 的异步接口。你坐下来写测试，立刻撞上三堵墙：

1. **真打 LLM API 不现实**。一次调用几秒、几分钱，测试套件跑一次几十次调用——又慢又贵，而且 LLM 每次返回都不一样，`assert reply == "..."` 永远失败。测试需要的是**确定性的输入和输出**，这只能靠 mock（用假对象替身替换真实的外部依赖）实现。
2. **接口是 `async def` 的，普通 pytest 根本测不了**。更糟的是它不报错，而是「假通过」——测试显示绿色，里面的断言一行都没执行。这是异步测试里最阴险的坑。
3. **测试之间互相污染**。这个测试改了环境变量，下个测试读到的就是脏值；这个测试用了共享的数据库连接，下个测试的状态就被污染了。需要一套机制保证每个测试拿到干净、独立的环境。

pytest 的 **fixture**（第 2、3 章）解决第 3 堵墙，**pytest-asyncio**（第 5 章）解决第 2 堵墙，**mock 工具**（第 6、7 章）解决第 1 堵墙，**参数化**（第 4 章）让你用一份代码测 N 组场景——W11 做 prompt 回归测试时全靠它。这四件事合起来，就是后面三个项目「有测试、测试全绿、测试不打真实网络」的全部地基。

---

## 重点标注（星级导航）

> 星级标准：★★★ 必须精通（学完当天就要能默写关键代码）｜★★☆ 需要理解（能讲清楚原理，细节可查）｜★☆☆ 了解即可（用到再回来看）

| 章节                            | 星级  | 为什么（对照学习目的/计划周次）                                                   |
| ----------------------------- | --- | ------------------------------------------------------------------ |
| 第 1 章 pytest 断言与发现            | ★★☆ | 你「做过但不熟」，本章把散的知识收敛成体系，半小时过完                                        |
| 第 2 章 fixture                 | ★★★ | 测试体系的第一块地基：后面每个项目的 conftest 都靠它；W2 项目一要求「pytest 覆盖（mock 掉 LLM 调用）」 |
| 第 3 章 conftest 与内置 fixture    | ★★★ | `ai-service-template` 骨架的共享测试设施就放这里；monkeypatch 管环境变量是每周都会用的       |
| 第 4 章 参数化                     | ★★★ | 一份断言测 N 组输入；W11「eval 接进 pytest，prompt 改动必须跑一遍」的载体就是它               |
| 第 5 章 pytest-asyncio          | ★★★ | async 测试「假通过」是隐形炸弹；W1 骨架全是 async 接口，不配它测试等于没写                      |
| 第 6 章 mock 概念 + unittest.mock | ★★★ | 「测试不打真实网络」的第一种手段；mock LLM SDK、管 API key 环境变量都靠它                    |
| 第 7 章 respx                   | ★★★ | mock httpx 的专用工具，W2 的 OpenAI 兼容调用、错误重试逻辑全用它测——比 unittest.mock 优雅得多 |
| 第 8 章 测试 FastAPI 应用           | ★★★ | W1 周五交付物直接抄综合示例：TestClient + 依赖覆盖 + respx 的组合拳                     |
| 综合示例                          | ★★★ | mini 版 `ai-service-template` 测试套件，可直接跑                             |

**一日学习路线（对应 W1 周四全天）**：

- 上午：第 1 章（快速过）→ 第 2 章（重点 yield 与 scope）→ 第 3 章 → 第 4 章 → 第 5 章
- 下午：第 6 章 → 第 7 章 → 第 8 章 → 综合示例亲手敲一遍
- 延后：fixture 的 `params=` 参数化（4.4 节）、respx 的 `M()` 组合模式（7.3 节）用到再回来看

---

## 第 1 章：pytest 断言与测试发现 ★★☆

### 1.1 概念解释

pytest 的哲学是「**裸 assert + 约定优于配置**」：不需要 `self.assertEqual` 这种断言方法，直接写 `assert`，失败时 pytest 会做**断言重写**（assertion rewriting），把表达式中间值都打印出来。测试文件和函数靠命名约定被发现，不需要注册。

### 1.2 代码示例

```python
# 文件名必须是 test_*.py 或 *_test.py，pytest 才会收集
def add(a, b):
    return a + b


def test_add():                      # 函数名必须 test_ 开头
    assert add(1, 2) == 3            # 裸 assert，失败时能看到 1+2 的实际值


def test_add_rejects_none():
    import pytest
    with pytest.raises(TypeError):   # 断言「应该抛异常」的标准写法
        add(None, 2)
```

常用命令行（够用了，别背更多）：

```bash
uv run pytest                      # 跑全部
uv run pytest tests/test_chat.py   # 只跑一个文件
uv run pytest -k "chat"            # 按名字过滤，只跑名字含 chat 的
uv run pytest -x                   # 第一个失败就停
uv run pytest --tb=short           # 失败堆栈用短格式
```

### 1.3 ⚠️ 常见坑

- **测试没被发现**：99% 是命名不符合约定——文件不是 `test_*.py`，或函数不是 `test_` 开头。跑 `pytest --collect-only` 看收集到了什么。
- **断言裸对象真假**：`assert result` 失败时只告诉你 `assert <对象>`，看不出为什么。写成 `assert result == expected` 或 `assert result.status == "ok"`，断言重写才能展示中间值。

---

## 第 2 章：fixture —— 测试的「准备与清理」★★★

### 2.1 概念解释

**fixture（夹具）**：为测试提供「前置准备 + 后置清理」的函数。你可以把它理解为**带自动回收的测试材料供给系统**——测试函数在参数列表里「点菜」，pytest 看到参数名就去找同名 fixture，执行它并把返回值递进来。

对比你已经熟悉的两种老办法：

| 方式 | 问题 |
| ---- | ---- |
| 每个测试开头复制粘贴 setup 代码 | 重复、改一处漏一处 |
| `unittest` 的 `setUp/tearDown` | 整个类共享一套，粒度粗，不能组合 |

fixture 的优势：**按需点菜**（要什么声明什么）、**自动组合**（fixture 可以依赖 fixture）、**每个测试默认拿到全新一份**（测试之间不互相污染）。

### 2.2 代码示例

最基本的用法——参数名即声明：

```python
import pytest


@pytest.fixture
def sample_prompt():
    return "把这段话翻译成英文：你好"


def test_prompt_not_empty(sample_prompt):   # 参数名 = fixture 名，pytest 自动注入
    assert len(sample_prompt) > 0
```

**yield fixture（setup/teardown 一体）**——这是你要默写的核心模式：

```python
@pytest.fixture
def llm_client():
    client = FakeLLMClient()        # setup：yield 之前
    yield client                    # 把 client 交给测试用
    client.close()                  # teardown：测试结束后自动执行
```

yield 之前是准备，yield 之后是清理。测试失败也**照样执行清理**，不会泄漏。真实项目里，`httpx.AsyncClient`、数据库连接、临时目录都这么管。

**scope（作用域）**——控制一个 fixture 在多大范围内只建一次：

```python
@pytest.fixture(scope="session")    # 整个测试会话只建一次，所有测试共享
def expensive_model():
    return load_model()
```

| scope 值 | 生命周期 |
| ---- | ---- |
| `function`（默认） | 每个测试函数一份——最安全，默认选它 |
| `class` / `module` / `package` | 每个类/模块/包一份 |
| `session` | 整次 pytest 运行一份——只用于真正昂贵的东西（加载大模型、起容器） |

**autouse**——不给参数也自动套用，适合「所有测试都必须有的环境」：

```python
@pytest.fixture(autouse=True)
def no_real_network():              # 见第 3 章：杜绝测试打真实网络
    ...
```

### 2.3 ⚠️ 常见坑

- **scope 调大后共享了可变对象**。`function` scope 下每个测试重新执行 fixture，互不干扰；改成 `module` 后 N 个测试共享同一个返回值——一个测试改了它（比如往 list 里 append），后面的测试全被污染。scope 越大越省时间，也越危险。
- **fixture 返回 `None` 还是没有生效**：写了 `return` 之外的分支忘了 return；yield fixture 里写了 `return client` ——yield fixture 里应该用 `yield`，别混 `return`。
- **fixture 名与参数名对不上**：pytest 报 `fixture 'xxx' not found`。检查拼写，并确认它定义在当前文件或 conftest.py（见第 3 章）里。

---

## 第 3 章：conftest.py 与内置 fixture ★★★

### 3.1 概念解释

**conftest.py** 是 pytest 的「共享配置文件」：放在哪个目录，那个目录（含子目录）下的所有测试都能自动使用其中的 fixture，**不需要 import**。这就是 fixture 体系的「集中供餐处」——跨文件复用的测试设施全放这里。

规则很简单：

- `tests/conftest.py` → 所有测试可用
- `tests/api/conftest.py` → 只有 `tests/api/` 下的测试可用，且可以**覆盖**上层同名 fixture（就近原则）
- conftest 可以层层嵌套，离测试最近的那个生效

同时认识三个**内置 fixture**（不用定义，直接当参数用）：

| 内置 fixture | 作用 | 典型场景 |
| ---- | ---- | ---- |
| `monkeypatch` | 安全地临时改属性/环境变量/字典，测试结束自动还原 | 设 `LLM_API_KEY`、替换函数 |
| `tmp_path` | 每个测试独享的临时目录（`pathlib.Path`） | 测试写文件（导出 Excel） |
| `capsys` | 捕获 stdout/stderr 输出 | 测日志/打印 |

### 3.2 代码示例

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient

from app.main import app


@pytest.fixture
def fake_api_key(monkeypatch):
    monkeypatch.setenv("LLM_API_KEY", "test-key-123")   # 测试结束自动还原


@pytest.fixture
def client(fake_api_key):                # fixture 依赖 fixture，自动组装
    with TestClient(app) as c:
        yield c


@pytest.fixture(autouse=True)            # 所有测试自动生效：禁止真网请求
def no_real_http(request):
    if "respx_mock" not in request.fixturenames:
        import httpx
        raise RuntimeError("网络相关测试必须使用 respx_mock 拦截（见第 7 章）")
```

monkeypatch 的常用操作（全部自动还原）：

```python
def test_env_and_attr(monkeypatch, tmp_path):
    monkeypatch.setenv("LLM_MODEL", "deepseek-chat")   # 设环境变量
    monkeypatch.delenv("OPENAI_API_KEY", raising=False)  # 删环境变量（不存在也不报错）

    monkeypatch.setattr("app.llm.API_BASE", "http://test.local/v1")  # 改模块属性

    out = tmp_path / "result.xlsx"                     # 每个测试独享的临时路径
    assert out.parent.exists()
```

### 3.3 ⚠️ 常见坑

- **patch 打错了位置**（mock 领域第一大坑，官方文档专门有一节叫 "where to patch"）：如果 `app/llm.py` 写了 `from openai import OpenAI`，那么要 patch 的是 `app.llm.OpenAI`（使用处的名字），而不是 `openai.OpenAI`（定义处的名字）——因为 `from ... import` 已经把名字复制进了 `app.llm` 的命名空间。monkeypatch.setattr 同理。
- **conftest 放错目录**：放在项目根目录而不是 `tests/` 下，或目录名拼错（必须是 `conftest.py`），fixture 就「神秘失踪」。
- **不要把 conftest 堆成垃圾场**：只放「多个文件共享」的 fixture；只有单个文件用的，放测试文件顶部，离使用处越近越好读。

---

## 第 4 章：参数化 —— 一份断言测 N 组场景 ★★★

### 4.1 概念解释

**参数化（parametrize）**：用装饰器给一个测试函数喂多组参数，pytest 会把它展开成 N 个独立测试，每组各跑一次、各报各的成败。解决的问题很直白：「翻译功能要测中→英、英→中、空串」总不能复制三个几乎一样的测试函数。它也是 W11 eval 回归测试的载体——eval 数据集（问题 + 期望答案）天然就是一张参数表。

### 4.2 代码示例

标准三件套：**参数名字符串、数据列表、测试函数**：

```python
import pytest


@pytest.mark.parametrize(
    "text,lang,expected",
    [
        ("hello", "en", "你好"),
        ("你好", "zh", "hello"),
        ("", "en", ""),          # 边界情况：空输入
    ],
)
def test_translate(text, lang, expected):
    assert translate(text, lang) == expected
```

跑起来是三个独立测试：`test_translate[hello-en-你好]`、`test_translate[你好-zh-hello]`、`test_translate[-en-]`——失败能精确定位到哪组数据。

给某组数据单独打标记（比如已知有 bug 的组合，先标 xfail 不挡 CI）：

```python
@pytest.mark.parametrize(
    "text,expected",
    [
        ("hello", "你好"),
        pytest.param("🚀", "🚀", marks=pytest.mark.xfail(reason="emoji 待修复")),  # 预期失败
    ],
)
def test_translate_zh(text, expected):
    assert translate(text, "zh") == expected
```

**堆叠装饰器 = 笛卡尔积**（2 × 3 = 6 个测试），适合测「多模型 × 多场景」组合：

```python
@pytest.mark.parametrize("model", ["deepseek-chat", "gpt-4o-mini"])
@pytest.mark.parametrize("temperature", [0.0, 1.0, 2.0])
def test_generate(model, temperature):
    ...
```

### 4.3 ⚠️ 常见坑

- **参数是按引用原样传入的，不做拷贝**（官方文档明确说明）。如果某组参数是 list/dict 且测试里改了它，后面的参数组会看到被改过的值。参数数据尽量用不可变对象（tuple、str）。
- **名字串与数据形状不匹配**：`"text,expected"` 是两个名字，数据却给单元素 tuple，会直接 TypeError。名字个数 = 每个 tuple 的元素个数。
- **中文参数在测试 ID 里显示成转义**：默认非 ASCII 会被转义显示。测试名可读性重要时，用 `ids=` 参数自定义：`parametrize(..., ids=["en2zh", "zh2en", "empty"])`。

### 4.4 fixture 的 params= 参数化 ★☆☆

fixture 也可以参数化（`@pytest.fixture(params=[...])`），让「准备的数据」变多份。语义上：**测函数用 `@pytest.mark.parametrize`，测基础设施（数据库、模型）多配置才用 fixture params**。初学先掌握前者，后者 W4 配 pgvector 时再回来看。

---

## 第 5 章：pytest-asyncio —— 让 async 测试真正执行 ★★★

### 5.1 概念解释

先看这个坑有多大。装了 FastAPI 但没装 pytest-asyncio 时：

```python
async def test_translate():
    assert await translate("hello", "zh") == "???"   # 断言永远不可能成立的值
```

跑 pytest：**1 passed**，只有一行小字警告 `RuntimeWarning: coroutine 'test_translate' was never awaited`。

原因：pytest 调用 `test_translate()` 只拿到了一个协程对象（复习 [asyncio 教程](asyncio-tutorial.md) 第 1 章：调用协程函数不执行它），协程对象是真值，于是「通过」。**断言一行都没跑**——这就是「假通过」，CI 全绿但啥也没测。

**pytest-asyncio** 就是解决这个的插件：它创建事件循环、把 async 测试函数真正 `await` 起来。它有两种**发现模式**：

| 模式           | 行为                                                 | 适用                         |
| ------------ | -------------------------------------------------- | -------------------------- |
| `strict`（默认） | 必须给每个 async 测试加 `@pytest.mark.asyncio` 才会被执行       | 项目里混用 trio 等其他异步框架         |
| `auto`       | 所有 `async def` 测试自动按 asyncio 跑，async fixture 也自动识别 | **纯 asyncio 项目（你的情况）——推荐** |

### 5.2 代码示例

一次性配置（写进 `pyproject.toml`，之后再也不用想这件事）：

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"   # 显式声明，消除未来版本警告
```

配置后写法极简，无需任何装饰器：

```python
# asyncio_mode = "auto" 下，直接写
async def test_translate():
    assert await translate("hello", "zh") == "你好"


async def test_llm_client(http_client):    # async fixture 也直接用
    result = await http_client.get("/health")
    assert result.status_code == 200
```

**async fixture**（准备/清理本身是异步的，比如开关 `httpx.AsyncClient`）：

```python
import pytest
import httpx


@pytest.fixture                       # auto 模式下普通装饰器即可
async def http_client():
    async with httpx.AsyncClient(base_url="http://test") as client:
        yield client                  # 清理由 async with 保证
```

若用 `strict` 模式，则要写 `@pytest.mark.asyncio` 装饰每个测试、`@pytest_asyncio.fixture` 装饰 async fixture。

**事件循环作用域（loop_scope）**：默认每个测试函数一个全新事件循环（隔离性最好）。当 async fixture 用 `scope="module"/"session"` 时，必须让测试跑在同一个循环里，否则对象「绑定」在旧循环上会报错：

```python
@pytest.fixture(scope="module")                     # 循环级 fixture
async def shared_client():
    async with httpx.AsyncClient() as c:
        yield c


@pytest.mark.asyncio(loop_scope="module")           # 测试也声明用 module 级循环
async def test_a(shared_client):
    ...
```

### 5.3 ⚠️ 常见坑

- **「假通过」**（见 5.1）：装完插件记得确认 pytest 输出的 plugins 行里有 `asyncio-x.x.x`。
- **`RuntimeError: ... attached to a different loop`**：session/module 级 async fixture 创建的对象（httpx 客户端、asyncpg 连接池），被 function 级循环里的测试使用。解法：fixture 和测试的 loop scope 保持一致（fixture 与测试都声明 `scope="module"` + `loop_scope="module"`）。FastAPI 官方也提醒：需要事件循环的对象要在循环内部创建。
- **老教程的 `event_loop` fixture 覆盖写法已过时**：pytest-asyncio 0.23 起废弃、1.0 已移除自定义 `event_loop` 的玩法。现在统一用 `loop_scope` 参数和 `asyncio_default_test_loop_scope` 配置。看到旧文章教你重定义 `event_loop`，直接关掉，[官方 concepts](https://pytest-asyncio.readthedocs.io/en/v0.26.0/concepts.html) 是唯一权威。

---

## 第 6 章：mock 概念 + unittest.mock ★★★

### 6.1 概念解释

**mock（替身测试）**：把测试中「不可控、慢、贵」的依赖换成可控的假对象。AI 应用里最典型的就是 LLM API：mock 之后，返回内容由你定（确定性）、零延迟、零成本。

先认识替身家族（test double，术语一次分清，后面文档都这么叫）：

| 替身类型 | 干什么 | AI 应用例子 |
| ---- | ---- | ---- |
| **stub** | 给固定回答，不理会输入 | `fake.complete() -> "你好"` |
| **fake** | 有简化但能用的真实现 | 内存版向量库替代 pgvector |
| **spy** | 包住真对象并记录调用 | 记录 LLM 被调了几次（测缓存命中率） |
| **mock** | stub + 记录调用 + 可断言调用方式 | 断言「重试时带了正确的 header」 |

Python 标准库 `unittest.mock` 提供这套工具（pytest 也能直接用，两者无缝）。核心三件：

- **`MagicMock`**：一个「万能假对象」——访问任何属性/方法都自动生成 mock，调用记录自动保存，魔法方法（`__str__`、`__len__` 等）也可配置。
- **`patch`**：在**限定范围内**把某个名字替换成 mock，范围结束自动还原。三种用法：装饰器 / `with` 上下文 / `start()`-`stop()`。
- **`AsyncMock`**：异步版 mock——`await mock()` 返回 `return_value`，并支持 `assert_awaited_once_with`。

### 6.2 代码示例

**MagicMock 基本功**：

```python
from unittest.mock import MagicMock, call

llm = MagicMock()
llm.complete.return_value = "你好"          # 预设返回值
llm.complete.with_model.return_value = "OK"

assert llm.complete("hello") == "你好"      # 怎么调都返回预设值

llm.complete.assert_called_once_with("hello")   # 断言：被调用一次、参数是 hello
llm.complete.assert_any_call("hello")           # 断言：至少这样调用过一次
llm.complete.assert_not_called()                # 断言：从未被调用（测缓存时用）
```

**patch 三种用法**（以替换 `app.llm.LLMClient` 为例）：

```python
from unittest.mock import patch

# 用法 1：装饰器（注意参数顺序 = 装饰器自下而上）
@patch("app.llm.LLMClient")            # 后生效
@patch("app.config.DEBUG", True)       # 先生效 → 第一个参数
def test_chat(MockClient, debug_true):
    MockClient.return_value.complete.return_value = "hi"
    ...

# 用法 2：上下文管理器（推荐，范围一目了然）
def test_chat():
    with patch("app.llm.LLMClient") as MockClient:
        MockClient.return_value.complete.return_value = "hi"
        ...

# 用法 3：手动 start/stop（fixture 里用，配合 yield 自动 stop）
@pytest.fixture
def mock_llm():
    p = patch("app.llm.LLMClient")
    mock = p.start()
    yield mock
    p.stop()
```

**AsyncMock —— mock 异步方法**（LLM SDK 的方法几乎都是 async）：

```python
from unittest.mock import AsyncMock

llm = AsyncMock()
llm.complete.return_value = "你好"

async def test_complete():
    assert await llm.complete("hello") == "你好"
    llm.complete.assert_awaited_once_with("hello")    # 异步专用断言
```

### 6.3 ⚠️ 常见坑

- **patch 了却不生效**（"where to patch" 问题，第 3 章讲过，这里再强调一次，因为 90% 的 mock 失败都是它）：永远 patch「**被测代码命名空间里的名字**」。`app/llm.py` 里 `from llm_sdk import Client` → patch `"app.llm.Client"`。判断方法：在被测代码文件里看到那个名字是从哪 import 的，就 patch 哪个模块的前缀。
- **没打 spec，拼错属性不报错**：`llm.complte()`（拼错）在裸 MagicMock 上静默返回一个新 mock，测试照样绿。给 mock 加规格：`MagicMock(spec=LLMClient)` 或 `patch(..., autospec=True)`，拼错立刻 AttributeError。
- **mock 断言打在错误对象上**：`patch` 装饰器注入的是「类/模块的 mock」，断言调用要打到 `MockClient.return_value.xxx`（实例方法）；`with ... as m` 拿到什么就断言什么，层级想清楚再写。

---

## 第 7 章：respx —— 在 HTTP 层 mock httpx ★★★

### 7.1 概念解释

第 6 章 mock 的是「对象」，还有一种思路：**在 HTTP 传输层拦截**。你的代码用 httpx（或基于它的 OpenAI SDK）发请求，respx 把 httpx 底层的传输层整个接管——代码以为在发真请求，实际请求被路由规则捕获、返回你预设的响应。**代码零改动**。

对比两种 mock 策略，这决定你在项目里怎么选：

| | mock 对象（第 6 章） | mock HTTP（respx） |
| ---- | ---- | ---- |
| 拦截层 | Python 对象/方法 | httpx 传输层（同步/异步客户端全拦） |
| 代码侵入 | 需要知道类名、方法名 | 零侵入，只认 URL |
| 能测到 | 业务逻辑与对象的交互 | **真实序列化、header、URL 拼装是否正确** |
| 适用 | mock 注入进来的依赖（配合 FastAPI `Depends`） | 测调用外部 API 的客户端封装（重试、限流、错误处理） |

**决策**：测自己的业务逻辑 → 依赖注入 + mock 对象（第 8 章）；测「我对上游 API 的调用是否正确」→ respx。W2 项目一两种都会用到。

### 7.2 代码示例

安装后 respx 自动注册为 pytest 插件，直接用 `respx_mock` fixture（也支持 `@respx.mock` 装饰器和 `with respx.mock:` 上下文，效果相同）：

```python
import httpx
import respx   # 装了就有 fixture；不 import 也能用 fixture，但导入便于类型检查

CHAT_URL = "https://api.deepseek.com/v1/chat/completions"


def test_chat_ok(respx_mock):
    # 1. 定路由：什么请求会被拦、返回什么
    route = respx_mock.post(CHAT_URL).mock(
        return_value=httpx.Response(200, json={
            "choices": [{"message": {"content": "你好"}}]
        })
    )

    # 2. 正常发请求（同步/异步客户端都会被拦下，不会出网）
    resp = httpx.post(CHAT_URL, json={"model": "deepseek-chat", "messages": []})

    assert resp.json()["choices"][0]["message"]["content"] == "你好"
    assert route.called            # 3. 断言：这条路由真的被命中了
```

**side_effect 三种玩法**——这是测错误处理的杀手锏：

```python
# 玩法 A：抛异常（测超时/断网降级）
respx_mock.post(CHAT_URL).mock(side_effect=httpx.ConnectTimeout)

# 玩法 B：异常类列表，第一次 429、第二次 200（测指数退避重试）
route = respx_mock.post(CHAT_URL).mock(
    side_effect=[httpx.Response(429), httpx.Response(200, json={"choices": [...]})]
)

# 玩法 C：函数，按请求动态返回（可校验请求体）
def echo(request):
    body = json.loads(request.content)
    assert body["model"] == "deepseek-chat"
    return httpx.Response(200, json={"choices": [{"message": {"content": "ok"}}]})
respx_mock.post(CHAT_URL).mock(side_effect=echo)
```

**两个内置保险**（默认开启，专治「测试不小心打了真网」）：

- `assert_all_mocked=True`（默认）：任何**没被路由覆盖**的请求直接报错——想真出网都出不去，这正是计划要求的「测试不打真实网络」。
- `assert_all_called=True`（默认）：测试结束时检查所有路由都被调用过——定义了 mock 却忘了让代码走到那条路，测试失败。有时一条路由只为「兜底」而定义，可以 `@respx.mock(assert_all_called=False)` 关掉。

### 7.3 ⚠️ 常见坑

- **`AllMockedError`**：说明代码请求了一个你没 mock 的 URL。这通常是好事——它告诉你代码的请求 URL 和你以为的不一样（拼错、少了 `/v1`）。把报错里的实际 URL 抄进路由定义即可。
- **路由按添加顺序匹配**：先加具体的（精确 URL、特定 `params`），后加宽泛的（仅 `host=`）。顺序反了，宽泛路由会吃掉本该命中具体路由的请求。
- **base_url 前缀问题**：客户端用 `AsyncClient(base_url="https://api.x.com/v1")` 时，代码里写相对路径 `/chat/completions`。mock 路由要写**完整 URL**，或配置 `@pytest.mark.respx(base_url="https://api.x.com/v1")` 后用相对路径。

### 7.4 进阶用法 ★☆☆

`respx_mock.route(method="POST", host="api.x.com", path__regex="/v1/.*")` 的完整 pattern/lookup 体系、`M()` 组合模式、命名路由跨测试复用——用到时查 [RESPX User Guide](https://lundberg.github.io/respx/guide/)，不必现在背。

---

## 第 8 章：测试 FastAPI 应用 ★★★

### 8.1 概念解释

FastAPI 应用本身是一个 ASGI 对象，不需要起真服务器就能测——**在进程内直接调用**。两条路，按需选：

| 工具 | 测试函数写法 | 特点 |
| ---- | ---- | ---- |
| `TestClient`（Starlette 提供） | 普通 `def` | 最简单；`with` 写法会触发 lifespan |
| `httpx.AsyncClient` + `ASGITransport` | `async def`（配 pytest-asyncio） | 测试里能 `await` 其他异步代码（如 asyncpg 查库校验） |

日常 90% 用 TestClient 就够了——**接口是 async 的，测试函数照样可以是普通 `def`**，TestClient 内部帮你跑事件循环。只有「测试本身要 await 别的东西」时才需要 async 测试。

第三件武器是**依赖覆盖（dependency overrides）**：FastAPI 的 `Depends` 依赖可以整包替换成假的，这是「mock 注入依赖」与 FastAPI 的标准结合点。

### 8.2 代码示例

**TestClient + lifespan**：

```python
from fastapi.testclient import TestClient
from app.main import app            # 你的 FastAPI 实例


def test_health():                           # 注意：普通 def 就行
    with TestClient(app) as client:          # with → 触发 lifespan（startup/shutdown）
        resp = client.get("/health")
        assert resp.status_code == 200
        assert resp.json() == {"status": "ok"}
```

lifespan 在 `with` 进入/退出时触发；模块级 `client = TestClient(app)` 不进 `with` 就**不会**触发。你的骨架在 lifespan 里创建 `httpx.AsyncClient`，所以测试要用 `with` 写法。

**依赖覆盖 —— 一行换掉真 LLM**：

```python
from app.main import app, get_llm
from app.llm import LLMClient


class FakeLLM(LLMClient):
    async def complete(self, prompt: str) -> str:
        return f"echo: {prompt}"


def test_chat_with_fake_llm():
    app.dependency_overrides[get_llm] = lambda: FakeLLM(http=None)
    try:
        with TestClient(app) as client:
            assert client.post("/chat", json={"prompt": "hi"}).json() == {"reply": "echo: hi"}
    finally:
        app.dependency_overrides.clear()     # 记得清理，别污染其他测试
```

**async 测试（需要 await 时）**：

```python
from httpx import ASGITransport, AsyncClient
from app.main import app


async def test_health_async():                       # asyncio_mode=auto
    transport = ASGITransport(app=app)               # 不走网络，直接调 ASGI 应用
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        resp = await ac.get("/health")
    assert resp.status_code == 200
```

**测 SSE 流式接口**（W2 项目一会反复用到）：

```python
def test_stream_chat():
    with TestClient(app) as client:
        with client.stream("POST", "/chat/stream", json={"prompt": "hi"}) as resp:
            assert resp.status_code == 200
            chunks = [chunk for chunk in resp.iter_text()]   # 逐块收
    assert len(chunks) > 1                                    # 确实是分块到达
```

### 8.3 ⚠️ 常见坑

- **`ASGITransport` 不触发 lifespan**（官方文档明确警告）：async 测试里 `app.state.xxx`（lifespan 里创建的）会 AttributeError。需要 lifespan 又要 async 测试时，用 `asgi-lifespan` 库的 `LifespanManager`，或把依赖改为惰性创建。本教程综合示例绕开此坑：走 TestClient 的接口测试 + 不依赖 lifespan 的纯 async 单测分开写。
- **httpx 0.27+ 不再支持 `AsyncClient(app=app)`** 简写，必须 `transport=ASGITransport(app=app)`。老教程里的 `app=` 参数写法直接报错。
- **`dependency_overrides` 忘了 clear**：覆盖会跨测试残留（app 是模块级单例）。用 fixture 的 yield 清理（见综合示例）。

---

## 综合示例：mini 版 ai-service-template 测试套件

把全天知识串起来。目录结构（对应 W1 周五交付物的雏形）：

```text
ai-service-template/
├── app/
│   ├── __init__.py
│   ├── llm.py          # LLM 客户端封装（被测的「外部调用层」）
│   └── main.py         # FastAPI 应用
├── tests/
│   ├── conftest.py     # 共享 fixture
│   ├── test_health.py  # 接口测试（TestClient）
│   └── test_chat.py    # 接口 + 客户端测试（respx 拦 LLM 上游）
└── pyproject.toml
```

```toml
# pyproject.toml —— 关键配置
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"

[dependency-groups]
dev = ["pytest", "pytest-asyncio", "respx", "httpx"]
# 安装：uv add --dev pytest pytest-asyncio respx httpx
```

```python
# app/llm.py —— OpenAI 兼容格式的极简客户端
import os

import httpx

API_BASE = os.getenv("LLM_API_BASE", "https://api.deepseek.com/v1")


class LLMClient:
    def __init__(self, http: httpx.AsyncClient):
        self.http = http

    async def complete(self, prompt: str) -> str:
        resp = await self.http.post(
            f"{API_BASE}/chat/completions",
            headers={"Authorization": f"Bearer {os.getenv('LLM_API_KEY', '')}"},
            json={
                "model": "deepseek-chat",
                "messages": [{"role": "user", "content": prompt}],
            },
        )
        resp.raise_for_status()
        return resp.json()["choices"][0]["message"]["content"]
```

```python
# app/main.py
from contextlib import asynccontextmanager

import httpx
from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel

from .llm import LLMClient


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(timeout=30)
    yield
    await app.state.http.aclose()


app = FastAPI(lifespan=lifespan)


class ChatRequest(BaseModel):
    prompt: str


def get_llm() -> LLMClient:
    return LLMClient(app.state.http)


@app.get("/health")
async def health():
    return {"status": "ok"}


@app.post("/chat")
async def chat(req: ChatRequest, llm: LLMClient = Depends(get_llm)):
    try:
        return {"reply": await llm.complete(req.prompt)}
    except httpx.HTTPStatusError as e:
        raise HTTPException(502, f"upstream error: {e.response.status_code}")
    except httpx.HTTPError:
        raise HTTPException(504, "upstream timeout or connection error")
```

```python
# tests/conftest.py —— fixture 体系
import pytest
from fastapi.testclient import TestClient

from app.main import app


@pytest.fixture
def fake_api_key(monkeypatch):
    monkeypatch.setenv("LLM_API_KEY", "test-key")     # 环境变量：测完自动还原


@pytest.fixture
def client(fake_api_key):
    with TestClient(app) as c:                        # with → lifespan 正常触发
        yield c
```

```python
# tests/test_health.py
def test_health(client):
    assert client.get("/health").json() == {"status": "ok"}
```

```python
# tests/test_chat.py —— respx 拦截上游 + 参数化测错误分支
import httpx
import pytest

from app.llm import API_BASE, LLMClient

CHAT_URL = f"{API_BASE}/chat/completions"


def _llm_reply(text: str) -> httpx.Response:
    return httpx.Response(200, json={"choices": [{"message": {"content": text}}]})


def test_chat_ok(client, respx_mock):
    respx_mock.post(CHAT_URL).mock(return_value=_llm_reply("你好"))
    resp = client.post("/chat", json={"prompt": "hi"})
    assert resp.status_code == 200
    assert resp.json() == {"reply": "你好"}


@pytest.mark.parametrize("upstream_status", [429, 500])       # 两个独立测试
def test_chat_upstream_error(client, respx_mock, upstream_status):
    respx_mock.post(CHAT_URL).mock(return_value=httpx.Response(upstream_status))
    resp = client.post("/chat", json={"prompt": "hi"})
    assert resp.status_code == 502


def test_chat_connect_error(client, respx_mock):
    respx_mock.post(CHAT_URL).mock(side_effect=httpx.ConnectTimeout)
    resp = client.post("/chat", json={"prompt": "hi"})
    assert resp.status_code == 504


async def test_llm_client_hits_openai_schema(respx_mock):
    """纯 async 单测：验证客户端发出的请求体符合 OpenAI 兼容格式。"""
    def check_request(request):
        import json
        body = json.loads(request.content)
        assert body["messages"] == [{"role": "user", "content": "ping"}]
        assert request.headers["Authorization"].startswith("Bearer ")
        return _llm_reply("pong")

    respx_mock.post(CHAT_URL).mock(side_effect=check_request)

    async with httpx.AsyncClient() as http:
        assert await LLMClient(http).complete("ping") == "pong"
```

跑起来：

```bash
uv run pytest -v
# test_health.py::test_health PASSED
# test_chat.py::test_chat_ok PASSED
# test_chat.py::test_chat_upstream_error[429] PASSED
# test_chat.py::test_chat_upstream_error[500] PASSED
# test_chat.py::test_chat_connect_error PASSED
# test_chat.py::test_llm_client_hits_openai_schema PASSED
```

这一套用到了：conftest 共享 fixture（第 3 章）、monkeypatch 环境变量（第 3 章）、yield fixture（第 2 章）、参数化（第 4 章）、async 自动模式测试（第 5 章）、respx 路由 + side_effect（第 7 章）、TestClient + lifespan（第 8 章）——没有一个测试碰过真实网络，全程毫秒级。把它扩一扩，W1 周五的「pytest 全绿」就有了。

---

## 附录 A：API 速查表

| API/写法 | 作用 | 关键注意点 |
| ---- | ---- | ---- |
| `@pytest.fixture` | 定义夹具 | 参数名匹配注入；function scope 默认每测试一份 |
| `@pytest.fixture(scope="session")` | 全程一份 | 共享可变对象有污染风险 |
| `@pytest.fixture(autouse=True)` | 自动生效 | 全局环境（如禁真网） |
| yield fixture | setup + teardown | yield 后的代码测试失败也执行 |
| fixture 依赖 fixture | 组装复杂环境 | 声明为参数即可 |
| `@pytest.mark.parametrize("a,b", [...])` | 参数化测试 | 名字串个数 = tuple 元素数；值按引用传递 |
| `pytest.param(..., marks=pytest.mark.xfail)` | 单组数据打标 | ids= 自定义测试名 |
| `monkeypatch.setenv / delenv` | 环境变量 | 测后自动还原 |
| `monkeypatch.setattr("pkg.mod.name", val)` | 替换名字 | patch 使用处，不是定义处 |
| `tmp_path` | 临时目录 fixture | 每测试独享 |
| `pytest.raises(Exc)` | 断言抛异常 | `with` 语法 |
| `asyncio_mode = "auto"` | pyproject 配置 | async 测试/fixture 零装饰器 |
| `@pytest.mark.asyncio(loop_scope="module")` | 循环作用域 | 与 async fixture scope 对齐，防 different loop |
| `MagicMock(return_value=...)` | 万能假对象 | 配 `spec=` 防拼错属性 |
| `mock.assert_called_once_with(...)` | 断言调用 | AsyncMock 用 `assert_awaited_*` |
| `patch("a.b.name")` | 限时替换 | 装饰器/with/start-stop 三用法 |
| `AsyncMock` | 异步方法替身 | `await mock()` 直接可用 |
| `respx_mock.post(url).mock(return_value=httpx.Response(...))` | mock httpx 响应 | 未覆盖的请求默认报 AllMockedError |
| `route.side_effect=[Response(429), Response(200)]` | 序列响应 | 测重试；异常类直接抛 |
| `route.called / route.call_count` | 断言命中 | 路由按添加顺序匹配，具体在前 |
| `TestClient(app)` | 同步测 FastAPI | `with` 才触发 lifespan |
| `app.dependency_overrides[dep] = fake` | 替换 Depends | 用完 clear |
| `ASGITransport(app=app)` | async 测 FastAPI | 不触发 lifespan；httpx 0.27+ 必须 transport= |

## 附录 B：官方文档精确链接

**pytest（2026-09 实际访问）**

- Fixtures 全部：https://docs.pytest.org/en/stable/how-to/fixtures.html
- 参数化：https://docs.pytest.org/en/stable/how-to/parametrize.html
- monkeypatch：https://docs.pytest.org/en/stable/how-to/monkeypatch.html

**pytest-asyncio**

- 首页与基本用法：https://pytest-asyncio.readthedocs.io/en/latest/
- 概念（模式 / loop_scope / 每收集器一循环，v0.26.0 版）：https://pytest-asyncio.readthedocs.io/en/v0.26.0/concepts.html
- 配置项（asyncio_mode / *_loop_scope，仓库 main 分支源文档）：https://github.com/pytest-dev/pytest-asyncio/blob/main/docs/reference/configuration.rst

**respx**

- 首页 + QuickStart（respx_mock fixture）：https://lundberg.github.io/respx/
- User Guide（路由 / side_effect / 断言开关 / 回滚）：https://lundberg.github.io/respx/guide/

**FastAPI**

- Testing（TestClient）：https://fastapi.tiangolo.com/tutorial/testing/
- Async Tests（ASGITransport、lifespan 警告）：https://fastapi.tiangolo.com/advanced/async-tests/

**unittest.mock（标准库）**

- 模块文档（Mock / patch / MagicMock / autospec / where to patch）：https://docs.python.org/3/library/unittest.mock.html
