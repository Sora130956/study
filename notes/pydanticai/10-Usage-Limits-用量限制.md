# 10 Usage Limits — 如何防止一次 Agent 调用烧光预算

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#usage-limits
> 重写日期：2026-09-21
> 学习目标：生产环境必备的成本保险

---

## <mark style="background: #BBFABBA6;">快速上手：最常用的三个限额</mark>

不想看长篇大论？记住这三个：

```python
from pydantic_ai import Agent, UsageLimits
from decimal import Decimal

agent = Agent('anthropic:claude-sonnet-4-6')

result = agent.run_sync(
    '帮我写一篇 10 万字的小说',
    usage_limits=UsageLimits(
        request_limit=10,              # 最多 10 次请求（防工具死循环）
        output_tokens_limit=4000,      # 最多输出 4000 token（防超长输出）
        cost_limit=Decimal('0.05'),    # 最多花 5 美分（直接控制成本）
    ),
)
```

**如果超限**：<mark style="background: #BBFABBA6;">抛出 `UsageLimitExceeded` 异常，你可以捕获它、记日志、跳过这条数据。
</mark>
---

## 场景 1：防止工具死循环（`request_limit`）

### 真实问题

你写了个数据提取 Agent，注册了一个 `retry_extraction()` 工具，让模型"提取失败时重试"。结果模型卡在循环里：

```
调用 retry_extraction → 失败 → 再调 retry_extraction → 失败 → ...
```

10 分钟后你收到 OpenAI 的账单：$127.34。

### 解决方案

```python
from pydantic_ai import Agent, UsageLimits, UsageLimitExceeded

agent = Agent('openai:gpt-4o')

@agent.tool
def retry_extraction(data: str) -> dict:
    # 假设这个工具有 bug，总是让模型重试
    raise ModelRetry('提取失败，请重试')

try:
    result = agent.run_sync(
        '提取这份 PDF 的数据',
        usage_limits=UsageLimits(request_limit=5),  # 最多 5 次请求
    )
except UsageLimitExceeded as e:
    print(f"超限了：{e}")
    # 输出：The next request would exceed the request_limit of 5.
    # 记日志、发警报、跳过这条数据
```

**什么时候会检查**：在发送**下一次请求前**检查，所以超限的请求**不会被发送**（不计费）。

---

## 场景 2：防止超长输出烧钱（`output_tokens_limit`）

### 真实问题

你让模型"用一句话总结"，结果它写了 5000 字。Output token 的价格是 input 的 3 倍，你这一次调用花了 $2.15。

### 解决方案

```python
from pydantic_ai import Agent, UsageLimits, UsageLimitExceeded

agent = Agent('anthropic:claude-sonnet-4-6')

# 正常情况：回答很短，不超限
result = agent.run_sync(
    '意大利的首都是哪？只说城市名',
    usage_limits=UsageLimits(output_tokens_limit=10),
)
print(result.output)  # 罗马
print(result.usage)   # output_tokens=1

# 异常情况：模型写太多，抛异常
try:
    result = agent.run_sync(
        '意大利的首都是哪？写一段话介绍',
        usage_limits=UsageLimits(output_tokens_limit=10),
    )
except UsageLimitExceeded as e:
    print(e)
    # Exceeded the output_tokens_limit of 10 (output_tokens=32).
```

**什么时候会检查**：在**响应返回后**检查（因为输出多少 token 只有生成完才知道），所以超限的请求**已经计费了**。

---

## 场景 3：直接控制成本（`cost_limit`）

### 为什么 token 限额不够用？

同样 1000 token：

- 在 GPT-3.5 上花 $0.001
- 在 Claude Opus 上花 $0.015
- 在 GPT-4o 上花 $0.03

你给 GPT-3.5 调的 `output_tokens_limit=5000` 在 GPT-4o 上可能导致单次调用花费 $0.15（你的整月预算）。

### 解决方案：直接限制美元

