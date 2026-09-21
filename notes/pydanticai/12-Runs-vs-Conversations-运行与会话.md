# 12 Runs vs. Conversations — 一次调用 vs 多轮对话

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#runs-vs-conversations
> 重写日期：2026-09-21
> 学习目标：做多轮对话产品时才需要

---

## 快速判断：你需要多轮对话吗？

| 场景 | run 次数 | 要不要传 `message_history` |
|------|---------|---------------------------|
| 数据提取（一份文档提取一次） | 1 次 | ❌ 不需要 |
| 批处理（100 份文档各提取一次，互不相关） | 100 次（独立） | ❌ 不需要 |
| 聊天机器人（用户连续问 3 个问题，后面的问题依赖前面的回答） | 3 次（相关） | ✅ 需要 |

**核心区别**：

- **单 run**：一次 `run_sync()` 或 `run()` 解决问题，结束后不需要记住这次对话
- **多 run 会话**：多次 run 之间要**记住历史**（如"他是谁？"里的"他"指代前面提到的人）

---

## 场景：多轮对话（聊天机器人）

### 真实需求

用户和客服机器人对话：

```
用户：爱因斯坦是谁？
机器人：爱因斯坦是德国理论物理学家，提出了相对论。

用户：他最著名的公式是什么？  ← "他"指代爱因斯坦
机器人：E = mc²
```

**问题**：第二次对话时，如果不传历史消息，模型不知道"他"指的是谁。

### 代码实现

```python
from pydantic_ai import Agent

agent = Agent('openai:gpt-4o')

# 第一次 run
result1 = agent.run_sync('爱因斯坦是谁？')
print(result1.output)
# 输出：爱因斯坦是德国理论物理学家，提出了相对论。

# 第二次 run：传入之前的消息历史
result2 = agent.run_sync(
    '他最著名的公式是什么？',
    message_history=result1.new_messages(),  # ← 关键：传历史消息
)
print(result2.output)
# 输出：E = mc²
```

**如果不传 `message_history`**：

```python
# ❌ 错误示范：不传历史
result2 = agent.run_sync('他最著名的公式是什么？')
print(result2.output)
# 输出：请问"他"指的是谁？
```

模型不知道上下文，无法回答。

---

## 关键 API：`new_messages()` vs `all_messages()`

### `new_messages()` — 只拿本轮新增的消息

```python
result1 = agent.run_sync('爱因斯坦是谁？')
print(result1.new_messages())
# 输出：
# [
#   UserPrompt(content='爱因斯坦是谁？'),
#   ModelResponse(content='爱因斯坦是德国理论物理学家...'),
# ]
```

**用途**：追加到你自己维护的会话历史列表。

### `all_messages()` — 拿完整历史（包括传入的）

```python
result1 = agent.run_sync('爱因斯坦是谁？')
result2 = agent.run_sync('他最著名的公式是什么？', message_history=result1.new_messages())

print(result2.all_messages())
# 输出：
# [
#   UserPrompt(content='爱因斯坦是谁？'),                    # ← 第一轮的
#   ModelResponse(content='爱因斯坦是德国理论物理学家...'),  # ← 第一轮的
#   UserPrompt(content='他最著名的公式是什么？'),            # ← 第二轮的
#   ModelResponse(content='E = mc²'),                       # ← 第二轮的
# ]
```

**用途**：调试、查看完整对话、存数据库。

---

## 生产环境实战：Web 聊天机器人

### FastAPI 实现（存储在内存）

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from pydantic_ai import Agent
from uuid import uuid4

app = FastAPI()
agent = Agent('openai:gpt-4o')

# 简单的内存存储（生产环境用 Redis/数据库）
conversations: dict[str, list] = {}

class ChatRequest(BaseModel):
    session_id: str | None = None  # 会话 ID
    message: str

class ChatResponse(BaseModel):
    session_id: str
    reply: str

@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest):
    # 获取或创建会话
    if req.session_id and req.session_id in conversations:
        history = conversations[req.session_id]
    else:
        req.session_id = str(uuid4())
        history = []
    
    # 运行 Agent
    result = agent.run_sync(
        req.message,
        message_history=history,  # ← 传入历史
    )
    
    # 更新历史（只存新增的消息）
    history.extend(result.new_messages())
    conversations[req.session_id] = history
    
    return ChatResponse(
        session_id=req.session_id,
        reply=result.output,
    )
