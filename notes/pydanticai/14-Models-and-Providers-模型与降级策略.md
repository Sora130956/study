# 14 Models & Providers — 模型、Provider 与降级策略【必读】

> 原文档：https://ai.pydantic.dev/models/overview/
> 相关文档：https://ai.pydantic.dev/models/openai/
> 精读日期：2026-09-21
> 学习目标：Day 2 的降级容错 + Day 4 的多模型切换 + 降低 API 成本

## 一、是什么

**大白话版本**：<mark style="background: #BBFABBA6;">Pydantic AI 不绑定任何一家大模型厂商。你写的 Agent 代码在 OpenAI、Claude、Gemini、DeepSeek、本地 Ollama 之间切换，只需要改一个字符串。而且它还内置了"主模型挂了自动换备用模型"的能力（`FallbackModel`）。</mark>

**技术定义**：Pydantic AI 用三个术语描述它和 LLM 的交互：

| 术语 | 含义 | 命名规律 | 举例 |
|-----|------|---------|-----|
| **Model** | 封装某个 LLM **API 协议**的类 | `<VendorSdk>Model` | `OpenAIChatModel`、`AnthropicModel`、`GoogleModel` |
| **Provider** | 处理**认证和连接**的类 | `<Vendor>Provider` | `OpenAIProvider`、`DeepSeekProvider`、`AzureProvider` |
| **Profile** | 描述某个模型**需要怎样构造请求**才能效果最好 | `ModelProfile` | JSON Schema 限制、是否支持 strict tool |

**三者关系**：
```
Agent('openai:gpt-5.2')
   ↓ 自动解析
Model (OpenAIChatModel)  ← 决定用什么 API 协议
   ├─ Provider (OpenAIProvider)  ← 决定连哪个地址、用什么 key
   └─ Profile (OpenAIModelProfile)  ← 决定 schema 怎么转换
```

<mark style="background: #BBFABBA6;">写 `Agent('<provider>:<model>')` 这种简写时，Pydantic AI 会自动帮你选好 Model 类、Provider 和 Profile。</mark>只有想换 Provider（比如走 Azure、走自建网关）或换 Profile 时，才需要手动实例化。

## 二、为什么需要

**实战场景**（Smart-Data-Extractor 的成本与可用性问题）：

| 问题 | 场景 | 解决手段 |
|-----|------|---------|
| **成本太高** | GPT-4o 提取一份简历 ~0.03 美元，demo 跑 100 次就 3 刀 | 换 DeepSeek / Qwen，成本降 10 倍 |
| **服务挂了** | OpenAI 半夜 529，你的 demo 视频正在录 | `FallbackModel` 自动切 Claude |
| **限流** | 批量处理 100 份简历，并发打爆 rate limit | `ConcurrencyLimitedModel` 限流 |
| **客户要私有化** | 数据不能出内网 | 换 Ollama 本地模型，一行代码搞定 |
| **输出被截断** | `finish_reason='length'`，拿到半截 JSON | 基于响应内容触发 fallback |

**核心价值**：<mark style="background: #BBFABBA6;">模型可替换性是 Portfolio 项目的硬加分项——"我的提取器支持 OpenAI / Claude / DeepSeek / 本地 Ollama，并且主模型故障时自动降级"，这句话在 Upwork 提案里的分量远大于"我用了 GPT-4"。</mark>

## 三、核心概念

### 3.1 内置支持的 Provider

**原生支持**（各有独立 Model 类）：
OpenAI、Anthropic、Gemini（Generative Language API + VertexAI 两套）、xAI、Bedrock、Cerebras、Cohere、Groq、Hugging Face、Mistral、OpenRouter、Outlines

**OpenAI 兼容**（用 `OpenAIChatModel` + 对应 Provider）：
<mark style="background: #BBFABBA6;">DeepSeek、阿里云百炼（DashScope/Qwen）、Ollama、Azure AI Foundry、MoonshotAI（Kimi）、GitHub Models、Perplexity、Fireworks AI、Together AI、Heroku、LiteLLM、Nebius、OVHcloud、SambaNova、Vercel AI Gateway</mark>

