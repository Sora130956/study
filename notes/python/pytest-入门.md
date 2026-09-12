# pytest 核心入门：fixture、conftest.py、参数化、测异步代码

- 目标读者：会写 Python 语法（函数、类、字典、`async def` 见过但不熟），几乎没写过自动化测试，平时靠 `print` 手动验证
- 前置知识：见第 3 节的十个概念（断言、三段结构、测试发现规则、fixture、fixture scope、conftest.py、参数化、测试替身、monkeypatch、协程与 await）。除此之外不需要别的准备
- 学习时长：约 2.5 小时（读 1 小时 + 亲手敲第 4、5 节 1.5 小时） / 可跳过：第 2 节是纯背景推演，急着写代码可以先跳，但建议回头补
- 环境：Python 3.12+、pytest 9.1.1、pytest-asyncio 1.4.0

---

## 第 1 节：它是什么

**pytest 是一个帮你自动跑「检查代码对不对」的工具**：你把「什么输入应该得到什么输出」写成一个个普通的 Python 函数，pytest 负责找到它们、全部跑一遍、告诉你哪个不符合预期。

类比：pytest 就像**汽车出厂前的检测流水线**。

你造了一辆车（你的代码）。以前你是自己坐进去开一圈，听听有没有异响——这叫手动验证。检测流水线的做法不一样：它有一排固定工位，刹车工位踩一脚看几米停下，灯光工位打开看亮不亮，涉水工位泼水看漏不漏。每个工位就是一个**测试函数**；工位上事先摆好的水桶、测距仪就是 **fixture**（第 3 节）；同一个刹车工位换 5 种车速各试一遍，就是**参数化**（第 3 节）；所有工位共用的那套仪器放在车间入口的架子上，就是 **conftest.py**（第 3 节）。

关键点：流水线的价值不在于第一次检出问题，而在于**你改完车再跑一遍只要几秒**。这句话是整篇教程的核心。

---

## 第 2 节：它解决什么问题

### 不写测试，你会怎么验证？

假设你在写 AI 应用的一个小函数：把用户消息估算成 token 数，好判断要不要截断。

朴素做法一，手动跑脚本看 `print`：

```python
# 你大概会在文件末尾临时加几行，跑一次，眼睛看输出对不对
def estimate_tokens(text: str) -> int:
    return len(text) // 4

print(estimate_tokens("hello world"))   # 眼睛看：输出 2，嗯……应该对吧？
print(estimate_tokens(""))              # 输出 0，对
```

朴素做法二，稍微「自动」一点，自己写 `if` 判断：

```python
result = estimate_tokens("hello world")
if result != 2:                      # 自己比对期望值
    print("错了！期望 2，实际", result)   # 错了就打印一行
```

这两种做法能用，但有四个痛点，而且会随项目变大迅速恶化：

1. **改了 A 处不知道 B 处坏了。** 你把 `// 4` 改成更准的算法，`estimate_tokens` 自己测过了，但三个星期前调用它的那个「超长对话自动截断」逻辑悄悄崩了——没人再跑一遍它。
2. **每次改动要手动点一遍 10 个场景。** 空字符串、纯中文、超长文本、None、emoji……第一次你认真试了 10 种，第二次改动你只试了 2 种，第三次一种都没试。
3. **上线后才发现边界情况崩。** 手动验证时你输的都是「正常」输入，因为脑子会自动避开麻烦的输入。用户不会。
4. **看 print 靠人眼，人眼会累会骗自己。** 输出一多，你只会扫一眼「没红字，应该没事」。

**根源一句话：手动验证的成本每次都是全价，所以你必然会越改越少验证——而自动化测试把这个成本一次性付清，之后每次重跑接近免费。**

### 经典应用场景

- **重构 / 改算法**：先有测试，改完跑一遍确认行为没变，这是重构唯一安全的姿势。
- **边界与异常**：空输入、超长输入、非法 JSON、上游超时——把它们钉成测试用例，一次写好永久生效。
- **调外部服务的代码**（本文重点，你正在学的 AI 应用就是这类）：调 LLM 的函数不能每跑一次测试就真的花钱、等 2 秒、还可能因为模型输出随机而结果不一样。测试要在**不联网**的前提下验证你的解析、重试、错误处理逻辑对不对。
- **Web 接口**：FastAPI 端点返回的状态码、字段、错误码是不是符合约定。
- **协作与 CI**：别人（或三个月后的你）改代码时，测试是唯一会替你说话的那道防线。

---

## 第 3 节：读下去之前，先搞懂这些概念

十个概念，每个一行定义 + 一句大白话 + 最小示例。不用背，看懂就往下走，第 4 节会全部用上。

### 3.1 断言 assert

**定义**：`assert 条件` 是 Python 内置语句，条件为真就什么都不发生，为假就抛 `AssertionError`。

**大白话**：「我断定这里应该是这样，不是就当场报错」。它替代了你手写的 `if result != 2: print(...)`——不用自己打印，不满足就自动炸。

```python
assert 1 + 1 == 2      # 成立，这行什么都不发生，程序继续
assert 1 + 1 == 3      # 不成立，抛 AssertionError，测试算失败
```

### 3.2 测试的三段结构（准备 / 执行 / 断言）

**定义**：一个测试函数通常分三段——准备数据、调用被测代码、断言结果。英文叫 Arrange-Act-Assert。

**大白话**：摆好东西 → 动手 → 检查。养成这个节奏，测试就不会写成一团。

```python
def test_add():
    a, b = 2, 3                # 准备：造输入
    result = add(a, b)         # 执行：调被测函数
    assert result == 5         # 断言：检查业务结果
```

### 3.3 测试发现规则

<mark style="background: #BBFABBA6;">**定义**：pytest 默认自动收集**文件名** `test_*.py` 或 `*_test.py` 中的**函数名** `test_*` 作为测试。</mark>

**大白话**：文件和函数都得叫 `test_` 开头，pytest 才认它。名字不对 = pytest 视而不见（第 6 节有这个坑）。

```python
# 文件必须叫 test_calc.py 这类名字
def test_add(): ...   # 会被收集
def check_add(): ...  # 不会被收集，pytest 当它是普通函数
```

### 3.4 fixture（测试夹具）

