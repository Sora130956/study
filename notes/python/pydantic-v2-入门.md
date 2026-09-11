# pydantic v2 核心入门（BaseModel、Field 校验、嵌套模型、model_validate）

- 目标读者：会写 Python 函数和类，但没系统用过类型注解，也没用过任何数据校验库
- 前置知识：见第 3 节的六个概念（类型注解 / 类属性与实例属性 / 校验 / 序列化与反序列化 / schema / 可选字段与默认值），另加一节 v1 与 v2 的改名对照
- 学习时长：约 60 分钟（动手敲一遍约 90 分钟） / 可跳过：第 2 节是纯背景推演，急着写代码可以先跳到第 4 节，卡住了再回来看

环境：Python 3.12+，pydantic 2.13.5（本文所有代码在这个版本上实际跑过）。

---

## 1. 它是什么

pydantic 是一个「按你写好的字段表，检查外面送进来的数据对不对，并把它变成一个能点出来用的 Python 对象」的库。

类比：**它像机场海关**。外面的人（数据）要进国门（你的程序），先在关口出示证件：名字必须是文字、年龄必须是数字、随行行李清单里每一件也得逐件查。证件不全或者不对，当场挡回去，并给你一张写清楚「第几个人、哪一项、错在哪」的单子；查过的人进来以后，你就可以放心用，不用在国内每个路口再查一遍。

这个类比后面会一一对上：

| 海关 | pydantic |
| --- | --- |
| 关口的检查表（谁该有什么证件） | 你定义的 `BaseModel` 类 |
| 逐个人过关 | `Model.model_validate(数据)` |
| 挡回去 + 那张问题清单 | 抛出 `ValidationError`，`.errors()` 就是清单 |
| 行李逐件查 | 嵌套模型 `list[LineItem]` |
| 额外的特殊规定（比如必须年满 18） | `Field(gt=18)` 和 `field_validator` |

---

## 2. 它解决什么问题

### 不用它的时候，代码长什么样

假设你让大模型（LLM）读一段发票文字，返回 JSON，你要拿到发票号、买方、金额。手写就是这样：

```python
import json

def parse_invoice(raw_text: str) -> dict:
    data = json.loads(raw_text)              # 第一关：JSON 本身可能就不合法

    # 检查发票号
    if "invoice_no" not in data:             # 缺字段
        raise ValueError("缺少 invoice_no")
    if not isinstance(data["invoice_no"], str):   # 类型不对
        raise ValueError("invoice_no 必须是字符串")
    if not data["invoice_no"].startswith("INV"):  # 业务规则
        raise ValueError("invoice_no 必须以 INV 开头")

    # 检查金额
    if "amount" not in data:
        raise ValueError("缺少 amount")
    try:
        amount = float(data["amount"])       # LLM 常返回 "12.50" 这种字符串
    except (TypeError, ValueError):
        raise ValueError("amount 不是数字")
    if amount <= 0:
        raise ValueError("amount 必须大于 0")

    # 买方可以没有，给个默认值
    buyer = data.get("buyer") or "未知"

    return {"invoice_no": data["invoice_no"], "amount": amount, "buyer": buyer}
```

三个字段，二十多行。痛点是能直接感受到的：

- **每个字段 3 到 5 行校验**：字段表一到 10 个，校验代码就 30 到 50 行，真正的业务逻辑被埋在里面。
- **校验散落各处**：另一个函数也收发票数据，这套 `if` 要么复制一遍，要么忘了写。规则一改（发票号前缀换了），得挨个文件去改。
- **只报第一个错**：`raise` 在第一个问题上就中断了。你想知道「这条数据一共错了几处」做不到，调 LLM 调一次几分钱，你却只能一次修一个错。
- **LLM 少给一个字段就当场崩**：`data["amount"]` 直接 `KeyError`，而且崩在生产环境里，日志只有一行 `KeyError: 'amount'`，看不出是哪条数据、原始文本长什么样。
- **拿到的还是 dict**：后面全程 `data["invoice_no"]`，字段名打错了 IDE 不提示，运行时才炸。

### 根源一句话

**「数据长什么样」这件事只写在你的脑子里和一堆 `if` 里，没有一个地方把它声明出来**，所以没法复用、没法自动检查、也没法自动告诉你哪里不对。

pydantic 的做法：把「数据长什么样」写成一个类，检查、报错、转换全部由这份声明自动生成。

### 经典应用场景

1. **FastAPI 的请求 / 响应模型**：FastAPI 内部就用 pydantic。你写好模型，客户端传错数据由框架自动挡掉并返回 422 错误，你的函数里拿到的一定是干净数据。
2. **LLM 结构化输出**（本文重点）：让模型返回 JSON → pydantic 校验 → 保证字段齐全类型正确 → 转 dict 喂给 pandas → 导出 Excel。非结构化文本抽取成表格的主力链路。
3. **配置读取**：读环境变量、YAML，启动时就校验，而不是跑到半夜才发现某个配置项拼错了。
4. **各种外部数据入口**：第三方 API 返回、上传的 CSV、消息队列的消息，凡是「数据不是你自己生成的」地方都适用。

---

## 3. 读下去之前，先搞懂这些概念

### 3.1 类型注解（type hints）

**定义**：在变量或函数参数后面用冒号写上它「应该是什么类型」的语法。