**测试专用**：`TestModel`、`FunctionModel`（见 [13-Testing](./13-Testing-测试最佳实践.md)）

> 用哪家模型就要装对应的包。装错了 Pydantic AI 会直接告诉你该 `pip install` 什么。

### 3.2 三种指定模型的写法（由简到繁）

```python
# 写法 1：字符串简写（90% 的场景用这个）
agent = Agent('openai:gpt-5.2')
agent = Agent('deepseek:deepseek-chat')
agent = Agent('anthropic:claude-sonnet-4-5')

# 写法 2：实例化 Model 类（需要传 settings 时）
from pydantic_ai.models.openai import OpenAIChatModel
model = OpenAIChatModel('gpt-5.2')
agent = Agent(model)

# 写法 3：Model + 自定义 Provider（换端点、换 key、走代理）
from pydantic_ai.providers.openai import OpenAIProvider
model = OpenAIChatModel(
    'model_name',
    provider=OpenAIProvider(
        base_url='https://your-endpoint/v1',
        api_key='your-api-key',
    ),
)
agent = Agent(model)
```

**环境变量约定**：<mark style="background: #BBFABBA6;">每家 Provider 都读 `<PROVIDER>_API_KEY`</mark>，例如 `OPENAI_API_KEY`、`DEEPSEEK_API_KEY`、`ALIBABA_API_KEY`（或 `DASHSCOPE_API_KEY`）、`MOONSHOTAI_API_KEY`。

### 3.3 常用 Provider 速查（Portfolio 相关）

| 场景 | 代码 | 环境变量 |
|-----|------|---------|
| OpenAI | `Agent('openai:gpt-5.2')` | `OPENAI_API_KEY` |
| DeepSeek（便宜） | `Agent('deepseek:deepseek-chat')` | `DEEPSEEK_API_KEY` |
| Qwen（国内） | `Agent('alibaba:qwen-max')` | `ALIBABA_API_KEY` |
| Kimi | `Agent('moonshotai:kimi-k2-0711-preview')` | `MOONSHOTAI_API_KEY` |
| 本地 Ollama | 见下方代码 | `OLLAMA_BASE_URL` |
| 任意 OpenAI 兼容端点 | `OpenAIProvider(base_url=..., api_key=...)` | — |

```python
# 本地 Ollama（完全离线、零成本，适合录 demo）
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.ollama import OllamaProvider

ollama_model = OpenAIChatModel(
    model_name='gpt-oss:20b',
    provider=OllamaProvider(base_url='http://localhost:11434/v1'),
)
agent = Agent(ollama_model, output_type=CityLocation)
result = agent.run_sync('Where were the olympics held in 2012?')
print(result.output)
#> city='London' country='United Kingdom'
```

### 3.4 自定义 HTTP 客户端（超时、重试）

```python
from httpx import AsyncClient
from pydantic_ai.providers.deepseek import DeepSeekProvider

custom_http_client = AsyncClient(timeout=30)   # 30 秒超时
model = OpenAIChatModel(
    'deepseek-chat',
    provider=DeepSeekProvider(
        api_key='your-key',
        http_client=custom_http_client,
    ),
)
```

也可以直接传厂商 SDK 的客户端：
```python
from openai import AsyncOpenAI
client = AsyncOpenAI(max_retries=3)
model = OpenAIChatModel('gpt-5.2', provider=OpenAIProvider(openai_client=client))
```

### 3.5 `ConcurrencyLimitedModel` — 并发限流

```python
from pydantic_ai import Agent, ConcurrencyLimitedModel

model = ConcurrencyLimitedModel('openai:gpt-4o', limiter=5)  # 最多 5 个并发 HTTP 请求
agent = Agent(model)

async def main():
    results = await asyncio.gather(*[agent.run(f'Question {i}') for i in range(20)])
    # 20 个任务，但同时最多只有 5 个在飞
```