<mark style="background: #BBFABBA6;">**定义**：用 `@pytest.fixture` 装饰的函数，负责为测试准备好某个东西（数据、客户端、临时目录），测试函数**通过把它的名字写成参数**来取用。</mark>

**大白话**：可复用的「准备工作」。放在工位上的水桶——需要它的工位报一声名字，它就出现在手边。

```python
@pytest.fixture
def user():                       # 定义一个叫 user 的 fixture
    return {"name": "小明"}

def test_name(user):              # 参数名写 user，pytest 就把上面的返回值传进来
    assert user["name"] == "小明"
```

> 参数名注入这件事对新手极其反直觉（你没有在任何地方「调用」`user()`）。第 4 节示例 2 会专门拆开讲机制。

### 3.5 fixture scope（作用域）

<mark style="background: #BBFABBA6;">**定义**：fixture 重建的频率。`function`（默认，每个测试函数重建一次）、`class`、`module`、`package`、`session`（整个测试会话只建一次）。</mark>

**大白话**：这个水桶是「每辆车换一桶新水」，还是「一整天用同一桶」。默认每次换新的，最安全。

```python
@pytest.fixture(scope="function")   # 默认值，写不写都一样
def cart(): return []               # 每个测试拿到的都是全新空列表

@pytest.fixture(scope="session")    # 整个测试会话共用一个，慎用（第 6 节的坑）
def cart(): return []
```

### 3.6 conftest.py

<mark style="background: #BBFABBA6;">**定义**：pytest 自动加载的特殊文件，放在里面的 fixture 不需要 import，同目录及子目录下的所有测试都能直接用。</mark>

**大白话**：车间入口那个公共仪器架。放这儿的东西，全车间随手可取。

```python
# tests/conftest.py —— 文件名必须一字不差
import pytest

@pytest.fixture
def api_key():                # tests/ 下任何测试直接写 api_key 参数就能用
    return "sk-test-fake"
```

### 3.7 参数化（parametrize）

<mark style="background: #BBFABBA6;">**定义**：用 `@pytest.mark.parametrize` 给一个测试函数喂多组输入，pytest 把它展开成多个独立测试。</mark>

**大白话**：同一个工位，换 5 种车速各试一遍，而且哪一种坏了它单独告诉你。

```python
@pytest.mark.parametrize("text,expected", [("abcd", 1), ("", 0)])  # 两组数据
def test_tokens(text, expected):        # 参数名要和上面字符串里的名字对上
    assert estimate_tokens(text) == expected   # 这个测试会跑 2 次
```

### 3.8 mock / fake / 测试替身

<mark style="background: #BBFABBA6;">**定义**：测试替身（test double）是替代真实依赖的假对象，统称。**fake** 是有简化实现的假货（假数据库=一个字典）；**mock** 是记录调用、返回预设值的空壳。</mark>

**大白话**：碰撞测试用假人，不用真人。你的代码要调 LLM，测试里给它一个「假 LLM」——立刻返回固定答案，不花钱不联网。

```python
class FakeLLM:                            # 一个 fake：结构像真的，行为是写死的
    async def chat(self, prompt): return "固定回答"
```

**⚠️ 边界在哪里（工作区硬规定，全篇最重要的一条纪律）：**

| 你在测什么        | 该替换什么                                              | 不该替换什么     |
| ------------ | -------------------------------------------------- | ---------- |
| 你自己的业务逻辑     | 它依赖的外部服务（注入 fake，FastAPI 用 `dependency_overrides`） | **被测对象本身** |
| 你的代码有没有正确调上游 | 在 HTTP 层拦截（respx，第 5 节提一句）                         | 被测对象本身     |
|              |                                                    |            |

**<mark style="background: #BBFABBA6;">禁止 mock 被测对象本身</mark>。** 如果你把要测的那个函数替换成假的，那测试跑的就是假货，你的真代码一行都没被验证过。这类「绿了但什么都没验证」的测试叫**装饰性测试**，第 6 节有正反例对照。

### 3.9 monkeypatch

**定义**：pytest 内置 fixture，<mark style="background: #BBFABBA6;">临时修改环境变量</mark>、属性、字典项，<mark style="background: #BBFABBA6;">**测试结束自动还原**</mark>。

**大白话**：借东西并保证归还。你把 `OPENAI_API_KEY` 改成假值，测试一结束它自动变回原样，不会污染下一个测试。

```python
def test_reads_key(monkeypatch):                        # 直接写 monkeypatch 参数即可
    monkeypatch.setenv("OPENAI_API_KEY", "sk-fake")     # 设环境变量，测完自动撤销
    assert load_settings().api_key == "sk-fake"
```

<mark style="background: #BBFABBA6;">**环境变量一律用 `monkeypatch.setenv`，禁止直接改 `os.environ</mark>`**（工作区规定，原因见第 6 节坑 4）。

### 3.10 协程与 await（一句话）

**定义**：`async def` 定义的函数调用后不执行，只返回一个**协程对象**，必须 `await`（或交给事件循环）它才真正跑。

**大白话**：`async def` 的函数调一下只是「下单」，`await` 才是「取货」。

```python
result = fetch_data()          # 只拿到协程对象，函数体一行都没执行
result = await fetch_data()    # 真正执行并拿到结果
```

> 这个区别是第 6 节「假通过」大坑的全部根源。想深入 asyncio 本身，另见 `asyncio-入门.md`。

### 术语速查表

| 术语 | 大白话 | 正文哪节 |
|---|---|---|
| assert 断言 | 我断定这里是这样，不是就报错 | 3.1、4 示例 1 |
| 三段结构 | 摆好 → 动手 → 检查 | 3.2、4 示例 1 |
| 测试发现 | 文件和函数都要 `test_` 开头 | 3.3、6 节 |
| fixture | 可复用的准备工作，靠参数名取用 | 3.4、4 示例 2 |
| scope | 这份准备工作多久重建一次 | 3.5、6 节坑 3 |
| conftest.py | 全车间公用的仪器架 | 3.6、4 示例 3、5 节 |
| parametrize | 一个测试喂多组数据 | 3.7、4 示例 2 |
| fake / mock | 替代真实依赖的假货 | 3.8、4 示例 3 |
| monkeypatch | 临时改，测完自动还原 | 3.9、4 示例 3 |
| 协程 / await | 下单 / 取货 | 3.10、6 节坑 1 |
| 装饰性测试 | 绿了但什么都没验证 | 3.8、6 节坑 5 |

