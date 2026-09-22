# 15 - Structured Output 结构化输出完整指南【必读】

> **原文档**：https://ai.pydantic.dev/output/
> **相关文档**：https://ai.pydantic.dev/api/output/
> **精读日期**：2026-09-21
> **学习目标**：掌握 `output_type` 的全部写法、三种输出模式的选型、输出校验与重试，让 Smart-Data-Extractor 稳定吐出可直接入库的结构化数据

---

## 一、是什么

### 大白话版本

LLM 天生只会吐字符串。**结构化输出**就是给它套一副"模具"——你声明一个 Pydantic 模型，Pydantic AI 负责把模型的自由发挥压进这副模具里，校验不过就让它重来。

一句话：<mark style="background: #BBFABBA6;">`output_type` 是你和模型之间的合同，`AgentRunResult.output` 是履约结果。</mark>

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class CityLocation(BaseModel):
    city: str
    country: str

agent = Agent('google:gemini-3-flash-preview', output_type=CityLocation)

result = agent.run_sync('Where were the olympics held in 2012?')
print(result.output)
#> city='London' country='United Kingdom'
```

没有 `output_type` 时 `result.output` 就是 `str`，有了它就是一个**已经过 Pydantic 校验**的对象。

### 三个部件

| 部件                        | 作用              | 典型写法                     |
| ------------------------- | --------------- | ------------------------ |
| `output_type`             | 声明"我要什么形状的数据"   | `output_type=Resume`     |
| **输出模式**                  | 决定"怎么逼模型给出这个形状" | Tool / Native / Prompted |
| `@agent.output_validator` | 声明"什么样的数据才算合格"  | raise `ModelRetry` 打回重做  |

### 三种输出模式速览

| 模式                  | 原理                                            | 可靠性 | 模型支持度                    | 何时用                                              |
| ------------------- | --------------------------------------------- | --- | ------------------------ | ------------------------------------------------ |
| **Tool Output**（默认） | 注册一个名为 `final_result` 的工具，模型通过 tool call 交付结果 | 高   | 所有支持函数调用的模型              | <mark style="background: #BBFABBA6;">默认首选</mark> |
| **Native Output**   | 用模型原生的 JSON Schema response format            | 最高  | 仅部分模型（OpenAI / Gemini 等） | 纯提取、不需要函数工具时                                     |
| **Prompted Output** | 把 schema 塞进 instructions，靠提示词约束               | 最低  | 所有模型                     | 前两者都不支持时的兜底                                      |

---

## 二、为什么需要（Smart-Data-Extractor 实战场景）

Portfolio 项目要做的事是"简历 PDF → 结构化 JSON → 入库"。如果不用结构化输出：

| 痛点   | 无结构化输出                           | 有结构化输出                     |
| ---- | -------------------------------- | -------------------------- |
| 字段缺失 | 模型今天给 `phone`，明天给 `phone_number` | schema 强约束，字段名固定           |
| 类型错乱 | `"工作 3 年"` 这种字符串塞进 `years: int`  | Pydantic 校验直接拦截            |
| 解析代码 | 手写正则 + `json.loads` + try/except | 零解析代码，拿到就是对象               |
| 错误恢复 | 解析失败只能整轮重跑                       | `ModelRetry` 把错误信息回传，模型自己修 |
| 下游对接 | FastAPI response_model 要再写一遍     | 同一个 Pydantic 模型直接复用        |

<mark style="background: #FFF3A3A6;">结构化输出不是"锦上添花"，它是提取类项目的地基。</mark>

---

## 三、核心概念

### 3.1 `output_type` 支持哪些类型

几乎所有 Pydantic 能校验的类型都能直接当 `output_type`：

```python
from dataclasses import dataclass
from typing import TypedDict
from pydantic import BaseModel

# 1. 标量
Agent('openai:gpt-5-mini', output_type=int)
Agent('openai:gpt-5-mini', output_type=bool)

# 2. 容器
Agent('openai:gpt-5-mini', output_type=list[str])
Agent('openai:gpt-5-mini', output_type=dict[str, int])

# 3. TypedDict
class Person(TypedDict):
    name: str
    age: int

# 4. dataclass
@dataclass
class Box:
    width: int
    height: int

# 5. Pydantic 模型（最常用）
class Resume(BaseModel):
    name: str
    email: str

# 6. Union —— 模型二选一
Agent('openai:gpt-5-mini', output_type=Box | str)

# 7. 列表形式的多选（推荐写法）
Agent('openai:gpt-5-mini', output_type=[Box, str])
```

**两个内部机制要知道**：

1. **非 object 的 schema 会被自动包一层**。`output_type=int` 时，Pydantic AI 会生成 `{"type": "object", "properties": {"response": {"type": "integer"}}}` 这样的单元素对象，因为大多数模型的 JSON Schema 接口只接受 object 顶层。你拿到的还是 `int`，包装是透明的。

2. **多个输出类型 = 多个独立的 output tool**，而不是一个带 `anyOf` 的大 schema。这样每个 schema 都很简单，<mark style="background: #BBFABBA6;">模型选对的概率显著提高</mark>。

### 3.2 <mark style="background: #BBFABBA6;">`str` 的特殊地位
</mark>

| 写法 | 行为 |
|------|------|
| `output_type=str`（默认） | 只接受纯文本 |
| `output_type=Box` | **强制**结构化，模型不能回纯文本 |
| `output_type=[Box, str]` | 能提取就给 `Box`，提不出来就用文本解释原因 |

<mark style="background: #BBFABBA6;">`[Box, str]` 这个组合在提取场景里极其实用：</mark>

```python
from pydantic_ai import Agent
from dataclasses import dataclass

