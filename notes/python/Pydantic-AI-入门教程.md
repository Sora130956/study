# Pydantic AI 入门教程

---

**元信息**
- **目标读者**: 会 Python，用过 OpenAI 原生 API，想在项目中使用更简洁的 AI 框架
- **前置知识**: Python 基础、面向对象、类型注解、async/await、OpenAI API 基本概念（prompt、tool calling）
- **学习时长**: 60-90 分钟
- **可跳过章节**: 第 2 节（为什么需要它）可快速浏览，第 6 节（常见坑）遇到问题再查

---

## 第 1 节：它是什么（一句话 + 一句类比）

**Pydantic AI 是一个类型安全的 Python AI 框架，让你用类和装饰器来定义 Agent，而不是手写 JSON schema 和循环解析模型返回。**

**生活化类比**：就像 FastAPI 让你用 Python 类型注解自动生成 API 文档一样，Pydantic AI 让你用 Python 函数自动生成 LLM 工具调用的 schema，并自动处理调用循环——你只需写业务逻辑，框架帮你搞定和模型的"对话协议"。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 反面推演：不用它会怎样？

当你用 OpenAI 原生 API 实现工具调用时，通常要做这些事：

```python
# 1. 手写工具 schema（容易出错，改参数要同步改 schema）
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "城市名称"},
                    "date": {"type": "string", "description": "日期"}
                },
                "required": ["location"]
            }
        }
    }
]

# 2. 手写调用循环（要处理 tool_calls、function_call、finish_reason）
messages = [{"role": "user", "content": "北京明天天气怎么样？"}]
while True:
    response = client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        tools=tools
    )
    
    # 3. 手动判断是否需要调用工具
    if response.choices[0].finish_reason == "tool_calls":
        tool_calls = response.choices[0].message.tool_calls
        messages.append(response.choices[0].message)
        
        # 4. 手动执行工具并拼接返回
        for tool_call in tool_calls:
            if tool_call.function.name == "get_weather":
                args = json.loads(tool_call.function.arguments)
                result = get_weather(args["location"], args.get("date"))
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result)
                })
    else:
        break

final_output = response.choices[0].message.content
```

### 痛点清单

1. **Schema 和函数定义分离**：改一个参数要同时改 schema 和函数签名，容易不同步
2. **循环逻辑重复**：每个 Agent 都要写 while True + finish_reason 判断
3. **类型不安全**：`json.loads(tool_call.function.arguments)` 拿到的是字典，IDE 不知道有什么字段
4. **错误处理繁琐**：工具调用失败、参数验证失败都要手动处理
5. **多模型切换麻烦**：OpenAI、Anthropic、Gemini 的 API 格式不同，换模型要改代码

### 问题根源

**根源是"协议细节"和"业务逻辑"混在一起**——你既要关心怎么和模型对话（schema、消息格式、循环），又要实现真正的业务功能（查天气、查数据库）。

### 经典应用场景

- **数据提取**：从非结构化文本中提取结构化数据（简历解析、合同信息提取）
- **智能客服 Agent**：需要调用多个工具（查订单、查库存、发邮件）
- **代码助手**：需要读文件、执行命令、搜索文档
- **多轮对话系统**：需要保持上下文、管理历史消息

---

## 第 3 节：读下去之前，先搞懂这些概念（前置知识）

| 术语 | 大白话解释 | 对应正文章节 |
|------|-----------|------------|
| **Agent** | 一个配置好的 AI 助手，包含它能用的工具、指令和输出格式 | 第 4 节示例 1 |
| **Tool / 工具** | Agent 可以调用的 Python 函数，模型会根据需要决定调用哪个 | 第 4 节示例 2 |
| **RunContext** | 工具函数的上下文对象，可以拿到依赖（数据库连接、API key 等） | 第 4 节示例 2 |
| **deps / 依赖** | 运行时传给 Agent 的动态数据（用户 ID、数据库连接等），不写死在代码里 | 第 4 节示例 2 |
| **output_type** | Agent 返回的结构化数据类型（Pydantic Model、dataclass 等） | 第 4 节示例 3 |
| **Tool Calling 循环** | 模型返回 → 执行工具 → 把结果回传给模型 → 模型继续生成，直到输出最终结果 | 第 4 节所有示例 |
| **Type Hints / 类型注解** | Python 的 `def func(x: int) -> str` 语法，框架用它自动生成 schema | 贯穿全文 |
| **装饰器** | `@agent.tool` 这种语法，用来把普通函数注册为工具 | 第 4 节示例 2 |