**大白话**：给变量贴一张标签纸，写着「这里应该放一个整数」。**Python 本身不检查这张标签**，它只是写给人和工具看的备注。

```python
name: str = "小明"          # 标签说是文字
age: int = "其实我是文字"    # Python 照样运行，不报错
```

后面你会看到 pydantic 的核心魔法就是：**把这张原本没人管的标签真正执行起来**。

常用写法，认识就够：

```python
list[str]        # 一个列表，里面每一项都是文字
dict[str, int]   # 一个字典，键是文字，值是整数
str | None       # 要么是文字，要么是 None（Python 3.10+ 写法，等价于 Optional[str]）
```

### 3.2 类属性与实例属性

**定义**：写在 `class` 缩进里、不在方法里的变量是类属性（属于这个类本身）；在方法里用 `self.x = ...` 赋值的是实例属性（属于某一个具体对象）。

**大白话**：类属性是「表格模板上印好的栏目名」，实例属性是「某一份填好的表格里的内容」。

```python
class User:
    kind = "人类"          # 类属性：所有 User 共用一份
    def __init__(self, name):
        self.name = name   # 实例属性：每个 user 各有一份
```

为什么要提这个：pydantic 模型里你写的 `name: str` 长得像类属性，**但它其实是字段声明**，实例化后 `user.name` 拿到的是这个对象自己的值。pydantic 在背后把声明改写成了实例属性。

### 3.3 校验（validation）

**定义**：检查一份数据是否满足预先规定的条件的过程。

**大白话**：进门前先查一遍，不合格就别让它进来。

```python
if not isinstance(age, int):     # 这就是最原始的校验
    raise ValueError("age 必须是整数")
```

### 3.4 序列化 / 反序列化（serialization / deserialization）

**定义**：把内存里的 Python 对象转成可以传输或保存的通用格式（dict、JSON 文本）叫序列化；反过来叫反序列化。

**大白话**：**序列化是把家具拆成一堆板子装箱寄出去，反序列化是收到箱子照说明书装回家具**。程序之间只能传板子（文本），不能直接传家具（对象）。

```python
import json
json.dumps({"name": "小明"})     # 序列化：对象 → '{"name": "小明"}' 文本
json.loads('{"name": "小明"}')   # 反序列化：文本 → 对象
```

在 pydantic 里对应：`model_dump()` 是序列化（模型 → dict），`model_validate()` 是反序列化 + 校验（dict → 模型）。

### 3.5 schema（数据结构说明书）

**定义**：一份描述「数据应该有哪些字段、每个字段是什么类型、哪些必填」的说明。

**大白话**：**就是一张空白表格的栏目定义**——有哪几栏、每栏填数字还是文字、哪几栏不能空着。

你写的 `BaseModel` 类就是 schema。pydantic 还能把它自动导出成 JSON Schema（一种通用格式的说明书），这份说明书可以直接交给 LLM，告诉它「按这个格式返回」。

### 3.6 可选字段与默认值

**定义**：必填字段是不给就报错的字段；可选字段是不给就用默认值的字段。

**大白话**：表格里带星号的栏目必须填，不带星号的空着也行，空着按预设值算。

```python
class User(BaseModel):
    name: str                    # 必填：不给就 ValidationError
    nickname: str = "无"          # 可选：不给就是 "无"
    email: str | None = None     # 可选，且允许显式传 None
```

<mark style="background: #BBFABBA6;">注意区分两件事：`str | None` 说的是**值可以是 None**；`= None` 说的是**不传时用什么**。要成为可选字段，关键是有等号右边的默认值。</mark>

### 3.7 v1 和 v2 的 API 改名（网上老教程的坑）

pydantic 2.0 把一批方法改了名，旧名字仍然能用但会警告，3.0 会彻底删除。网上大量教程还是 v1 写法，你一定会撞上。对照表：

| v1 老写法（别用）           | v2 正确写法                          | 干什么            |
| -------------------- | -------------------------------- | -------------- |
| `m.dict()`           | `m.model_dump()`                 | 模型 → dict      |
| `m.json()`           | `m.model_dump_json()`            | 模型 → JSON 文本   |
| `Model.parse_obj(d)` | `Model.model_validate(d)`        | dict → 模型（带校验） |
| `Model.parse_raw(s)` | `Model.model_validate_json(s)`   | JSON 文本 → 模型   |
| `Model.schema()`     | `Model.model_json_schema()`      | 导出 JSON Schema |
| `@validator("x")`    | `@field_validator("x")`          | 单字段自定义校验       |
| `@root_validator`    | `@model_validator`               | 跨字段校验          |
| `class Config:`      | `model_config = ConfigDict(...)` | 模型配置           |

记忆规律：**v2 把模型自带的方法统一加了 `model_` 前缀**，这样字段名就不会和方法名撞车（你完全可以有一个叫 `json` 的字段）。

### 术语速查表

| 术语 | 大白话 | 正文位置 |
| --- | --- | --- |
| 类型注解 type hints | 给变量贴的类型标签，Python 自己不检查 | 3.1 / 4.1 原理 |
| 字段 field | 模型里的一个栏目 | 4.1 |
| 校验 validation | 进门前查一遍合不合格 | 3.3 / 4.1 |
| ValidationError | 校验失败时抛的异常，附带问题清单 | 4.1 / 6.1 |
| 序列化 | 对象拆成 dict / 文本寄出去 | 3.4 / 4.3 |
| 反序列化 | dict / 文本装回对象 | 3.4 / 4.2 |
| schema | 空白表格的栏目定义 | 3.5 / 5 |
| 嵌套模型 | 表格里的某一栏本身是另一张表 | 4.3 |
| 可选字段 | 不填就用默认值的栏目 | 3.6 / 4.2 |
| lax / strict 模式 | 宽松模式会顺手转类型，严格模式不转 | 6.4 |

