# 06 模型设置与并发配置 — 实战指南

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#additional-configuration  
> 精读日期：2026-09-21  
> 核心场景：控制模型行为、防止 API 超限、跟踪成本

---

## 快速上手：最常用的三个配置

### 1. 控制模型输出的确定性 — `temperature`

**场景**：你在做数据提取、结构化输出，需要结果稳定、可复现。

```python
from pydantic_ai import Agent, ModelSettings

# 提取任务：每次运行结果必须一致
extract_agent = Agent(
    'openai:gpt-4o-mini',
    model_settings=ModelSettings(temperature=0.0),  # 0 = 完全确定性
)

# 创意任务：需要多样性
creative_agent = Agent(
    'openai:gpt-4o',
    model_settings=ModelSettings(temperature=0.7),  # 越高越随机
)
```

**实用规则**：
- <mark style="background: #BBFABBA6;">数据提取、JSON 生成、测试用例 → `temperature=0.0`</mark>
- <mark style="background: #BBFABBA6;">文案生成、头脑风暴 → `temperature=0.7-1.0`</mark>
- <mark style="background: #BBFABBA6;">默认值通常是 `0.7`，不设置时由模型决定</mark>

---

### 2. 防止 API 被打爆 — <mark style="background: #BBFABBA6;">`max_concurrency`</mark>

**场景**：<mark style="background: #BBFABBA6;">你要并行处理 100 个文档，但 OpenAI 免费账号只有 3 RPM（每分钟 3 次请求）。</mark>

```python
from pydantic_ai import Agent

# 方式 1：简单限流（最多 3 个并发）
agent = Agent('openai:gpt-4o-mini', max_concurrency=3)

# 方式 2：带队列深度控制（防止内存爆炸）
from pydantic_ai import ConcurrencyLimit

agent = Agent(
    'openai:gpt-4o-mini',
    max_concurrency=ConcurrencyLimit(
        max_running=3,      # 同时运行 3 个
        max_queued=100,     # 最多排队 100 个，超过直接报错
    ),
)

# 实际使用（假设要处理 100 个文档）
import asyncio

async def process_docs():
    tasks = [agent.run(f'提取文档 {i} 的关键信息') for i in range(100)]
    results = await asyncio.gather(*tasks)  # 自动限流，不会超过 3 并发
```

**什么时候用？**
- <mark style="background: #BBFABBA6;">免费/低配额账号并发处理大量任务</mark>
- 调用外部 API（防止打爆下游服务）
- 共享账号多人使用（防止互相干扰）

**不用会怎样？**
- 触发 `429 Too Many Requests` 错误
- 账号被临时封禁
- 内存占用过高（所有任务同时创建）

---

### 3. 给任务打标签方便追踪 — `metadata`

**场景**：<mark style="background: #BBFABBA6;">多租户 SaaS，需要在日志/trace 中区分是哪个客户的请求。</mark>

```python
from dataclasses import dataclass
from pydantic_ai import Agent

@dataclass
class Deps:
    tenant_id: str
    user_id: str

agent = Agent[Deps](
    'openai:gpt-4o',
    deps_type=Deps,
    metadata=lambda ctx: {
        'tenant': ctx.deps.tenant_id,  # 租户标识
        'user': ctx.deps.user_id,      # 用户标识
    },
)

# 单次运行时追加额外标签
result = agent.run_sync(
    'Query user data',
    deps=Deps(tenant_id='acme-corp', user_id='user-123'),
    metadata={'request_id': 'req-456'},  # 与 Agent 级 metadata 合并
)

print(result.metadata)
#> {'tenant': 'acme-corp', 'user': 'user-123', 'request_id': 'req-456'}
```

**实用价值**：
- 在 OpenTelemetry/Datadog 中按租户筛选 trace
- 成本归因（哪个客户消耗了多少 token）
- 调试时快速定位问题请求

---

## 进阶：三层配置的优先级（像 CSS 层叠）

Pydantic AI 的 `ModelSettings` 可以在三个地方设置，**后面的会覆盖前面的**（像 CSS 的层叠规则）：