### 术语速查表

- **Agent**：第 4 节示例 1-3，核心概念
- **Tool**：第 4 节示例 2，工具调用机制
- **RunContext**：第 4 节示例 2，依赖注入
- **output_type**：第 4 节示例 3，结构化输出

**提示**：如果你已经用过 FastAPI 或 Pydantic，这些概念会很熟悉，可以直接进第 4 节。

---

## 第 4 节：从最简单的代码示例开始（循序渐进）

### 示例 1：最小可运行版本 —— 一个不用工具的 Agent

**目标**：创建一个最简单的 Agent，体验 Pydantic AI 的基本用法。

```python
from pydantic_ai import Agent

# 1. 创建 Agent，指定模型
agent = Agent('openai:gpt-4')

# 2. 同步运行（适合脚本和测试）
result = agent.run_sync('你好，用一句话介绍 Python')

# 3. 获取输出
print(result.output)
# 输出：Python 是一种简洁易学、功能强大的编程语言。
```

#### 逐行解读

1. **`Agent('openai:gpt-4')`**
   - <mark style="background: #BBFABBA6;">创建一个 Agent 实例，绑定 OpenAI 的 GPT-4 模型</mark>
   - 模型名格式：`提供商:模型名`（支持 `openai:`、`anthropic:`、`gemini:` 等）
   - <mark style="background: #BBFABBA6;">也可以写 `Agent('anthropic:claude-opus-4')` 无需改其他代码</mark>

2. **`agent.run_sync(...)`**
   - <mark style="background: #BBFABBA6;">同步执行一次对话，返回 `RunResult` 对象</mark>
   - <mark style="background: #BBFABBA6;">参数是用户输入的 prompt</mark>
   - 内部自动处理：构造消息 → 调用模型 → 解析返回

3. **`result.output`**
   - 拿到模型的最终输出（默认是字符串）
   - <mark style="background: #BBFABBA6;">`result` 还有其他字段：`result.all_messages()`（完整对话历史）、`result.cost()`（花费）</mark>

#### 等价的原生 API 写法对照

```python
# Pydantic AI 一行
result = agent.run_sync('你好')

# 原生 API 需要
response = client.chat.completions.create(
    model='gpt-4',
    messages=[{'role': 'user', 'content': '你好'}]
)
output = response.choices[0].message.content
```

#### 这一步你得到了什么

✅ 掌握了 Agent 的创建和基本调用  
✅ 理解了 `run_sync` 返回的 `RunResult` 对象  
✅ 知道如何切换模型（只改模型名，代码不变）

---

### 示例 2：加一个真实小需求 —— 调用工具查天气

**目标**：让 Agent 能调用一个 Python 函数获取天气信息。

```python
from pydantic_ai import Agent, RunContext
from datetime import date

# 1. 创建 Agent，添加系统提示
agent = Agent(
    'openai:gpt-4',
    deps_type=str,  # 声明依赖类型是字符串（这里用 API key）
    system_prompt='你是一个天气助手，使用 get_weather 工具查询天气。'
)

# 2. 用装饰器注册工具（需要访问依赖）
@agent.tool
async def get_weather(ctx: RunContext[str], location: str, forecast_date: date) -> str:
    """
    获取指定地点和日期的天气预报。
    
    Args:
        location: 城市名称，如"北京"
        forecast_date: 日期，格式 YYYY-MM-DD
    """
    # ctx.deps 拿到运行时传入的依赖（这里是 API key）
    api_key = ctx.deps
    
    # 实际项目中这里会调用真实天气 API
    # response = requests.get(f'https://api.weather.com?key={api_key}&city={location}')
    return f'{location} 在 {forecast_date} 的天气是晴天，25°C'

# 3. 运行 Agent，传入依赖
result = agent.run_sync(
    '北京明天天气怎么样？',
    deps='your-api-key-here'  # 运行时注入依赖
)

print(result.output)
# 输出：北京明天天气是晴天，25°C
```

#### 逐段解读