**`limiter` 接受三种值**：

| 类型 | 用途 |
|-----|------|
| `int`（如 `5`） | 简单限流 |
| `ConcurrencyLimit` | 高级配置，带背压控制 |
| `ConcurrencyLimiter` | **多个模型共享同一份配额** |

```python
# 多模型共享配额（同一家 provider 的总限额）
from pydantic_ai import ConcurrencyLimiter

shared_limiter = ConcurrencyLimiter(max_running=10, name='openai-pool')
model1 = ConcurrencyLimitedModel('openai:gpt-4o', limiter=shared_limiter)
model2 = ConcurrencyLimitedModel('openai:gpt-4o-mini', limiter=shared_limiter)
# 两个模型加起来最多 10 个并发
```

> 开了 instrumentation 后，<mark style="background: #BBFABBA6;">等待并发槽位的请求会以 span 形式出现在 trace 里，`name` 参数用来在 trace 里区分不同的限流器。</mark>

### 3.6 <mark style="background: #BBFABBA6;">`FallbackModel` — 降级策略【重点】</mark>

**基本用法**：
```python
from pydantic_ai.models.fallback import FallbackModel
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.models.anthropic import AnthropicModel

openai_model = OpenAIChatModel('gpt-5.2')
anthropic_model = AnthropicModel('claude-sonnet-4-5')
fallback_model = FallbackModel(openai_model, anthropic_model)

agent = Agent(fallback_model)
response = agent.run_sync('What is the capital of France?')
print(response.data)
#> Paris
```

<mark style="background: #BBFABBA6;">按顺序尝试，第一个成功的就返回。从 `ModelResponse.model_name` 可以看出最终是谁答的。</mark>

**默认触发条件**：`ModelAPIError`（包含 `ModelHTTPError`，即 4xx/5xx）。<mark style="background: #BBFABBA6;">最常见的场景不需要任何配置。</mark>

#### 关键约束 1：校验错误**不会**触发 fallback

> 官方明确说明：<mark style="background: #BBFABBA6;">结构化输出或工具参数的**校验错误不会触发降级**，它们走的是[重试机制](./05-Reflection-反思与自我纠错.md)，即重新 prompt **同一个**模型。</mark>
>
> **设计理由**：校验错误源于 LLM 的不确定性，重试同一个模型很可能成功；而 API 错误（4xx/5xx）通常重试同一个请求也解决不了。

| 错误类型 | 处理机制 | 结果 |
|---------|---------|-----|
| 输出 JSON 不符合 schema | **重试**（`ModelRetry`） | 同一模型重来 |
| 工具参数类型错误 | **重试** | 同一模型重来 |
| 401 / 429 / 500 | **降级**（`FallbackModel`） | 换下一个模型 |

#### 关键约束 2：要关掉 SDK 自带的重试

> 官方提醒：<mark style="background: #BBFABBA6;">OpenAI、Anthropic 等 SDK 自带重试逻辑，会**拖延 `FallbackModel` 的触发**。建议设置 `max_retries=0`。</mark>

```python
from openai import AsyncOpenAI
client = AsyncOpenAI(max_retries=0)   # ✅ 关掉 SDK 重试，让 fallback 立即生效
openai_model = OpenAIChatModel('gpt-5.2', provider=OpenAIProvider(openai_client=client))
```

#### 每个模型独立配置 settings

```python
openai_model = OpenAIChatModel(
    'gpt-5.2',
    settings=ModelSettings(temperature=0.7, max_tokens=1000),   # OpenAI 用高创造性
)
anthropic_model = AnthropicModel(
    'claude-sonnet-4-5',
    settings=ModelSettings(temperature=0.2, max_tokens=1000),   # Claude 用低温度保稳定
)
fallback_model = FallbackModel(openai_model, anthropic_model)
```

<mark style="background: #BBFABBA6;">`FallbackModel` 本身没有 settings，用的是最终成功那个模型的 settings。`base_url`、`api_key`、自定义 client 也都要配在各自的模型上。</mark>

