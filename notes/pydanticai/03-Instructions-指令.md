# 03 Instructions — 指令【必读】

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#instructions
> 精读日期：2026-09-19
> 学习目标：Day 1 核心，决定提取质量的关键

## Instructions（对应原文 H2）

Instructions 与 system prompts 类似。**主要区别**：当调用 `Agent.run` 等方法时显式传入了 `message_history`，历史消息中已有的 instructions **不会**被包含在发给模型的请求里——只有**当前 Agent 的 instructions** 会被包含。

选择原则（原文建议）：

- <mark style="background: #BBFABBA6;">用 **`instructions`**：当你希望发给模型的请求只包含当前 Agent 的系统提示时。</mark>
- <mark style="background: #BBFABBA6;">用 **`system_prompt`**：当你希望保留之前请求（可能由其他 Agent 发出）中使用过的系统提示时。</mark>

> <mark style="background: #BBFABBA6;">**原文推荐**：除非有特定理由用 `system_prompt`，一般**推荐用 `instructions`**。</mark>

### 三种指定时机

Instructions 和 system prompts 一样，可以在不同时机指定：

1. <mark style="background: #BBFABBA6;">**静态 instructions**：写代码时已确定，通过 `Agent` 构造函数的 `instructions` 参数定义。</mark>
2. <mark style="background: #BBFABBA6;">**动态 instructions**：依赖运行时才可知的上下文，用 `@agent.instructions` 装饰的函数定义。与动态 system prompt 不同——动态 system prompt 在存在 `message_history` 时可能被复用，而**动态 instructions 总是重新求值**。</mark>
3. <mark style="background: #BBFABBA6;">**运行时 instructions**：针对某一次特定 run 的额外指令，通过 `instructions` 参数传给各 [run 方法](#)（`run()` / `run_sync()` 等）。</mark>

### 排序与 prompt 缓存（重要行为）

<mark style="background: #BBFABBA6;">三种 instructions 可以加到同一个 Agent 上，运行时**按定义顺序追加**。每条 instruction 在内部被分类为静态（`instructions` 参数的字面字符串）或动态（`@agent.instructions` 函数、运行时 instructions、toolset instructions）。**静态的总是排在动态的前面**。</mark>

这个排序是有意为之：支持 prompt caching 的提供商（如 Anthropic、Bedrock）可以**缓存稳定的静态前缀**，把动态 instructions 留在缓存边界之外。→ 省钱关键点。

### <mark style="background: #BBFABBA6;">原文示例：instructions.py</mark>

```python
from datetime import date

from pydantic_ai import Agent, RunContext

agent = Agent(
    'openai:gpt-5.2',
    deps_type=str,  # (1)
    instructions="Use the customer's name while replying to them.",  # (2)
)

@agent.instructions  # (3)
def add_the_users_name(ctx: RunContext[str]) -> str:
    return f"The user's name is {ctx.deps}."

@agent.instructions
def add_the_date() -> str:  # (4)
    return f'The date is {date.today()}.'

result = agent.run_sync('What is the date?', deps='Frank')
print(result.output)
#> Hello Frank, the date today is 2032-01-02.
```

**原文注释讲解**：

1. **(1)** Agent 期望 `str` 类型的依赖。
2. **(2)** 创建 Agent 时定义的**静态** instruction。
3. **(3)** 通过装饰器定义的**动态** instruction，带 `RunContext`：<mark style="background: #BBFABBA6;">它在 `run_sync` 之后才被调用（不是 Agent 创建时）</mark>，所以能利用本次 run 的运行时信息（如 `deps`）。
4. **(4)** 另一个动态 instruction——<mark style="background: #BBFABBA6;">**instructions 函数不强制要求 `RunContext` 参数**。</mark>

**其他行为**：

- <mark style="background: #BBFABBA6;">动态 instruction 函数**返回空字符串时不会添加 instruction 消息**。</mark>
- Instructions 还可以来自 capabilities（`get_instructions()`）、toolsets（`get_instructions()`），或针对 Agent 依赖渲染的模板字符串（template strings）。

---

## Instruction parts（对应原文 H3）

每个来源（agent / toolset / capability）贡献自己的 instruction **part**。这些 parts 以**一个字符串**形式发给模型，之间用**空行**分隔；同时也可以在 `ModelRequestParameters.instruction_parts` 上以 `InstructionPart` 对象列表的形式单独获取。

### 命名与 id 规则

- 你声明 `name`；框架签发 `id`。
- **命名只需相对于你拥有的来源**，如 `'limits'` 而非 `'toolset:weather:limits'`——贡献它的来源会补全其余部分，所以你永远不会重复自己的身份，也不可能冒用其他来源的 key。
- 返回的 `InstructionId` 把贡献者的 `source` 与 `name`（如果有）配对，渲染时用 `:` 连接各段：

| `str(part.id)` | 指向 |
|---|---|
| `'agent'` | Agent 自己的 `instructions` |
| `'toolset:<toolset_id>'` | 带 `id` 的 toolset 贡献的所有内容 |
| `'capability:<capability_id>'` | 带 `id` 的 capability 贡献的所有内容 |
| `'agent:<name>'` | Agent 命名的某一个 part |
| `'toolset:<toolset_id>:<name>'` | 该 toolset 命名的某一个 part |
| `'capability:<capability_id>:<name>'` | 该 capability 命名的某一个 part |

### 声明 name 的两种方式

1. **函数**在注册处命名：`@agent.instructions(name=...)`、`@capability.instructions(name=...)`。
2. **字面文本**把名字带在文本上：在任何接受 instructions 的地方传一个 `InstructionPart`——`Agent(instructions=...)`、`Capability(instructions=...)`、`FunctionToolset(instructions=...)`、或 capability/toolset 的 `get_instructions()` 实现。该 part 还决定自己是否算 `dynamic`（这决定它是否被排除在可缓存前缀之外，见 prompt caching）；**part 永远保持完整，不会与相邻 part 合并**。

### 设计意图与约束

- 因为 part 的 id **跨 run 稳定**，把 instruction 配置存在别处的应用（例如让用户编辑某个 MCP server 贡献的 instructions 的 UI）可以以 id 为 key，而不必依赖 part 的位置或措辞——这两者都会随 Agent 演化而变化。
- 来源 key 是一切的基石，语义**永久保持**：以后给更多来源加 id 只能增加 key，永远不会改变已有 key 的指向。
- capability id、toolset id、instruction name **不能包含 `:`**（保留为分隔符）。名字 `'agent'` 也是保留字，因为它单独就是 Agent 自身 instructions 的 key。

### 原文示例：instruction_parts.py

```python
from pydantic_ai import Agent, RunContext
from pydantic_ai.capabilities import Capability
from pydantic_ai.models.test import TestModel

model = TestModel()
agent = Agent(
    model,
    instructions='Be concise.',
    capabilities=[Capability(instructions='Cite your sources.', id='research')],
)

@agent.instructions(name='local_time')
def local_time() -> str:
    return 'The time is 10:00.'

@agent.instructions
def user_name(ctx: RunContext[None]) -> str:
    return 'The user is Frank.'

agent.run_sync('What is the capital of Italy?')

parts = model.last_model_request_parameters.instruction_parts or []
print([(part.name, str(part.id) if part.id is not None else None, part.content) for part in parts])
"""
[
    (None, 'agent', 'Be concise.'),
    (None, 'capability:research', 'Cite your sources.'),
    ('local_time', 'agent:local_time', 'The time is 10:00.'),
    (None, None, 'The user is Frank.'),
]
"""
```

（此示例完整，可直接运行。）

### 以 id 为 key 之前的两个注意点（原文强调）

1. **key 覆盖其下的一切**：一个来源贡献了多个 parts 且都未命名时，它们都带来源 key，替换该 key 的文本会替换全部（包括计算生成的 parts）。`'agent'` 是刻意的例外：它只覆盖 Agent 构造时的字面 instructions，因此接管基础 prompt 不会悄悄吞掉注入日期或用户名的 `@agent.instructions` 函数。
2. **有些 parts 根本无法寻址**：未命名的 instructions 函数（函数自身名字不唯一，lambda 或模板字符串没有名字）、传给 `Agent(instructions=...)` 的可调用对象、某次 run 的运行时 instructions、来自无 `id` 的 toolset/capability 的内容。若需要自己的某个可调用对象可被覆盖，用 `@agent.instructions(name=...)` 注册，而不是传给构造函数。

另外：给**来源没有 `id`** 的 part 命名时，其 `id` 为 `None`（没有来源 key 可限定）——name 仍随 part 保留可见，但无法被寻址。

---

**实际开发经验**（Smart-Data-Extractor）：

- 提取规则（字段定义、格式约束、不要编造等）放**静态 instructions**——稳定、可缓存、跨 run 复用。
- 文档类型、目标语言等运行时信息用**动态 instructions** 注入。
- 单文档级别的临时要求（如"本次只提取表格数据"）用**运行时 instructions**：`agent.run_sync(prompt, instructions='...')`。
- 面试话术：instructions vs system_prompt 的区别在 message_history 场景下的行为 + 静态前缀缓存友好。

---

## 自测问题

1. 传入 `message_history` 时，instructions 和 system_prompt 的行为有何不同？
2. 三种 instructions 指定时机分别是什么？排序规则是什么，为什么这样设计？
3. 动态 instruction 函数返回空字符串会怎样？
4. instruction part 的 id 由哪几段组成？`'agent'` 这个 key 为什么是特例？
