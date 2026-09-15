# 流式输出：实时返回 LLM 生成内容

- 目标读者：会调用 OpenAI API，想实现打字机效果
- 前置知识：迭代器、for 循环、HTTP 流式响应概念
- 学习时长：1 小时 / 可跳过：第 5 节（SSE 接口）属于进阶应用

---

## 第 1 节：它是什么

**流式输出（Streaming）** 是让 LLM 边生成边返回内容，而不是等全部生成完才一次性返回。

**类比**：像视频播放的"边下载边播放"——不用等整个视频下载完，看到一部分就能开始播，用户体验更流畅。对 LLM 来说，就是"边生成边显示"，避免用户盯着空白屏幕等十几秒。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 不用它会怎样？

默认的非流式调用：

```python
response = client.chat.completions.create(...)
print(response.choices[0].message.content)  # 等 5-10 秒后一次性打印全部内容
```

**痛点**：
- 用户等待时间长（生成 500 字可能要 10 秒），页面卡死无反馈
- 无法提前展示部分结果（比如生成了一半就知道方向不对，想中断）
- 用户体验差（感觉"卡住了"）

### 问题根源

**等待全部生成完才返回，把"生成时间"变成了"阻塞时间"**，而实际上内容是逐字生成的，完全可以逐步返回。

### 经典应用场景

- 聊天界面（ChatGPT 式打字机效果）
- 长文本生成（文章、代码、翻译）
- 需要实时反馈的场景（用户可以边看边决定是否继续）
- SSE（Server-Sent Events）接口（后端接收 LLM 流，转发给前端）

---

## 第 3 节：读下去之前，先搞懂这些概念

| 术语 | 大白话解释 | 示例 | 对应正文节 |
|------|-----------|------|-----------|
| **stream=True** | 告诉 API"别等全部生成完，边生成边发给我" | `stream=True` | 第 4 节示例 1 |
| **chunk** | 流式响应的"每一块"数据，包含一小段新生成的文本 | 每次循环拿到一个 chunk | 第 4 节示例 1 |
| **delta** | chunk 里的"增量内容"，即本次新生成的那部分文字 | `chunk.choices[0].delta.content` | 第 4 节示例 1 |
| **迭代器** | 可以用 for 循环遍历的对象，流式响应返回的就是迭代器 | `for chunk in response: ...` | 第 4 节示例 1 |
| **SSE** | Server-Sent Events，一种 HTTP 协议，让服务端持续推送数据给前端 | `Content-Type: text/event-stream` | 第 5 节 |

**迭代器原理补充**：
```python
# 普通调用：一次性返回完整对象
response = client.chat.completions.create(...)
print(response.choices[0].message.content)  # 等全部生成完，拿到完整文本

# 流式调用：返回迭代器，每次 yield 一小块
response = client.chat.completions.create(..., stream=True)
for chunk in response:  # 每次循环拿到一小块新内容
    print(chunk.choices[0].delta.content, end="")  # 逐字打印
```

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本

**目标**：开启流式输出，逐块打印内容（打字机效果）。

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# stream=True 开启流式输出
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "用 100 字介绍 Python"}
    ],
    stream=True  # 关键参数：开启流式
)

# response 现在是一个迭代器，每次 yield 一个 chunk
for chunk in response:
    # chunk.choices[0].delta.content 是本次新增的内容（可能是 None）
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)  # 实时打印，不换行
```

**逐段解读**：

1. **stream=True**：
   - 开启后，API 不再等全部生成完才返回
   - 而是每生成几个字就发送一个 chunk

2. **for chunk in response**：
   - `response` 是一个迭代器（不是普通对象）
   - 每次循环拿到一个 chunk（一小块数据）

3. **chunk.choices[0].delta.content**：
   - `delta` 表示"增量"，即本次新增的文本
   - 注意可能是 `None`（比如第一个和最后一个 chunk 通常是空的），所以要判断

4. **print(..., end="", flush=True)**：
   - `end=""` 避免自动换行（让文字连在一起）
   - `flush=True` 立即刷新输出缓冲区（否则可能攒一堆才显示）

**这一步你得到了什么**：能看到 LLM 像打字机一样逐字输出，用户体验提升。

---

### 示例 2：收集完整内容 + 实时显示

**目标**：既要实时显示，又要保存完整文本。

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "你是代码助手"},
        {"role": "user", "content": "写一个 Python 函数计算斐波那契数列"}
    ],
    stream=True,
    temperature=0
)

full_content = ""  # 用于拼接完整内容

for chunk in response:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)  # 实时显示
        full_content += delta              # 同时拼接到完整文本

print("\n\n--- 完整内容 ---")
print(full_content)

# 后续可以把 full_content 存数据库、写文件等
with open("output.txt", "w", encoding="utf-8") as f:
    f.write(full_content)
```

**逐段解读**：

- **full_content += delta**：
  - 每次把新增的片段拼接到字符串里
  - 循环结束后 `full_content` 就是完整的生成内容

- **适用场景**：
  - 前端显示打字机效果，同时后端记录完整对话
  - 生成代码后要执行，需要完整代码字符串

**这一步你得到了什么**：既满足用户体验（实时看到），又满足业务需求（保存完整文本）。

---

### 示例 3：统计 token 和处理结束标志

**目标**：判断流是否结束，统计 token 消耗。

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "介绍一下 FastAPI"}],
    stream=True,
    stream_options={"include_usage": True}  # 开启 usage 统计
)

full_content = ""

