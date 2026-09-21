# 13 Unit testing — 测试最佳实践【必读】

> 原文档：https://ai.pydantic.dev/testing/
> 相关 API：https://ai.pydantic.dev/api/models/test/ 、https://ai.pydantic.dev/api/models/function/
> 精读日期：2026-09-21
> 学习目标：Day 5 的测试方案（**替代原计划的 respx mock HTTP 方案**）

## 一、是什么

**大白话版本**：<mark style="background: #BBFABBA6;">给 Agent 写测试时，你不想每跑一次测试就烧一次 API 的钱，也不想因为模型这次心情不好输出变了导致测试红了。Pydantic AI 提供了两个"假模型"——`TestModel` 和 `FunctionModel`——它们完全离线运行，不发任何网络请求，但能让你的 Agent 代码正常跑完整个流程。</mark>

**技术定义**：Pydantic AI 的单元测试策略由 4 个部件组成：

| 部件                               | 作用      | 一句话说明                      |
| -------------------------------- | ------- | -------------------------- |
| **`TestModel`**                  | 假模型（自动） | 按 JSON Schema 自动编造能通过校验的数据 |
| **`FunctionModel`**              | 假模型（手动） | 你写一个函数，完全控制模型每一步返回什么       |
| **`Agent.override()`**           | 替换器     | 在不改业务代码的前提下把真模型换成假模型       |
| **`ALLOW_MODEL_REQUESTS=False`** | 保险丝     | 全局禁止真实 API 请求，防止漏网之鱼烧钱     |