```python
from pydantic_ai import Agent, ModelSettings
from pydantic_ai.models.openai import OpenAIChatModel

# 第 1 层：模型级（全局默认值）
model = OpenAIChatModel(
    'gpt-4o',
    settings=ModelSettings(temperature=0.8, max_tokens=500)
)

# 第 2 层：Agent 级（这个 Agent 的默认值）
agent = Agent(
    model,
    model_settings=ModelSettings(temperature=0.5)  # 覆盖模型级的 temperature
)
# 此时实际配置：temperature=0.5, max_tokens=500

# 第 3 层：运行时（单次调用覆盖）
result = agent.run_sync(
    'Query',
    model_settings=ModelSettings(temperature=0.0)  # 最终优先级最高
)
# 此时实际配置：temperature=0.0, max_tokens=500
```

**层叠规则**（和 CSS 一样）：
- 只覆盖设置了的字段（`temperature` 被覆盖了，`max_tokens` 保留）
- 越靠近运行时，优先级越高
- 没有设置的字段从上一层继承

**实用场景**：
- **模型级**：放硬约束（如 `max_tokens=4000` 防止超限）
- **Agent 级**：放任务默认值（如提取任务 `temperature=0.0`）
- **运行时**：按文档大小动态调整（如长文档 `max_tokens=8000`）

---

## 动态配置：让 Agent 自己调整策略

**场景**：第一步用确定性输出（`temperature=0`），后续步骤需要创意（`temperature=0.7`）。

```python
from pydantic_ai import Agent, ModelSettings

agent = Agent(
    'openai:gpt-4o',
    model_settings=lambda ctx: ModelSettings(
        temperature=0.0 if ctx.run_step == 1 else 0.7,  # 根据步骤动态调整
    ),
)
```

**工作原理**：
- `model_settings` 可以传**函数**而不是固定值
- **每次模型请求前**都会调用这个函数
- 函数内可以访问 `ctx.run_step`（当前步骤）、`ctx.usage`（已消耗 token）等

**更复杂的例子**（成本控制）：

```python
agent = Agent(
    'openai:gpt-4o',
    model_settings=lambda ctx: ModelSettings(
        max_tokens=2000 if ctx.usage.total_tokens < 10000 else 500,  # 快用完配额时缩小输出
    ),
)
```

**注意事项**：
- Agent 级的函数只能看到模型默认值
- 运行时的函数能看到**模型 + Agent + Capability 三层合并后**的配置
- 要清空上一层的设置，显式传 `None`（如 `{'temperature': None}`）

---

## 并发控制详解

### 基础用法

```python
from pydantic_ai import Agent, ConcurrencyLimit

# 简单版本：只限制并发数
agent = Agent('openai:gpt-4o', max_concurrency=10)

# 完整版本：限制并发 + 队列深度
agent = Agent(
    'openai:gpt-4o',
    max_concurrency=ConcurrencyLimit(
        max_running=10,   # 最多 10 个同时运行
        max_queued=100,   # 最多 100 个排队等待
    ),
)
```

### 行为说明

**达到并发上限时**：
- 新的 `agent.run()` 调用会**阻塞等待**（不会报错）
- 在 OpenTelemetry trace 中会显示 "waiting for concurrency" span

**队列满了时**：
- 抛出 `ConcurrencyLimitExceeded` 异常
- 用于防止内存爆炸（1000 个任务同时排队会占用大量内存）

**实际使用**：

```python
import asyncio
from pydantic_ai import ConcurrencyLimitExceeded

agent = Agent(
    'openai:gpt-4o',
    max_concurrency=ConcurrencyLimit(max_running=3, max_queued=5),
)

async def process_with_backpressure():
    tasks = []
    for i in range(100):
        try:
            task = agent.run(f'Process {i}')
            tasks.append(task)
        except ConcurrencyLimitExceeded:
            print(f'队列满了，等待一会儿...')
            await asyncio.sleep(1)  # 等待一些任务完成
    
    results = await asyncio.gather(*tasks)
```

---

## 模型特定设置（Gemini 安全过滤等）

**场景**：你用 Google Gemini，需要调整内容安全过滤级别。

```python
from pydantic_ai import Agent, UnexpectedModelBehavior
from pydantic_ai.models.google import GoogleModelSettings

agent = Agent('google:gemini-3-flash-preview')

try:
    result = agent.run_sync(
        '生成 5 句讽刺命运的话',
        model_settings=GoogleModelSettings(
            temperature=0.0,  # 通用设置仍然可用
            gemini_safety_settings=[  # Gemini 特有的安全设置
                {
                    'category': 'HARM_CATEGORY_HARASSMENT',
                    'threshold': 'BLOCK_LOW_AND_ABOVE',  # 低风险就阻止
                },
                {
                    'category': 'HARM_CATEGORY_HATE_SPEECH',
                    'threshold': 'BLOCK_LOW_AND_ABOVE',
                },
            ],
        ),
    )
except UnexpectedModelBehavior as e:
    print(f'内容被安全过滤器拦截：{e}')
```