---

## 第 4 节：从最简单的代码示例开始

三个示例递进：能跑 → 有真实需求 → 像项目里的样子。**请亲手敲，不要只看。**

### 示例 1：最小可运行版本

<mark style="background: #BBFABBA6;">先装依赖（工作区用 uv）：</mark>

```bash
uv add --dev pytest==9.1.1 pytest-asyncio==1.4.0
```

<mark style="background: #BBFABBA6;">两个文件。被测代码：</mark>

```python
# tokens.py —— 被测代码，普通模块，和测试无关，不需要 import pytest
def estimate_tokens(text: str) -> int:
    """粗估 token 数：英文大约 4 个字符 1 个 token。"""
    return len(text) // 4          # 整除，向下取整；"abcde"(5 字符) → 1
```

<mark style="background: #BBFABBA6;">测试代码：</mark>

```python
# test_tokens.py —— 文件名必须 test_ 开头，否则 pytest 根本不会看它（见 3.3）
from tokens import estimate_tokens        # 像平常一样 import 被测函数

def test_estimate_tokens_basic():         # 函数名必须 test_ 开头，才会被收集
    result = estimate_tokens("abcdefgh")  # 执行：8 个字符，期望 8//4 = 2
    assert result == 2                    # 断言：不等于 2 就抛 AssertionError，测试失败
```

<mark style="background: #BBFABBA6;">跑它：</mark>

```bash
uv run pytest -q          # -q 是 quiet，只输出摘要，日常最常用
```

真实输出（全绿时）：

```
.                                                                        [100%]
1 passed in 0.01s
```

那个 `.` 就是一个通过的测试。现在**故意把断言改成 `assert result == 3`** 再跑一次，看失败长什么样——这一步很重要，你要认识它的报错格式：

```
F                                                                        [100%]
=================================== FAILURES ===================================
_____________________________ test_estimate_tokens_basic ______________________

    def test_estimate_tokens_basic():
        result = estimate_tokens("abcdefgh")
>       assert result == 3
E       assert 2 == 3

test_tokens.py:6: AssertionError
=========================== short test summary info ============================
FAILED test_tokens.py::test_estimate_tokens_basic - assert 2 == 3
1 failed in 0.02s
```

**逐段解读：**

- `F` 代替了 `.`，表示 failed。
- `>` 那一行是出错的语句，`E` 那一行是 pytest 替你算出的实际值对比：`assert 2 == 3`。**这是 pytest 最贴心的地方**——你只写了 `assert result == 3`，它自动告诉你 `result` 其实是 2。这个功能叫断言重写（assertion rewriting），所以你永远不需要写 `assert result == 3, f"实际是{result}"` 这种手动消息。
- 最后 `FAILED 文件::函数名` 可以直接复制去单独重跑：`uv run pytest test_tokens.py::test_estimate_tokens_basic`。

改回 `== 2`，继续。

**这一步你得到了什么**：会写并运行一个测试，看得懂通过（`.`）和失败（`F` + `E assert 2 == 3`）两种输出。你已经比 `print` 前进了一大步——判断对错的活从你的眼睛转给了机器。

### 示例 2：加真实需求（fixture + 参数化 + 断异常）

需求升级：写一个「对话历史裁剪」函数，超出 token 预算就从最旧的消息开始丢，预算是负数要报错。

```python
# history.py
from tokens import estimate_tokens

def trim_history(messages: list[dict], budget: int) -> list[dict]:
    """从最旧的消息开始丢，直到总 token 不超过 budget。"""
    if budget < 0:                                    # 前置校验：负预算是调用方的 bug
        raise ValueError("budget 不能为负数")           # 主动抛异常，让错误尽早暴露
    kept = list(messages)                             # 拷一份，不改调用方传进来的列表
    while kept and sum(estimate_tokens(m["content"]) for m in kept) > budget:
        kept.pop(0)                                   # pop(0) 丢掉最旧的一条（列表头部）
    return kept
```

```python
# test_history.py
import pytest                                  # 用到 pytest.fixture / mark / raises 才需要 import
from history import trim_history

@pytest.fixture
def messages():
    """准备一组固定的对话历史，三条各 20 字符 = 各 5 token，合计 15 token。"""
    return [                                   # 返回值就是测试里拿到的东西
        {"role": "user", "content": "a" * 20},      # 5 token
        {"role": "assistant", "content": "b" * 20}, # 5 token
        {"role": "user", "content": "c" * 20},      # 5 token
    ]

def test_keeps_all_when_budget_enough(messages):   # 参数名 messages 对上 fixture 名，自动注入
    result = trim_history(messages, budget=100)    # 预算远超 15，应该一条都不丢
    assert len(result) == 3                        # 断言业务结果：条数没变

def test_drops_oldest_first(messages):
    result = trim_history(messages, budget=10)     # 只够放 10 token = 2 条，得丢 1 条
    assert len(result) == 2                        # 丢了一条
    assert result[0]["content"].startswith("b")    # 关键：留下的是较新的两条，最旧的 a 被丢

@pytest.mark.parametrize(                          # 一个测试函数，喂 4 组边界数据
    "budget,expected_count",                       # 参数名清单（逗号分隔的字符串）
    [
        (100, 3),      # 预算充足：全留
        (15, 3),       # 刚好等于总量：边界，不该丢（条件是 > budget 才丢）
        (14, 2),       # 差一点：丢最旧一条
        (0, 0),        # 预算为 0：全丢光
    ],
)
def test_trim_budget_boundaries(messages, budget, expected_count):
    # fixture 参数和 parametrize 参数可以混用，pytest 按名字分别注入
    assert len(trim_history(messages, budget)) == expected_count

def test_negative_budget_raises(messages):
    with pytest.raises(ValueError, match="不能为负数"):   # 断言这段代码必须抛 ValueError
        trim_history(messages, budget=-1)               # 如果它没抛，测试失败
```

跑：`uv run pytest -q`，输出 `7 passed`（3 个普通 + 4 个参数化展开）。

想看参数化展开成了什么，加 `-v`：