```python
from decimal import Decimal
from pydantic_ai import Agent, UsageLimits, UsageLimitExceeded

agent = Agent('anthropic:claude-sonnet-4-6')

try:
    result = agent.run_sync(
        '意大利的首都是哪？',
        usage_limits=UsageLimits(cost_limit=Decimal('0.0001')),  # 最多 0.01 美分
    )
except UsageLimitExceeded as e:
    print(e)
    # Exceeded the `cost_limit` of 0.0001 (`usage.cost`=Decimal('0.000201')).
```

**什么时候会检查**：

- 默认在**响应返回后**检查（此时已计费）
- 设置 `count_tokens_before_request=True` 可以**请求前**预估输入成本，如果输入成本就超限则提前拒绝

```python
result = agent.run_sync(
    prompt,
    usage_limits=UsageLimits(
        cost_limit=Decimal('0.05'),
        count_tokens_before_request=True,  # 请求前先预估输入成本
    ),
)
```

---

## 进阶：限制单次请求的上下文大小（`per_request_input_tokens_limit`）

### 真实问题

你的 Agent 用了 prompt caching（提示词缓存），前 10 次请求都命中缓存，input token 很便宜。第 11 次请求时上下文太长（20000 token），缓存失效，单次请求花了 $0.18。

### 解决方案

```python
from pydantic_ai import Agent, UsageLimits, UsageLimitExceeded

agent = Agent('anthropic:claude-sonnet-4-6')

try:
    result = agent.run_sync(
        prompt,
        usage_limits=UsageLimits(
            per_request_input_tokens_limit=10000,  # 单次请求最多 10k 输入
        ),
    )
except UsageLimitExceeded as e:
    print(e)
    # Exceeded the per_request_input_tokens_limit of 10000 (request_input_tokens=20000).
```

**与 `input_tokens_limit` 的区别**：

- `input_tokens_limit` — 限制**整个 run 的累计输入**（所有请求加起来）
- `per_request_input_tokens_limit` — 限制**单次请求的输入大小**

---

## 进阶：限制工具调用次数（`tool_calls_limit`）

### 真实问题

你的 Agent 注册了 10 个工具，某次运行时模型疯狂调工具（一共调了 47 次），虽然每次调用很快，但累计花费巨大。

### 解决方案

```python
from pydantic_ai import Agent, UsageLimits, UsageLimitExceeded

agent = Agent('anthropic:claude-sonnet-4-6')

@agent.tool_plain
def do_work() -> str:
    return 'ok'

try:
    result = agent.run_sync(
        '请调用工具两次',
        usage_limits=UsageLimits(tool_calls_limit=1),  # 最多调 1 次工具
    )
except UsageLimitExceeded as e:
    print(e)
    # The next tool call(s) would exceed the tool_calls_limit of 1 (tool_calls=2).
```

**注意**：限制的是**成功执行的工具调用数**，不包括失败重试的次数。

---

## 生产环境最佳实践

### 示例 1：批处理文档提取（三重保险）

```python
from pydantic_ai import Agent, UsageLimits, UsageLimitExceeded
from decimal import Decimal

agent = Agent('openai:gpt-4o')

documents = load_documents()  # 100 份文档

for doc in documents:
    try:
        result = agent.run_sync(
            f'提取这份文档的数据：{doc.content}',
            usage_limits=UsageLimits(
                request_limit=10,               # 防工具死循环
                output_tokens_limit=4000,       # 防超长输出
                cost_limit=Decimal('0.05'),     # 单文档成本硬顶：5 美分
            ),
        )
        save_result(doc.id, result.output)
    
    except UsageLimitExceeded as e:
        # 超限了：跳过这份文档，记入失败清单
        log_failure(doc.id, str(e))
        continue  # 不要中断整个批次
```

### 示例 2：用户输入的 Agent（防恶意输入）

