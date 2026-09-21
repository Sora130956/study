# 09 Streaming Events — 什么时候需要"边运行边看"

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#streaming-events-and-final-output
> 重写日期：2026-09-21
> 学习目标：客户要实时反馈时才需要补这一章

---

## 快速判断：你需要流式吗？

三个场景，对号入座：

| 场景 | 用什么 | 为什么 |
|------|--------|--------|
| 只要最终结果（不管是文本还是结构化数据） | `run_sync()` 或 `run()` | 最简单，等结果到了一次拿到 |
| 要在 Web 界面逐字显示 ChatGPT 式的打字效果 | `run_stream()` + `stream_text()` | 前端 SSE 推送，用户体验好 |
| 要看到工具调用过程（如"正在查询数据库…"） | `run_stream_events()` | 拿到所有中间事件，自己决定显示什么 |

**重点**：如果你只是想"等结果出来"，根本不需要流式——`run_sync()` 就够了。只有当你要**在 Agent 运行期间**给用户反馈时，才需要这一章。

---

## 场景 1：边打字边显示文本（ChatGPT 效果）

### 真实需求

你做一个 Web 客服机器人，客户问"我的订单在哪？"，你希望回答像 ChatGPT 那样**逐字出现**，而不是等 3 秒后突然蹦出整段话。

### 代码实现

```python
from pydantic_ai import Agent

agent = Agent('openai:gpt-4o', system_prompt='你是客服机器人')

# 流式获取文本
async with agent.run_stream('我的订单在哪？') as run:
    async for chunk in run.stream_text():
        print(chunk, end='', flush=True)  # 逐字打印
        # 前端实现：yield f"data: {chunk}\n\n"  # SSE 格式
```

**输出效果**：

```
您好！（立即出现）
请提供您的订单号（0.5秒后）
，我帮您查询物流信息。（1秒后）
```

而不是等 1.5 秒后突然显示整段。

### 常见问题：为什么有时候流式"卡住"了？

**问题**：模型同时返回了工具调用和文本，但工具没执行，文本也不完整。

**原因**：`run_stream()` 默认认为"文本出现 = 任务完成"，会忽略后面的工具调用。

**解决**：

```python
# 方案 1：让 Agent 先把所有工具跑完再输出文本
agent = Agent('openai:gpt-4o', end_strategy='exhaustive')

# 方案 2：改用 run_stream_events()（见场景 2）
```

---

## 场景 2：显示工具调用进度（"正在查询…"）

### 真实需求

用户问"帮我订明天去北京的机票"，Agent 要调三个工具：

1. `search_flights()` — 查询航班
2. `check_price()` — 比价
3. `create_order()` — 下单

你希望前端显示：

```
🔍 正在查询航班...
💰 正在比价...
✅ 订单已创建！
```

而不是让用户干等 10 秒。

### 代码实现

```python
from pydantic_ai import Agent, FunctionToolCallEvent, FunctionToolResultEvent, AgentRunResultEvent

agent = Agent('openai:gpt-4o')

@agent.tool
async def search_flights(destination: str) -> list[str]:
    # 模拟耗时操作
    await asyncio.sleep(2)
    return ['CA1234', 'MU5678']

@agent.tool
async def check_price(flight_id: str) -> float:
    await asyncio.sleep(1)
    return 1200.0

@agent.tool
async def create_order(flight_id: str) -> str:
    await asyncio.sleep(1)
    return 'ORDER-20260921-001'

# 流式获取所有事件
async with agent.run_stream_events('帮我订明天去北京的机票') as events:
    async for event in events:
        if isinstance(event, FunctionToolCallEvent):
            tool_name = event.part.tool_name
            print(f"🔍 正在执行 {tool_name}...")
            # 前端：yield f"data: {{\"status\": \"calling\", \"tool\": \"{tool_name}\"}}\n\n"
        
        elif isinstance(event, FunctionToolResultEvent):
            print(f"✅ 工具执行完成，结果：{event.part.content}")
        
        elif isinstance(event, AgentRunResultEvent):
            print(f"📦 最终结果：{event.result.output}")
```

**输出效果**：