**官方推荐的测试技术栈**：
- [`pytest`](https://docs.pytest.org/en/stable/) — 测试框架
- [`inline-snapshot`](https://15r10nk.github.io/inline-snapshot/latest/) — 断言太长时自动生成快照
- [`dirty-equals`](https://dirty-equals.helpmanual.io/latest/) — 比较大型数据结构（如 `IsNow()`、`IsStr()`）

## 二、为什么需要

**实战场景**（Smart-Data-Extractor 的 `/extract` 接口测试）：

```python
# 你要测的业务代码
async def extract_resume(text: str, db: DatabaseConn) -> dict:
    result = await resume_agent.run(text, deps=...)
    await db.save(result.output)
    return result.output
```

**如果用真实模型测试**：

| 问题 | 后果 |
|-----|------|
| 每次跑测试都调 OpenAI | CI 每天跑 50 次 → 烧钱 |
| 模型输出不确定 | 今天测试过了，明天同样代码红了 |
| 网络延迟 | 一个测试 3 秒，20 个测试 1 分钟 |
| 需要 API Key | CI 环境要配密钥，新人 clone 下来跑不了 |

**如果用 respx mock HTTP**（原计划方案）：
- 要手写 OpenAI 的响应 JSON（几十行嵌套结构）
- OpenAI 改了响应格式 → 你的 mock 全部失效
- <mark style="background: #BBFABBA6;">本质上你在 mock 框架的内部实现，而不是 mock 你的依赖</mark>

**用 `TestModel`**：
```python
with resume_agent.override(model=TestModel()):
    result = await extract_resume('张三，13800138000...', conn)
# 0 秒、0 成本、100% 确定性
```

**核心价值**：<mark style="background: #BBFABBA6;">在"模型边界"上做替换，而不是在"HTTP 边界"上做替换。你的工具调用逻辑、重试逻辑、输出校验逻辑全部真实执行，只有模型本身是假的。</mark>

## 三、核心概念

### 3.1 `TestModel` — 自动生成数据

**默认行为**（不传任何参数时）：
1. <mark style="background: #BBFABBA6;">调用 Agent 上注册的**所有**工具</mark>
2. <mark style="background: #BBFABBA6;">工具参数按 JSON Schema **自动编造**</mark>（`str` → `'a'`，`date` → `'2024-01-01'`，`int` → `0`）
3. 最后返回：
   - 有 `output_type` → 调用 output tool，参数同样自动编造
   - 无 `output_type`（纯文本）且无工具调用 → 返回固定字符串 `'success (no tool calls)'`
   - 无 `output_type` 但有工具返回 → 把所有工具的返回值打包成 JSON 字符串

> **`TestModel` 不是魔法**
> 官方原文强调：里面**没有任何 ML/AI**，就是一段普通的 Python 代码，按 JSON Schema 生成"能通过 Pydantic 校验"的数据。
> 生成的数据**不好看也不相关**（比如 `location='a'`），但大多数情况下能过校验。想要更真实的数据，请用 `FunctionModel`。

<mark style="background: #BBFABBA6;">**常用构造参数**：
</mark>

| 参数                   | 类型                     | 作用                                           |
| -------------------- | ---------------------- | -------------------------------------------- |
| `call_tools`         | `list[str]` \| `'all'` | 指定只调哪些工具，默认 `'all'`                          |
| `custom_output_text` | `str \| None`          | 直接指定最终文本输出                                   |
| `custom_output_args` | `Any \| None`          | 直接指定传给 output tool 的参数（**结构化输出测试的关键**）       |
| `seed`               | `int`                  | 随机数据的种子，默认 `0`                               |
| `model_name`         | `str`                  | 模型名，默认 `'test'`（断言 `model_name='test'` 时用得上） |

**可读属性**：
- `last_model_request_parameters` — 最后一次请求时的 `ModelRequestParameters`，<mark style="background: #BBFABBA6;">可以用来断言"Agent 到底把哪些工具暴露给了模型"</mark>

```python
m = TestModel()
with my_agent.override(model=m):
    await my_agent.run('...')
assert m.last_model_request_parameters.function_tools == []
```

**两个约束**（源码里的 assert）：
- `custom_output_text` 和 `custom_output_args` **不能同时设置**
- `output_type` 强制走 tool 模式时，不能设 `custom_output_text`

### 3.2 `FunctionModel` — 手动控制每一步

**定位**：<mark style="background: #BBFABBA6;">比 `TestModel` 更强的控制力，用于 `TestModel` 做不到的高级测试。</mark>

**核心签名**：
```python
def my_model_func(messages: list[ModelMessage], info: AgentInfo) -> ModelResponse:
    ...
FunctionModel(my_model_func)
```

- `messages` — 到目前为止的完整消息历史（**用 `len(messages)` 判断这是第几轮**）
- `info: AgentInfo` — 这一轮 Agent 暴露了什么：

| `AgentInfo` 字段 | 含义 |
|----------------|------|
| `function_tools` | 通过 `@agent.tool` / `@agent.tool_plain` 注册的工具定义 |
| `output_tools` | 用于产出最终输出的工具定义 |
| `allow_text_output` | 是否允许纯文本输出 |
| `model_settings` | run 时传入的模型设置 |
| `instructions` | 传给模型的 instructions |

**支持 async 和 sync 两种函数**，也支持传 `stream_function` 来测试流式场景。

### 3.3 <mark style="background: #BBFABBA6;">`Agent.override()` — 不改业务代码做替换</mark>

```python
with weather_agent.override(model=TestModel()):
    await run_weather_forecast(...)   # 调用你的真实业务函数
```

**关键点**：
- <mark style="background: #BBFABBA6;">`override` 是上下文管理器，作用域内所有 `run()` 都用替换后的模型</mark>
- 不需要把 agent 作为参数传来传去，也不需要改造业务代码的签名
- 除了 `model`，还能覆盖 `deps`、`toolsets`

### 3.4 <mark style="background: #BBFABBA6;">`ALLOW_MODEL_REQUESTS=False` — 全局保险丝</mark>

```python
from pydantic_ai import models
models.ALLOW_MODEL_REQUESTS = False
```

<mark style="background: #BBFABBA6;">放在 `conftest.py` 或测试文件顶部，任何试图请求真实模型的代码都会直接报错。</mark>防止某个测试忘了 `override` 导致偷偷调用真实 API。

### 3.5 <mark style="background: #BBFABBA6;">`capture_run_messages()` — 抓取完整消息流</mark>

```python
with capture_run_messages() as messages:
    with weather_agent.override(model=TestModel()):
        await run_weather_forecast(...)

assert messages == [...]  # 完整的请求/响应序列 
```

用于断言 <mark style="background: #BBFABBA6;">"模型被调了几次、工具被调了什么参数、instructions 是否正确注入"</mark>。

## 四、官方示例（带中文注释）

### 被测业务代码

```python
# weather_app.py
import asyncio
from datetime import date
from pydantic_ai import Agent, RunContext
from fake_database import DatabaseConn
from weather_service import WeatherService

weather_agent = Agent(
    'openai:gpt-5.2',
    deps_type=WeatherService,
    instructions='Providing a weather forecast at the locations the user provides.',
)

@weather_agent.tool
def weather_forecast(
    ctx: RunContext[WeatherService], location: str, forecast_date: date
) -> str:
    # 注意这个分支：过去的日期走历史接口，未来的日期走预报接口
    if forecast_date < date.today():
        return ctx.deps.get_historic_weather(location, forecast_date)
    else:
        return ctx.deps.get_forecast(location, forecast_date)

async def run_weather_forecast(user_prompts: list[tuple[str, int]], conn: DatabaseConn):
    """这就是我们要测的函数，连同它内部使用的 agent 一起测。"""
    async with WeatherService() as weather_service:
        async def run_forecast(prompt: str, user_id: int):
            result = await weather_agent.run(prompt, deps=weather_service)
            await conn.store_forecast(user_id, result.output)
        await asyncio.gather(
            *(run_forecast(prompt, user_id) for (prompt, user_id) in user_prompts)
        )
```

**关键诉求**（官方原文加粗）：<mark style="background: #BBFABBA6;">我们想测这段代码，但**不想 mock 任何对象，也不想为了塞测试对象去改造代码结构**。</mark>

### 用 `TestModel` 测试

```python
# test_weather_app.py
from datetime import timezone
import pytest
from dirty_equals import IsNow, IsStr
from pydantic_ai import models, capture_run_messages, RequestUsage
from pydantic_ai.models.test import TestModel
from pydantic_ai import (
    ModelResponse, TextPart, ToolCallPart,
    ToolReturnPart, UserPromptPart, ModelRequest,
)
from fake_database import DatabaseConn
from weather_app import run_weather_forecast, weather_agent

pytestmark = pytest.mark.anyio        # 整个文件的测试都用 anyio 跑异步
models.ALLOW_MODEL_REQUESTS = False   # 保险丝：禁止任何真实模型请求

async def test_forecast():
    conn = DatabaseConn()
    user_id = 1
    with capture_run_messages() as messages:          # 抓取消息流
        with weather_agent.override(model=TestModel()):  # 替换模型
            prompt = 'What will the weather be like in London on 2024-11-28?'
            await run_weather_forecast([(prompt, user_id)], conn)  # 跑真实业务函数

    forecast = await conn.get_forecast(user_id)
    assert forecast == '{"weather_forecast":"Sunny with a chance of rain"}'

    assert messages == [
        ModelRequest(
            parts=[UserPromptPart(
                content='What will the weather be like in London on 2024-11-28?',
                timestamp=IsNow(tz=timezone.utc),   # dirty-equals：不比较具体时间
            )],
            instructions='Providing a weather forecast at the locations the user provides.',
            timestamp=IsNow(tz=timezone.utc),
            run_id=IsStr(),
        ),
        ModelResponse(
            parts=[ToolCallPart(
                tool_name='weather_forecast',
                args={
                    'location': 'a',              # TestModel 编造的 str
                    'forecast_date': '2024-01-01' # TestModel 编造的 date
                },
                tool_call_id=IsStr(),
            )],
            usage=RequestUsage(input_tokens=60, output_tokens=7),
            model_name='test',
            timestamp=IsNow(tz=timezone.utc),
            run_id=IsStr(),
        ),
        ModelRequest(
            parts=[ToolReturnPart(
                tool_name='weather_forecast',
                content='Sunny with a chance of rain',
                tool_call_id=IsStr(),
                timestamp=IsNow(tz=timezone.utc),
            )],
            instructions='Providing a weather forecast at the locations the user provides.',
            timestamp=IsNow(tz=timezone.utc),
            run_id=IsStr(),
        ),
        ModelResponse(
            parts=[TextPart(content='{"weather_forecast":"Sunny with a chance of rain"}')],
            usage=RequestUsage(input_tokens=66, output_tokens=16),
            model_name='test',
            timestamp=IsNow(tz=timezone.utc),
            run_id=IsStr(),
        ),
    ]
```

**执行流程分解**：

1. **第 1 轮**：`TestModel` 看到 Agent 注册了 `weather_forecast` 工具 → 自动调用它，参数按 schema 编造（`location='a'`、`forecast_date='2024-01-01'`）
2. **工具真实执行**：`weather_forecast` 是你的真实代码，`2024-01-01 < today` → 走 `get_historic_weather` 分支 → 返回 `'Sunny with a chance of rain'`
3. **第 2 轮**：`TestModel` 发现已经有工具返回了，且 Agent 允许文本输出 → 把工具返回打包成 JSON 字符串 `'{"weather_forecast":"Sunny with a chance of rain"}'`
4. **业务代码继续**：`run_weather_forecast` 把结果存进 `conn`

### 用 `FunctionModel` 覆盖漏掉的分支

**问题**：上面的测试里，`get_forecast`（未来日期分支）**从来没被执行过**，因为 `TestModel` 编的日期永远是 `2024-01-01`（过去）。

```python
# test_weather_app2.py
import re
import pytest
from pydantic_ai import models
from pydantic_ai import ModelMessage, ModelResponse, TextPart, ToolCallPart
from pydantic_ai.models.function import AgentInfo, FunctionModel
from fake_database import DatabaseConn
from weather_app import run_weather_forecast, weather_agent

pytestmark = pytest.mark.anyio
models.ALLOW_MODEL_REQUESTS = False

def call_weather_forecast(messages: list[ModelMessage], info: AgentInfo) -> ModelResponse:
    if len(messages) == 1:
        # 第 1 轮：从用户 prompt 里提取日期，用真实日期调工具
        user_prompt = messages[0].parts[-1]
        m = re.search(r'\d{4}-\d{2}-\d{2}', user_prompt.content)
        assert m is not None
        args = {'location': 'London', 'forecast_date': m.group()}  # 未来日期
        return ModelResponse(parts=[ToolCallPart('weather_forecast', args)])
    else:
        # 第 2 轮：工具返回了，包装成最终文本
        msg = messages[-1].parts[0]
        assert msg.part_kind == 'tool-return'
        return ModelResponse(parts=[TextPart(f'The forecast is: {msg.content}')])

async def test_forecast_future():
    conn = DatabaseConn()
    user_id = 1
    with weather_agent.override(model=FunctionModel(call_weather_forecast)):
        prompt = 'What will the weather be like in London on 2032-01-01?'
        await run_weather_forecast([(prompt, user_id)], conn)

    forecast = await conn.get_forecast(user_id)
    assert forecast == 'The forecast is: Rainy with a chance of sun'
```

**关键点**：<mark style="background: #BBFABBA6;">`len(messages) == 1` 判断"这是第几轮"，是 `FunctionModel` 里最常用的分支逻辑。</mark>

### 用 pytest fixture 复用

```python
# test_agent.py
import pytest
from pydantic_ai.models.test import TestModel
from weather_app import weather_agent

@pytest.fixture
def override_weather_agent():
    with weather_agent.override(model=TestModel()):
        yield          # yield 之前进入 override，测试结束后自动退出

async def test_forecast(override_weather_agent: None):
    ...  # 测试代码，模型已经被替换
```

## 五、生产级模板代码

### 场景：Smart-Data-Extractor 的测试套件

```python
# conftest.py —— 全局配置
import pytest
from pydantic_ai import models
from pydantic_ai.models.test import TestModel

# 保险丝：整个测试套件禁止真实 API 请求
models.ALLOW_MODEL_REQUESTS = False

@pytest.fixture
def anyio_backend():
    return 'asyncio'

@pytest.fixture
def fake_resume_text() -> str:
    return """张三
    手机：13800138000
    邮箱：zhangsan@example.com
    工作经历：
    2020.03 - 2023.06  字节跳动  后端工程师
    """
```

```python
# test_extractor.py
import pytest
from pydantic_ai import models, capture_run_messages
from pydantic_ai.models.test import TestModel
from pydantic_ai.models.function import FunctionModel, AgentInfo
from pydantic_ai import ModelMessage, ModelResponse, ToolCallPart

from app.extractor import resume_agent, extract_resume

pytestmark = pytest.mark.anyio


# ---------- 冒烟测试：流程能跑通 ----------
async def test_pipeline_runs(fake_resume_text):
    """只验证整条链路不报错，不关心数据内容。"""
    with resume_agent.override(model=TestModel()):
        result = await extract_resume(fake_resume_text)
    assert result is not None


# ---------- 断言工具暴露：Agent 注册了预期的工具 ----------
async def test_tools_registered(fake_resume_text):
    m = TestModel()
    with resume_agent.override(model=m):
        await extract_resume(fake_resume_text)

    tool_names = {t.name for t in m.last_model_request_parameters.function_tools}
    assert tool_names == {'normalize_phone', 'parse_date_range'}


# ---------- 精确控制输出：验证后处理逻辑 ----------
async def test_output_postprocessing(fake_resume_text):
    """用 custom_output_args 指定模型"返回"什么，测试下游处理。"""
    m = TestModel(custom_output_args={
        'name': '张三',
        'phone': '13800138000',
        'email': 'zhangsan@example.com',
        'work_experience': [{
            'company': '字节跳动',
            'position': '后端工程师',
            'start_date': '2020-03',
            'end_date': '2023-06',
            'description': '负责后端服务',
        }],
    })
    with resume_agent.override(model=m):
        result = await extract_resume(fake_resume_text)

    assert result.name == '张三'
    assert len(result.work_experience) == 1
    assert result.work_experience[0].company == '字节跳动'


# ---------- 测试重试逻辑：FunctionModel 模拟"先错后对" ----------
def retry_then_succeed(messages: list[ModelMessage], info: AgentInfo) -> ModelResponse:
    output_tool = info.output_tools[0]
    if len(messages) == 1:
        # 第 1 轮：故意返回非法手机号，触发 field_validator 重试
        args = {'name': '张三', 'phone': '不详', 'email': 'a@b.com', 'work_experience': []}
    else:
        # 第 2 轮：修正后的数据
        args = {'name': '张三', 'phone': '13800138000', 'email': 'a@b.com', 'work_experience': []}
    return ModelResponse(parts=[ToolCallPart(output_tool.name, args)])


async def test_retry_on_invalid_phone(fake_resume_text):
    """验证 05 笔记里的重试机制真的生效。"""
    with capture_run_messages() as messages:
        with resume_agent.override(model=FunctionModel(retry_then_succeed)):
            result = await extract_resume(fake_resume_text)

    assert result.phone == '13800138000'
    # 消息流里应该出现一次 RetryPromptPart
    retry_count = sum(
        1 for msg in messages
        for part in getattr(msg, 'parts', [])
        if part.__class__.__name__ == 'RetryPromptPart'
    )
    assert retry_count == 1
```

**模板特点**：

| 测试类型 | 用什么 | 验证什么 |
|---------|-------|---------|
| 冒烟测试 | `TestModel()` | 链路跑通不报错 |
| 工具注册 | `TestModel().last_model_request_parameters` | Agent 配置正确 |
| 后处理逻辑 | `TestModel(custom_output_args=...)` | 拿到数据后的业务处理 |
| 重试机制 | `FunctionModel` + `capture_run_messages` | 校验器和重试真的生效 |

## 六、常见坑点与解决方案

### 坑点 1：`TestModel` 的自动数据只走到一个分支

**场景**（官方示例就踩了这个坑）：工具内部有 `if date < today` 分支，`TestModel` 永远生成 `'2024-01-01'`，导致另一个分支的代码**从未被测试覆盖**。

**解决方案**：
```python
# ❌ 只能覆盖历史分支
with agent.override(model=TestModel()):
    ...

# ✅ 用 FunctionModel 精确控制参数，覆盖未来分支
def future_date_call(messages, info):
    return ModelResponse(parts=[ToolCallPart(
        'weather_forecast', {'location': 'London', 'forecast_date': '2032-01-01'}
    )])

with agent.override(model=FunctionModel(future_date_call)):
    ...
```

<mark style="background: #BBFABBA6;">经验法则：`TestModel` 做冒烟测试，`FunctionModel` 做分支覆盖。</mark>

### 坑点 2：忘了设 `ALLOW_MODEL_REQUESTS=False` 导致 CI 偷偷烧钱

**错误示例**：
```python
# test_foo.py
async def test_something():
    result = await my_agent.run('...')   # ❌ 忘了 override，真实调用 OpenAI
```

**正确做法**：在 `conftest.py` 里全局关闭：
```python
# conftest.py
from pydantic_ai import models
models.ALLOW_MODEL_REQUESTS = False   # ✅ 任何漏网之鱼都会直接报错
```

### 坑点 3：`custom_output_text` 与结构化输出冲突

**错误示例**：
```python
agent = Agent('openai:gpt-4o', output_type=Resume)   # 强制 tool 输出模式

with agent.override(model=TestModel(custom_output_text='随便一段文本')):
    ...
# ❌ AssertionError: Plain response not allowed, but `custom_output_text` is set.
```

**正确做法**：
```python
# 有 output_type 时用 custom_output_args
TestModel(custom_output_args={'name': '张三', 'phone': '13800138000', ...})

# 没有 output_type（纯文本）时才用 custom_output_text
TestModel(custom_output_text='hello')
```

同理，<mark style="background: #BBFABBA6;">两者不能同时设置</mark>，会触发 `Cannot set both custom_output_text and custom_output_args`。

### 坑点 4：断言时间戳导致测试不稳定

**错误示例**：
```python
assert messages[0].timestamp == datetime(2026, 9, 21, 10, 30, 0)  # ❌ 永远对不上
```

**正确做法**：用 `dirty-equals`：
```python
from dirty_equals import IsNow, IsStr
from datetime import timezone

assert messages[0].timestamp == IsNow(tz=timezone.utc)  # ✅ 只断言"是刚刚"
assert messages[0].run_id == IsStr()                    # ✅ 只断言"是个字符串"
```

断言太长写不动时，用 `inline-snapshot` 让它自动生成。

### 坑点 5：`FunctionModel` 里忘了处理"第二轮"导致死循环

**错误示例**：
```python
def bad_func(messages, info):
    # ❌ 每轮都返回工具调用 → 模型永远不产出最终输出
    return ModelResponse(parts=[ToolCallPart('my_tool', {})])
```

**正确做法**：必须按 `len(messages)` 分支，最后一轮返回 `TextPart` 或 output tool 调用：
```python
def good_func(messages, info):
    if len(messages) == 1:
        return ModelResponse(parts=[ToolCallPart('my_tool', {...})])
    return ModelResponse(parts=[TextPart('final answer')])   # ✅ 有终止条件
```

## 七、自测问题

1. **基础理解**：`TestModel` 和 `FunctionModel` 的分工是什么？各自适合什么场景？

2. **API 细节**：`TestModel(custom_output_text=...)` 和 `TestModel(custom_output_args=...)` 分别在什么情况下用？为什么不能同时设置？

3. **覆盖率**：为什么官方示例里 `WeatherService.get_forecast` 用 `TestModel` 测不到？怎么解决？

4. **安全防护**：`ALLOW_MODEL_REQUESTS = False` 应该放在哪个文件？它能防止什么问题？

5. **实战应用**：在 Smart-Data-Extractor 里，如何用 `FunctionModel` 验证"模型第一次输出非法手机号、第二次纠正"的重试流程？

---

## 八、与其他笔记的关联

- **重试测试**：[05-Reflection-反思与自我纠错](./05-Reflection-反思与自我纠错.md) — 用 `FunctionModel` 构造"先错后对"来验证 `ModelRetry` 逻辑
- **结构化输出**：[15-Structured-Output-结构化输出完整指南](./15-Structured-Output-结构化输出完整指南.md) — `custom_output_args` 的数据必须符合 `output_type` 的 schema
- **模型替换**：[14-Models-and-Providers-模型与降级策略](./14-Models-and-Providers-模型与降级策略.md) — `override(model=...)` 与 `FallbackModel` 是同一套模型抽象
- **消息结构**：[12-Runs-vs-Conversations-运行与会话](./12-Runs-vs-Conversations-运行与会话.md) — 理解 `capture_run_messages()` 抓到的 `ModelRequest`/`ModelResponse` 结构
- **错误断言**：[08-Model-errors-模型错误](./08-Model-errors-模型错误.md) — 用 `pytest.raises(UnexpectedModelBehavior)` 测试预算耗尽场景
