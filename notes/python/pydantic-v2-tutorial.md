# Pydantic v2 零基础教程

> **来源:** https://docs.pydantic.dev/latest/ （官方文档精读 + 实践总结）
> **对应计划周次:** 第 1 周 · 周一下午 pydantic v2 深入（`BaseModel`、`Field` 校验、嵌套模型、`model_validate`）

## 核心理解

Pydantic 做的事一句话概括：**用 Python 类型注解声明「数据应该长什么样」，运行时自动校验 + 转换，不合格就报错，合格就给你一个类型正确的对象**。

没有 pydantic 时，拿到一个外部输入（API 响应、JSON 文件、表单数据）只能手写一堆 `if 'name' not in data: ...` 防御性代码。有了 pydantic，校验规则声明在类上，解析一行 `User.model_validate(data)` 搞定——**数据进系统的第一道关口交给它守，后面的代码可以放心假设数据是干净的**。

为什么它在 AI 应用里是地基：LLM 返回的是不可靠的文本，把它约束成结构化输出靠的就是 pydantic 模型（JSON Schema 由模型自动生成）；FastAPI 的请求体/响应体校验全部建立在 pydantic 之上。学 FastAPI 前必须先过这关。

v2 相对 v1 是**重写过的 Rust 内核**，速度提升 5–50 倍，但 API 变了：方法从 `.dict()` / `.parse_obj()` 换成了 `model_dump()` / `model_validate()`。**网上 v1 教程很多，学的时候认准 v2 写法**，文末有对照表。

## 安装

```bash
pip install pydantic
# 或项目用 uv 管理
uv add pydantic
```

邮箱、URL 等特殊类型需要额外装：

```bash
pip install "pydantic[email]"   # EmailStr 需要
```

## 关键点

### 1. BaseModel：定义一个模型

继承 `BaseModel`，用类型注解声明字段。实例化时自动校验 + 转换：

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str

user = User(name="张三", age="25", email="zhangsan@example.com")
print(user.age)   # 25 —— 注意：字符串 "25" 被自动转成了 int
print(type(user.age))  # <class 'int'>
```

⚠️ 默认模式下 pydantic 会**尽力做合理转换**（lax mode）：`"25"` → `25`、`"3.14"` → `3.14`、`1` → `True` 这种都能转。不想要这种宽容就开严格模式（见第 8 节）。

数据不合格时抛 `ValidationError`，错误信息极其详细（哪个字段、期望什么、实际收到什么）：

```python
from pydantic import ValidationError

try:
    User(name="张三", age="abc", email="x@x.com")
except ValidationError as e:
    print(e)
    # 1 validation error for User
    # age
    #   Input should be a valid integer, unable to parse string as an integer
    #   [type=int_parsing, input_value='abc', input_type=str]
```

接单时的典型用法：**外部数据进来第一件事就是过模型**，错误在这里集中爆发，而不是在代码深处莫名其妙炸掉。

### 2. 可选字段与默认值

```python
from typing import Optional

class User(BaseModel):
    name: str                            # 必填
    age: Optional[int] = None            # 可选，缺省 None
    tags: list[str] = []                 # 可选，缺省空列表
    is_active: bool = True               # 可选，缺省 True
```

⚠️ 三个容易踩的坑：

1. `Optional[int]` 不带默认值时**仍是必填字段**（只是允许传 `None`）。要「可以不传」必须给默认值：`Optional[int] = None`。
2. **可变默认值（`list` / `dict` / `set`）直接写是安全的**——pydantic 会自动为每个实例深拷贝一份，不像原生 dataclass 那样共享。但习惯上更稳妥的写法是 `Field(default_factory=list)`（见下节）。
3. `Optional` 的 v2 推荐写法是 `int | None`（Python 3.10+），更简洁：

```python
class User(BaseModel):
    age: int | None = None
```

### 3. Field：给字段加校验规则

类型注解只能约束「类型」，`Field()` 负责约束「值」：范围、长度、正则、别名、说明。

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    price: float = Field(gt=0, description="必须大于 0")
    quantity: int = Field(default=0, ge=0)          # ge = >=
    sku: str = Field(pattern=r"^[A-Z]{3}-\d{4}$")   # 正则
    category: str = Field(default="general", alias="cat")  # 外部叫 cat，内部叫 category
```

常用参数速查：

| 参数 | 作用 | 适用类型 |
|------|------|---------|
| `default` | 默认值 | 所有 |
| `default_factory` | 默认值工厂（调用一次生成） | 所有，常用于 list/dict/datetime |
| `gt` / `ge` / `lt` / `le` | 大于 / 大于等于 / 小于 / 小于等于 | 数字、日期 |
| `min_length` / `max_length` | 长度 | str、list 等 |
| `pattern` | 正则约束 | str |
| `alias` | 外部数据里的字段名映射 | 所有 |
| `description` | 字段说明（会进 JSON Schema） | 所有 |
| `examples` | 示例值（会进 JSON Schema） | 所有 |