---

## 4. 从最简单的代码示例开始

### 4.1 示例一：最小可运行版本

```python
from pydantic import BaseModel, ValidationError   # BaseModel 是所有模型的父类；ValidationError 是校验失败时抛的异常

class User(BaseModel):        # 继承 BaseModel，这一步让这个类获得校验能力
    name: str                 # 声明字段 name，类型是文字，没有默认值 → 必填
    age: int                  # 声明字段 age，类型是整数 → 必填
    city: str = "未知"         # 有默认值 → 可选，不传就是 "未知"

u = User(name="小明", age=18)  # 关键字参数实例化；这一行的同时就在做校验
print(u)                       # 打印模型，pydantic 已经生成好了可读的输出
print(u.name, u.age, u.city)   # 用点号取值，不是 u["name"]；IDE 能补全、拼错会提示

try:
    User(name="小红", age="十八")   # 故意传错：中文"十八"没法变成整数
except ValidationError as e:        # 校验失败不会静默通过，而是抛异常
    print(e)                        # 打印出人类可读的错误说明
```

运行结果：

```
name='小明' age=18 city='未知'
小明 18 未知
1 validation error for User
age
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='十八', input_type=str]
```

**逐段解读**

- 第 1 行导入：`BaseModel` 是你所有模型的起点，`ValidationError` 是唯一你需要捕获的异常类型。
- `class User(BaseModel)` 这段负责**定义 schema**。三行字段声明就是全部的检查表：两个必填、一个可选。没有 `__init__`，pydantic 自动帮你生成了。
- `u = User(...)` 这段负责**造对象 + 校验**。关键点：校验发生在实例化的瞬间，一旦这行过了，`u` 里面的数据一定是干净的，后面不需要再检查。
- `try / except` 这段负责**演示失败路径**。注意错误信息有三段有用的东西：出错的字段名（`age`）、错误类型（`int_parsing`）、你实际传进来的值（`'十八'`）。手写 `if` 想输出这么全的信息，得费不少劲。

#### 展开讲：类型注解为什么能驱动校验

第 3.1 节说过，Python 自己不检查类型注解。那 pydantic 是怎么做到的？

Python 会把类里所有的注解收集到一个特殊字典 `__annotations__` 里，你可以自己看：

```python
class User(BaseModel):
    name: str
    age: int

print(User.__annotations__)   # {'name': <class 'str'>, 'age': <class 'int'>}
```

`BaseModel` 在类被定义的时候（不是实例化的时候）读取这个字典，为每个字段生成一段对应的检查代码，然后拼装出 `__init__`。所以：

- **注解不是装饰，它是 pydantic 的输入**。你写 `age: int` 就等于在下单「给我一段把输入变成整数、变不了就报错的检查」。
- 这些检查代码是用 Rust 写的核心（`pydantic-core`）执行的，所以比你手写一堆 `if isinstance` 还快。

**等价的普通写法对照**，前面那个 `User` 大约相当于：

```python
class User:
    def __init__(self, name, age, city="未知"):
        errors = []
        if not isinstance(name, str):                  # 对应 name: str
            errors.append({"loc": ("name",), "msg": "Input should be a valid string"})
        if isinstance(age, bool) or not isinstance(age, (int, str)):
            errors.append({"loc": ("age",), "msg": "Input should be a valid integer"})
        else:
            try:
                age = int(age)                         # 对应 age: int 的宽松转换
            except ValueError:
                errors.append({"loc": ("age",), "msg": "unable to parse string as an integer"})
        if errors:
            raise ValueError(errors)                   # 一次性报出所有错，不是遇到第一个就停
        self.name, self.age, self.city = name, age, city

    def __repr__(self):
        return f"name={self.name!r} age={self.age!r} city={self.city!r}"
```

三行声明 = 上面这二十行。而且字段一多，上面这段会线性膨胀，声明式写法不会。

**这一步你得到了什么**：你能定义一个模型、造出实例、用点号取值，并且知道数据不合格时会抛 `ValidationError` 而不是悄悄放过。

### 4.2 示例二：加约束、可选字段、用 model_validate 吃一个 dict

真实需求：LLM 返回一个 JSON，转成 dict 后交给你。字段要求：商品名不超过 50 字、数量必须大于 0、备注可以没有、而且 <mark style="background: #BBFABBA6;">LLM 那边用的是驼峰命名 `unitPrice`，你的 Python 代码想用 `unit_price`。</mark>