```bash
uv run pytest -v
```

```
test_history.py::test_trim_budget_boundaries[100-3] PASSED
test_history.py::test_trim_budget_boundaries[15-3] PASSED
test_history.py::test_trim_budget_boundaries[14-2] PASSED
test_history.py::test_trim_budget_boundaries[0-0] PASSED
```

方括号里是那组参数值——**每组是一个独立测试，一组失败不影响其他组**，报错也会直接告诉你是哪组挂的。这就是参数化比在测试里写 `for` 循环强的地方（for 循环第一组挂了后面就不跑了）。

**逐段解读：**

- `messages` fixture 负责「准备」，三个测试共用它，各自拿到**独立的新列表**（默认 function scope），所以 `test_drops_oldest_first` 里的 `pop` 不会影响别的测试。
- `test_drops_oldest_first` 的第二个断言 `startswith("b")` 是重点：只断言 `len == 2` 的话，函数丢错了一头（丢掉最新的）测试照样绿。**断言要咬住业务语义**，不只是形状。
- `pytest.raises` 是「我期望它出错」的写法，`match=` 用正则匹配异常消息。注意：出错才算通过，不出错算失败。

#### 单独展开 1：fixture 靠「参数名」注入的机制（最反直觉的一点）

你写 `def test_keeps_all_when_budget_enough(messages)`，从没调用 `messages()`，它却有值。这不是魔法，是 pytest 做了一件很朴素的事：

1. 收集阶段，pytest 记下所有 `@pytest.fixture` 函数的**名字** → 建一张登记表。
2. 准备运行某个测试前，用 `inspect` 读出这个测试函数的**参数名列表**（这在 Python 里是可读的元数据）。
3. 每个参数名去登记表里查，查到就调用那个 fixture 函数，把返回值作为该参数的实参。
4. 用关键字参数把测试函数调起来。

等价的普通写法，帮你彻底去掉魔法感：

```python
# pytest 实际帮你做的事，约等于：
def messages():                              # 就是个普通函数
    return [...]

def test_keeps_all_when_budget_enough(messages):
    ...

test_keeps_all_when_budget_enough(messages=messages())   # pytest 自动做了这一行
```

所以两条推论要记牢：**fixture 名字写错就注入失败**（报 `fixture 'xxx' not found`，第 6 节坑 2）；**fixture 名和参数名是唯一的连接方式**，跟 import、跟定义顺序都无关。

#### 单独展开 2：fixture scope 为什么默认该用 `function`

`function` scope 意味着每个测试函数都重新执行一遍 fixture，拿到全新对象。代价是慢一点，换来的是**测试之间完全隔离**。

隔离为什么值这个价：测试的价值建立在「失败信息可信」之上。如果 A 测试往共享列表里塞了数据，B 测试读到脏数据挂了，你会花半小时排查 B——而 bug 根本不在 B。更糟的是这类失败**跟执行顺序有关**，单独跑 B 是绿的，一起跑就红，这是最难查的一类问题。

```python
@pytest.fixture               # 默认 function：每个测试一个新列表，互不干扰
def cart(): return []

@pytest.fixture(scope="session")   # 整个会话一个列表：A 测试 append 了，B 测试就能看见
def cart(): return []
```

**规则**：默认写 `function`（即不写 scope）。只有当准备工作确实昂贵**且**对象确实只读（比如加载一个大模型的分词器、启一个容器）时才放大 scope，并且要明确保证没人改它。工作区验收清单第 5 步专门查这一条。

**这一步你得到了什么**：会用 fixture 抽出准备工作、用参数化批量覆盖边界、用 `pytest.raises` 验证错误路径，并且理解了 fixture 注入不是魔法而是「按参数名查表」。

### <mark style="background: #BBFABBA6;">示例 3：贴近项目的样子（conftest + monkeypatch + async + FastAPI 注入 fake）</mark>

现在做一个像真项目的结构：一个调 LLM 的异步服务，加一个 FastAPI 端点。

```
myapp/
├── pyproject.toml
├── app/
│   ├── llm.py          # LLM 客户端（真实实现，测试里不会真跑）
│   ├── service.py      # 业务逻辑（被测重点）
│   ├── deps.py         # FastAPI 依赖
│   └── main.py         # FastAPI 应用
└── tests/
    ├── conftest.py     # 共享 fixture
    ├── test_service.py
    └── test_api.py
```

<mark style="background: #BBFABBA6;">先开启异步测试支持，这一行是本节的前提：</mark>

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"        # 让 pytest 自动把 async def test_ 当异步测试跑（下面单独展开）
pythonpath = ["."]           # 让 tests/ 里能直接 import app.xxx，不用装成包
```

业务代码：

```python
# app/llm.py
import os
import httpx

class LLMClient:
    """真实的 LLM 客户端。测试里不会用它，而是用同形状的 fake 替代。"""
    def __init__(self, api_key: str | None = None):
        # 从环境变量读密钥；这行是后面 monkeypatch.setenv 的作用对象
        self.api_key = api_key or os.environ["LLM_API_KEY"]

    async def chat(self, prompt: str) -> str:
        async with httpx.AsyncClient() as client:                 # 真实网络请求
            resp = await client.post(
                "https://api.example.com/v1/chat",
                headers={"Authorization": f"Bearer {self.api_key}"},
                json={"prompt": prompt},
            )
            resp.raise_for_status()                                # 4xx/5xx 抛异常
            return resp.json()["reply"]
```

```python
# app/service.py
class SummaryService:
    def __init__(self, llm):
        # 依赖注入：llm 从外部传进来，不在内部 new。这一行让测试有机会塞 fake 进来
        self.llm = llm

    async def summarize(self, text: str) -> dict:
        if not text.strip():                        # 空输入不该浪费一次 LLM 调用
            return {"summary": "", "truncated": False}
        truncated = len(text) > 100                 # 超长就截断，并如实标记
        payload = text[:100] if truncated else text
        reply = await self.llm.chat(f"总结：{payload}")   # await：真正拿到结果（见 3.10）
        return {"summary": reply.strip(), "truncated": truncated}
```

<mark style="background: #BBFABBA6;">共享 fixture 放 conftest：</mark>

```python
# tests/conftest.py —— 文件名固定，pytest 自动加载，测试里不用 import
import pytest
from app.service import SummaryService

