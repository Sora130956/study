# 08 模型错误处理 — 失败时如何保存现场

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#model-errors  
> 精读日期：2026-09-21  
> 核心场景：API 失败、重试耗尽、拿到完整上下文排查

---

## <mark style="background: #BBFABBA6;">什么时候会抛 `UnexpectedModelBehavior`？</mark>

**三种常见场景**：

1. **重试预算耗尽**：工具一直抛 `ModelRetry`，超过 `retries` 上限
2. **API 返回错误**：OpenAI 返回 503 Service Unavailable
3. **内容过滤触发**：Gemini 的安全过滤器拦截了输出

**统一出口**：这三种情况都抛 `UnexpectedModelBehavior`，你只需捕获这一个异常。

---

## 核心问题：失败时如何知道发生了什么？

**传统做法**（没有上下文）：
```python
try:
    result = agent.run_sync('Extract data')
except UnexpectedModelBehavior as e:
    print(e)  # "Tool exceeded max retries"
    # 然后呢？不知道模型调了哪个工具、传了什么参数、为什么失败
```

**Pydantic AI 方案**：用 `capture_run_messages` **保存完整对话历史**。

```python
from pydantic_ai import capture_run_messages, UnexpectedModelBehavior

with capture_run_messages() as messages:
    try:
        result = agent.run_sync('Extract data')
    except UnexpectedModelBehavior as e:
        print('失败:', e)
        print('底层原因:', e.__cause__)  # 原始异常（如 ModelRetry）
        print('消息历史:', messages)     # 完整对话，包含所有请求/响应
        
        # 现在你知道：
        # - 用户提问是什么
        # - 模型调了哪个工具
        # - 工具的参数和返回值
        # - 重试了几次，每次的参数
```

---

## 实战场景：工具一直失败（调试重试循环）

**问题**：你的工具 `calc_volume` 定义了，但模型调用时一直重试失败。

### 完整示例

```python
from pydantic_ai import Agent, ModelRetry, UnexpectedModelBehavior, capture_run_messages

agent = Agent('openai:gpt-4o', retries=2)  # 允许重试 2 次

@agent.tool_plain
def calc_volume(size: int) -> int:
    if size == 42:
        return size ** 3
    else:
        raise ModelRetry('Size must be 42')  # 故意让模型重试

with capture_run_messages() as messages:
    try:
        result = agent.run_sync('Calculate volume of a box with size 10')
    except UnexpectedModelBehavior as e:
        print(f'失败: {e}')
        # 失败: Tool 'calc_volume' exceeded max retries count of 2
        
        print(f'底层原因: {e.__cause__}')
        # 底层原因: ModelRetry('Size must be 42')
        
        # 查看消息历史
        for msg in messages:
            print(msg)
```

### 消息历史长什么样？

```python
[
    # 第 1 轮：用户提问
    ModelRequest(
        parts=[UserPromptPart(content='Calculate volume of a box with size 10')],
        timestamp=datetime(...),
    ),
    
    # 第 1 轮：模型调用工具
    ModelResponse(
        parts=[ToolCallPart(tool_name='calc_volume', args={'size': 10})],
        usage=RequestUsage(input_tokens=50, output_tokens=5),
    ),
    
    # 第 1 次重试：工具返回错误提示
    ModelRequest(
        parts=[RetryPromptPart(content='Size must be 42', tool_name='calc_volume')],
    ),
    
    # 第 2 轮：模型再次调用（可能改了参数，也可能没改）
    ModelResponse(
        parts=[ToolCallPart(tool_name='calc_volume', args={'size': 10})],  # 还是 10！
    ),
    
    # 第 2 次重试：又失败了
    ModelRequest(
        parts=[RetryPromptPart(content='Size must be 42')],
    ),
    
    # 第 3 轮：还是失败
    ModelResponse(
        parts=[ToolCallPart(tool_name='calc_volume', args={'size': 10})],
    ),
    
    # 重试预算耗尽，run 中断
    ModelRequest(
        parts=[],
        state='interrupted',  # 标记：这次 run 没有正常完成
    ),
]
```