#### 全部失败时：`FallbackExceptionGroup`

```python
# Python >= 3.11
from pydantic_ai import Agent, ModelAPIError

try:
    response = agent.run_sync('What is the capital of France?')
except* ModelAPIError as exc_group:
    for exc in exc_group.exceptions:
        print(exc)
```

```python
# Python < 3.11：用 exceptiongroup 回填包
from exceptiongroup import catch
from pydantic_ai import ModelAPIError

def model_status_error_handler(exc_group: BaseExceptionGroup) -> None:
    for exc in exc_group.exceptions:
        print(exc)

with catch({ModelAPIError: model_status_error_handler}):
    response = agent.run_sync('What is the capital of France?')
```

### 3.7 基于响应内容的降级（Response-Based Fallback）

**场景**：HTTP 200 没报错，但响应内容本身就是失败的——比如输出被长度截断、内置工具执行失败。

> **仅支持非流式**：只对 `agent.run()` 和 `agent.run_sync()` 生效。流式请求（`run_stream()`）只支持异常触发的降级。

**`fallback_on` 接受的四种形式**：

| 形式 | 示例 |
|-----|------|
| 异常类型元组 | `(ModelAPIError, ModelHTTPError)` |
| 异常处理器（同步/异步） | `lambda exc: isinstance(exc, MyError)` |
| 响应处理器（同步/异步） | `def check(r: ModelResponse) -> bool` |
| 混合列表 | `[ModelAPIError, exc_handler, response_handler]` |

**类型是怎么区分的**：<mark style="background: #BBFABBA6;">通过检查第一个参数的类型标注。标注成 `ModelResponse` 就是响应处理器；其他情况（包括没标注和 lambda）都当异常处理器。</mark>

```python
from pydantic_ai.messages import FinishReason, ModelResponse
from pydantic_ai.models.fallback import FallbackModel

def bad_finish_reason(response: ModelResponse) -> bool:
    """输出被长度限制截断、被内容过滤、或出错时降级。"""
    reason: FinishReason | None = response.finish_reason
    return reason in ('length', 'content_filter', 'error')

fallback_model = FallbackModel(
    'openai:gpt-5.2',
    'anthropic:claude-sonnet-4-5',
    fallback_on=bad_finish_reason,
)
```

> ⚠️ **大坑**：<mark style="background: #BBFABBA6;">单独传一个响应处理器会**完全替换**默认的 `(ModelAPIError,)` 异常降级</mark>。也就是说上面这段代码里，4xx/5xx 错误**不再触发降级**，而是直接抛出来。
>
> 想两者都要，必须传列表：
> ```python
> fallback_on=[ModelAPIError, bad_finish_reason]
> ```

## 四、生产级模板代码

### 场景：Smart-Data-Extractor 的多模型配置层