```python
from pydantic import BaseModel, Field, ConfigDict, ValidationError

class LineItem(BaseModel):
    model_config = ConfigDict(populate_by_name=True)   # 允许两种键名都能传进来：unitPrice 和 unit_price

    name: str = Field(max_length=50)                   # 文字，且最长 50 字，超了报错
    qty: int = Field(gt=0)                             # 整数，且必须 > 0（gt = greater than）
    unit_price: float = Field(alias="unitPrice", gt=0)  # 外部叫 unitPrice，内部叫 unit_price，且要 > 0
    note: str | None = None                            # 可选：可以不传，也可以显式传 None

raw = {                          # 假设这是 LLM 返回的 JSON 解析出来的 dict
    "name": "A4 打印纸",
    "qty": "5",                  # 注意是字符串，pydantic 会自动转成整数 5
    "unitPrice": 23.5,           # 用的是驼峰键名
}

item = LineItem.model_validate(raw)   # 从一个 dict 造模型（v1 的 parse_obj，别用）
print(item)
print(type(item.qty), item.qty)       # 确认真的转成了 int
print(item.note)                      # 没传 → None

try:
    LineItem.model_validate({"name": "笔", "qty": 0, "unitPrice": 1})   # qty=0 违反 gt=0
except ValidationError as e:
    print(e.errors())                 # 用 .errors() 拿到结构化的问题清单，而不是一串文字
```

运行结果：

```
name='A4 打印纸' qty=5 unit_price=23.5 note=None
<class 'int'> 5
None
[{'type': 'greater_than', 'loc': ('qty',), 'msg': 'Input should be greater than 0', 'input': 0, 'ctx': {'gt': 0}, 'url': '...'}]
```

**逐段解读**

- `model_config = ConfigDict(populate_by_name=True)` 这段负责**模型级配置**。`Field(alias=...)` 默认只认外部名（`unitPrice`），加了这个配置后字段名（`unit_price`）也能传，你自己写单元测试时会方便很多。
- 四行字段声明这段负责**schema + 约束**。类型注解管「是什么类型」，`Field(...)` 管「还要满足什么额外条件」。两者分工清楚：注解定形状，`Field` 定规则。
- `LineItem.model_validate(raw)` 这段负责**从 dict 反序列化**。和示例一的 `LineItem(name=..., qty=...)` 效果一样，区别只是数据源是现成的字典。你的数据来自 JSON、数据库行、CSV 一行时，用它。
- `e.errors()` 这段负责**结构化取错**。返回的是一个列表，每项是一个 dict，含 `loc`（哪个字段）、`type`（错误类型代号）、`msg`（说明）、`input`（你传进来的原值）。这就是第 6.1 节要细讲的东西。

#### 展开讲：`Field` 到底是什么，以及 `model_config`

`Field(...)` 不是「给字段赋值」，它是**在默认值的位置放一份「这个字段的附加说明」**。常用参数：

```python
Field(default=0)              # 默认值（等价于直接写 = 0）
Field(default_factory=list)   # 默认值需要每次新建时用它（见第 6.2 节，重要）
Field(gt=0, ge=0, lt=100, le=100)      # 数字范围：大于 / 大于等于 / 小于 / 小于等于
Field(min_length=1, max_length=50)     # 文字或列表的长度范围
Field(pattern=r"^INV")                 # 文字必须匹配这个正则
Field(alias="unitPrice")               # 外部键名和内部字段名不一致时用
Field(description="发票号")             # 写给人和 LLM 看的说明，会进 JSON Schema
```

<mark style="background: #BBFABBA6;">`model_config` 是**整个模型的开关面板**，用 `ConfigDict(...)` 赋值。你早期只需要认识三个：</mark>

```python
model_config = ConfigDict(
    populate_by_name=True,   # 有 alias 时，字段名也能用来传值
    extra="forbid",          # 数据里出现模型没声明的多余字段就报错（默认是忽略）
    str_strip_whitespace=True,  # 所有字符串自动去掉首尾空格，处理 LLM 输出很实用
)
```

**等价的普通写法对照**：`gt=0` 相当于在 `__init__` 里加一句 `if qty <= 0: raise ...`；`alias="unitPrice"` 相当于 `qty = data.get("unitPrice", data.get("qty"))`。声明式的好处是这些规则**跟字段长在一起**，看模型就知道全部规则，不用去读 `__init__` 的实现。

**这一步你得到了什么**：你能给字段加范围和长度约束、处理外部命名和内部命名不一致、区分「不传」和「传 None」，并且会用 `model_validate` 从 dict 造模型、用 `.errors()` 取结构化错误。

### 4.3 示例三：贴近项目的样子（嵌套模型 + 自定义校验 + 喂给 pandas）

<mark style="background: #BBFABBA6;">真实需求：一张发票有发票号、买方、开票日期，还有**一条或多条明细**，每条明细是「商品名 / 数量 / 单价」。发票号必须以 `INV` 开头。最后要把它变成 DataFrame 导出 Excel。</mark>