**从历史能看出什么？**
- 模型调了 3 次工具，每次传的参数都是 `{'size': 10}`
- 虽然工具提示 "Size must be 42"，但模型**没有改参数**
- 原因：模型理解不了你的错误提示（可能太隐晦）

**解决方案**：
```python
raise ModelRetry('Invalid size. Please call calc_volume again with size=42')
# 更明确的提示，模型更容易理解
```

---

## 实战场景：API 失败（503 错误）

**场景**：OpenAI API 偶尔返回 503，你需要区分是 API 问题还是代码问题。

```python
with capture_run_messages() as messages:
    try:
        result = agent.run_sync('Query database')
    except UnexpectedModelBehavior as e:
        # 检查底层原因
        if isinstance(e.__cause__, httpx.HTTPStatusError):
            status = e.__cause__.response.status_code
            if status >= 500:
                # API 服务端错误，过会儿重试
                logger.warning('OpenAI API error: %s, will retry later', status)
                schedule_retry(task_id)
            else:
                # 4xx 客户端错误（如 token 超限），不要重试
                logger.error('Client error: %s', e)
                raise
        else:
            # 其他错误（如重试耗尽），记录消息历史排查
            logger.error('Run failed: %s, messages=%s', e, messages)
            raise
```

---

## 被中断的消息（Interrupted Messages）

**场景**：工具执行到一半时 run 失败了（如网络超时、用户取消），部分工具已完成、部分未完成。

### 示例：两个工具顺序执行，第二个失败

```python
agent = Agent('openai:gpt-4o')

@agent.tool_plain(sequential=True)  # 顺序执行
def get_volume(size: int) -> int:
    return size ** 3

@agent.tool_plain(sequential=True)
def get_mass(size: int) -> int:
    raise RuntimeError('Missing density parameter')  # 第二个工具故意失败

with capture_run_messages() as messages:
    try:
        agent.run_sync('Calculate volume and mass of a box with size 5')
    except RuntimeError:
        # 查看被中断的消息
        interrupted_request = next(
            msg for msg in messages 
            if isinstance(msg, ModelRequest) and msg.state == 'interrupted'
        )
        
        print(interrupted_request.parts)
        # [
        #     ToolReturnPart(tool_name='get_volume', content=125),  # 第一个工具成功了
        #     # 第二个工具的返回不在这里（因为失败了）
        # ]
```

**关键点**：
- `state='interrupted'` 标记这次 run 没有正常完成
- **已完成的工具结果会被保留**（`get_volume` 返回了 125）
- 未完成的工具调用不会有假的结果（不会凭空造一个 `ToolReturnPart`）

### 为什么要保留部分结果？

**续传场景**：
```python
# 第一次 run 失败，但保留了部分进度
with capture_run_messages() as messages:
    try:
        result = agent.run_sync('Multi-step task')
    except Exception:
        save_to_db(messages)  # 保存进度

# 用户稍后重试，传入上次的历史
saved_messages = load_from_db()
result = agent.run_sync(
    'Continue the task',
    message_history=saved_messages,  # 恢复上下文
)
# 框架会自动修复被中断的消息，让模型从断点继续
```

---

## 生产环境最佳实践

### 1. 统一错误处理（FastAPI 示例）

```python
from fastapi import HTTPException
from pydantic_ai import UnexpectedModelBehavior, capture_run_messages

@app.post('/extract')
async def extract_data(doc: str):
    with capture_run_messages() as messages:
        try:
            result = await extract_agent.run(doc, usage_limits=...)
            return {'data': result.output}
        
        except UnexpectedModelBehavior as e:
            # 记录完整上下文
            logger.error(
                'Extraction failed: %s, cause=%r, messages=%s',
                e, e.__cause__, messages
            )
            
            # 返回用户友好的错误
            if 'exceeded max retries' in str(e):
                raise HTTPException(502, 'Model retry limit exceeded')
            elif 'Content filter' in str(e):
                raise HTTPException(400, 'Content blocked by safety filter')
            else:
                raise HTTPException(502, 'AI service temporarily unavailable')
```