```

**前端调用**：

```javascript
let sessionId = null;

async function sendMessage(message) {
  const response = await fetch('/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ session_id: sessionId, message }),
  });
  const data = await response.json();
  sessionId = data.session_id;  // 保存会话 ID
  return data.reply;
}

// 用户对话
await sendMessage('爱因斯坦是谁？');        // 第一轮
await sendMessage('他最著名的公式是什么？');  // 第二轮（能理解"他"）
```

---

## 进阶：什么时候清空历史？

### 场景 1：用户点击"新对话"

```python
@app.post("/new_conversation")
async def new_conversation():
    session_id = str(uuid4())
    conversations[session_id] = []  # 空历史
    return {"session_id": session_id}
```

### 场景 2：历史太长（超过上下文窗口）

```python
MAX_MESSAGES = 20  # 保留最近 20 条消息

history.extend(result.new_messages())
if len(history) > MAX_MESSAGES:
    history = history[-MAX_MESSAGES:]  # 只保留最近的
```

**注意**：简单截断会丢失早期上下文。更好的方案：

- 用 summarization（让另一个 Agent 总结早期对话）
- 用 vector DB（语义搜索相关历史，只传最相关的）

---

## 与笔记 03 的联动：`instructions` vs `system_prompt`

回顾笔记 03 的结论：

- **`instructions`**：只保留当前 Agent 的，不会出现在 `message_history` 里
- **`system_prompt`**：会保留在历史消息里，传给下一次 run

### 多轮对话的影响

```python
agent = Agent(
    'openai:gpt-4o',
    system_prompt='你是客服机器人，请礼貌回答。',
)

result1 = agent.run_sync('你好')
# system_prompt 会在 result1.all_messages() 里

result2 = agent.run_sync('订单在哪？', message_history=result1.new_messages())
# result2 的请求里会包含 result1 的 system_prompt（因为在历史里）
```

**最佳实践**：多轮对话时用 `system_prompt`（不用 `instructions`），保证每轮都有相同的身份设定。

---

## 常见问题 Q&A

### Q1: `new_messages()` 和 `all_messages()` 什么时候用？

| 场景 | 用哪个 | 为什么 |
|------|--------|--------|
| 传给下一次 `run()` 的 `message_history` | `new_messages()` | 避免重复（你已经传过历史了，只需要新增的） |
| 调试、查看完整对话 | `all_messages()` | 看到完整上下文 |
| 存数据库 | `new_messages()` | 增量存储，不重复 |

### Q2: 多轮对话会增加成本吗？

**会**。每次 run 都要把完整历史传给模型：

- 第 1 轮：2 条消息（用户 + 模型）
- 第 2 轮：4 条消息（前 2 条 + 新 2 条）
- 第 3 轮：6 条消息
- ...
- 第 10 轮：20 条消息

**input token 会线性增长**，导致成本累积。

**优化方案**：

1. 截断历史（只保留最近 N 条）
2. 总结早期对话（用另一个 Agent 压缩）
3. 语义搜索（只传最相关的历史）

### Q3: 可以跨 Agent 传历史吗？

**可以，但要小心**。不同 Agent 的 `system_prompt` 可能冲突：

```python
agent1 = Agent('openai:gpt-4o', system_prompt='你是客服机器人')
result1 = agent1.run_sync('你好')

agent2 = Agent('openai:gpt-4o', system_prompt='你是技术支持机器人')
result2 = agent2.run_sync(
    '我的电脑坏了',
    message_history=result1.new_messages(),  # ← 包含 agent1 的 system_prompt
)
# result2 的请求里会同时有两个 system_prompt，可能导致身份混乱
```

**建议**：跨 Agent 传历史时，只传用户和模型的消息，过滤掉 system_prompt。

---

## 三个必记要点

| 要点 | 说明 |
|------|------|
| 单 run 场景不需要 `message_history` | 数据提取、批处理等独立任务 |
| `new_messages()` 拿本轮新增 | 用于追加到自己维护的历史列表 |
| 多轮对话成本线性增长 | 需要截断、总结或语义搜索优化 |

---

## 自测问题

1. 什么时候需要传 `message_history`？什么时候不需要？
2. `new_messages()` 和 `all_messages()` 的区别是什么？各自适合什么场景？
3. 多轮对话为什么会导致成本增长？有哪三种优化方案？