```python
from datetime import date
from decimal import Decimal
from pydantic import BaseModel, Field, ConfigDict, field_validator, model_validator

class LineItem(BaseModel):                              # 先定义"小表"：一条明细
    name: str = Field(max_length=50)
    qty: int = Field(gt=0)
    unit_price: Decimal = Field(gt=0)                   # 金额用 Decimal 而不是 float，避免 0.1+0.2 的浮点误差

    @property                                           # property 是普通 Python 特性，不是 pydantic 的
    def amount(self) -> Decimal:                        # 派生值：能算出来的就别当字段存，避免和源数据不一致
        return self.qty * self.unit_price

class Invoice(BaseModel):                               # 再定义"大表"：一张发票
    model_config = ConfigDict(populate_by_name=True, str_strip_whitespace=True)

    invoice_no: str = Field(alias="invoiceNo")          # LLM 常输出驼峰，这里做名字映射
    buyer: str | None = None                            # 可选：抽不到买方也不该让整条数据失败
    issued_at: date | None = Field(default=None, alias="issuedAt")  # "2026-01-05" 会自动变成 date 对象
    items: list[LineItem]                               # 嵌套：这一栏本身是一串 LineItem，逐条校验

    @field_validator("invoice_no")                      # 给单个字段加自定义规则
    @classmethod                                        # v2 要求 field_validator 必须配 classmethod
    def check_invoice_no(cls, v: str) -> str:           # v 是这个字段已经通过类型校验后的值
        if not v.startswith("INV"):
            raise ValueError("发票号必须以 INV 开头")     # 抛 ValueError，pydantic 会包装成 ValidationError
        return v                                        # 必须 return，返回值才是最终存进模型的值

    @model_validator(mode="after")                      # 跨字段校验：所有字段都验完之后再跑
    def check_not_empty(self):                          # mode="after" 时参数是 self，能点出所有字段
        if not self.items:
            raise ValueError("发票至少要有一条明细")
        return self                                     # 同样必须 return self

    @property
    def total(self) -> Decimal:
        return sum(i.amount for i in self.items)

raw = {                                                 # 模拟 LLM 抽出来的结构
    "invoiceNo": "  INV-2026-001  ",                    # 首尾有空格，str_strip_whitespace 会清掉
    "buyer": "某某科技有限公司",
    "issuedAt": "2026-01-05",                           # 字符串日期
    "items": [
        {"name": "A4 打印纸", "qty": "5", "unit_price": "23.50"},   # 数量和单价都是字符串
        {"name": "签字笔", "qty": 20, "unit_price": "3.00"},
    ],
}

inv = Invoice.model_validate(raw)                        # 一行完成：整张发票 + 每条明细全部校验
print(inv.invoice_no, inv.issued_at, inv.total)          # INV-2026-001 2026-01-05 177.50

rows = [                                                 # 转成 pandas 要的"一行一条明细"结构
    {"发票号": inv.invoice_no, "买方": inv.buyer, **it.model_dump()}   # model_dump() 把模型变成 dict
    for it in inv.items
]
print(rows)
```

运行结果（末行整理后）：

```
INV-2026-001 2026-01-05 177.50
[{'发票号': 'INV-2026-001', '买方': '某某科技有限公司', 'name': 'A4 打印纸', 'qty': 5, 'unit_price': Decimal('23.50')},
 {'发票号': 'INV-2026-001', '买方': '某某科技有限公司', 'name': '签字笔',   'qty': 20, 'unit_price': Decimal('3.00')}]
```

**逐段解读**

- `LineItem` 这段负责**最小数据单元**。它是一个完整的独立模型，可以单独使用，也可以被别的模型当类型用。`amount` 用 `@property` 而不是字段，因为它能从 `qty * unit_price` 算出来——存下来反而可能和源数据打架。
- `items: list[LineItem]` 这一行是**嵌套的全部秘密**。类型注解里放另一个模型，pydantic 就会对列表里每一项递归执行 `LineItem` 的校验。数据里第 2 条明细的 `qty` 是 0，报错的 `loc` 会精确到 `('items', 1, 'qty')`——第 1 条（下标 1）明细的 qty，定位非常准。
- `check_invoice_no` 这段负责**单字段业务规则**。类型校验（是不是 str）已经由注解完成，它只管业务规则（前缀）。关键三点：抛 `ValueError`、必须 `return v`、必须配 `@classmethod`。
- `check_not_empty` 这段负责**跨字段规则**。单字段校验器看不到别的字段，要「A 和 B 得对得上」这种规则就用 `model_validator(mode="after")`，此时模型已经建好，直接 `self.字段名` 访问。
- 最后的列表推导这段负责**转交 pandas**。`model_dump()` 把模型摊平成 dict，`**it.model_dump()` 把它展开合并进外层 dict，得到「宽表」结构，`pd.DataFrame(rows)` 直接就能用。

#### 展开讲：`field_validator` 的装饰器机制

`@field_validator("invoice_no")` 这行没有魔法，拆开看就是普通 Python：

```python
def check_invoice_no(cls, v): ...                              # 1. 先定义一个普通函数
check_invoice_no = classmethod(check_invoice_no)                # 2. @classmethod 把它变成类方法
check_invoice_no = field_validator("invoice_no")(check_invoice_no)   # 3. field_validator 返回一个装饰器，包一层
```

装饰器就是「拿一个函数，返回一个（通常包装过的）东西」。`field_validator("invoice_no")` 先执行，返回真正的装饰器，这个装饰器给你的函数**打上一个标记：我是 invoice_no 的校验器**。模型类被定义时，`BaseModel` 扫描所有带标记的方法，把它们挂到对应字段的检查流程末尾。

**顺序很重要**：`@field_validator` 必须在 `@classmethod` 上面（外层）。写反了标记会打在错误的对象上。

两种模式：

```python
@field_validator("qty", mode="before")   # 在类型转换之前跑，v 是原始输入，可能是任何类型
@field_validator("qty", mode="after")    # 默认。在类型转换之后跑，v 已经是 int，能放心用
```

处理 LLM 输出时 `mode="before"` 特别有用，比如模型返回 `"1,234.50"` 这种带千分位的金额，可以在转换前先把逗号去掉：