class FakeLLM:
    """假 LLM：形状和 LLMClient 一样（有 async chat），行为写死，不联网不花钱。

    注意它是「被测对象的依赖」的替身，不是被测对象 SummaryService 的替身。
    """
    def __init__(self, reply: str = "  这是摘要  "):
        self.reply = reply
        self.calls: list[str] = []       # 记下收到的 prompt，供测试断言「调用是否正确」

    async def chat(self, prompt: str) -> str:
        self.calls.append(prompt)        # 留痕
        return self.reply                # 固定返回，不做真实推理

@pytest.fixture
def fake_llm():
    """默认 function scope：每个测试一个全新 FakeLLM，calls 不会串（见 3.5）。"""
    return FakeLLM()

@pytest.fixture
def service(fake_llm):
    """fixture 可以依赖另一个 fixture：参数写 fake_llm 就注入进来。"""
    return SummaryService(llm=fake_llm)      # 真的业务对象 + 假的依赖

@pytest.fixture(autouse=True)
def fake_env(monkeypatch):
    """autouse=True：不用测试主动申请，自动对每个测试生效。

    作用是保证没有任何测试意外读到你机器上的真实密钥；测完 monkeypatch 自动还原。
    """
    monkeypatch.setenv("LLM_API_KEY", "sk-test-fake")   # 禁止写 os.environ[...]=...
```

<mark style="background: #BBFABBA6;">业务逻辑测试（异步）：</mark>

```python
# tests/test_service.py
import pytest

async def test_summarize_strips_reply(service, fake_llm):
    # asyncio_mode="auto" 下直接写 async def，不需要任何装饰器
    result = await service.summarize("讲讲 pytest")   # await 不能漏！漏了就是假通过（第 6 节坑 1）

    assert result["summary"] == "这是摘要"     # 断言业务结果：service 把 fake 返回的空格去掉了
    assert result["truncated"] is False       # 短文本不该被标记截断

async def test_summarize_truncates_long_text(service, fake_llm):
    long_text = "字" * 300                     # 300 字符，远超 100 的阈值

    result = await service.summarize(long_text)

    assert result["truncated"] is True                  # 业务结果 1：如实标记了截断
    assert len(fake_llm.calls) == 1                     # 业务结果 2：只调了一次 LLM
    assert len(fake_llm.calls[0]) < 150                 # 业务结果 3：发出去的 prompt 确实被截短了

async def test_summarize_skips_llm_for_blank_input(service, fake_llm):
    result = await service.summarize("   ")             # 纯空格

    assert result == {"summary": "", "truncated": False}
    assert fake_llm.calls == []          # 关键：一次 LLM 都没调，省钱逻辑生效了
```

FastAPI 端点 + 依赖覆盖：

```python
# app/deps.py
from app.llm import LLMClient

def get_llm() -> LLMClient:
    """FastAPI 依赖工厂。测试里用 dependency_overrides 把它换成返回 FakeLLM 的函数。"""
    return LLMClient()
```

```python
# app/main.py
from fastapi import Depends, FastAPI
from pydantic import BaseModel
from app.deps import get_llm
from app.service import SummaryService

app = FastAPI()

class SummarizeIn(BaseModel):
    text: str

@app.post("/summarize")
async def summarize(body: SummarizeIn, llm=Depends(get_llm)):   # Depends 声明依赖
    service = SummaryService(llm=llm)                           # 用注入进来的 llm
    return await service.summarize(body.text)
```

```python
# tests/test_api.py
import pytest
from fastapi.testclient import TestClient
from app.deps import get_llm
from app.main import app
from tests.conftest import FakeLLM

@pytest.fixture
def client(fake_llm):
    # 把 get_llm 这个依赖换掉：FastAPI 解依赖时不再调真实的，改调这个 lambda
    app.dependency_overrides[get_llm] = lambda: fake_llm
    yield TestClient(app)                  # yield 之前是准备，之后是清理
    app.dependency_overrides.clear()       # 必须清！app 是模块级对象，不清会漏给下一个测试

def test_summarize_endpoint_returns_summary(client):
    response = client.post("/summarize", json={"text": "讲讲 pytest"})

    assert response.status_code == 200                       # 业务结果：状态码
    assert response.json()["summary"] == "这是摘要"           # 业务结果：响应体字段

def test_summarize_endpoint_rejects_missing_field(client):
    response = client.post("/summarize", json={})            # 缺 text 字段

    assert response.status_code == 422       # FastAPI 校验失败的标准码，不是 400
```

**逐段解读：**

- `conftest.py` 里三个 fixture 形成链条：`fake_llm` → `service`（用它构造真业务对象）→ 测试。**被测对象 `SummaryService` 始终是真的**，只有它的依赖是假的。这就是 3.8 那条边界的落地形态。
- `fake_env` 用了 `autouse=True`，所以每个测试自动生效，不需要在参数里写它。这是「安全网」类 fixture 的典型用法。
- `test_summarize_truncates_long_text` 三个断言分别咬住：返回的标记、调用次数、发出的内容。注意 `fake_llm.calls` 是**你的代码产生的行为记录**，断言它是合法的；但断言 `fake_llm.reply == "  这是摘要  "` 就非法了——那是你自己设的值（第 6 节坑 5）。
- `client` fixture 用 `yield` 分隔准备和清理。这是 fixture 做收尾工作的标准写法，比 `return` 多了「测完还要做点事」的能力。
- TestClient 是同步的（内部自己跑事件循环），所以 `test_api.py` 里的测试函数写普通 `def` 就行，不用 `async def`。

#### 单独展开 3：`asyncio_mode = "auto"` 到底做了什么

pytest 本身**不会**跑 `async def` 测试函数。它调用测试函数，拿到一个协程对象，然后……就没有然后了（这正是坑 1 的成因）。pytest-asyncio 的作用是补上「拿到协程后创建事件循环、把它跑完」这一步。

它有两种模式：

```python
# strict 模式（pytest-asyncio 的默认值）：每个异步测试都要显式打标记
import pytest

@pytest.mark.asyncio            # 少这一行，这个测试就不会被真正执行
async def test_something():
    result = await fetch()
    assert result == 1