```
🔍 正在执行 search_flights...
✅ 工具执行完成，结果：['CA1234', 'MU5678']
🔍 正在执行 check_price...
✅ 工具执行完成，结果：1200.0
🔍 正在执行 create_order...
✅ 工具执行完成，结果：ORDER-20260921-001
📦 最终结果：已为您预订 CA1234 航班，订单号 ORDER-20260921-001
```

---

## 进阶：事件类型速查表

`run_stream_events()` 能拿到这些事件：

| 事件类型 | 含义 | 你能拿到什么 |
|----------|------|--------------|
| `PartStartEvent` | 模型开始输出一个 part（文本或工具调用） | `event.part` — 初始内容 |
| `PartDeltaEvent` | 文本逐字增量 / 工具参数增量 | `event.delta.content_delta` — 新增的文本块 |
| `FunctionToolCallEvent` | 模型决定调用工具了 | `event.part.tool_name`、`event.part.args` |
| `FunctionToolResultEvent` | 工具执行完毕 | `event.part.content` — 工具返回值 |
| `AgentRunResultEvent` | **整个 run 结束** | `event.result.output` — 最终结果 |

**关键点**：`AgentRunResultEvent` 只在流的**最后**出现一次，代表 Agent 完成了所有工作。

---

## <mark style="background: #BBFABBA6;">生产环境实战：FastAPI SSE 端点</mark>

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from pydantic_ai import Agent, FunctionToolCallEvent, PartDeltaEvent, TextPartDelta, AgentRunResultEvent

app = FastAPI()
agent = Agent('openai:gpt-4o')

@app.get("/chat")
async def chat_stream(query: str):
    async def event_generator():
        async with agent.run_stream_events(query) as events:
            async for event in events:
                if isinstance(event, PartDeltaEvent) and isinstance(event.delta, TextPartDelta):
                    # 文本增量 → SSE 推送
                    yield f"data: {event.delta.content_delta}\n\n"
                
                elif isinstance(event, FunctionToolCallEvent):
                    # 工具调用 → 发送进度提示
                    yield f"data: [调用 {event.part.tool_name}]\n\n"
                
                elif isinstance(event, AgentRunResultEvent):
                    # 结束标记
                    yield "data: [DONE]\n\n"
    
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

**前端代码**（JavaScript）：

```javascript
const eventSource = new EventSource('/chat?query=帮我查天气');
eventSource.onmessage = (e) => {
  if (e.data === '[DONE]') {
    eventSource.close();
  } else {
    document.getElementById('output').innerText += e.data;
  }
};
```

---

## 常见问题 Q&A

### Q1: `run_stream()` 和 `run_stream_events()` 有什么区别？

| 维度 | `run_stream()` | `run_stream_events()` |
|------|----------------|------------------------|
| 用途 | 只要流式文本输出 | 要看到所有中间事件（工具调用、思考过程） |
| 工具调用 | 默认可能忽略（如果文本先出现） | 总是执行到完成 |
| 流的结尾 | 文本输出完就结束 | 以 `AgentRunResultEvent` 结尾 |
| 典型场景 | ChatGPT 式打字效果 | 进度提示、调试、监控 |

### Q2: 如果我只要最终结果，为什么不用 `run_sync()`？

**完全正确！** `run_sync()` 最简单：

```python
result = agent.run_sync('查天气')
print(result.output)  # 直接拿结果
```

只有当你需要**在等待期间**给用户反馈时，才用流式。

### Q3: 流式模式会影响性能吗？

**不会**。模型本身就是流式输出的（一个 token 一个 token 生成），`run_stream()` 只是把这个过程暴露给你。不用流式反而要等所有 token 生成完才返回。

---

## 三个必记场景

| 需求 | 用什么 | 一句话总结 |
|------|--------|-----------|
| 只要结果 | `run_sync()` | 最简单，等就完了 |
| ChatGPT 打字效果 | `run_stream()` + `stream_text()` | 逐字显示，用户体验好 |
| 显示工具调用进度 | `run_stream_events()` | 拿到所有事件，自己决定显示什么 |

---

## 自测问题

1. 什么时候需要用流式？什么时候 `run_sync()` 就够了？
2. `run_stream()` 默认配置下，如果模型同时返回工具调用和文本会发生什么？
3. 如何实现"正在查询数据库…"这样的进度提示？用哪个方法？监听哪个事件？