**创建 Agent**
```python
agent = Agent(
    'openai:gpt-4',
    deps_type=str,  # 这行很关键！
    system_prompt='你是一个天气助手，使用 get_weather 工具查询天气。'
)
```
- **`deps_type=str`**：告诉框架"这个 Agent 运行时需要一个字符串类型的依赖"
- 为什么要声明类型？因为 Python 是动态类型，框架需要这个信息来做类型检查
- **`system_prompt`**：给模型的"人设"，影响它如何使用工具

**注册工具函数**
```python
@agent.tool
async def get_weather(ctx: RunContext[str], location: str, forecast_date: date) -> str:
    """获取指定地点和日期的天气预报..."""
    api_key = ctx.deps  # 拿到依赖
    return f'{location} 在 {forecast_date} 的天气是...'
```
- **`@agent.tool`**：装饰器，把这个函数注册为工具
- **`ctx: RunContext[str]`**：上下文对象，泛型参数 `[str]` 必须和 `deps_type` 一致
- **`location: str, forecast_date: date`**：工具参数，框架会自动从这里生成 JSON schema
- **docstring**：函数的文档字符串会被提取给模型，帮助它理解工具用途
- **`ctx.deps`**：拿到 `run_sync` 时传入的 `deps`

**执行流程（框架自动完成）**
1. 用户输入："北京明天天气怎么样？"
2. 模型判断需要调用 `get_weather` 工具
3. 模型生成参数：`{"location": "北京", "forecast_date": "2024-09-18"}`
4. 框架执行 `get_weather(ctx, "北京", date(2024, 9, 18))`
5. 框架把返回值传回模型
6. 模型生成最终回复："北京明天天气是晴天，25°C"

#### 复杂度展开：`RunContext` 的作用

**为什么不直接 `def get_weather(api_key: str, location: str) ...`？**

因为 `api_key` 不是模型应该知道的参数，它是运行时环境提供的。如果把它放在函数参数里：
- 模型会看到 `api_key` 这个参数，可能会尝试生成它（不安全）
- 每次运行要把 key 写死在代码里（不灵活）

**`RunContext` 的设计**：
- 模型只看到 `location` 和 `forecast_date`
- `api_key` 通过 `ctx.deps` 注入，模型看不到
- 可以传数据库连接、用户 session 等任何运行时依赖

#### 等价的原生 API 写法对照

```python
# Pydantic AI 自动处理
@agent.tool
async def get_weather(ctx: RunContext[str], location: str, forecast_date: date) -> str:
    """获取天气"""
    return f'{location} 天气是...'

# 原生 API 要手写
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "获取天气",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "城市名称"},
                "forecast_date": {"type": "string", "format": "date", "description": "日期"}
            },
            "required": ["location", "forecast_date"]
        }
    }
}]

# 还要手写循环判断和执行...
```

#### 这一步你得到了什么

✅ 掌握了如何用 `@agent.tool` 注册工具  
✅ 理解了 `deps_type` 和 `RunContext` 的依赖注入机制  
✅ 知道了框架如何从函数签名自动生成 schema  
✅ 明白了工具调用的完整流程（模型判断 → 执行函数 → 返回结果 → 模型继续）

---

### 示例 3：<mark style="background: #BBFABBA6;">贴近项目的样子 —— 结构化输出 + 多个工具</mark>

<mark style="background: #BBFABBA6;">**目标**：返回一个 Pydantic Model 而不是字符串，添加多个工具。</mark>

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext
from datetime import date
from typing import Literal

# 1. 定义结构化输出格式
class WeatherReport(BaseModel):
    """天气报告"""
    location: str = Field(description='城市名称')
    date: date = Field(description='日期')
    temperature: int = Field(description='温度（摄氏度）')
    condition: Literal['晴', '雨', '雪', '阴'] = Field(description='天气状况')
    suggestion: str = Field(description='出行建议')

# 2. 定义依赖类型（真实项目中可能是数据库连接）
class Dependencies:
    def __init__(self, api_key: str, user_id: int):
        self.api_key = api_key
        self.user_id = user_id

# 3. 创建 Agent，指定输出类型
agent = Agent(
    'openai:gpt-4',
    deps_type=Dependencies,
    output_type=WeatherReport,  # 关键：指定返回类型
    system_prompt='你是天气助手，查询天气并给出出行建议。'
)

