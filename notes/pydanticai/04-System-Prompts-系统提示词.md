# 04 System Prompts — 系统提示词【必读】

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#system-prompts
> 精读日期：2026-09-19
> 学习目标：配合 Instructions 使用，定义 Agent 行为

## System Prompts（对应原文 H2）

System prompt 乍看简单——只是字符串（或被拼接的字符串序列）——但**写出正确的 system prompt 是让模型按预期行为的关键**。

一般来说，system prompt 分两类：

1. <mark style="background: #BBFABBA6;">**静态 system prompt**：写代码时已确定，通过 `Agent` 构造函数的 `system_prompt` 参数定义。</mark>
2. <mark style="background: #BBFABBA6;">**动态 system prompt**：依赖运行时才可知的上下文，通过 `@agent.system_prompt` 装饰的函数定义。</mark>

两类可以**同时加到同一个 Agent 上**；运行时按定义顺序追加。

### 原文示例：system_prompts.py（同时使用两类）

```python
from datetime import date

from pydantic_ai import Agent, RunContext

agent = Agent(
    'openai:gpt-5.2',
    deps_type=str,  # (1)
    system_prompt="Use the customer's name while replying to them.",  # (2)
)

@agent.system_prompt  # (3)
def add_the_users_name(ctx: RunContext[str]) -> str:
    return f"The user's name is {ctx.deps}."

@agent.system_prompt
def add_the_date() -> str:  # (4)
    return f'The date is {date.today()}.'

result = agent.run_sync('What is the date?', deps='Frank')
print(result.output)
#> Hello Frank, the date today is 2032-01-02.
```

**原文注释讲解**：

1. **(1)** Agent 期望 `str` 类型的依赖。
2. **(2)** 创建 Agent 时定义的**静态** system prompt。
3. **(3)** 通过装饰器定义的**动态** system prompt，带 `RunContext`：它在 `run_sync` 之后才被调用（而非 Agent 创建时），所以能利用本次 run 的运行时信息（如依赖）。
4. **(4)** 另一个动态 system prompt——**system prompt 函数不强制要求 `RunContext` 参数**。

（此示例完整，可直接运行。）

---

## <mark style="background: #BBFABBA6;">与 Instructions 的关系（配合笔记 03）</mark>

| 维度 | `system_prompt` | `instructions` |
|------|-----------------|----------------|
| 传入 `message_history` 时 | 历史中其他 Agent 的 system prompt 会**保留**在请求里 | 只包含**当前 Agent** 的 instructions |
| 动态函数求值 | 有 `message_history` 时可能被复用 | **每次 run 都重新求值** |
| 原文建议 | 有特定理由才用 | **默认推荐** |

**实际开发经验**：

- <mark style="background: #BBFABBA6;">单 Agent 项目（如 Smart-Data-Extractor）两者效果几乎一样，但按官方推荐统一用 `instructions`，避免以后接多轮对话/多 Agent 时踩坑。</mark>
- 动态 system prompt 的典型用途：注入当前日期、用户名、租户配置、文档元数据。
- 动态函数可以是 sync 也可以是 async；返回空字符串则不产生消息（与 instructions 一致）。

---

## 自测问题

1. 静态和动态 system prompt 各自的定义方式是什么？
2. 动态 system prompt 函数什么时候被调用？为什么这很重要？
3. 同一个 Agent 上两类 prompt 混用时，顺序如何确定？