```

```python
# auto 模式（工作区要求）：pytest-asyncio 自动接管所有 async def test_，无需标记
async def test_something():     # 干净，且不可能忘记打标记
    result = await fetch()
    assert result == 1
```

`auto` 模式下 pytest-asyncio 会在收集阶段把每个 `async def` 测试函数自动加上等价于 `@pytest.mark.asyncio` 的处理，同时也自动处理 `async def` 的 fixture。

**为什么工作区强制 `auto`**：strict 模式下漏写 `@pytest.mark.asyncio` 的后果不是报错，而是**测试显示通过但一行都没执行**。这是一种沉默的失败，靠人自觉不可靠。`auto` 把这个陷阱从根上拆掉。

配置只有一行，放 `pyproject.toml`：

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

验证它真的生效：跑 `uv run pytest -q`，输出头部的 `plugins` 行必须包含 `asyncio-`。这是工作区验收清单第 1 步。

**这一步你得到了什么**：一套能测异步业务逻辑和 FastAPI 端点的完整结构，并且理解了 mock 的边界在哪（换依赖不换被测对象）、`asyncio_mode = "auto"` 挡掉了什么坑。

---

## <mark style="background: #BBFABBA6;">第 5 节：最小可用的实际用法（生产场景模板）</mark>

<mark style="background: #BBFABBA6;">下面这套可以直接复制进新项目。注释标了「必须」和「可选」。</mark>

**1. `pyproject.toml`**

```toml
[dependency-groups]
dev = [
    "pytest>=9.1.1",           # 必须
    "pytest-asyncio>=1.4.0",   # 必须（只要有 async 代码）
    "respx>=0.23.1",           # 可选：在 HTTP 层拦截 httpx，见本节末
]

[tool.pytest.ini_options]
asyncio_mode = "auto"          # 必须：不写就是 strict，漏标记会假通过
pythonpath = ["."]             # 必须（除非项目已装成可导入包）
testpaths = ["tests"]          # 可选：限定收集范围，跑得快一点
addopts = "-q"                 # 可选可删：默认安静输出，省得每次敲 -q
```

**2. `tests/conftest.py`**

```python
import pytest

@pytest.fixture(autouse=True)
def fake_env(monkeypatch):
    """必须：兜住所有环境变量，防止测试读到真实密钥或真实端点。

    monkeypatch 会在每个测试结束后自动还原，禁止改用 os.environ（见第 6 节坑 4）。
    """
    monkeypatch.setenv("LLM_API_KEY", "sk-test-fake")
    monkeypatch.setenv("LLM_BASE_URL", "https://api.invalid")   # 故意指向不存在的域名：
                                                                # 万一漏了拦截，会立刻失败而不是偷偷联网

class FakeLLM:
    """必须（或换成你项目的等价 fake）：被测对象的依赖的替身。"""
    def __init__(self, reply: str = "摘要内容"):
        self.reply = reply
        self.calls: list[str] = []

    async def chat(self, prompt: str) -> str:
        self.calls.append(prompt)
        return self.reply

@pytest.fixture
def fake_llm():
    return FakeLLM()          # 不写 scope = function scope，每个测试独立（见 3.5）
```

**3. `tests/test_service.py` —— 业务逻辑测试**

```python
from app.service import SummaryService

async def test_summarize_marks_truncation(fake_llm):
    service = SummaryService(llm=fake_llm)      # 真业务对象 + 假依赖

    result = await service.summarize("字" * 300)   # await 必须有

    assert result["truncated"] is True           # 断言业务结果
    assert len(fake_llm.calls) == 1              # 断言产生的行为
```

**4. `tests/test_api.py` —— FastAPI 端点测试**

```python
import pytest
from fastapi.testclient import TestClient
from app.deps import get_llm
from app.main import app

@pytest.fixture
def client(fake_llm):
    app.dependency_overrides[get_llm] = lambda: fake_llm   # 必须：注入 fake
    yield TestClient(app)
    app.dependency_overrides.clear()                       # 必须：不清会污染后续测试

def test_summarize_ok(client):
    response = client.post("/summarize", json={"text": "hello"})

    assert response.status_code == 200
    assert response.json()["summary"] == "摘要内容"

def test_summarize_validation_error(client):
    assert client.post("/summarize", json={}).status_code == 422
```

跑起来：

```bash
uv run pytest -q
```

### ⚠️ 交付前必做：改坏一行，看它变红

写完测试**先别提交**。全绿不代表测试有效——一套什么都不验证的测试也是全绿的。按下面四步自证：

**第 1 步**，确认基线全绿，并确认 pytest-asyncio 真的装上了：

```bash
uv run pytest -q
```

输出头部的 `plugins` 行必须包含 `asyncio-`（例：`plugins: asyncio-1.4.0, respx-0.23.1`）。同时**输出里不能有** `RuntimeWarning: coroutine ... was never awaited`。

**第 2 步**，故意改坏一行业务逻辑。挑被测函数里一个有实际作用的表达式，做最小改动：

```python
# app/service.py，把阈值改小，让截断判断出错
truncated = len(text) > 10000      # 原本是 > 100
```

**第 3 步**，再跑一次：

```bash
uv run pytest -q
```

**必须有测试变红**。看到 `FAILED tests/test_service.py::test_summarize_marks_truncation` 才算合格——这证明你的测试真的在盯着这行逻辑。

如果**仍然全绿**，说明这套测试是装饰品，没有覆盖到这行逻辑。不要放过它：回去补断言（或者你会发现断言咬的是自己 set 的 mock 值，见第 6 节坑 5），重写。

**第 4 步**，把那行改回来，重跑确认恢复全绿。

对每个关键业务分支重复一遍（截断、空输入、错误处理各改坏一次）。这套流程就是工作区验收清单的第 1、2 步，机器可判，没有讨论空间。

另外两条自查，建议顺手做：

- **断网再跑一次**（或关掉代理），必须仍然全绿。报连接错误 = 有测试在打真实 API。
- **随机挑一个测试，只读它的 assert 行**，问自己：它验证的是业务结果，还是我刚造的输入/mock 值？后者要重写。

### 实际项目通常还会配（属进阶，先知道有这回事）

- **覆盖率**：`pytest-cov`，`uv run pytest --cov=app --cov-report=term-missing` 看哪些行没被测到。注意覆盖率高不等于测试有效——装饰性测试也能刷出高覆盖率，「改坏一行」才是真的检验。
- **CI**：GitHub Actions 里跑 `uv run pytest`，PR 挂了就不许合。
- **测试标记**：`@pytest.mark.slow` 加上 `-m "not slow"` 分快慢两档。

### HTTP 层拦截（respx）

上面用的是「注入 fake 依赖」，适合测你自己的业务逻辑。还有一类需求是**验证你的代码有没有正确地发出 HTTP 请求**（URL、请求头、重试、超时处理）——这时候不换对象，而是用 **respx** 在 httpx 的传输层拦截，代码零改动，请求一个字节都不出网。工作区要求所有 httpx 请求都被 `respx_mock` 覆盖，并保持 `assert_all_mocked` 开启（未覆盖的请求直接报错，这是防止偷偷联网的保险，不要关）。详见 `respx-http-mock.md` 专篇。

---

## 第 6 节：常见坑与易错点

### <mark style="background: #BBFABBA6;">坑 1（最重要）：只调协程不 await —— 测试显示 pass，实际什么都没跑</mark>

**报错信息原文**（注意它是 Warning，不是 Error）：

```
=============================== warnings summary ===============================
tests/test_service.py::test_summarize
  /path/to/test_service.py:5: RuntimeWarning: coroutine 'SummaryService.summarize' was never awaited
    service.summarize("hello")