`default_factory` 的典型场景——时间戳和可变容器：

```python
from datetime import datetime, timezone
from pydantic import BaseModel, Field

class Order(BaseModel):
    items: list[str] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```

⚠️ `alias` 有个坑：设了 `alias="cat"` 后，**默认必须用 `cat` 来传参**，用 `category=` 反而报错。想两个都接受，加配置 `model_config = ConfigDict(populate_by_name=True)`（见第 8 节）。

### 4. 常用类型库

pydantic 支持的远不止 `str / int`。这一节挑接单最常用的：

```python
from datetime import date, datetime
from decimal import Decimal
from enum import Enum
from typing import Literal
from uuid import UUID

from pydantic import BaseModel, EmailStr, HttpUrl

class PayMethod(str, Enum):
    CARD = "card"
    BANK = "bank"

class Invoice(BaseModel):
    id: UUID                                  # 自动校验并转成 UUID 对象
    amount: Decimal                           # 金额用 Decimal，别用 float（精度丢失）
    status: Literal["pending", "paid", "void"]  # 只允许这三个字符串
    method: PayMethod                         # 枚举，传 "card" 自动转 PayMethod.CARD
    issued_on: date                           # "2026-09-04" → date 对象
    paid_at: datetime | None = None
    buyer_email: EmailStr                     # 校验邮箱格式（需 pydantic[email]）
    callback: HttpUrl | None = None           # 校验 URL 格式
```

选类型的心法：

- 金额 → `Decimal`，永远不用 `float`。
- 有限取值 → `Literal[...]` 或 `Enum`；Literal 轻量，Enum 便于在代码里引用。
- 日期时间 → `date` / `datetime`，ISO 格式字符串自动解析。
- 格式类（邮箱 / URL / UUID）→ 用 pydantic 内置类型，别自己写正则。

### 5. 嵌套模型：接单场景的主力

模型字段可以是另一个模型，校验会递归到底。非结构化文本 → 结构化数据（W2 周三的主力接单场景）全靠它：

```python
from pydantic import BaseModel, Field

class LineItem(BaseModel):
    description: str
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)

class Invoice(BaseModel):
    invoice_no: str
    items: list[LineItem]           # 嵌套模型列表
    total: float

data = {
    "invoice_no": "INV-001",
    "items": [
        {"description": "咨询费", "quantity": 2, "unit_price": 150.0},
        {"description": "差旅", "quantity": "1", "unit_price": "800"},  # 字符串自动转
    ],
    "total": 1100.0,
}

invoice = Invoice.model_validate(data)
print(invoice.items[0].quantity)   # 2 (int)
print(invoice.items[1].unit_price) # 800.0 (float)
```

「文本 → pydantic → pandas → Excel」流水线里，LLM 抽出来的 JSON 就是这样灌进模型里清洗、然后 `model_dump()` 成 dict 交给 pandas 的。

### 6. 四个必背方法（v2 的核心 API）

```python
class User(BaseModel):
    name: str
    age: int

# ① model_validate：dict → 模型（最常用，外部数据入口）
user = User.model_validate({"name": "张三", "age": "25"})

# ② model_validate_json：JSON 字符串 → 模型
user2 = User.model_validate_json('{"name": "李四", "age": 30}')

# ③ model_dump：模型 → dict（交给 pandas / 存库前）
d = user.model_dump()          # {'name': '张三', 'age': 25}
d2 = user.model_dump(exclude={"age"})         # 排除字段
d3 = user.model_dump(exclude_none=True)       # 排除 None 值（API 出参常用）

# ④ model_dump_json：模型 → JSON 字符串
s = user.model_dump_json()     # '{"name":"张三","age":25}'
```

记忆法：**validate 是「进来」，dump 是「出去」**。`model_validate*` 把外部脏数据变成干净模型；`model_dump*` 把模型还原成 dict / JSON 供序列化。

外加一个 AI 场景高频方法：

```python
schema = User.model_json_schema()
# {'properties': {'name': {'title': 'Name', 'type': 'string'},
#                 'age':  {'title': 'Age',  'type': 'integer'}},
#  'required': ['name', 'age'], 'title': 'User', 'type': 'object'}
```

`model_json_schema()` 生成的 JSON Schema 直接喂给 LLM 的 structured output / function calling——**这就是「用 pydantic 模型约束 LLM 输出」的机理**，W2 周三会用到。

### 7. 自定义校验：field_validator 与 model_validator

内置参数不够时，写自己的校验逻辑。

`@field_validator` 针对**单个字段**，在类型转换后运行：

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    name: str
    email: str

    @field_validator("name")
    @classmethod
    def name_not_blank(cls, v: str) -> str:
        v = v.strip()                    # 顺便做清洗：去首尾空格
        if not v:
            raise ValueError("name 不能是空白")
        return v                         # 返回的是最终存入的值