```python
@field_validator("unit_price", mode="before")
@classmethod
def strip_comma(cls, v):
    if isinstance(v, str):          # 只处理字符串，其他类型原样放过
        return v.replace(",", "")
    return v
```

**这一步你得到了什么**：你能定义嵌套模型并让 pydantic 递归校验、能加单字段和跨字段的自定义业务规则、知道校验器的装饰器是怎么生效的，并且能用 `model_dump()` 把结果交给 pandas。

---

## 5. 最小可用的实际用法：LLM 输出 → pydantic 校验 → 落 Excel

下面是一条完整可跑的链路。LLM 调用用一个返回固定 JSON 字符串的假函数代替，真实项目里替换掉它就行。

```python
"""发票文本 → 结构化 → Excel。依赖：pydantic、pandas、openpyxl"""
from decimal import Decimal
from pathlib import Path

import pandas as pd
from pydantic import BaseModel, ConfigDict, Field, ValidationError, field_validator


# ---------- 1. 定义 schema（必须） ----------
class LineItem(BaseModel):
    name: str = Field(max_length=50, description="商品名称")   # description 可选，但建议写：它会进 JSON Schema 给 LLM 看
    qty: int = Field(gt=0, description="数量")
    unit_price: Decimal = Field(gt=0, description="不含税单价")


class Invoice(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)      # 可选：自动去空格，LLM 输出常带空格

    invoice_no: str = Field(description="发票号，以 INV 开头")
    buyer: str | None = Field(default=None, description="买方名称，没有则为 null")
    items: list[LineItem] = Field(description="明细列表")

    @field_validator("invoice_no")                            # 可选：业务规则，你的场景没有就删掉
    @classmethod
    def check_no(cls, v: str) -> str:
        if not v.startswith("INV"):
            raise ValueError("发票号必须以 INV 开头")
        return v


# ---------- 2. 调 LLM（必须，真实项目替换这个函数） ----------
def call_llm(invoice_text: str) -> str:
    """真实项目里换成 OpenAI / Claude 等 SDK 调用，把 prompt 和 schema 传过去，返回模型的 JSON 文本。

    提示词里带上 schema 能显著提高 LLM 的格式正确率：
        schema_text = json.dumps(Invoice.model_json_schema(), ensure_ascii=False)
        prompt = f"把下面发票文本抽成 JSON，严格符合这个 schema：\\n{schema_text}\\n\\n文本：\\n{invoice_text}"
    """
    return """
    {
      "invoice_no": "INV-2026-001",
      "buyer": "某某科技有限公司",
      "items": [
        {"name": "A4 打印纸", "qty": 5,  "unit_price": "23.50"},
        {"name": "签字笔",    "qty": 20, "unit_price": "3.00"}
      ]
    }
    """


# ---------- 3. 校验 + 落表（必须） ----------
def extract_to_excel(invoice_text: str, out_path: Path) -> Path:
    raw_json = call_llm(invoice_text)                       # 必须：拿到 LLM 的原始文本

    invoice = Invoice.model_validate_json(raw_json)         # 必须：JSON 文本 → 模型，一步完成解析 + 校验
                                                            # 不需要先 json.loads，v2 直接吃 JSON 文本更快

    rows = [                                                # 必须：把嵌套模型摊平成"一行一条明细"的宽表
        {
            "发票号": invoice.invoice_no,
            "买方": invoice.buyer or "",                     # 可选：None 换成空字符串，Excel 里更干净
            "商品": item.name,
            "数量": item.qty,
            "单价": float(item.unit_price),                  # 必须：Decimal 转 float，否则 Excel 里会变成文本
            "金额": float(item.qty * item.unit_price),        # 可选：派生列，方便直接看
        }
        for item in invoice.items
    ]

    df = pd.DataFrame(rows)                                 # 必须：dict 列表 → DataFrame
    out_path.parent.mkdir(parents=True, exist_ok=True)      # 可选：目录不存在时自动建
    df.to_excel(out_path, index=False)                      # 必须：index=False 避免多一列没用的行号
    return out_path


if __name__ == "__main__":
    try:
        path = extract_to_excel("（这里放原始发票文本）", Path("out/invoices.xlsx"))
        print(f"已导出：{path}")
    except ValidationError as e:                            # 必须：校验失败要单独处理，不能让它裸奔到用户面前
        print(f"LLM 返回的数据不合格，共 {e.error_count()} 处问题：")
        for err in e.errors():
            loc = " → ".join(str(x) for x in err["loc"])     # loc 是元组，如 ('items', 0, 'qty')
            print(f"  [{loc}] {err['msg']}（收到的值：{err['input']!r}）")
```

### 实际项目里通常还会配什么（进阶，可先跳过）