# 4. 注册多个工具
@agent.tool
async def get_weather(ctx: RunContext[Dependencies], location: str, query_date: date) -> str:
    """获取实时天气数据"""
    # 实际项目中调用天气 API
    return f'{location} 在 {query_date} 是晴天，温度 25°C'

@agent.tool
async def get_user_preferences(ctx: RunContext[Dependencies]) -> str:
    """获取用户偏好设置"""
    user_id = ctx.deps.user_id
    # 实际项目中查询数据库
    return f'用户 {user_id} 喜欢户外活动'

# 5. 运行并获取结构化输出
deps = Dependencies(api_key='sk-xxx', user_id=123)
result = agent.run_sync('北京明天天气怎么样？', deps=deps)

# 6. result.output 是 WeatherReport 类型，有类型提示！
print(result.output.location)      # IDE 有自动补全
print(result.output.temperature)   # 类型是 int，不是字符串
print(result.output.suggestion)

# 输出示例：
# 北京
# 25
# 明天天气晴朗，适合户外活动，建议您多喝水防晒。
```

#### 逐段解读

**定义输出结构**
```python
class WeatherReport(BaseModel):
    location: str = Field(description='城市名称')
    temperature: int = Field(description='温度（摄氏度）')
    condition: Literal['晴', '雨', '雪', '阴']
```
- 用 Pydantic 的 `BaseModel` 定义数据结构
- **`Field(description=...)`**：这些描述会传给模型，帮助它理解每个字段的含义
- **`Literal['晴', '雨', ...]`**：限制字段只能是这几个值，模型必须遵守
- 框架会自动把这个类转成 JSON schema 给模型

**指定输出类型**
```python
agent = Agent(
    'openai:gpt-4',
    deps_type=Dependencies,
    output_type=WeatherReport,  # 告诉 Agent 返回这个类型
    system_prompt='...'
)
```
- **`output_type=WeatherReport`**：Agent 会要求模型返回符合这个结构的数据
- 如果模型返回的数据不符合（比如 temperature 是字符串），Pydantic 会报错并自动重试

**获取类型安全的输出**
```python
result = agent.run_sync('北京明天天气怎么样？', deps=deps)
print(result.output.temperature)  # IDE 知道这是 int 类型
```
- `result.output` 的类型是 `WeatherReport`，不是 `Any`
- IDE 会提供自动补全和类型检查
- 如果你写 `result.output.temperatur`（拼写错误），IDE 会报错

#### 关键变量和执行流程

**关键变量**：
- `agent`：Agent 实例，配置了模型、依赖类型、输出类型
- `deps`：运行时依赖，包含 API key 和用户 ID
- `result.output`：类型是 `WeatherReport` 的实例

**执行流程**：
1. 用户输入："北京明天天气怎么样？"
2. 模型决定调用 `get_weather` 工具
3. 框架执行工具，返回天气数据
4. 模型可能再调用 `get_user_preferences` 获取用户偏好
5. 模型根据所有信息生成 `WeatherReport` 格式的 JSON
6. 框架用 Pydantic 验证 JSON，转成 `WeatherReport` 对象
7. 如果验证失败，框架会把错误信息发给模型重试（最多 N 次）

#### 复杂度展开：结构化输出的底层原理

**模型看到的 schema**（框架自动生成）：
```json
{
  "type": "object",
  "properties": {
    "location": {"type": "string", "description": "城市名称"},
    "date": {"type": "string", "format": "date"},
    "temperature": {"type": "integer", "description": "温度（摄氏度）"},
    "condition": {"enum": ["晴", "雨", "雪", "阴"]},
    "suggestion": {"type": "string"}
  },
  "required": ["location", "date", "temperature", "condition", "suggestion"]
}
```

**模型的实际返回**（OpenAI Function Calling 格式）：
```json
{
  "location": "北京",
  "date": "2024-09-18",
  "temperature": 25,
  "condition": "晴",
  "suggestion": "明天天气晴朗，适合户外活动。"
}
```

**框架的验证和转换**：
```python
# 1. 框架拿到 JSON
raw_json = model_response['tool_calls'][0]['function']['arguments']

# 2. Pydantic 验证并转换
output = WeatherReport.model_validate_json(raw_json)