1 passed, 1 warning in 0.01s
```

**现象**：`1 passed`。绿的。但你的业务代码一行都没执行过，断言也从未真正验证任何东西。这是**假通过**——测试套件里最危险的东西，因为它给你虚假的安全感。

**原因**：`async def` 函数调用后只返回协程对象，不执行函数体（见 3.10）。两种典型写法都会中招：

```python
# 错法 A：忘了 await
async def test_summarize(service):
    result = service.summarize("hello")     # 只拿到协程对象，函数体没跑
    assert result                           # 协程对象是真值，断言恒成立！绿了

# 错法 B：strict 模式下漏了标记（整个测试函数本身都没被执行）
async def test_summarize(service):          # 没有 @pytest.mark.asyncio，且 mode 不是 auto
    assert await service.summarize("") == {"summary": "", "truncated": False}
    # pytest 拿到协程就丢掉了，测试标记为 passed
```

**解决**（三层防护，都要有）：

1. <mark style="background: #BBFABBA6;">`pyproject.toml` 里配 `asyncio_mode = "auto"`，从根上消灭错法 B。</mark>
2. <mark style="background: #BBFABBA6;">调异步函数一律带 `await`。断言写成 `assert await f() == 具体值`，不要写 `assert f()`——断言一个具体值，协程对象不可能等于它，漏 await 会立刻失败。</mark>
3. <mark style="background: #BBFABBA6;">跑测试时扫一眼有没有 `RuntimeWarning: coroutine ... was never awaited`。</mark><mark style="background: #BBFABBA6;">想更狠一点，把它升级成错误：</mark>

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
filterwarnings = ["error::RuntimeWarning"]   # 让这类警告直接让测试失败，不再是绿的
```

### 坑 2：`fixture 'xxx' not found`

**报错信息原文**：

```
E       fixture 'fake_llm' not found
>       available fixtures: cache, capfd, capsys, monkeypatch, pytestconfig, record_property, recwarn, tmp_path, tmpdir, ...
>       use 'pytest --fixtures [testpath]' for help on them.
```

**现象**：测试直接 error（不是 fail），一行业务代码没跑。

**原因**（按出现频率）：

1. 参数名和 fixture 名拼错了（`fake_llm` 写成 `fake_lmm`）。fixture 靠名字连接，名字就是接口（见示例 2 的展开 1）。
2. fixture 定义在**另一个测试文件**里。测试文件之间不共享 fixture——共享要放 `conftest.py`。
3. `conftest.py` 拼错了（`conftests.py`、`confest.py`）或放错了目录。它必须在测试文件的同级或上级目录。
4. 你以为某个是内置 fixture，其实不是（内置的常用就 `monkeypatch`、`tmp_path`、`capsys`）。

**解决**：先跑 `uv run pytest --fixtures` 列出当前所有可用 fixture 和它们来自哪个文件，对着名字核对。

### 坑 3：session scope fixture 导致测试间脏数据

**现象**：单独跑 `uv run pytest tests/test_b.py` 是绿的，一起跑 `uv run pytest` 就红；或者换个执行顺序结果就变。

**原因**：放大 scope 的 fixture 返回的是**同一个可变对象**，前面的测试改了它，后面的测试读到脏数据。

```python
@pytest.fixture(scope="session")     # 整个会话只建一次
def history():
    return []                        # 所有测试共用这一个列表

async def test_a(history):
    history.append("a")              # 塞了一条
    assert len(history) == 1         # 绿

async def test_b(history):
    assert len(history) == 0         # 红！里面已经有 test_a 塞的 "a" 了
```

**解决**：默认就用 `function` scope（不写 scope 即是）。确实需要放大 scope 时（启容器、加载大模型），要么保证对象只读，要么再加一个 `function` scope 的 fixture 负责每次清理：

```python
@pytest.fixture(scope="session")
def db_engine():                     # 昂贵、只建一次：合理
    engine = create_engine(...)
    yield engine
    engine.dispose()

@pytest.fixture                      # function scope：每个测试独立事务并回滚
def db_session(db_engine):
    conn = db_engine.connect()
    tx = conn.begin()
    yield conn
    tx.rollback()                    # 每个测试的写入都撤销，隔离恢复
    conn.close()
```

### 坑 4：直接改 `os.environ` 污染后续测试

**现象**：没有报错，但某个测试单独跑是绿的、和别人一起跑就红；或者跑完测试后你的 shell 里环境变量莫名变了。

**原因**：`os.environ` 是进程级全局状态，你改了不会自动恢复，之后所有测试都活在被你改过的环境里。

