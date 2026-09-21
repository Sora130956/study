# 02 Running Agents — 运行 Agent【必读】

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#running-agents
> 精读日期：2026-09-19
> 学习目标：项目需要 `run_sync()`（CLI）和 `run()`（API）两种实现
>
> 说明：原文本节（H2）下还有 Streaming、Custom Events、iter()、Cancelling、Additional Configuration 等 H3 子章节；流式部分见笔记 09，配置部分见笔记 06。本笔记聚焦"五种运行方式"这一核心。

## Running Agents（对应原文 H2）

运行 Agent 有**五种方式**：

1. **`agent.run()`** — 异步函数，返回包含完整响应的 [`RunResult`](https://pydantic.dev/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult)。
2. **`agent.run_sync()`** — 普通的同步函数，返回 `RunResult`（内部实现就是 `loop.run_until_complete(self.run())`）。
3. **`agent.run_stream()`** — 异步上下文管理器，返回 [`StreamedRunResult`](https://pydantic.dev/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.StreamedRunResult)，其中的方法可以以异步迭代器方式流式输出文本和结构化输出。另有同步变体 **`agent.run_stream_sync()`**，返回 `StreamedRunResultSync`，提供同一组方法的同步版本。
4. **`agent.run_stream_events()`** — 异步上下文管理器，产出一个异步迭代器，逐个产出 [`AgentStreamEvent`](https://pydantic.dev/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)，**最后一个事件是 `AgentRunResultEvent`**，内含本次 run 的最终结果。
5. **`agent.iter()`** — 上下文管理器，返回 [`AgentRun`](https://pydantic.dev/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun)，可异步迭代 Agent 底层 [`Graph`](https://pydantic.dev/docs/ai/api/pydantic_graph/graph_builder/) 的节点（用于深度控制执行流程）。

### 原文示例：run_agent.py（演示前四种方式）

```python
from pydantic_ai import Agent, AgentRunResultEvent, AgentStreamEvent

agent = Agent('openai:gpt-5.2')

result_sync = agent.run_sync('What is the capital of Italy?')
print(result_sync.output)
#> The capital of Italy is Rome.

async def main():
    result = await agent.run('What is the capital of France?')
    print(result.output)
    #> The capital of France is Paris.

    async with agent.run_stream('What is the capital of the UK?') as response:
        async for text in response.stream_text():
            print(text)
            #> The capital of
            #> The capital of the UK is
            #> The capital of the UK is London.

    collected: list[AgentStreamEvent | AgentRunResultEvent] = []
    async with agent.run_stream_events('What is the capital of Mexico?') as events:
        async for event in events:
            collected.append(event)
    print(collected)
    """
    [
        PartStartEvent(index=0, part=TextPart(content='The capital of ')),
        FinalResultEvent(tool_name=None, tool_call_id=None),
        PartDeltaEvent(index=0, delta=TextPartDelta(content_delta='Mexico is Mexico ')),
        PartDeltaEvent(index=0, delta=TextPartDelta(content_delta='City.')),
        PartEndEvent(
            index=0, part=TextPart(content='The capital of Mexico is Mexico City.')
        ),
        AgentRunResultEvent(
            result=AgentRunResult(output='The capital of Mexico is Mexico City.')
        ),
    ]
    """
```

（运行此示例：确保导入 `asyncio` 并加上 `asyncio.run(main())`，无需其他改动。）

**关键观察**：
- `stream_text()` 默认产出的是**累计文本**（每段快照越来越长），不是增量；要增量需用 `stream_text(delta=True)`。
- `run_stream_events()` 的事件流以 `AgentRunResultEvent` 结尾，里面带着完整 `AgentRunResult`。

### 继续对话

可以把上一次 run 的消息传入下一次，实现多轮对话或提供上下文：

```python
result2 = agent.run_sync(
    'What was his most famous equation?',
    message_history=result1.new_messages(),  # 传入上一轮的新消息
)
```

详见 [Messages and Chat History](https://pydantic.dev/docs/ai/core-concepts/message-history/) 与笔记 12（Runs vs. Conversations）。

---

**实际开发经验**（对应你的 CLI + API 需求）：

| 场景 | 选用 | 理由 |
|------|------|------|
| CLI 批处理脚本 | `run_sync()` | 同步函数直接调用，无需 asyncio 样板代码 |
| FastAPI 接口 | `await run()` | async 端点里绝不能调 `run_sync()`——它会阻塞事件循环 |
| 给客户实时反馈（SSE） | `run_stream()` | 配 `stream_text(delta=True)` 逐块推送 |
| 需要过程可观测（工具调用进度） | `run_stream_events()` | 能拿到工具调用/结果事件 |

⚠️ 常见坑：`run_sync()` 不能在**已在运行的**事件循环里调用（如 Jupyter、FastAPI 端点内），会抛 `RuntimeError`。CLI 入口用 `run_sync()`，异步上下文一律 `await run()`。

---

## 自测问题

1. 五种运行方式分别是什么？哪两种返回完整结果、哪两种是流式、哪一种是图迭代？
2. `run_sync()` 的内部实现是什么？为什么不能在 FastAPI 端点里用？
3. `run_stream_events()` 产出的事件流以什么事件结尾？