for chunk in response:
    # 检查是否是最后一个 chunk（包含 usage 信息）
    if chunk.choices[0].finish_reason == "stop":
        print("\n[生成结束]")
        # 获取 token 使用情况
        if hasattr(chunk, "usage") and chunk.usage:
            print(f"输入 token: {chunk.usage.prompt_tokens}")
            print(f"输出 token: {chunk.usage.completion_tokens}")
            print(f"总计: {chunk.usage.total_tokens}")
        break
    
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
        full_content += delta
```

**逐段解读**：

- **finish_reason**：
  - `"stop"` 表示正常结束（生成完毕）
  - `"length"` 表示因 `max_tokens` 限制被截断
  - `None` 表示还在生成中

- **stream_options={"include_usage": True}**：
  - 流式输出默认不返回 token 统计
  - 加这个参数后，最后一个 chunk 会包含 `usage` 信息

- **hasattr(chunk, "usage")**：
  - 只有最后一个 chunk 才有 `usage` 属性，要先判断是否存在

**这一步你得到了什么**：可以在流式场景下也监控成本、判断是否被截断。

---

## 第 5 节：最小可用的实际用法（对接 SSE 接口）

**场景**：你是后端开发，前端要求返回 SSE 格式的流式响应（类似 ChatGPT Web）。

### FastAPI + SSE 示例

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from openai import OpenAI
import os

app = FastAPI()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

@app.post("/chat/stream")
async def chat_stream(user_message: str):
    """
    接收前端消息，返回 SSE 流式响应
    """
    def generate():
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": user_message}],
            stream=True,
            temperature=0
        )
        
        for chunk in response:
            delta = chunk.choices[0].delta.content
            if delta:
                # SSE 格式：data: <内容>\n\n
                yield f"data: {delta}\n\n"
        
        # 发送结束标志
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")

# 运行：uvicorn main:app --reload
# 前端用 EventSource 或 fetch 接收流
```

**逐段解读**：

- **def generate()**：
  - 生成器函数，每次 `yield` 返回一小块数据
  - 不能用普通 `return`，因为要持续返回多次

- **StreamingResponse**：
  - FastAPI 的流式响应类
  - `media_type="text/event-stream"` 是 SSE 协议的标准 Content-Type

- **SSE 格式规范**：
  - 每条消息必须是 `data: <内容>\n\n`（两个换行）
  - 前端会逐条接收这些消息

**前端接收示例（JavaScript）**：

```javascript
const eventSource = new EventSource('/chat/stream?user_message=介绍Python');

eventSource.onmessage = (event) => {
    if (event.data === '[DONE]') {
        eventSource.close();
        return;
    }
    document.getElementById('output').innerText += event.data;
};
```

**这一步你得到了什么**：一个完整的"后端接 OpenAI 流 → 转发给前端"的链路，可以直接用于生产。

---

## 第 6 节：常见坑与易错点

### 坑 1：忘记 `flush=True` 导致不实时

**现象**：代码写了流式输出，但终端里还是等很久才一次性显示。

**原因**：Python 的 `print()` 默认有缓冲区，攒够一定量才输出。

**解决**：加 `flush=True` 立即刷新缓冲区。

---

### 坑 2：delta.content 是 None 导致拼接出错

**现象**：`full_content += delta` 报错 `TypeError: can only concatenate str (not "NoneType")`。

**原因**：第一个和最后一个 chunk 的 `delta.content` 通常是 `None`。

**解决**：先判断 `if delta:`。

---

### 坑 3：流式输出不返回 usage

**现象**：想统计 token 但 `chunk.usage` 不存在。

**原因**：默认流式不包含 usage 信息（减少开销）。

**解决**：加参数 `stream_options={"include_usage": True}`，最后一个 chunk 会包含。

---

### 坑 4：SSE 格式错误导致前端解析失败

**现象**：前端 EventSource 报错或收不到消息。

**原因**：
- 忘记 `\n\n`（SSE 必须两个换行）
- 忘记 `data:` 前缀
- Content-Type 不是 `text/event-stream`

**解决**：严格按照 `data: <内容>\n\n` 格式，并设置正确的 media_type。

---

### 坑 5：流式输出中断后无法重试

**现象**：网络抖动导致流中断，程序崩溃。

**原因**：流式响应一旦开始，无法"重新来过"（不像普通请求可以重试）。

**解决**：
- 客户端记录已接收的内容，中断后用 `messages` 续传（续写）
- 或用 WebSocket 替代 SSE（支持双向通信和重连）

---

## 第 7 节：验证你学会了

### 问题 1：用一句话说清流式输出解决什么问题

**提示**：回查第 2 节。

---

### 问题 2：不看文档，默写一个最小流式输出示例

**要求**：
- 调用 OpenAI API，问"什么是机器学习"
- 开启流式输出
- 逐块打印到终端（打字机效果）

**提示**：参考第 4 节示例 1。

---

### 问题 3：变体场景

**任务**：写一个函数 `stream_to_list(prompt: str) -> list[str]`，输入 prompt，返回一个列表，列表的每个元素是一个 chunk 的内容（不是拼接后的完整文本）。

要求：
- 用流式输出获取内容
- 把每个 `delta.content`（非 None）加入列表
- 返回列表

**提示**：
- 如何开启流式？（`stream=True`）
- 如何判断 delta 非空？（`if delta:`）
- 如何往列表追加元素？（`list.append()`）

**答案位置**：第 4 节示例 1、2。