# 3. 如果失败，生成错误消息回传给模型
try:
    output = WeatherReport.model_validate_json(raw_json)
except ValidationError as e:
    # 框架会把错误发给模型重试
    error_msg = "返回数据不符合格式：temperature 必须是整数"
```

#### 这一步你得到了什么

✅ 掌握了如何定义和使用结构化输出  
✅ 理解了 `output_type` 的作用和类型安全的价值  
✅ 知道了如何注册多个工具  
✅ 明白了框架如何自动验证和重试  
✅ 学会了用 Pydantic Model 定义复杂的输出格式

---

## 第 5 节：<mark style="background: #BBFABBA6;">最小可用的实际用法（生产场景模板）</mark>

这是一个可以直接复制到项目中的完整示例，包含了实际开发中的常见模式。

```python
# weather_agent.py
import asyncio
from pydantic import BaseModel, Field
from pydantic_ai import Agent, RunContext
from datetime import date
import httpx  # 用于 HTTP 请求
from typing import Optional

# ==================== 1. 定义数据结构 ====================

class WeatherReport(BaseModel):
    """天气报告输出格式"""
    location: str
    temperature: int
    condition: str
    suggestion: str

class AppDependencies:
    """应用依赖（数据库、API 客户端等）"""
    def __init__(self, weather_api_key: str, db_connection=None):
        self.weather_api_key = weather_api_key
        self.db_connection = db_connection  # 可选：数据库连接
        self.http_client = httpx.AsyncClient()  # HTTP 客户端
    
    async def close(self):
        """清理资源"""
        await self.http_client.aclose()

# ==================== 2. 创建 Agent ====================

weather_agent = Agent(
    'openai:gpt-4',  # ← 必须：模型名
    deps_type=AppDependencies,  # ← 必须：依赖类型
    output_type=WeatherReport,  # ← 必须：输出类型
    system_prompt=(  # ← 必须：系统提示
        '你是专业的天气助手。使用 get_weather 工具查询天气，'
        '然后根据天气情况给出个性化的出行建议。'
    ),
    retries=2,  # ← 可选：工具调用失败重试次数（默认 1）
)

# ==================== 3. 注册工具 ====================

@weather_agent.tool
async def get_weather(
    ctx: RunContext[AppDependencies], 
    location: str, 
    query_date: Optional[date] = None
) -> str:
    """
    查询指定城市的天气信息。
    
    Args:
        location: 城市名称
        query_date: 查询日期，不填默认今天
    """
    # ← 必须：从 ctx.deps 获取依赖
    api_key = ctx.deps.weather_api_key
    http_client = ctx.deps.http_client
    
    # 实际项目中调用真实 API
    # response = await http_client.get(
    #     f'https://api.weather.com/v1/forecast',
    #     params={'location': location, 'date': query_date},
    #     headers={'Authorization': f'Bearer {api_key}'}
    # )
    # data = response.json()
    # return f'温度 {data["temp"]}°C，{data["condition"]}'
    
    # 这里用模拟数据
    return f'{location} 的天气是晴天，温度 25°C'

# ==================== 4. 运行 Agent ====================

async def main():
    # 初始化依赖
    deps = AppDependencies(
        weather_api_key='your-api-key',  # ← 从环境变量读取更安全
        db_connection=None  # ← 如果需要数据库，这里传连接
    )
    
    try:
        # 运行 Agent
        result = await weather_agent.run(
            '北京明天天气怎么样？',
            deps=deps
        )
        
        # 获取结构化输出
        report: WeatherReport = result.output
        print(f'城市：{report.location}')
        print(f'温度：{report.temperature}°C')
        print(f'建议：{report.suggestion}')
        
        # ← 可选：查看完整对话历史（调试用）
        # for msg in result.all_messages():
        #     print(msg)
        
        # ← 可选：查看花费
        # print(f'花费：${result.cost()}')
        
    finally:
        # ← 必须：清理资源
        await deps.close()

if __name__ == '__main__':
    asyncio.run(main())
