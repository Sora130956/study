# 结构化输出：用 Pydantic 约束 LLM 输出

- 目标读者：会调用 OpenAI API，想让输出更可控
- 前置知识：Python 类、类型注解、JSON、字典
- 学习时长：1 小时 / 可跳过：第 6 节（坑点）可在遇到时再回查

---

## 第 1 节：它是什么

**Pydantic 结构化输出**是用 Python 类定义"期望的数据格式"，让 LLM 严格按这个格式返回，并自动校验、转换类型。

**类比**：像给 AI 发了一张"表单模板"——表单上写明"姓名填这里、电话必须是数字、日期格式 YYYY-MM-DD"，AI 填完后，Pydantic 自动检查有没有漏填、格式对不对，不对就报错。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 不用它会怎样？

传统做法是让 LLM 返回 JSON 字符串，然后手工解析：

```python
response = client.chat.completions.create(...)
data = json.loads(response.choices[0].message.content)
name = data.get("name")  # 可能是 None
phone = data.get("phone")  # 可能是字符串、可能是数字、可能不存在
```

**痛点**：
- LLM 可能返回字段名拼写错误（`phon` 而非 `phone`）
- 类型不确定（`quantity` 可能是字符串 `"3"` 而非数字 `3`）
- 缺少字段时 `data.get()` 返回 `None`，下游代码崩溃
- 每次都要手写一堆 `if data.get(...) is not None` 的防御代码

### 问题根源

**数据校验逻辑分散在各处代码里**，而 LLM 输出本质是"不可信的外部输入"，需要一个统一的校验层。

### 经典应用场景

- 发票、简历、合同等文档信息提取（字段固定、类型明确）
- 结构化数据生成（如生成 SQL、生成配置文件）
- API 参数构造（从自然语言生成 API 调用参数）

---

## 第 3 节：读下去之前，先搞懂这些概念

| 术语 | 大白话解释 | 示例 | 对应正文节 |
|------|-----------|------|-----------|
| **BaseModel** | Pydantic 提供的基类，你的数据类要继承它 | `class User(BaseModel): ...` | 第 4 节示例 1 |
| **类型注解** | 用 `: int`、`: str` 标明字段类型 | `age: int` 表示 age 必须是整数 | 第 4 节示例 1 |
| **Field** | 给字段加额外约束（如描述、默认值、范围） | `Field(description="用户年龄")` | 第 4 节示例 2 |
| **model_json_schema()** | 把 Pydantic 模型转成 JSON Schema 字典 | 传给 OpenAI 的 `response_format` | 第 4 节示例 2 |
| **model_validate()** | 从字典创建模型实例并校验 | `User.model_validate(data)` | 第 4 节示例 1 |
| **ValidationError** | 校验失败时抛出的异常 | `except ValidationError as e: ...` | 第 6 节坑 2 |

**类型注解原理补充**：
```python
# 普通 Python 变量（类型注解不强制）
age: int = "18"  # 不会报错，只是提示

# Pydantic 模型（会真正校验）
class User(BaseModel):
    age: int  # 如果传入 "18"，Pydantic 自动转成 18；如果传入 "abc"，报错
```

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本

**目标**：定义一个数据结构，让 LLM 按格式返回并自动校验。

```python
from pydantic import BaseModel
from openai import OpenAI
import os

# 1. 定义数据模型：这是你期望的输出格式
class Invoice(BaseModel):
    invoice_no: str      # 发票号，字符串类型
    amount: float        # 金额，浮点数
    date: str            # 日期，字符串格式

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 2. 调用 API，告诉模型要返回的格式
response = client.beta.chat.completions.parse(  # 注意是 parse 方法
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "从文本中提取发票信息"},
        {"role": "user", "content": "发票号INV-001，金额1250.5元，日期2026-09-15"}
    ],
    response_format=Invoice  # 直接传 Pydantic 类
)

# 3. 获取解析后的对象（已经是 Invoice 实例）
invoice = response.choices[0].message.parsed
print(invoice.invoice_no)  # INV-001
print(invoice.amount)      # 1250.5（自动转成了 float）
print(type(invoice))       # <class '__main__.Invoice'>
```

**逐段解读**：

1. **定义模型**：
   - 继承 `BaseModel`，里面每个字段都有类型注解
   - 这相当于告诉 Pydantic："我要一个有这三个字段的对象，且类型必须对"