1. **校验失败重试 + 错误回喂**：这是 LLM 抽取场景最有价值的一招。校验失败时，把 `e.errors()` 的内容拼进提示词再问一次模型——「你上次返回的 `items[0].qty` 是 0，但要求大于 0，请修正后重新输出」。模型看到具体错误后的修正成功率很高。控制重试次数（一般 2 到 3 次）防止无限循环。
2. **`model_json_schema()` 喂给 LLM**：别自己手写「请返回这样的 JSON」，直接把 `Invoice.model_json_schema()` 序列化后放进提示词，字段说明来自 `Field(description=...)`。模型改了，提示词自动跟着改，不会不同步。
3. **日志**：失败时把**原始 LLM 输出**完整记下来（不只是错误信息）。没有原文你没法复现问题。
4. **结构化输出 API**：主流模型厂商都提供了「强制按 schema 返回」的功能（function calling / JSON mode / structured outputs），很多 SDK 可以直接传入 pydantic 模型类。用上以后格式错误会大幅减少，但**校验一步不能省**——模型仍然可能在业务规则上出错（比如日期不合逻辑、金额和明细加不起来）。
5. **批量处理**：多份发票就是 `list[Invoice]`，逐份 `try / except` 并收集失败清单，别让一份坏数据毁掉整批任务。

---

## 6. 常见坑与易错点

### 6.1 ValidationError 怎么读：`.errors()` 的结构

**现象**：`print(e)` 打出来一大段文字，程序里没法用；想自动处理（比如回喂给 LLM）不知道从哪下手。

**原因**：`str(e)` 是给人看的，`e.errors()` 才是给代码看的。

**解决**：用 `e.errors()`，它返回一个列表，每项固定这几个键：

```python
try:
    Invoice.model_validate({"invoice_no": "X", "items": [{"name": "笔", "qty": 0, "unit_price": 1}]})
except ValidationError as e:
    print(e.error_count())        # 2 —— 一次报出所有问题，不是只报第一个
    for err in e.errors():
        print(err["loc"], "|", err["type"], "|", err["msg"], "|", err["input"])
```

实测输出：

```
2
('invoice_no',) | value_error | Value error, 发票号必须以 INV 开头 | X
('items', 0, 'qty') | greater_than | Input should be greater than 0 | 0
```

每个键的含义：

| 键 | 含义 | 用途 |
| --- | --- | --- |
| `loc` | 出错位置的**元组**路径，嵌套时含列表下标 | 定位到具体哪条明细的哪个字段 |
| `type` | 错误类型代号，如 `int_parsing` / `missing` / `greater_than` | 程序分支判断用（比如只对 `missing` 重试） |
| `msg` | 人类可读说明 | 回喂给 LLM、展示给用户 |
| `input` | 你实际传进来的原始值 | 排查「模型到底返回了什么」 |

两个细节：`loc` 是元组不是字符串，拼接展示要 `" → ".join(str(x) for x in err["loc"])`；**字段有 alias 时 `loc` 用的是 alias 名**（外部名），因为报错是给数据提供方看的。

### 6.2 可变默认值：`= []` 在 pydantic 里安全，但仍推荐 `default_factory`

**现象**：老教程和 Python 通用建议都说「别用列表当默认值，会在实例之间共享」。

**真实情况**（在 2.13.5 实测）：**pydantic 会深拷贝可变默认值，所以 `= []` 不会共享**。

```python
class Item(BaseModel):
    tags: list[str] = []      # 在 pydantic 里这样写是安全的

a, b = Item(), Item()
a.tags.append("x")
print(b.tags)                 # [] —— 没有被污染，pydantic 帮你拷贝了
```

对比一下普通 dataclass，同样写法直接拒绝启动：

```
ValueError: mutable default <class 'list'> for field tags is not allowed: use default_factory
```

**那什么时候必须用 `default_factory`**：当默认值需要**每次创建实例时重新计算**的时候。深拷贝只能复制那个「定义时算出来的值」，不能重新算。

```python
from datetime import datetime

class Log(BaseModel):
    at: datetime = datetime.now()                    # 错：定义类的那一刻算一次，之后所有实例都是同一个时间
    at2: datetime = Field(default_factory=datetime.now)   # 对：每次实例化都调用一次 datetime.now()
    ids: list[int] = Field(default_factory=list)     # 推荐：意图明确，换成 dataclass / 别的库也不会出错
```

**解决**：容器类默认值统一写 `Field(default_factory=list)`（或 `dict`、`set`），动态值必须写 `default_factory`。这不是为了绕开 bug，而是为了意图清楚和跨库一致。

### 6.3 v1 方法名在 v2 上的警告和报错

**报错信息原文**：

```
PydanticDeprecatedSince20: The `dict` method is deprecated; use `model_dump` instead.
Deprecated in Pydantic V2.0 to be removed in V3.0.
```

或者，跟着老教程写 `@validator` 时：

```
PydanticDeprecatedSince20: Pydantic V1 style `@validator` validators are deprecated.
You should migrate to Pydantic V2 style `@field_validator` validators.
```

**现象**：`.dict()`、`.json()`、`.parse_obj()` 这些还能跑，但每次都刷警告；`class Config:` 写法在某些配置项上直接不生效，且不报错，最坑。

**原因**：网上大量教程是 pydantic 1.x 时代写的。v2 保留旧名做过渡，v3 会删掉。

**解决**：对照第 3.7 节的表逐个替换。判断教程新旧的快速方法——看它有没有 `model_` 前缀的方法，有就是 v2，没有就直接跳过这篇教程。想把警告变成硬错误提早暴露问题，跑测试时加：

```bash
python -W error::DeprecationWarning -m pytest
```

### 6.4 `str` 字段会不会把 `123` 自动转成 `"123"`？—— 不会（和直觉相反）

**现象**：你以为 pydantic 「宽松模式什么都能转」，结果 LLM 返回 `{"invoice_no": 20260001}`（数字），校验直接失败：