**其他模型的特定设置**：
- `OpenAIModelSettings`：`response_format`（JSON mode）
- `AnthropicModelSettings`：`thinking_settings`（Claude 的思考过程控制）
- 查看对应模型的文档了解更多

---

## 成本监控：更新模型价格表

**场景**：OpenAI 刚发布 `gpt-5o-mini`，但你的 Pydantic AI 版本是 3 个月前安装的，价格表里没有这个模型。

```python
from pydantic_ai import prices

# 启动时下载最新价格表（立即更新 + 每小时自动更新）
updater = prices.update_in_background()

try:
    # 运行你的应用
    agent = Agent('openai:gpt-5o-mini')
    result = agent.run_sync('Query')
    print(f'本次请求成本：${result.usage.cost}')
finally:
    updater.stop()  # 应用退出时停止后台更新
```

**行为说明**：
- 价格表会**立即更新一次**，失败了用内置的旧价格表
- 之后**每小时自动更新**一次
- 下载失败时**保留上次成功的价格表**

**什么时候用？**
- 需要给客户出**成本报表**（按租户统计消耗）
- 用最新发布的模型（价格表还没进入你安装的版本）
- 开发期可跳过（成本估算不准确也无所谓）

---

## 生产环境最佳实践（来自 Smart-Data-Extractor）

```python
from pydantic_ai import Agent, ModelSettings, ConcurrencyLimit

# 数据提取 Agent 的推荐配置
extract_agent = Agent(
    'openai:gpt-4o-mini',
    output_type=list[ExtractedRecord],
    instructions='Extract structured data from documents',
    
    # 1. 确定性输出（提取任务必须）
    model_settings=ModelSettings(
        temperature=0.0,   # 完全确定性
        timeout=30,        # 30 秒超时（防止卡死）
    ),
    
    # 2. 重试机制（网络抖动容错）
    retries=2,
    
    # 3. 并发控制（OpenAI Tier 1 免费账号 3 RPM）
    max_concurrency=3,
    
    # 4. 成本归因（多租户场景）
    metadata=lambda ctx: {
        'tenant': ctx.deps.tenant_id,
        'document_id': ctx.deps.doc_id,
    },
)
```

**关键决策理由**：
- **`temperature=0.0`**：提取/结构化任务要可复现，不是创意写作
- **`timeout=30`**：防止某个文档卡死整个队列
- **`retries=2`**：网络抖动时自动重试，不用手动处理
- **`max_concurrency=3`**：匹配 OpenAI 免费账号限额（实测约 3 RPM）

---

## 常见问题

### Q1：为什么我的 Agent 一直报 429 错误？
**A**：并发太高，加上 `max_concurrency` 限制：

```python
agent = Agent('openai:gpt-4o', max_concurrency=3)  # 根据你的账号配额调整
```

### Q2：三层配置太复杂，我该怎么用？
**A**：按这个规则：
- **模型级**：只放**硬约束**（如 `max_tokens=4000`）
- **Agent 级**：放**任务默认值**（如提取任务 `temperature=0.0`）
- **运行时**：按**文档大小动态调整**（如长文档临时调大 `max_tokens`）

### Q3：`metadata` 和日志有什么区别？
**A**：
- **日志**：记录执行过程，开发者看
- **metadata**：标记任务属性，给**可观测性工具**（OpenTelemetry）用，可以按租户/用户筛选 trace

### Q4：价格更新失败了会怎样？
**A**：保留上次成功的价格表继续用（或回退到内置价格表），不会影响 Agent 运行。

---

## 总结：三个必记场景

| 场景 | 配置 | 代码 |
|------|------|------|
| 数据提取（要稳定） | `temperature=0.0` | `ModelSettings(temperature=0.0)` |
| 防止 API 超限 | `max_concurrency` | `Agent(..., max_concurrency=3)` |
| 多租户成本归因 | `metadata` | `metadata=lambda ctx: {'tenant': ctx.deps.tenant_id}` |

**记住这三个，就能解决 80% 的生产问题。**