2. **调用 API**：
   - 用 `client.beta.chat.completions.parse()`（注意是 `parse` 而非 `create`）
   - `response_format=Invoice`：把 Pydantic 类直接传进去，SDK 自动生成 JSON Schema

3. **获取结果**：
   - `.message.parsed` 返回的已经是 `Invoice` 对象（不是字典）
   - 可以用 `invoice.amount` 直接访问，且类型已经正确（字符串 `"1250.5"` 自动转成了 `float`）

**这一步你得到了什么**：不用手写 `json.loads()` 和类型检查，输出自动变成强类型对象。

---

### 示例 2：加字段描述和约束

**目标**：给字段加说明（帮助 LLM 理解）和验证规则。

```python
from pydantic import BaseModel, Field
from openai import OpenAI
import os

class Order(BaseModel):
    order_id: str = Field(description="订单编号")
    customer_name: str = Field(description="客户姓名")
    quantity: int = Field(description="购买数量", ge=1)  # ge=1 表示 >= 1
    phone: str = Field(description="联系电话，11位数字")

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "提取订单信息"},
        {"role": "user", "content": "订单ORD-001，客户李四，买了3个，电话13800138000"}
    ],
    response_format=Order
)

order = response.choices[0].message.parsed
print(order.model_dump())  # 转成字典查看
# {'order_id': 'ORD-001', 'customer_name': '李四', 'quantity': 3, 'phone': '13800138000'}
```

**逐段解读**：

- **Field(description=...)**：这个描述会被转换成 JSON Schema 的 `description` 字段，LLM 能看到，帮助它理解字段含义
- **ge=1**：`greater than or equal`，表示数量必须 >= 1。如果 LLM 返回 0 或负数，Pydantic 会报 `ValidationError`
- **model_dump()**：把 Pydantic 对象转成普通字典（方便存数据库或传给其他函数）

**这一步你得到了什么**：可以给 LLM "提示"每个字段的意义，并加上数值范围等约束，输出质量更高。

---

### 示例 3：嵌套结构和可选字段

**目标**：处理复杂数据（对象里包含对象、某些字段可能缺失）。

```python
from pydantic import BaseModel, Field
from typing import Optional
from openai import OpenAI
import os

# 地址是一个嵌套对象
class Address(BaseModel):
    city: str = Field(description="城市")
    district: str = Field(description="区县")

class Person(BaseModel):
    name: str = Field(description="姓名")
    age: int = Field(description="年龄")
    email: Optional[str] = Field(default=None, description="邮箱，可能没有")  # 可选字段
    address: Address = Field(description="地址信息")

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "从简历中提取信息"},
        {"role": "user", "content": "张三，28岁，住在北京市朝阳区"}
    ],
    response_format=Person
)

person = response.choices[0].message.parsed
print(person.name)              # 张三
print(person.email)             # None（因为文本里没提到）
print(person.address.city)      # 北京市
print(person.address.district)  # 朝阳区
```

**逐段解读**：

- **嵌套模型**：`Address` 本身也是一个 `BaseModel`，可以嵌在 `Person` 里。LLM 会返回 `{"address": {"city": "...", "district": "..."}}`，Pydantic 自动递归解析
- **Optional[str]**：表示这个字段可以是字符串或 `None`。如果 LLM 没返回 `email`，不会报错，而是赋值 `None`
- **default=None**：当字段缺失时的默认值

**和我直觉相反的点**：`Optional[str]` 不表示"可以不传这个字段"，而是"传了也可以是 None"。如果想让字段真正可选（不传也行），必须同时加 `default=None`。

**这一步你得到了什么**：可以处理真实场景里的复杂结构（地址、订单明细、多层嵌套），且容忍部分字段缺失。

---

## 第 5 节:最小可用的实际用法

以下是**生产场景模板**：批量提取文本并写入 CSV。