```
1 validation error for Invoice
invoice_no
  Input should be a valid string [type=string_type, input_value=20260001, input_type=int]
```

**原因**：v2 默认的 lax（宽松）模式**不是无脑转换，它的转换是有方向性的**。实测结论：

| 输入 → 目标类型 | lax 模式（默认） | strict 模式 |
| --- | --- | --- |
| `"123"` → `int` | 通过，得到 `123` | 报错 `int_type` |
| `123` → `str` | **报错 `string_type`** | 报错 |
| `"23.50"` → `Decimal` / `float` | 通过 | 报错 |
| `"2026-01-05"` → `date` | 通过 | 报错 |
| `1` / `0` → `bool` | 通过 | 报错 |

<mark style="background: #BBFABBA6;">规律：**pydantic 允许「字符串解析成结构化类型」，但不允许「结构化类型退化成字符串」**。因为前者是明确的解析规则（JSON 里数字常被引号包住），后者会掩盖真正的数据错误——如果 `int` 能变 `str`，那所有类型都能变 `str`，`str` 注解就等于没写。</mark>

**解决**：字段确实可能收到数字又想存成文字，显式写一个 `mode="before"` 的校验器把意图讲清楚：

```python
@field_validator("invoice_no", mode="before")   # before：在类型检查之前动手
@classmethod
def to_str(cls, v):
    return str(v) if isinstance(v, int) else v  # 只把整数转文字，别的原样放过
```

反过来，想连 `"123" → int` 这种也禁掉（比如做金融数据，输入必须严格），开 strict：

```python
model_config = ConfigDict(strict=True)          # 整个模型严格
# 或者只对单个字段：qty: int = Field(strict=True)
# 或者单次调用：Model.model_validate(data, strict=True)
```

### 6.5 改了字段值，校验不会自动重跑

**现象**：模型建好后直接赋值，明显不合法的值也进去了：

```python
item = LineItem(name="笔", qty=1, unit_price=1)
item.qty = -999          # 违反 gt=0，但默认不会报错
print(item.qty)          # -999
```

<mark style="background: #BBFABBA6;">**原因**：pydantic 默认只在**创建**时校验，赋值不校验（为了性能，也因为大多场景数据是一次性入口进来的）。</mark>

**解决**：需要每次赋值都校验就开 `validate_assignment`：

```python
class LineItem(BaseModel):
    model_config = ConfigDict(validate_assignment=True)   # 赋值也走校验
    qty: int = Field(gt=0)

item = LineItem(qty=1)
item.qty = -999          # 现在会抛 ValidationError
```

或者干脆禁止修改，让模型只读（数据入口场景推荐，能避免下游偷改数据）：

```python
model_config = ConfigDict(frozen=True)   # 任何赋值都抛 ValidationError
```

顺带一个相关陷阱：`Model.model_construct(...)` 会**跳过所有校验**直接造对象（实测传 `x="abc"` 给 `int` 字段也能建成）。它是给「数据已经验过、追求性能」的场景准备的，别拿它处理外部数据。

---

## 7. 验证你学会了（自测题）

不看正文做，卡住了再回去查标注的小节。

### 第 1 题：概念复述（答案在第 1、2 节）

用自己的话回答，每题两三句，不许用「schema」「序列化」这类词：

1. pydantic 是干什么的？为什么说它像海关？
2. 不用 pydantic，手写校验会遇到哪三个具体麻烦？
3. 你的 LLM 抽取场景里，如果不校验直接把 LLM 返回的 dict 扔给 pandas，最可能在什么时候出问题？

### 第 2 题：默写最小示例（答案在第 4.1、4.2 节）

不看正文，从零写出一个 `Product` 模型并跑通：

- `name`：文字，长度 1 到 100
- `price`：数字，必须大于 0
- `stock`：整数，不能为负，**不传时默认 0**
- `tags`：文字列表，不传时默认空列表（注意第 6.2 节的推荐写法）

然后：
1. 用 `model_validate` 从 `{"name": "鼠标", "price": "89.9", "stock": "5"}` 造一个实例（注意 price 和 stock 都是字符串，想清楚哪个能过、为什么）。
2. 故意传 `price=-1` 且不传 `name`，捕获 `ValidationError`，用 `.errors()` 打印出「一共几处错、分别是哪个字段」。
3. 把实例转成 dict 打印出来（想清楚该用哪个方法，别写成 `.dict()`）。

### 第 3 题：组合应用变体（答案在第 4.3、6.1 节）

在第 4.3 节的 `Invoice` 基础上改造，三个要求递进：

1. 加一个字段 `total_declared: Decimal`（发票上印的合计金额），再加一个**跨字段校验**：它必须等于所有明细 `qty * unit_price` 的和，不等就报错「合计金额与明细不符」。想清楚该用 `field_validator` 还是 `model_validator`，以及为什么。
2. 加一个字段 `due_at: date | None`（付款截止日），要求它**不能早于** `issued_at`。注意 `issued_at` 可能是 `None`，校验器里要处理这种情况。
3. 写一个函数 `safe_extract(raw_json: str) -> Invoice | None`：校验通过返回模型，失败时把 `e.errors()` 整理成一句话（格式如 `items → 0 → qty: Input should be greater than 0`）打印出来并返回 `None`。

做完第 3 题，你已经能独立写第 5 节那条完整链路了。