### 2. 区分可重试和不可重试错误

```python
def is_retryable(error: UnexpectedModelBehavior) -> bool:
    """判断错误是否值得重试"""
    cause = error.__cause__
    
    # API 5xx 错误：可重试
    if isinstance(cause, httpx.HTTPStatusError) and cause.response.status_code >= 500:
        return True
    
    # 网络超时：可重试
    if isinstance(cause, httpx.TimeoutException):
        return True
    
    # 重试耗尽：不可重试（重试也没用）
    if 'exceeded max retries' in str(error):
        return False
    
    # 内容过滤：不可重试（改输入才行）
    if 'Content filter' in str(error):
        return False
    
    return False
```

### 3. 批处理容错（处理 100 份文档）

```python
results = []
failed = []

for doc in documents:
    with capture_run_messages() as messages:
        try:
            result = agent.run_sync(doc, usage_limits=...)
            results.append({'doc_id': doc.id, 'data': result.output})
        
        except UnexpectedModelBehavior as e:
            # 记录失败，但不中断整个批次
            failed.append({
                'doc_id': doc.id,
                'error': str(e),
                'messages': messages,  # 保存现场，事后排查
            })
            continue  # 继续处理下一份文档

print(f'成功: {len(results)}, 失败: {len(failed)}')
save_failed_to_db(failed)  # 失败的单独处理
```

---

## 取消 run 的特殊情况

**场景**：用户主动取消了一个长时间运行的 Agent（如前端点了"停止"按钮）。

```python
from pydantic_ai import RunCancelled

try:
    result = await agent.run(prompt)
except RunCancelled as e:
    # RunCancelled 自带消息历史（不需要 capture_run_messages）
    print('Run was cancelled')
    print('Progress before cancellation:', e.messages)
    
    # 可以保存进度，让用户稍后继续
    save_progress(e.messages)
```

**区别**：
- `UnexpectedModelBehavior`：失败（需要 `capture_run_messages` 获取历史）
- `RunCancelled`：取消（自带 `.messages` 属性）

---

## 常见问题

### Q1：`capture_run_messages` 会影响性能吗？
**A**：几乎没有影响（只是引用消息对象，不复制）。开发期和生产期都建议用。

### Q2：消息历史会包含敏感数据吗？
**A**：会。工具参数和返回值都会被记录。如果有敏感字段：
- 用 `@agent.tool(..., redact_result=True)` 屏蔽返回值
- 或在记录日志前手动过滤 `messages`

### Q3：`state='interrupted'` 的消息能再次传入 run 吗？
**A**：能。框架会自动修复它们（补全缺失的 parts，调整格式），让模型从断点继续。

### Q4：`e.__cause__` 里可能是什么异常？
**A**：
- `ModelRetry`：工具重试
- `httpx.HTTPStatusError`：API 错误
- `ValidationError`：输出验证失败
- 自定义异常：工具里抛的任意异常

---

## 总结：错误处理三步法

| 步骤 | 做什么 | 代码 |
|------|--------|------|
| 1. 捕获异常 | 统一捕获 `UnexpectedModelBehavior` | `except UnexpectedModelBehavior as e` |
| 2. 保存现场 | 用 `capture_run_messages` 拿到完整历史 | `with capture_run_messages() as messages` |
| 3. 分类处理 | 检查 `e.__cause__` 区分 API 错误/重试耗尽/过滤 | `if isinstance(e.__cause__, HTTPStatusError)` |

**记住**：没有 `capture_run_messages`，你只能看到错误信息；有了它，你能看到完整对话过程。