```

### 必须的部分（不能省）

1. **创建 Agent**：`Agent(model, deps_type, output_type, system_prompt)`
2. **注册工具**：`@agent.tool` + 函数定义
3. **运行 Agent**：`agent.run(prompt, deps=...)`
4. **类型定义**：`deps_type` 和 `output_type` 对应的类

### 可选但推荐的部分

1. **资源清理**：`deps.close()` 关闭 HTTP 客户端、数据库连接
2. **错误处理**：用 `try/except` 捕获 `ModelRetry`、`UnexpectedModelBehavior` 等异常
3. **环境变量**：用 `os.getenv('WEATHER_API_KEY')` 而不是硬编码 key
4. **日志记录**：用 `logfire` 或标准库 `logging` 记录调用

### 进阶配置（属于进阶，可先跳过）

```python
# 环境变量管理
import os
api_key = os.getenv('OPENAI_API_KEY')

# 错误处理
from pydantic_ai.exceptions import ModelRetry, UnexpectedModelBehavior

try:
    result = await agent.run(prompt, deps=deps)
except ModelRetry as e:
    print(f'工具调用失败重试：{e}')
except UnexpectedModelBehavior as e:
    print(f'模型行为异常：{e}')

# 流式输出（适合聊天界面）
async with agent.run_stream(prompt, deps=deps) as response:
    async for text in response.stream_text():
        print(text, end='', flush=True)

# 多模型支持
agent_gpt4 = Agent('openai:gpt-4', ...)
agent_claude = Agent('anthropic:claude-opus-4', ...)
# 其他代码完全一样，只需改模型名
```

---

## 第 6 节：常见坑与易错点

### 坑 1：忘记传 `deps` 导致 AttributeError

**现象**：
```python
result = agent.run_sync('北京天气')  # 忘记传 deps
# 报错：AttributeError: 'NoneType' object has no attribute 'api_key'
```

**原因**：工具函数里用了 `ctx.deps.api_key`，但运行时没传 `deps` 参数。

**解决**：
```python
result = agent.run_sync('北京天气', deps=your_deps)  # 必须传
```

---

### 坑 2：`deps_type` 和 `RunContext` 泛型不一致

**现象**：
```python
agent = Agent('openai:gpt-4', deps_type=str)

@agent.tool
async def my_tool(ctx: RunContext[int], x: str) -> str:  # 这里写了 int
    return ctx.deps  # IDE 报类型错误
```

**原因**：`Agent` 声明依赖是 `str`，但工具函数期望 `int`。

**解决**：保持一致
```python
agent = Agent(..., deps_type=str)
@agent.tool
async def my_tool(ctx: RunContext[str], x: str) -> str:  # 改成 str
```

---

### 坑 3：工具函数返回类型不能被 JSON 序列化

**现象**：
```python
@agent.tool
async def get_data(ctx: RunContext[str]) -> MyCustomClass:
    return MyCustomClass()  # 自定义类
# 报错：TypeError: Object of type MyCustomClass is not JSON serializable
```

**原因**：工具返回值会被序列化成 JSON 传给模型，必须是可序列化的类型。

**解决**：返回字符串、字典或 Pydantic Model
```python
@agent.tool
async def get_data(ctx: RunContext[str]) -> str:
    obj = MyCustomClass()
    return str(obj)  # 或 obj.model_dump_json()
```

---

### 坑 4：`output_type` 字段缺少描述导致模型乱填

**现象**：
```python
class Report(BaseModel):
    title: str  # 没有 Field(description=...)
    content: str

# 模型返回的 title 是 "Untitled" 或其他奇怪的值
```

**原因**：模型不知道 `title` 应该填什么，只能瞎猜。

**解决**：所有字段都加描述
```python
class Report(BaseModel):
    title: str = Field(description='报告标题，简洁概括主题')
    content: str = Field(description='报告正文内容')
```

---

### 坑 5：异步函数和同步函数混用

**现象**：
```python
@agent.tool
async def async_tool(ctx, x: str) -> str:
    return await some_async_call()

result = agent.run_sync(...)  # 同步运行
# 可能报错：RuntimeError: This event loop is already running
```

**原因**：`run_sync` 内部会创建新事件循环，可能和外层循环冲突。

**解决**：
- 在脚本里用 `run_sync`，在 FastAPI/异步框架里用 `await agent.run(...)`
- 如果工具是异步的，尽量用异步运行

---

### 和我直觉相反的提示

**提示 1**：`@agent.tool` 装饰的函数不需要手动调用，框架会自动执行。

```python
@agent.tool
async def my_tool(...):
    return 'result'