```python
# 错：改了不还原，后面所有测试都看到这个假 key
def test_uses_fake_key():
    os.environ["LLM_API_KEY"] = "sk-fake"     # 泄漏到整个测试进程
    assert LLMClient().api_key == "sk-fake"

# 对：monkeypatch 在测试结束时自动还原
def test_uses_fake_key(monkeypatch):
    monkeypatch.setenv("LLM_API_KEY", "sk-fake")
    assert LLMClient().api_key == "sk-fake"
```

**解决**：环境变量一律 `monkeypatch.setenv` / `monkeypatch.delenv`。同理，改属性用 `monkeypatch.setattr`，改字典项用 `monkeypatch.setitem`。**这是工作区硬规定，没有例外。**

### 坑 5（和直觉相反）：断言自己刚 set 的 mock 值 —— 装饰性测试

**现象**：测试全绿，覆盖率还挺高，但你把业务逻辑改坏它依然全绿。它什么都没验证。

**原因**：断言的对象是你自己设进去的值，等于「我设 x=5，然后断言 x==5」。这条链路完全绕过了被测代码。

**反例（错，绝对不要写）**：

```python
from unittest.mock import AsyncMock, patch

async def test_summarize_bad():
    # 错误 1：patch 掉了被测对象 SummaryService.summarize 本身
    with patch.object(SummaryService, "summarize", new=AsyncMock()) as mock_sum:
        mock_sum.return_value = {"summary": "摘要", "truncated": True}   # 自己设的值

        service = SummaryService(llm=None)
        result = await service.summarize("字" * 300)

        # 错误 2：断言的正是自己刚设的返回值。真实的 summarize 一行都没跑
        assert result == {"summary": "摘要", "truncated": True}
        # 你把 service.py 里的截断阈值改成任何数字，这个测试都是绿的
```

**正例（对）**：

```python
async def test_summarize_good(fake_llm):
    # 被测对象是真的 SummaryService，只有它的依赖 llm 是 fake
    service = SummaryService(llm=fake_llm)

    result = await service.summarize("字" * 300)

    # 断言的都是「真实业务逻辑算出来的东西」，不是自己设的值
    assert result["truncated"] is True          # 截断判断由 service.py 算出
    assert len(fake_llm.calls) == 1             # 调用次数由 service.py 的控制流决定
    assert len(fake_llm.calls[0]) < 150         # prompt 长度由 service.py 的切片决定
```

对照着看差别：

| | 反例 | 正例 |
|---|---|---|
| 被替换的是 | 被测对象自己（`SummaryService.summarize`） | 被测对象的依赖（`llm`） |
| 断言的是 | 自己 set 的 `return_value` | 真实代码算出的结果和行为 |
| 改坏业务逻辑 | 依然全绿 | 立刻变红 |

**解决**：两条判据，每写完一个测试自问一遍。**第一，被替换的东西必须是被测对象的依赖，不能是被测对象本身。第二，每个断言里的值必须是被测代码算出来的，不能是我在这个测试里写过的字面量返回值。** 拿不准就跑第 5 节的「改坏一行」——那是这条坑的终极检测器。

---

## 第 7 节：验证你学会了（自测题）

合上文档做，不许翻。每题标了答案在哪节，卡住了回去看。

### 第 1 题：复述（检验概念）

不看文档，用自己的话回答：

1. pytest 是什么？用一句话说清它和「手动跑脚本看 print」的本质区别。
2. 手动验证有哪四个痛点？根源那一句话是什么？
3. 解释这三个词的区别：mock 被测对象本身 / 注入 fake 依赖 / 在 HTTP 层拦截。为什么第一个是禁止的？

> 答案在第 1 节、第 2 节、3.8 节

### 第 2 题：默写（检验基本功）

不看文档，从零写出一对文件，要求：

- 被测函数 `split_batches(items, size)`：把列表按 size 切成若干批，`size <= 0` 时抛 `ValueError`
- 一个 `@pytest.fixture` 提供输入数据
- 一个 `@pytest.mark.parametrize` 覆盖至少 4 组边界：能整除、不能整除、size 大于列表长度、空列表
- 一个 `pytest.raises` 验证 `size=0` 抛异常
- 写出你会敲的运行命令，以及全绿时的输出大概长什么样

写完做一件事：**把 `split_batches` 里的切片逻辑改坏一行，跑一遍确认变红，再改回来。** 如果没变红，你的参数化数据没覆盖到那行，回去补。

> 答案在第 4 节示例 1、示例 2，自证步骤在第 5 节

### 第 3 题：组合应用（检验能不能上手项目）

给下面这个函数写一套测试。它调外部 API，**测试不许打真实网络**：

```python
# app/translate.py
class TranslateService:
    def __init__(self, llm):
        self.llm = llm

    async def translate(self, text: str, target: str = "en") -> dict:
        if not text.strip():
            return {"text": "", "cached": False}
        if len(text) > 500:
            raise ValueError("文本超长")
        reply = await self.llm.chat(f"把下面内容翻译成{target}：{text}")
        return {"text": reply.strip(), "cached": False}
```

要求：

1. 在 `conftest.py` 里写一个 fake，说明为什么它替换的是 `llm` 而不是 `TranslateService`
2. `pyproject.toml` 要加哪一行才能直接写 `async def test_...`？不加会发生什么（说出具体的警告文字和「测试显示什么结果」）
3. 至少四个测试：正常翻译、空输入不调 LLM、超长抛 `ValueError`、`target` 参数确实进了发给 LLM 的 prompt
4. 假设这个服务还要读 `TRANSLATE_API_KEY` 环境变量，测试里怎么设？为什么不能用 `os.environ`
5. 检查你的每一条断言：有没有哪一条在断言你自己 set 的 fake 返回值？如果有，改成断言业务结果
6. 跑完「改坏一行 → 变红 → 改回来」，写下你改坏了哪一行、哪个测试红了

> 答案在第 4 节示例 3、第 5 节、3.9 节、第 6 节坑 1 和坑 5

---

## 下一步

- 异步本身不熟 → `asyncio-入门.md`
- 要验证「发出去的 HTTP 请求对不对」、测重试和超时 → `respx-http-mock.md`
- FastAPI 依赖注入想搞明白 → `fastapi-入门.md`

最后再强调一遍那条判据，它比本文任何 API 都重要：**一套测试的价值，等于你改坏业务逻辑时它变红的能力。** 全绿不是目标，能红才是。