@dataclass
class Box:
    width: int
    height: int
    depth: int
    units: str

agent = Agent(
    'openai:gpt-5-mini',
    output_type=[Box, str],
    instructions=(
        "Extract me the dimensions of a box, "
        "if you can't extract all data, ask the user to try again."
    ),
)

result = agent.run_sync('The box is 10x20x30')
print(result.output)
#> Please provide the units for the dimensions (e.g., cm, in, m).

result = agent.run_sync('The box is 10x20x30 cm')
print(result.output)
#> Box(width=10, height=20, depth=30, units='cm')
```

拿到结果后用 `isinstance` 分支处理即可。

### 3.3 Union 类型的类型检查坑

静态类型检查器目前无法从 `output_type` 推断出 Union 的结果类型（PEP-747 尚未落地），所以需要**显式标注泛型参数**：

```python
from pydantic_ai import Agent

agent = Agent[object, list[str] | list[int]](
    'openai:gpt-5-mini',
    output_type=list[str] | list[int],  # type: ignore
    instructions='Extract either colors or sizes from the shapes provided.',
)

result = agent.run_sync('red square, blue circle, green triangle')
print(result.output)
#> ['red', 'blue', 'green']
```

`Agent[DepsT, OutputT]` 两个泛型参数：第一个是依赖类型（没有依赖就写 `object`），第二个是输出类型。

⚠️ **额外注意**：mypy 对**列表形式**（`output_type=[A, B]`）和 **async output function** 的推断也有误，pyright 则正常。用 mypy 的项目准备好加 `# type: ignore`。

### 3.4 三种输出模式详解

#### 为什么需要三种模式？

LLM 获取结构化输出的本质是**把自由文本压进 JSON Schema 模具**，但不同模型对"模具"的支持方式各不相同：

| 模型特性                              | 需要用哪种模式               |
| --------------------------------- | --------------------- |
| 支持函数调用（Function Calling / Tool Use） | Tool Output（默认）       |
| 支持原生 Structured Output API          | Native Output         |
| 啥高级特性都不支持，只会吐文本                   | Prompted Output（兜底方案） |

Pydantic AI 的设计是**自动适配**——你声明 `output_type=Resume`，它会根据模型能力选最可靠的方式。但你也可以显式指定用哪种模式来获得更精细的控制。

---

#### Tool Output（默认）—— 把输出伪装成"工具调用"

**原理**：把你的 `output_type` 注册成一个名为 `final_result` 的"假工具"，模型通过调用这个工具来交付结果。

**为什么这样设计？** 因为大多数现代模型（OpenAI / Anthropic / Gemini）的函数调用机制已经做了严格的 JSON Schema 校验，借用它就能获得可靠的结构化输出。

```python
from pydantic_ai import Agent, ToolOutput

# 简单形式（单一输出类型）
agent = Agent('openai:gpt-4o', output_type=Resume)
# 内部会注册一个名为 final_result 的工具，参数就是 Resume 的字段

# 复杂形式（多个输出类型，需要显式配置）
agent = Agent(
    'openai:gpt-4o',
    output_type=[
        ToolOutput(Fruit, name='return_fruit', description='返回水果信息'),
        ToolOutput(Vehicle, name='return_vehicle', description='返回交通工具信息'),
    ],
)
```

**ToolOutput 可配置项**：

| 参数            | 说明                                       | 默认值              |
| ------------- | ---------------------------------------- | ---------------- |
| `name`        | 工具名，模型调用时会看到这个名字                         | `final_result`   |
| `description` | 工具描述，帮助模型判断"什么情况下该调用这个工具"                | 自动从模型的 docstring |
| `max_retries` | 该输出工具的重试上限（可以和其他工具的重试预算分开配置）             | 继承 agent 的 retries |
| `strict`      | 是否启用 provider 的 strict mode（解码层面保证格式合法） | `False`          |

**适用场景**：
- ✅ 你需要同时注册函数工具（如 `search_database`）和输出类型
- ✅ 模型支持函数调用（几乎所有主流模型）
- ❌ 模型不支持函数调用（如早期的 text-davinci-003）

---

#### Native Output —— 用模型的原生 Structured Output API

**原理**：直接调用模型 API 的原生 `response_format` / `schema` 参数，让模型在**解码阶段**就保证输出符合 schema。

```python
from pydantic_ai import Agent, NativeOutput

agent = Agent(
    'openai:gpt-4o',
    output_type=NativeOutput(
        Resume,  # 或者 [Fruit, Vehicle]
        name='Resume',  # 可选：给 schema 起个名字
        description='Extract structured resume data',
    ),
)
```