# ❌ 错误：你不需要也不应该手动调用
# result = my_tool(ctx, 'arg')

# ✅ 正确：只需运行 Agent，框架会自动调用
result = agent.run_sync('user input', deps=...)
```

**提示 2**：工具函数的 docstring 很重要，模型会读它来决定是否调用。

```python
@agent.tool
async def get_user(ctx, user_id: int):
    """根据用户 ID 获取用户信息"""  # ← 这行模型能看到
    return {'name': 'Alice'}
```

如果 docstring 写得不清楚，模型可能不会用这个工具。

---

## 第 7 节：验证你学会了（自测题）

### 问题 1：复述概念

**题目**：用一两句话说清楚：
1. Pydantic AI 是什么？
2. 它解决了什么问题？
3. `RunContext` 的作用是什么？

**答案位置**：第 1 节、第 2 节、第 3 节术语表

---

### 问题 2：默写最小示例

**题目**：不看文档，写一个最简单的 Agent，要求：
1. 使用 GPT-4 模型
2. 有一个工具函数 `add(a: int, b: int) -> int` 计算加法
3. 用户输入 "1 + 2 等于多少？"，Agent 调用工具返回结果

**答案位置**：参考第 4 节示例 2 的结构

---

### 问题 3：变体场景

**题目**：现在要实现一个订单查询 Agent：
- 输入：用户 ID 和订单 ID
- 工具 1：`get_order(order_id: str) -> dict` 查订单详情
- 工具 2：`check_stock(product_id: str) -> bool` 查库存
- 输出：一个 `OrderReport` 对象，包含订单状态、商品名称、是否有货

要求写出：
1. `OrderReport` 的定义
2. `Agent` 的创建（包括 deps_type、output_type）
3. 两个工具函数的定义

**提示**：
- 用户 ID 应该通过 `deps` 传入，不让模型看到
- 参考第 4 节示例 3 的结构

**答案位置**：组合第 4 节示例 2 和示例 3 的模式

---

## 附录：与 OpenAI 原生 API 对比速查表

| 功能 | OpenAI 原生 API | Pydantic AI |
|------|----------------|-------------|
| **定义工具** | 手写 JSON schema | 用 Python 函数 + 类型注解 |
| **工具调用循环** | 手写 while True 循环 | 框架自动处理 |
| **结构化输出** | `response_format` + 手动解析 | `output_type` + 自动验证 |
| **类型安全** | 无，返回 `dict[str, Any]` | 有，IDE 自动补全和检查 |
| **依赖注入** | 全局变量或闭包 | `deps_type` + `RunContext` |
| **多模型支持** | 改 `model` 和请求格式 | 只改模型名字符串 |
| **错误重试** | 手写 try/except | 内置重试机制 |

---

## 下一步学习

1. **官方文档**：https://pydantic.dev/docs/ai/
2. **进阶主题**：
   - 流式输出：`agent.run_stream()`
   - 多轮对话：`messages` 参数
   - 自定义模型：实现 `Model` 接口
   - Logfire 监控：集成可观测性平台
3. **实战项目**：
   - 实现一个 SQL 查询助手
   - 构建智能客服机器人
   - 开发代码审查 Agent

---

## 总结速览

**Pydantic AI 的核心价值**：
1. **类型安全**：IDE 知道每个变量的类型，减少运行时错误
2. **自动化**：schema 生成、调用循环、输出验证全自动
3. **简洁**：用 Python 类和装饰器替代 JSON 配置
4. **模型无关**：同一套代码支持 OpenAI、Anthropic、Gemini 等

**记住这个公式**：
```
Agent = 模型 + 工具 + 系统提示 + 输出格式
工具 = Python 函数 + 类型注解 + Docstring
依赖 = 运行时数据（API key、数据库连接、用户 session）
```

**实际项目检查清单**：
- ✅ API key 从环境变量读取
- ✅ 所有 Pydantic 字段都有 `Field(description=...)`
- ✅ 工具函数有清晰的 docstring
- ✅ 用 `try/finally` 清理资源
- ✅ `deps_type` 和 `RunContext[T]` 类型一致
- ✅ 在异步环境用 `await agent.run()`，脚本用 `run_sync()`

现在你可以在项目中使用 Pydantic AI 了！