```

`@model_validator` 针对**整个模型**，用于跨字段校验（单字段做不到的事）：

```python
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start: str
    end: str

    @model_validator(mode="after")
    def check_order(self) -> "DateRange":
        if self.start > self.end:
            raise ValueError("start 必须早于 end")
        return self
```

规则很简单：**单字段用 field_validator，多字段联动用 model_validator(mode="after")**。

### 8. model_config：模型级配置

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        strict=True,                # 严格模式："25" 不再自动转 int，必须给 int
        populate_by_name=True,      # 设了 alias 也允许用原字段名传参
        extra="forbid",             # 多出未声明的字段直接报错（默认是忽略）
        frozen=True,                # 实例不可变（赋值报错），类似 frozen dataclass
    )
    name: str
```

按场景挑着用，不要全抄：

- 接外部 API 数据 → `extra="ignore"`（默认）或 `"forbid"`，看你要不要防数据漂移。
- 对接财务类数据 → `strict=True`，拒绝隐式转换。
- 内部领域模型 → `frozen=True` 防止意外修改。

### 9. computed_field：派生字段

由其他字段算出来的字段，会出现在序列化结果里，但不需要传入：

```python
from pydantic import BaseModel, computed_field

class Rectangle(BaseModel):
    width: float
    height: float

    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height

r = Rectangle(width=3, height=4)
print(r.area)                       # 12.0
print(r.model_dump())               # {'width': 3.0, 'height': 4.0, 'area': 12.0}
```

### 10. v1 → v2 对照表（看旧教程/旧代码时防迷路）

| v1（已废弃） | v2（现在用） |
|--------------|--------------|
| `.parse_obj(d)` | `.model_validate(d)` |
| `.parse_raw(s)` | `.model_validate_json(s)` |
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `.schema()` | `.model_json_schema()` |
| `@validator("f")` | `@field_validator("f")` |
| `@root_validator` | `@model_validator(mode="after")` |
| `class Config: ...` | `model_config = ConfigDict(...)` |
| `Field(regex=...)` | `Field(pattern=...)` |
| `Field(min_items=...)` | `Field(min_length=...)` |

看到左边列的写法，说明资料是 v1 的，直接换算成右边再学。

## 和 FastAPI / AI 接单的关系

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None

@app.post("/chat")
def chat(req: ChatRequest):       # FastAPI 自动用 pydantic 校验请求体
    return {"echo": req.message}  # 请求不合格时自动返回 422，不用你写一行校验代码
```

- FastAPI 请求体/响应模型 = pydantic 模型，W1 学的 FastAPI 全程在用它。
- LLM 结构化输出 = 把 `model_json_schema()` 喂给模型 + 把模型返回的 JSON 用 `model_validate()` 收口（W2 周三）。
- 工具调用 schema = pydantic 模型（W2 周四）。

## 动手练习（建议全做）

1. 定义 `Employee(name, age, salary, email)`：salary 必须 > 0，email 用 `EmailStr`，age 默认 18。
2. 写一个嵌套模型 `Order(items: list[Item])`，用 `model_validate` 解析一段手写 dict，故意写错一个字段类型，读 `ValidationError` 的报错。
3. 给 `User` 加 `field_validator`：密码至少 8 位且含数字。
4. 用 `model_json_schema()` 打印模型的 JSON Schema，看一眼长什么样——这是 W2 给 LLM 用的东西。
5. 把第 1 题的模型实例 `model_dump(exclude_none=True)` 后 `pandas.DataFrame([...])`，跑通「模型 → 表格」链路。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| BaseModel | n. | pydantic 模型基类，所有模型继承它 |
| ValidationError | n. | 校验失败时抛的异常，含每个字段的详细错误 |
| model_validate | v. | 把 dict 校验并转成模型实例（v2 核心入口） |
| model_dump | v. | 把模型实例转回 dict |
| model_json_schema | v. | 生成模型的 JSON Schema，供 LLM 结构化输出用 |
| Field | n. | 字段级校验参数（范围/长度/正则/别名等） |
| field_validator | n. | 单字段自定义校验装饰器 |
| model_validator | n. | 跨字段联动校验装饰器 |
| ConfigDict | n. | 模型级配置（严格模式/别名/额外字段策略） |
| strict mode | n. | 严格模式，禁用隐式类型转换 |
| alias | n. | 外部数据字段名与模型字段名的映射 |
| default_factory | n. | 默认值工厂，实例化时调用生成默认值 |
| computed_field | n. | 由其他字段派生的计算字段 |
| lax mode | n. | 默认宽松模式，尽力做类型转换（如 "25"→25） |
| EmailStr / HttpUrl | n. | pydantic 内置格式校验类型（邮箱 / URL） |
| Literal | n. | 限定字符串取值的类型注解 |
| JSON Schema | n. | 描述 JSON 数据结构的标准，LLM 结构化输出依赖它 |