```python
from pydantic_ai import Agent, UsageLimits
from decimal import Decimal

agent = Agent('anthropic:claude-sonnet-4-6')

@app.post("/api/chat")
async def chat(user_input: str):
    # 用户可能输入 "写一篇 10 万字的小说"
    result = agent.run_sync(
        user_input,
        usage_limits=UsageLimits(
            request_limit=5,                  # 防恶意循环
            output_tokens_limit=1000,         # 免费用户只能生成 1000 token
            cost_limit=Decimal('0.01'),       # 单次对话最多 1 美分
        ),
    )
    return {"response": result.output}
```

### 示例 3：从工具内部读取剩余预算

```python
from pydantic_ai import Agent, RunContext

agent = Agent('openai:gpt-4o')

@agent.tool
def expensive_operation(ctx: RunContext) -> str:
    # 检查剩余预算
    remaining_cost = ctx.usage_limits.cost_limit - ctx.usage.cost
    
    if remaining_cost < Decimal('0.01'):
        # 预算不够了，提前返回简化结果
        return '预算不足，返回简化结果'
    
    # 执行耗时操作
    return do_expensive_work()
```

**关键点**：`ctx.usage_limits` 是只读的，反映的是本次 run 正在强制执行的限额；`ctx.usage` 是到目前为止的用量。

---

## 常见问题 Q&A

### Q1: 超限后能拿到部分结果吗？

**不能**。`UsageLimitExceeded` 是异常，`result` 不存在。如果你想保存部分结果，需要在工具内部或事件流里自己记录。

### Q2: 限额参数在什么时候检查？

| 限额参数 | 检查时机 | 超限的请求会计费吗？ |
|----------|----------|---------------------|
| `request_limit` | 下次请求**前** | ❌ 不会（提前拒绝） |
| `tool_calls_limit` | 工具调用**前** | ❌ 不会 |
| `output_tokens_limit` | 响应**后** | ✅ 会（已经生成完了） |
| `input_tokens_limit` | 响应**后** | ✅ 会 |
| `per_request_input_tokens_limit` | 默认响应**后** | ✅ 会；但设 `count_tokens_before_request=True` 可改成请求前 |
| `cost_limit` | 响应**后** | ✅ 会；但设 `count_tokens_before_request=True` 可提前拒绝（如果输入成本就超限） |

### Q3: `count_tokens_before_request=True` 会影响性能吗？

**会**。需要在请求前跑一遍 tokenizer 计数，增加 50-200ms 延迟。生产环境权衡：

- ✅ 用：对成本非常敏感（如免费用户），宁可慢一点也要提前拒绝
- ❌ 不用：对延迟敏感（如实时对话），依赖响应后检查 + 异常处理

---

## 六个限额参数汇总表

| 参数 | 限制对象 | 典型值 | 用来防什么 |
|------|----------|--------|-----------|
| `request_limit` | 模型请求次数 | 5-10 | 工具死循环 |
| `tool_calls_limit` | 工具调用次数 | 20-50 | 疯狂调工具 |
| `output_tokens_limit` | 累计输出 token | 2000-5000 | 超长输出 |
| `input_tokens_limit` | 累计输入 token | 50000-100000 | 上下文爆炸 |
| `per_request_input_tokens_limit` | 单次请求输入 token | 10000-30000 | 缓存失效导致的高成本 |
| `cost_limit` | run 成本（USD） | `Decimal('0.01')` ~ `Decimal('0.10')` | 直接控制花费 |

---

## 三个必记场景

| 场景 | 用哪个限额 | 一句话总结 |
|------|-----------|-----------|
| 批处理几百份文档 | `cost_limit` + `request_limit` | 单文档成本硬顶 + 防死循环 |
| 用户输入的 Agent | `output_tokens_limit` + `cost_limit` | 防恶意输入烧钱 |
| 调试工具调用 | `tool_calls_limit` | 防疯狂调工具 |

---

## 自测问题

1. `UsageLimits` 能限制哪六个维度的用量？各自防什么问题？
2. 默认情况下 `per_request_input_tokens_limit` 超限的请求会不会被发送计费？如何改成发送前拦截？
3. 为什么 `cost_limit` 比 `output_tokens_limit` 更适合生产环境？
4. 批处理 100 份文档时，如何保证一份文档超限不会中断整个批次？