**与 Tool Output 的关键区别**：

| 特性       | Tool Output               | Native Output           |
| -------- | ------------------------- | ----------------------- |
| 实现方式     | 注册成一个"假工具"，模型通过函数调用交付结果  | 直接用模型 API 的 schema 参数   |
| 模型支持度    | 所有支持函数调用的模型               | 仅 OpenAI / Gemini / 部分模型 |
| 能否同时用函数工具 | ✅ 可以                      | ❌ 大多数模型不支持（Gemini 3 例外） |
| 可靠性      | 高                         | 最高（解码层面保证）              |

**限制**：
- 并非所有模型支持（查看 [Model capabilities](https://ai.pydantic.dev/models/#model-capabilities) 表格）
- <mark style="background: #FF5582A6;">多数模型在 Native Output 模式下不能同时使用函数工具</mark>（Gemini 3 支持，更早的 Gemini 不行）

**适用场景**：
- ✅ 纯提取任务（不需要函数工具）
- ✅ 模型明确支持原生 Structured Output
- ❌ 需要同时调用函数工具

---

#### Prompted Output —— 靠提示词约束（兜底方案）

**原理**：把 JSON Schema 塞进 system prompt，靠提示词让模型"自觉"输出 JSON。

```python
from pydantic_ai import Agent, PromptedOutput

agent = Agent(
    'openai:gpt-4o',
    output_type=PromptedOutput(
        Resume,
        template='Please respond with JSON matching this schema: {schema}',
    ),
)
```

**内部行为**：
1. 把 `Resume` 的 JSON Schema 序列化成字符串
2. 替换 `{schema}` 占位符，注入到 instructions
3. 如果模型支持 JSON Mode（如 OpenAI 的 `response_format={"type": "json_object"}`），自动启用

**可配置项**：

| 参数         | 说明                                         |
| ---------- | ------------------------------------------ |
| `template` | 提示词模板，`{schema}` 是占位符                      |
| `template=False` | 完全禁用 schema 注入（你自己在 instructions 里写提示词） |

**适用场景**：
- ✅ 模型不支持函数调用，也不支持原生 Structured Output
- ✅ 你想完全自定义提示词（设 `template=False`）
- ❌ 需要高可靠性（这是三种模式里最不可靠的）

---

#### 选型决策树

```
┌─ 需要函数工具？
│   ├─ 是 ────────────────────► Tool Output（默认）
│   └─ 否
│       ├─ 模型支持 Native？
│       │   ├─ 是 ──────────► Native Output
│       │   └─ 否 ──────────► Prompted Output
```

**大多数情况直接用默认的 Tool Output 即可**。只有在以下场景才需要显式指定：
- 纯提取任务 + 模型支持 Native → 用 `NativeOutput` 获得最高可靠性
- 模型啥都不支持 → 用 `PromptedOutput` 兜底
- 想完全控制提示词 → 用 `PromptedOutput(template=False)`

### 3.5 Output Functions —— 把"执行动作"当作输出

#### 什么是 Output Function？

前面的 `output_type` 都是数据模型（`Resume`、`Box`），但有时你希望**模型的最终输出是一个动作**，比如：
- 执行一条 SQL 查询
- 调用第三方 API
- 触发一个业务流程

这时就用 **Output Function**——把函数直接当作 `output_type`。

#### 示例：SQL 查询 Agent

```python
from pydantic_ai import Agent, RunContext, ModelRetry

def run_sql_query(ctx: RunContext[DatabaseConn], query: str) -> list[dict]:
    """Execute a SQL query against the database."""
    try:
        return ctx.deps.execute(query)
    except QueryError as e:
        raise ModelRetry(f'Invalid query: {e}') from e

agent = Agent('openai:gpt-4o', deps_type=DatabaseConn, output_type=run_sql_query)

# 用户问："统计每个部门的人数"
result = agent.run_sync('Count employees by department', deps=db_conn)
print(result.output)  # [{'dept': 'Engineering', 'count': 42}, ...]
```

**关键点**：
- 模型生成 SQL 语句 → 作为 `query` 参数传给 `run_sql_query`
- 函数执行 → 返回值直接成为 `result.output`
- 如果 SQL 语法错误 → `raise ModelRetry` 让模型重新生成

#### Output Function vs 普通 Tool 的核心区别

| 特性       | Output Function                | 普通 Tool (`@agent.tool`) |
| -------- | ------------------------------ | ------------------------- |
| 是否强制调用   | **是**，模型必须调用其中之一               | 否，模型自行决定要不要调用             |
| 调用后的行为   | **直接结束 run**，返回值成为 `result.output` | 结果回传给模型，模型继续推理            |
| 返回值去哪儿了  | 成为 `result.output`             | 作为 tool result 回传给模型      |
| 首参 `RunContext` | 支持（可选）                         | 支持（可选）                    |
| `raise ModelRetry` | 支持，触发重试                        | 支持                        |

**大白话理解**：
- **普通 Tool**：工具是模型的"助手"，模型调用工具拿到结果后还会继续思考
- **Output Function**：工具是模型的"终局动作"，一旦调用就意味着任务完成

#### 实战场景：API 调用 Agent

```python
from pydantic_ai import Agent
from httpx import AsyncClient

async def call_weather_api(city: str, date: str) -> dict:
    """Fetch weather data from external API."""
    async with AsyncClient() as client:
        resp = await client.get(f'https://api.weather.com/v1/{city}/{date}')
        return resp.json()

agent = Agent('openai:gpt-4o', output_type=call_weather_api)

# 用户问："北京明天天气"
result = await agent.run('What\'s the weather in Beijing tomorrow?')
print(result.output)  # {'temp': 25, 'condition': 'sunny', ...}
```

模型的工作是**把自然语言转成 API 参数**（`city='Beijing'`, `date='2026-09-22'`），剩下的由函数完成。

#### 多个 Output Function

可以注册多个 output function，模型根据任务选择调用哪一个：

```python
def search_products(query: str, max_results: int = 10) -> list[dict]:
    """Search for products in the database."""
    ...

def create_order(product_id: str, quantity: int) -> dict:
    """Create a new order."""
    ...

agent = Agent(
    'openai:gpt-4o',
    output_type=[search_products, create_order]
)

# 用户说："搜索蓝牙耳机" → 模型调用 search_products
# 用户说："买 3 个" → 模型调用 create_order
```

#### ⚠️ 重要警告

**不要把同一个函数既注册为 output function 又注册为普通 tool**，语义会冲突：

```python
# ❌ 错误示例
@agent.tool
def process_data(data: str) -> str:
    ...

agent = Agent('openai:gpt-4o', output_type=process_data)  # 同一个函数两种角色
```

如果需要函数既能"中途调用"又能"作为最终输出"，应该**写两个独立的函数**，逻辑可以复用。

### 3.6 TextOutput —— 纯文本 + 后处理

#### 为什么需要它？

有时你希望模型输出的是**纯文本**（不是结构化数据），但你需要对文本做后处理：
- 分词、切句
- Markdown → HTML 转换
- 提取关键词
- 统计字数

这时就用 `TextOutput`——它让模型输出文本，然后自动调用你的处理函数。

#### 基本用法

```python
from pydantic_ai import Agent, TextOutput

def split_into_words(text: str) -> list[str]:
    """把文本切成单词列表"""
    return text.split()

agent = Agent('openai:gpt-4o', output_type=TextOutput(split_into_words))

result = agent.run_sync('Who was Albert Einstein?')
print(result.output)
#> ['Albert', 'Einstein', 'was', 'a', 'German-born', 'theoretical', 'physicist.']
```

**内部流程**：
1. 模型输出纯文本：`"Albert Einstein was a German-born theoretical physicist."`
2. Pydantic AI 调用 `split_into_words(text)`
3. `result.output` 是处理后的结果：`list[str]`

#### 实战场景：Markdown 转 HTML

```python
import markdown

def md_to_html(text: str) -> str:
    """Convert markdown to HTML."""
    return markdown.markdown(text)

agent = Agent(
    'openai:gpt-4o',
    output_type=TextOutput(md_to_html),
    instructions='Write a short blog post about Python typing.',
)

result = agent.run_sync('Generate the post')
print(result.output)
#> <h1>Python Typing</h1><p>Python's type system...</p>
```

#### ⚠️ 流式输出的坑

**问题**：`stream_text()` **不会**应用 `TextOutput` 里的函数，它只吐原始文本。

```python
agent = Agent('openai:gpt-4o', output_type=TextOutput(split_into_words))

async with agent.run_stream('Who was Einstein?') as result:
    async for chunk in result.stream_text():
        print(chunk)  # ❌ 是原始字符串 "Albert Einstein..."，不是 list
```

**正确做法**：用 `stream_output()` 才能拿到处理后的结果。

```python
async with agent.run_stream('Who was Einstein?') as result:
    final_output = await result.get_output()  # ✅ 是 list[str]
```

#### TextOutput vs Output Function

| 特性      | TextOutput           | Output Function     |
| ------- | -------------------- | ------------------- |
| 输入      | 模型输出的**纯文本**         | 模型输出的**结构化参数**      |
| 函数签名    | `(text: str) -> T`   | `(param1, ...) -> T` |
| 何时用     | 后处理文本                | 执行动作（SQL / API 调用）  |
| 能访问 deps | ❌ 不能                 | ✅ 可以（首参 `RunContext`）  |

大白话：`TextOutput` 只是"文本 → 文本"的管道，`Output Function` 是模型驱动的动作执行。

### 3.7 <mark style="background: #BBFABBA6;">StructuredDict —— 运行时动态 Schema</mark>

#### 什么场景需要它？

前面的 `output_type=Resume` 都是**编译时已知**的 Pydantic 模型。但有些场景下，schema 是**运行时才知道**的：
- 用户上传自己的 JSON Schema 文件
- 根据数据库表结构动态生成 schema
- 多租户系统，每个租户有不同的字段定义

这时就用 `StructuredDict`——让你用 JSON Schema 字典动态定义输出格式。

#### 基本用法

```python
from pydantic_ai import Agent, StructuredDict

# 在运行时定义 schema（可能来自文件、数据库）
HumanDict = StructuredDict(
    {
        'type': 'object',
        'properties': {
            'name': {'type': 'string'},
            'age': {'type': 'integer'},
        },
        'required': ['name', 'age'],
    },
    name='Human',
    description='A human with a name and age',
)

agent = Agent('openai:gpt-4o', output_type=HumanDict)
result = agent.run_sync('Create a person')
print(result.output)
#> {'name': 'John Doe', 'age': 30}
print(type(result.output))
#> <class 'pydantic_ai.output.StructuredDictSubclass'>
```

**关键点**：
- 返回的是 `dict[str, Any]` 的子类（可以当普通字典用）
- Schema 在运行时传给模型，告诉它该生成什么形状的 JSON

#### 实战场景：用户自定义提取字段

```python
# app/extractors.py
from pydantic_ai import Agent, StructuredDict

def create_extractor(user_schema: dict) -> Agent:
    """根据用户上传的 schema 创建提取 Agent"""
    ExtractedData = StructuredDict(
        user_schema,
        name='UserDefinedData',
        description='Extract data matching user schema',
    )
    
    return Agent(
        'openai:gpt-4o',
        output_type=ExtractedData,
        instructions='Extract data from text according to the schema.',
    )

# 用户上传的 schema
user_schema = {
    'type': 'object',
    'properties': {
        'product_name': {'type': 'string'},
        'price': {'type': 'number'},
        'in_stock': {'type': 'boolean'},
    },
    'required': ['product_name', 'price'],
}

agent = create_extractor(user_schema)
result = agent.run_sync('iPhone 15 Pro, $999, available')
print(result.output)
#> {'product_name': 'iPhone 15 Pro', 'price': 999, 'in_stock': True}
```

#### ⚠️ 重大限制：Pydantic AI 不做校验

<mark style="background: #FF5582A6;">**Pydantic AI 对 StructuredDict 不做任何校验**</mark>。

**这意味着什么？**

| 阶段         | Pydantic 模型（`BaseModel`） | StructuredDict           |
| ---------- | ------------------------ | ------------------------ |
| 发给模型       | ✅ 转成 JSON Schema 发送      | ✅ 直接发送                   |
| 模型返回后      | ✅ 用 Pydantic 校验字段类型和必填项  | ❌ **不校验**，原样返回           |
| 字段类型错误时    | 抛异常或触发 `ModelRetry`      | ❌ 静默通过，你拿到错误数据           |
| `age` 是字符串 | 自动转成 `int`（如果可以）         | ❌ 保持字符串，调用方 `age + 1` 会炸 |

**实际行为示例**：

```python
# Schema 声明 age 是 integer
HumanDict = StructuredDict({
    'type': 'object',
    'properties': {'name': {'type': 'string'}, 'age': {'type': 'integer'}},
    'required': ['name', 'age'],
})

agent = Agent('openai:gpt-4o', output_type=HumanDict)
result = agent.run_sync('Create a person')

# 如果模型返回 {"name": "Alice", "age": "25"}  ← age 是字符串
print(result.output['age'] + 1)  # ❌ TypeError: can only concatenate str
```

#### <mark style="background: #BBFABBA6;">正确做法：配 output validator 手动校验</mark>

```python
from pydantic_ai import Agent, RunContext, ModelRetry
import jsonschema

human_schema = {
    'type': 'object',
    'properties': {
        'name': {'type': 'string'},
        'age': {'type': 'integer', 'minimum': 0, 'maximum': 150},
    },
    'required': ['name', 'age'],
}

HumanDict = StructuredDict(human_schema, name='Human')
agent = Agent('openai:gpt-4o', output_type=HumanDict)

@agent.output_validator
def validate_schema(ctx: RunContext, output: dict) -> dict:
    """用 jsonschema 库手动校验"""
    try:
        jsonschema.validate(output, human_schema)
    except jsonschema.ValidationError as e:
        raise ModelRetry(f'Invalid data: {e.message}') from e
    return output
```

**为什么 Pydantic AI 不自动校验？**

因为 `StructuredDict` 是为**灵活性**设计的——运行时 schema 可能来自不受信任的源，自动校验可能引入安全风险（Schema 注入攻击）。把校验权交给开发者，让你自己决定要不要校验、怎么校验。

#### StructuredDict vs Pydantic 模型

| 特性        | Pydantic 模型              | StructuredDict         |
| --------- | ------------------------ | ---------------------- |
| Schema 来源 | 编译时已知（代码中的类定义）           | 运行时动态（文件、数据库、用户上传）     |
| 类型安全      | ✅ 静态类型检查 + 运行时校验         | ❌ 返回 `dict[str, Any]` |
| 字段校验      | ✅ 自动校验类型、必填项、取值范围        | ❌ 需手动用 `jsonschema` 校验 |
| IDE 自动补全  | ✅ `result.output.name` 有提示 | ❌ `result.output['name']` 无提示 |
| 适用场景      | 固定的业务模型（简历、订单）           | 用户自定义字段、动态表结构          |

**选型建议**：
- 80% 的场景用 Pydantic 模型（类型安全 + 自动校验）
- 只有真正需要运行时动态 schema 时才用 `StructuredDict`

### 3.8 Optional Output —— 允许"无结果"

#### 什么场景需要？

有时模型可能**合理地**无法给出结果：
- 提取任务：文档里没有目标信息
- 分类任务：输入不属于任何已知类别
- 搜索任务：没有找到匹配项

这时用 `output_type=Box | None` 允许模型返回 `None`。

#### 基本用法

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class Box(BaseModel):
    width: int
    height: int
    depth: int

agent = Agent('openai:gpt-4o', output_type=Box | None)

result = agent.run_sync('Extract box dimensions from: "The sky is blue."')
print(result.output)  # None（文本里没有尺寸信息）
```

#### 不同模式下的行为差异

| 模式                | `Box \| None` 的实现                                    | 模型如何表达"无结果"               |
| ----------------- | ----------------------------------------------------- | ------------------------ |
| **Tool Output**   | 注册两个工具：`final_result_Box` 和 `final_result_NoneType` | 显式调用 `final_result_NoneType` |
| **Native Output** | 用模型的原生 Union 支持                                      | 返回 `null`                |
| **Prompted Output** | 在提示词里说明"可以返回 null"                                   | 输出 `{"response": null}`  |

**关键区别**：

在 **Tool 模式**下，`None` 不是"空响应"，而是模型主动调用一个"表示无结果"的工具：

```python
# 内部注册的两个工具
final_result_Box(width: int, height: int, depth: int)  # 返回 Box
final_result_NoneType()  # 返回 None（无参数）
```

在 **Native / Prompted 模式**下，如果模型返回空响应（什么都不输出），会触发重试——因为这可能是模型出错了，而不是主动表达"无结果"。

#### 实战场景：带降级的提取

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class ProductInfo(BaseModel):
    name: str
    price: float
    brand: str

agent = Agent(
    'openai:gpt-4o',
    output_type=ProductInfo | None,
    instructions=(
        'Extract product information. '
        'If the text does not contain product details, return None.'
    ),
)

# 有产品信息
result = agent.run_sync('iPhone 15 Pro, $999, Apple')
print(result.output)
#> ProductInfo(name='iPhone 15 Pro', price=999.0, brand='Apple')

# 无产品信息
result = agent.run_sync('The weather is nice today.')
print(result.output)
#> None
```

#### Optional Output vs `[Box, str]`

| 写法            | 模型返回 `None` 时的语义     | 适用场景         |
| ------------- | -------------------- | ------------ |
| `Box \| None` | "没找到，确实没有"           | 提取、搜索任务      |
| `[Box, str]`  | "没找到，这是原因：{str}"（文本解释） | 需要告诉用户为什么提取失败 |

**选型建议**：
- 用 `Box | None`：调用方只需要知道"有 or 没有"
- 用 `[Box, str]`：调用方需要知道"为什么没有"

### 3.9 `@agent.output_validator` —— 自定义校验

Pydantic 只能做**形状校验**，业务规则校验（查数据库、调 API）要靠 output validator：

```python
from pydantic_ai import Agent, RunContext, ModelRetry

agent = Agent('google:gemini-3-pro-preview', deps_type=DatabaseConn, output_type=Output)

@agent.output_validator
async def validate_sql(ctx: RunContext[DatabaseConn], output: Output) -> Output:
    if isinstance(output, InvalidRequest):
        return output
    try:
        await ctx.deps.execute(f'EXPLAIN {output.sql_query}')
    except QueryError as e:
        raise ModelRetry(f'Invalid query: {e}') from e
    else:
        return output
```

要点：
- 可以是 `async` 函数
- 签名：`(ctx: RunContext[DepsT], output: T) -> T`（`ctx` 可省略）
- **必须返回 output**（可以是修改过的版本）
- `raise ModelRetry(msg)` 把 `msg` 回传给模型，触发重试
- <mark style="background: #FFF3A3A6;">重试消耗的是 run 级别的 retry 预算</mark>

重试预算可以分开配置：

```python
strict_output = Agent(
    'openai:gpt-5.2',
    retries={'tools': 5, 'output': 1},  # 工具重试 5 次，输出只给 1 次机会
)
```

### 3.10 `end_strategy` —— 结果已出但还有工具在跑

模型可能在同一个响应里既调用了 output tool 又调用了其他函数工具。怎么办？

| 策略 | 行为 | 适用 |
|------|------|------|
| `'graceful'`（默认） | 采用 output tool 结果结束 run，其余工具**不执行** | 大多数场景 |
| `'early'` | 一看到 output tool 立刻结束 | 追求最低延迟 |
| `'exhaustive'` | 所有工具都执行完再结束 | 工具有必要的副作用时 |

⚠️ **隐藏规则**：在 `graceful` / `exhaustive` 下，如果同一响应里有函数工具 `raise ModelRetry`，<mark style="background: #FF5582A6;">即使 output tool 成功了，它的结果也不会被采用</mark>——因为模型显然是在信息不全的情况下过早下结论了。

### 3.11 流式结构化输出与 `partial_output`

```python
from datetime import date
from typing import TypedDict
from typing_extensions import NotRequired
from pydantic_ai import Agent

class UserProfile(TypedDict):
    name: str
    dob: NotRequired[date]
    bio: NotRequired[str]

agent = Agent('openai:gpt-5.2', output_type=UserProfile)

async def main():
    user_input = 'My name is Ben, I was born on January 28th 1990, I like the chain the dog and the pyramid.'
    async with agent.run_stream(user_input) as result:
        async for profile in result.stream_output():
            print(profile)
            #> {'name': 'Ben'}
            #> {'name': 'Ben', 'dob': date(1990, 1, 28)}
            #> {'name': 'Ben', 'dob': date(1990, 1, 28), 'bio': 'Likes the chain the dog and the pyramid'}
```

每次 yield 的是**累积快照**，用 partial validation 保证中间态也能过校验（所以非必填字段要用 `NotRequired`）。

⚠️ **流式下 output function / output validator 会被调用多次**，有副作用时必须用 `ctx.partial_output` 保护：

```python
def save_to_database(ctx: RunContext, record: DatabaseRecord) -> DatabaseRecord:
    if ctx.partial_output:
        return record          # 还在流式过程中，跳过写库
    print(f'Saving to database: {record.name} = {record.value}')
    return record
```

### 3.12 Strict Mode

`strict=True` 让 provider 在**解码层面**保证输出符合 schema：

| Provider | 支持情况 |
|----------|---------|
| OpenAI | 支持 |
| Anthropic | 支持 |
| Google（Gemini） | 支持（VALIDATED 模式） |
| Bedrock | 支持 |
| 其他 | 静默忽略该参数 |

---

## 四、生产级模板：简历提取

```python
# app/schemas.py
from datetime import date
from pydantic import BaseModel, EmailStr, Field

class WorkExperience(BaseModel):
    company: str = Field(description='公司全称')
    title: str = Field(description='职位名称')
    start_date: date
    end_date: date | None = Field(default=None, description='至今则为 None')

class Resume(BaseModel):
    name: str
    email: EmailStr
    phone: str = Field(description='仅保留数字，无法确定填 "unknown"')
    work_experience: list[WorkExperience]

class ExtractionFailed(BaseModel):
    """无法提取时返回，说明缺了什么。"""
    reason: str
    missing_fields: list[str]
```

```python
# app/extractor.py
from pydantic_ai import Agent, RunContext, ModelRetry
from app.schemas import Resume, ExtractionFailed
from app.models_config import build_model

resume_agent = Agent(
    build_model('production'),
    output_type=[Resume, ExtractionFailed],   # 提取成功 or 明确失败
    retries={'tools': 3, 'output': 2},
    instructions=(
        '你是简历解析器。从文本中提取结构化信息。\n'
        '若关键字段（姓名、邮箱）缺失，返回 ExtractionFailed 并说明缺什么，'
        '不要编造数据。'
    ),
)

@resume_agent.output_validator
def validate_resume(ctx: RunContext, output: Resume | ExtractionFailed):
    if isinstance(output, ExtractionFailed):
        return output

    # 业务规则：工作经历时间必须合法
    for exp in output.work_experience:
        if exp.end_date and exp.end_date < exp.start_date:
            raise ModelRetry(
                f'{exp.company} 的 end_date({exp.end_date}) 早于 '
                f'start_date({exp.start_date})，请重新提取。'
            )
    return output


async def extract_resume(text: str) -> Resume | ExtractionFailed:
    result = await resume_agent.run(text)
    return result.output
```

```python
# app/api.py
from fastapi import FastAPI, HTTPException
from app.schemas import Resume, ExtractionFailed

app = FastAPI()

@app.post('/extract', response_model=Resume)     # 同一个模型直接复用
async def extract(text: str):
    output = await extract_resume(text)
    if isinstance(output, ExtractionFailed):
        raise HTTPException(422, detail={
            'reason': output.reason,
            'missing_fields': output.missing_fields,
        })
    return output
```

<mark style="background: #BBFABBA6;">设计要点</mark>：用 `[Resume, ExtractionFailed]` 而不是 `[Resume, str]`——失败路径也是结构化的，前端可以直接拿 `missing_fields` 做高亮提示。

---

## 五、常见坑点与解决方案

### 坑点 1：Union 输出的类型检查报错

**错误示例**
```python
agent = Agent('openai:gpt-5-mini', output_type=list[str] | list[int])
result = agent.run_sync('...')
result.output[0].upper()   # mypy: 报错
```

**实际行为**：类型检查器无法从 `output_type` 反推结果类型。

**正确做法**
```python
agent = Agent[object, list[str] | list[int]](
    'openai:gpt-5-mini',
    output_type=list[str] | list[int],  # type: ignore
)
```

### 坑点 2：以为 StructuredDict 会校验

**错误示例**
```python
agent = Agent('openai:gpt-5.2', output_type=HumanDict)
result = agent.run_sync('Create a person')
print(result.output['age'] + 1)   # 可能 TypeError：age 是字符串
```

**实际行为**：Pydantic AI 只把 schema 发给模型，**不校验返回值**。

**正确做法**：配一个 output validator 手动跑 `jsonschema.validate`，失败时 `raise ModelRetry`。

### 坑点 3：NativeOutput 和函数工具一起用

**错误示例**
```python
agent = Agent('google:gemini-2.0-flash', output_type=NativeOutput(Resume))

@agent.tool_plain
def lookup_company(name: str) -> str: ...
```

**实际行为**：多数模型在 native structured output 模式下**不支持同时调用函数工具**，直接报错或工具被忽略。

**正确做法**：需要函数工具就用默认的 Tool Output 模式。

### 坑点 4：`stream_text()` 吞掉了 TextOutput 函数

**错误示例**
```python
agent = Agent('openai:gpt-5.2', output_type=TextOutput(split_into_words))
async with agent.run_stream('...') as result:
    async for chunk in result.stream_text():
        print(chunk)   # 是原始字符串，不是 list[str]
```

**正确做法**：用 `stream_output()`。

### 坑点 5：流式下副作用执行了 N 次

**错误示例**
```python
def save(ctx: RunContext, record: DatabaseRecord) -> DatabaseRecord:
    db.insert(record)     # 流式过程中每个快照都会插一次
    return record
```

**正确做法**
```python
def save(ctx: RunContext, record: DatabaseRecord) -> DatabaseRecord:
    if ctx.partial_output:
        return record
    db.insert(record)
    return record
```

### 坑点 6：以为 output function 的返回值会回传给模型

**错误示例**：写了个 output function 想让模型看到结果后继续加工。

**实际行为**：<mark style="background: #FF5582A6;">output function 一被调用，run 立刻结束</mark>，返回值直接成为 `result.output`，模型看不到。

**正确做法**：需要模型继续推理就用普通 `@agent.tool`，output function 只用于"终局动作"。

### 坑点 7：模型不支持文本输出时静默失败

**实际行为**：`supports_text_output=False` 的模型（如 TypeSafe Jev）遇到 `str` / `TextOutput` / `PromptedOutput`，会在**发请求前**抛 `UserError`。

**正确做法**：换模型或换成 Tool / Native Output。

---

## 六、自测问题

1. `output_type=[Box, str]` 和 `output_type=Box | str` 在 Tool 模式下生成的工具有什么区别？
2. 为什么多个输出类型会注册成多个独立的 output tool，而不是一个带 `anyOf` 的 schema？
3. output validator 里 `raise ModelRetry` 消耗的是哪个重试预算？怎么单独配置它？
4. `end_strategy='graceful'` 下，模型同时调用了 output tool 和一个 raise 了 `ModelRetry` 的函数工具，最终会怎样？
5. 流式输出结构化数据时，为什么 TypedDict 的非必填字段要标 `NotRequired`？

<details>
<summary>参考答案</summary>

1. 前者是"两个独立工具，各自 schema 简单"；后者是裸 Union，语义上等价，但 `| None` 这类会额外暴露 `final_result_NoneType` 工具。推荐列表形式。
2. 降低单个 schema 的复杂度。模型面对一个简单 schema 的正确率远高于一个嵌套 `anyOf` 的复杂 schema。
3. 消耗 run 级别的 **output** 重试预算。用 `retries={'tools': 5, 'output': 1}` 分开配置。
4. 即使 output tool 成功，结果也**不会被采用**——因为工具报错说明模型是在信息不全时下的结论，会进入重试。
5. 流式过程中数据是逐步累积的，中间快照必然缺字段。用 `NotRequired` 才能让 partial validation 通过，否则每个中间态都校验失败。

</details>

---

## 七、与其他笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [05-Reflection-反思与自我纠错](./05-Reflection-反思与自我纠错.md) | `ModelRetry` 与重试预算的完整机制，本篇的 output validator 正是其应用场景之一 |
| [08-Model-errors-模型错误](./08-Model-errors-模型错误.md) | 重试耗尽后抛出的 `UnexpectedModelBehavior` 如何处理 |
| [09-Streaming-Events-流式事件](./09-Streaming-Events-流式事件.md) | `stream_output()` 与事件流的配合 |
| [11-Type-safe-by-design-类型安全设计](./11-Type-safe-by-design-类型安全设计.md) | `Agent[DepsT, OutputT]` 泛型参数的完整说明 |
| [13-Testing-测试最佳实践](./13-Testing-测试最佳实践.md) | `TestModel` 会按 output schema 自动编数据，`FunctionModel` 可精确构造 `ToolCallPart` 测试输出路径 |
| [14-Models-and-Providers-模型与降级策略](./14-Models-and-Providers-模型与降级策略.md) | 校验失败走重试（同模型）vs API 错误走降级（换模型）的区分 |
