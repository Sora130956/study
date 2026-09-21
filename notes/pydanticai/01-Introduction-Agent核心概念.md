# 01 Introduction — Agent 核心概念【必读】

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#introduction
> 精读日期：2026-09-19
> 学习目标：理解 `output_type`、`deps_type`、tools 的关系（Smart-Data-Extractor 的基础）
>
> ⚠️ 术语提醒：当前版本文档已使用 `output_type` / `result.output`。旧版 API 的 `result_type` / `result.data` 已弃用，看到旧教程时注意替换。

## Introduction（对应原文 H2）

Agent 是 Pydantic AI 与 LLM 交互的**主要接口**。

- 某些场景下，一个 Agent 就能控制整个应用或组件；
- 也可以让**多个 Agent 相互协作**，组成更复杂的工作流。

`Agent` 类有完整的 API 文档，但概念上可以把 Agent 看作一个**容器**，装着以下组件：

| 组件 | 说明 |
|------|------|
| [Instructions](#)（指令） | 开发者写给 LLM 的一组指令 |
| [Function tool(s)](https://pydantic.dev/docs/ai/tools-toolsets/tools/) 和 [toolsets](https://pydantic.dev/docs/ai/tools-toolsets/toolsets/) | LLM 在生成响应过程中可以调用的函数（用来获取信息） |
| [Structured output type](https://pydantic.dev/docs/ai/core-concepts/output/)（结构化输出类型） | 如果指定了，LLM 在一次 run 结束时必须返回的结构化数据类型 |
| [Dependency type constraint](https://pydantic.dev/docs/ai/core-concepts/dependencies/)（依赖类型约束） | 动态指令函数、工具、输出函数在运行时都可以使用依赖 |
| [LLM model](https://pydantic.dev/docs/ai/api/models/base/) | Agent 关联的默认 LLM 模型（可选）；运行时也可以另行指定 |
| [Model Settings](#)（模型设置） | 可选的默认模型设置，用于微调请求；运行时也可以另行指定 |
| [Capabilities](https://pydantic.dev/docs/ai/capabilities/overview/) | 可复用的能力包，打包了 tools、hooks、instructions 和 model settings，用于扩展 Agent 行为 |

这些组件都可以单独配置；而 capabilities 让你能把相关行为打包成可复用单元，更容易组合、共享，并能从配置文件加载。

**类型层面**：Agent 在依赖类型和输出类型上是泛型的。例如一个需要 `Foobar` 类型依赖、产出 `list[str]` 类型输出的 Agent，其类型为 `Agent[Foobar, list[str]]`。实践中你不需要关心这个——它的意义在于 IDE 能提示你类型是否正确，并且配合静态类型检查（mypy/pyright）工作良好。

### 原文示例：roulette_wheel.py（轮盘赌 Agent）

```python
from pydantic_ai import Agent, RunContext

roulette_agent = Agent(  # (1)
    'openai:gpt-5.2',
    deps_type=int,
    output_type=bool,
    system_prompt=(
        'Use the `roulette_wheel` function to see if the '
        'customer has won based on the number they provide.'
    ),
)

@roulette_agent.tool
async def roulette_wheel(ctx: RunContext[int], square: int) -> str:  # (2)
    """check if the square is a winner"""
    return 'winner' if square == ctx.deps else 'loser'

# 运行 Agent
success_number = 18  # (3)
result = roulette_agent.run_sync('Put my money on square eighteen', deps=success_number)
print(result.output)  # (4)
#> True

result = roulette_agent.run_sync('I bet five is the winner', deps=success_number)
print(result.output)
#> False
```

**原文注释讲解**：

1. **(1)** 创建 Agent：期望 `int` 类型的依赖，产出 `bool` 类型的输出。这个 Agent 的类型是 `Agent[int, bool]`。
2. **(2)** 定义一个工具，检查押注的格子是否中奖。`RunContext` 用依赖类型 `int` 参数化；如果依赖类型写错了，会在类型检查时报错。
3. **(3)** 真实场景中这里应该用随机数，例如 `random.randint(0, 36)`。
4. **(4)** `result.output` 是布尔值，表示是否中奖。**Pydantic 负责输出校验**，其类型推导自 Agent 的 `output_type` 泛型参数，所以被推断为 `bool`。

**三要素关系总结（对应你的学习目标）**：

- `deps_type` → 运行时通过 `deps=` 传入，工具和动态指令里通过 `ctx.deps` 访问（本例：中奖号码 `18`）。
- `tools`（`@agent.tool`）→ LLM 在生成回答时可调用的函数；第一个参数是 `RunContext[deps_type]`，其余参数由 LLM 填充。
- `output_type` → run 结束时 LLM 必须返回的结构；Pydantic 校验后放在 `result.output` 上。

**实际开发经验**（Smart-Data-Extractor 映射）：你的提取 Agent 就是 `Agent[None, ExtractionResult]` 或 `Agent[ExtractionDeps, list[ExtractedRecord]]`——`output_type` 是你的 Pydantic schema（决定提取质量），`deps_type` 可传入配置对象（如目标字段列表），tools 可用于回查原文片段。提取场景务必配 `temperature=0.0`（见 06 笔记）。

---

## 自测问题

1. Agent 这个"容器"里装了哪 7 类组件？
2. `Agent[int, bool]` 这两个泛型参数分别对应构造时的哪两个参数？
3. `result.output` 的类型由什么决定？谁来校验？