```python
from pydantic import BaseModel, Field, ValidationError
from openai import OpenAI
import os
import csv

class Invoice(BaseModel):
    invoice_no: str = Field(description="发票号")
    amount: float = Field(description="金额")
    date: str = Field(description="日期 YYYY-MM-DD")

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

texts = [
    "发票INV-001，金额500元，日期2026-09-10",
    "发票INV-002，金额1200.5元，日期2026-09-12"
]

results = []

for text in texts:
    try:
        response = client.beta.chat.completions.parse(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "提取发票信息"},
                {"role": "user", "content": text}
            ],
            response_format=Invoice,
            temperature=0  # ✅ 必须：保证输出稳定
        )
        
        invoice = response.choices[0].message.parsed
        results.append(invoice.model_dump())  # 转成字典
        
    except ValidationError as e:
        # ⚠️ 强烈建议：记录校验失败的案例
        print(f"校验失败：{text}\n错误：{e}")
        continue

# 写入 CSV
with open("invoices.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["invoice_no", "amount", "date"])
    writer.writeheader()
    writer.writerows(results)

print(f"成功提取 {len(results)} 条记录")
```

**哪几行是必须的**：
- ✅ 必须：继承 `BaseModel`、字段类型注解、`Field(description=...)`
- ✅ 必须：用 `client.beta.chat.completions.parse()` 而非 `create()`
- ✅ 必须：`temperature=0`（保证稳定输出）
- ⚠️ 强烈建议：`try-except ValidationError`（处理格式错误）

**实际项目里通常会配什么**（进阶）：
- 自定义校验器（如 `@field_validator` 检查电话号格式）
- 日志记录失败案例（保存原始文本和错误，后续人工审核）
- 重试机制（校验失败时重新调用 LLM，换个 prompt）

---

## 第 6 节：常见坑与易错点

### 坑 1：忘记用 `parse()` 方法

**现象**：用了 `client.chat.completions.create()` 传 `response_format=Invoice`，报错。

**原因**：结构化输出必须用 `client.beta.chat.completions.parse()`，这是专门的解析方法。

**解决**：把 `create` 改成 `parse`。

---

### 坑 2：字段缺失但没设 Optional

**现象**：LLM 没返回某个字段，抛 `ValidationError: field required`。

**原因**：Pydantic 默认所有字段都是必填。

**解决**：
- 字段可能缺失时，用 `Optional[类型]` 并加 `default=None`
- 或设 `Field(default="未知")` 给一个默认值

---

### 坑 3：LLM 返回的类型不对

**现象**：定义 `amount: float`，LLM 返回 `"1250.5"`（字符串），但没报错。

**原因**：Pydantic 会**自动尝试类型转换**（字符串 `"123"` → 整数 `123`）。只有转换失败时才报错（如 `"abc"` → int）。

**解决**：这通常是好事（容忍 LLM 的小瑕疵）。但如果不想自动转换，用 `Field(strict=True)`。

---

### 坑 4：description 写得不清楚，LLM 理解错误

**现象**：字段 `date`，description 是"日期"，LLM 返回 `"九月十五号"`。

**原因**：description 太模糊，LLM 自由发挥。

**解决**：description 要具体，如 `"日期，格式 YYYY-MM-DD"`。必要时在 system prompt 里也强调一遍。

---

### 坑 5：嵌套模型的 Field description 被忽略

**现象**：给嵌套的 `Address` 类加了 `Field(description=...)`，但 LLM 提取错误。

**原因**：嵌套模型的 description 也会传给 LLM，但优先级低于外层的 system prompt。

**解决**：关键字段的格式要求同时写在：① Field description ② system prompt ③ user message 示例（三管齐下）。

---

## 第 7 节：验证你学会了

### 问题 1：用一句话说清 Pydantic 结构化输出解决什么问题

**提示**：回查第 2 节。

---

### 问题 2：不看文档，默写一个最小示例

**要求**：
- 定义一个 `Product` 类，包含 `name: str` 和 `price: float`
- 调用 API 提取 "iPhone 15，价格5999元"
- 打印 `product.name` 和 `product.price`

**提示**：参考第 4 节示例 1。

---

### 问题 3：变体场景

**任务**：定义一个 `Resume` 类，包含：
- `name: str`（姓名）
- `skills: list[str]`（技能列表，如 `["Python", "SQL"]`）
- `years_of_experience: int`（工作年限）
- `email: Optional[str]`（邮箱，可能没有）

从文本 "李四，会 Python 和 SQL，工作 3 年" 中提取信息，打印技能列表。

**提示**：
- 列表类型怎么写？（`list[str]`）
- 可选字段怎么设置？（第 4 节示例 3）
- 如何访问提取后的列表？（`.skills`）