```python
# app/models_config.py
import os
from httpx import AsyncClient
from openai import AsyncOpenAI

from pydantic_ai import ModelAPIError, ModelSettings
from pydantic_ai.messages import ModelResponse
from pydantic_ai.models.fallback import FallbackModel
from pydantic_ai.models.openai import OpenAIChatModel
from pydantic_ai.providers.openai import OpenAIProvider
from pydantic_ai.providers.deepseek import DeepSeekProvider
from pydantic_ai.providers.ollama import OllamaProvider


def _openai_model() -> OpenAIChatModel:
    """主力模型：质量最好，但最贵。关掉 SDK 重试让 fallback 立即生效。"""
    client = AsyncOpenAI(api_key=os.environ['OPENAI_API_KEY'], max_retries=0)
    return OpenAIChatModel(
        'gpt-4o',
        provider=OpenAIProvider(openai_client=client),
        settings=ModelSettings(temperature=0.0, max_tokens=4096),  # 提取任务要确定性
    )


def _deepseek_model() -> OpenAIChatModel:
    """备用模型：成本约为 GPT-4o 的 1/10。"""
    return OpenAIChatModel(
        'deepseek-chat',
        provider=DeepSeekProvider(
            api_key=os.environ['DEEPSEEK_API_KEY'],
            http_client=AsyncClient(timeout=60),   # 提取长文本给足超时
        ),
        settings=ModelSettings(temperature=0.0, max_tokens=4096),
    )


def _ollama_model() -> OpenAIChatModel:
    """最后兜底：本地模型，零成本、零网络依赖，录 demo 时用。"""
    return OpenAIChatModel(
        model_name='qwen2.5:14b',
        provider=OllamaProvider(base_url=os.getenv('OLLAMA_BASE_URL', 'http://localhost:11434/v1')),
    )


def _truncated_or_filtered(response: ModelResponse) -> bool:
    """响应虽然 200，但内容不可用时也要降级。"""
    return response.finish_reason in ('length', 'content_filter', 'error')


def build_model(profile: str = 'production'):
    """按环境返回不同的模型配置。"""
    if profile == 'local':
        return _ollama_model()

    if profile == 'cheap':
        return _deepseek_model()

    # production：三级降级链
    return FallbackModel(
        _openai_model(),
        _deepseek_model(),
        _ollama_model(),
        # 关键：列表形式同时保留 API 错误降级 + 响应内容降级
        fallback_on=[ModelAPIError, _truncated_or_filtered],
    )
```

```python
# app/extractor.py
from pydantic_ai import Agent, ConcurrencyLimitedModel
from app.models_config import build_model

# 批量提取时限制并发，避免打爆 rate limit
model = ConcurrencyLimitedModel(build_model('production'), limiter=5)

resume_agent = Agent(
    model,
    output_type=Resume,
    retries={'output': 2},
    instructions='从简历文本中提取结构化数据...',
)
```

```python
# app/api.py —— FastAPI 层的错误处理
from fastapi import HTTPException
from pydantic_ai import ModelAPIError

@app.post('/extract')
async def extract(text: str):
    try:
        result = await resume_agent.run(text)
        return result.output
    except* ModelAPIError as exc_group:          # Python 3.11+
        details = '; '.join(str(e) for e in exc_group.exceptions)
        raise HTTPException(503, f'所有模型均不可用：{details}')
```

**模板特点**：

| 设计点 | 理由 |
|-------|------|
| `max_retries=0` | 让 `FallbackModel` 立即接管，不被 SDK 重试拖延 |
| `temperature=0.0` | 数据提取要确定性，不要创造性 |
| 三级降级链 | 质量优先 → 成本优先 → 离线兜底 |
| `fallback_on=[ModelAPIError, handler]` | 列表形式，避免响应处理器覆盖默认异常降级 |
| `ConcurrencyLimitedModel` 包在最外层 | 对整条降级链统一限流 |
| `profile` 参数 | demo 用 `local`，测试用 `cheap`，线上用 `production` |

## 五、常见坑点与解决方案

### 坑点 1：以为校验失败会自动换模型

**错误认知**：
```python
# ❌ 期望：GPT 输出的 JSON 不合法 → 自动换 Claude
fallback_model = FallbackModel(openai_model, anthropic_model)
agent = Agent(fallback_model, output_type=Resume)
```

**实际行为**：JSON 校验失败走的是**重试机制**，会重新 prompt **同一个 GPT**，直到 `retries` 预算耗尽后抛 `UnexpectedModelBehavior`，全程**不会**碰 Claude。

**正确做法**：两套机制分开配置
```python
agent = Agent(
    FallbackModel(openai_model, anthropic_model),  # 管 API 故障
    output_type=Resume,
    retries={'output': 2},                          # 管校验失败
)
```

### 坑点 2：单独传响应处理器，API 错误不再降级

**错误示例**：
```python
fallback_on=bad_finish_reason   # ❌ 完全替换了默认的 ModelAPIError 降级
```
结果：OpenAI 返回 429，直接抛异常，根本不去试 Claude。

**正确做法**：
```python
fallback_on=[ModelAPIError, bad_finish_reason]   # ✅ 两种都保留
```

### 坑点 3：SDK 重试把 fallback 拖成"假死"

**现象**：OpenAI 挂了，请求卡了 40 秒才切到备用模型。

**原因**：`AsyncOpenAI` 默认 `max_retries=2`，每次重试还带指数退避。

**解决方案**：
```python
client = AsyncOpenAI(max_retries=0)   # ✅
```

### 坑点 4：把配置写在 `FallbackModel` 上

**错误示例**：
```python
# ❌ FallbackModel 没有这些参数
fallback_model = FallbackModel(model_a, model_b, api_key='...', base_url='...')
```

**正确做法**：<mark style="background: #BBFABBA6;">`base_url`、`api_key`、`settings`、自定义 client 都要配在**每个模型自己**身上。</mark>

### 坑点 5：OpenAI 兼容端点 schema 报错

**现象**：接了某个 OpenAI 兼容的第三方模型，结构化输出一直失败，提示 schema 不支持 `$defs` 或 strict 模式。

**解决方案**：手动指定 Profile
```python
from pydantic_ai import InlineDefsJsonSchemaTransformer
from pydantic_ai.profiles.openai import OpenAIModelProfile

model = OpenAIChatModel(
    'model_name',
    provider=OpenAIProvider(base_url='https://xxx/v1', api_key='...'),
    profile=OpenAIModelProfile(
        json_schema_transformer=InlineDefsJsonSchemaTransformer,  # 把 $defs 内联展开
        openai_supports_strict_tool_definition=False,             # 该模型不支持 strict
    ),
)
```

### 坑点 6：流式场景用了响应内容降级

**错误示例**：
```python
fallback_model = FallbackModel(..., fallback_on=bad_finish_reason)
async with agent.run_stream(...) as stream:   # ❌ 响应降级在流式下不生效
    ...
```

**说明**：<mark style="background: #BBFABBA6;">响应内容降级只在 `run()` / `run_sync()` 生效，流式请求只支持异常降级。</mark>流式场景要做内容降级只能自己在消费流时判断。

## 六、自测问题

1. **概念区分**：Model、Provider、Profile 三者分别负责什么？什么时候需要手动指定 Provider？

2. **降级边界**：模型输出的 JSON 不符合 `output_type`，`FallbackModel` 会换模型吗？为什么？

3. **配置陷阱**：`fallback_on=my_response_handler` 和 `fallback_on=[ModelAPIError, my_response_handler]` 有什么区别？

4. **性能调优**：为什么用 `FallbackModel` 时官方建议设 `max_retries=0`？

5. **实战应用**：为 Smart-Data-Extractor 设计一条三级降级链，说明每一级的选型理由和对应的 `ModelSettings`。

---

## 七、与其他笔记的关联

- **设置合并**：[06-Model-Settings-模型设置与并发配置](./06-Model-Settings-模型设置与并发配置.md) — 每个模型的 `settings` 如何与 Agent 级、run 级合并
- **重试 vs 降级**：[05-Reflection-反思与自我纠错](./05-Reflection-反思与自我纠错.md) — 校验错误走重试，API 错误走降级，两套机制互不干扰
- **异常体系**：[08-Model-errors-模型错误](./08-Model-errors-模型错误.md) — `ModelAPIError`、`ModelHTTPError`、`FallbackExceptionGroup` 的继承关系
- **测试替换**：[13-Testing-测试最佳实践](./13-Testing-测试最佳实践.md) — `TestModel`/`FunctionModel` 也是 `Model` 的子类，所以 `override(model=...)` 能无缝替换
- **用量控制**：[10-Usage-Limits-用量限制](./10-Usage-Limits-用量限制.md) — `ConcurrencyLimitedModel` 控并发，`UsageLimits` 控 token
- **链路追踪**：[07-Debugging-and-Monitoring-调试与监控](./07-Debugging-and-Monitoring-调试与监控.md) — 用 Logfire 观察降级实际发生在哪一步
