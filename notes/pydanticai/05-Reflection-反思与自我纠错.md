# 05 Reflection and self-correction — 反思与自我纠错【必读】

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#reflection-and-self-correction
> 精读日期：2026-09-19
> 学习目标：Day 2 需要的重试逻辑和错误处理

## 一、是什么

**大白话版本**：模型也会犯错——<mark style="background: #BBFABBA6;">给工具传错参数、输出的 JSON 格式不对、提取的数据缺字段……Pydantic AI 会自动把这些错误回传给模型，让它"反思"后重新生成响应。你也可以主动告诉模型"这次不行，重来"。</mark>

**技术定义**：Pydantic AI 支持三种重试触发方式：
1. **<mark style="background: #BBFABBA6;">工具参数校验失败</mark>** — 模型调用工具时传的参数不符合类型要求
2. **<mark style="background: #BBFABBA6;">结构化输出校验失败</mark>** — 模型输出的 JSON 不符合 `output_type` 定义
3. **<mark style="background: #BBFABBA6;">业务逻辑主动触发</mark>** — 在工具或输出校验器内抛出 <mark style="background: #BBFABBA6;">`ModelRetry`</mark>

<mark style="background: #BBFABBA6;">每次重试都会把错误信息作为 prompt 回传给模型，让它根据错误提示自我纠正。</mark>

## 二、为什么需要

**实战场景**（Smart-Data-Extractor 简历提取）：

```python
class Resume(BaseModel):
    name: str
    phone: str
    email: str
    work_experience: list[WorkExperience]

agent = Agent('openai:gpt-4o', output_type=Resume)
```

**可能出现的问题**：
- 模型输出 `"phone": "不详"` → Pydantic 校验失败（期望 `str`，但可能你的校验器要求手机号格式）
- 模型提取的工作经历里缺 `company` 字段 → 校验失败
- 业务规则：必须从原文中找到出处才能填字段 → 需要你主动触发重试

**没有重试机制会怎样**：
- 直接抛异常 → 用户看到错误信息 → 开发者手动调试 → 费时费力
- 有了自动重试 → 模型看到"缺 company 字段"提示 → 重新生成正确的输出 → 用户无感知

**核心价值**：<mark style="background: #BBFABBA6;">把模型的"一次性交付"变成"迭代改进"，提高输出质量</mark>。

## 三、核心概念

### 3.1 <mark style="background: #BBFABBA6;">三种触发方式对比</mark>

| 触发方式 | 何时发生 | 谁触发 | 错误信息来源 | 示例场景 |
|---------|---------|-------|------------|---------|
| **参数校验** | 模型调工具时参数类型不对 | Pydantic 自动 | Pydantic 验证错误 | `get_user(age="abc")` → 期望 `int` |
| **输出校验** | 模型最终输出不符合 `output_type` | Pydantic 自动 | Pydantic 验证错误 | 缺必填字段、类型错误 |
| **业务触发** | 你的代码主动抛 `ModelRetry` | 开发者手动 | 你写的错误提示 | "金额字段必须从原文找到出处" |

**关键点**：
- <mark style="background: #BBFABBA6;">前两种是 Pydantic 自动的"语法检查"</mark>
- <mark style="background: #BBFABBA6;">第三种是你的"业务规则检查"</mark>
- <mark style="background: #BBFABBA6;">三种方式都会把错误信息作为 `RetryPromptPart` 回传给模型</mark>

### 3.2 重试预算配置（三级优先级）

<mark style="background: #BBFABBA6;">**默认行为**：所有重试次数默认为 **1 次**（即模型有 2 次机会：首次 + 1 次重试）。</mark>

<mark style="background: #BBFABBA6;">**三级配置**（后者覆盖前者）：</mark>

| 配置级别 | 作用范围 | 代码写法 | 适用场景 |
|---------|---------|---------|---------|
| **Agent 级** | 该 Agent 的所有工具 + 输出 | `Agent(..., retries=2)` | 全局默认值 |
| **工具级** | 单个工具 | `@agent.tool(retries=3)` | 某个工具特别容易出错 |
| **运行时** | 单次 `run()` 调用 | `agent.run(..., retries={'tools': 2, 'output': 3})` | 特定任务需要更多重试 |

**配置语法细节**：
- **裸 `int`**：`retries=2` → 同时配置工具和输出的重试次数
- **字典分别配置**：`retries={'tools': 2, 'output': 3}` → 分别控制
- **批量覆盖**：`agent.override(retries=2)` → 影响后续所有 run

### 3.3 在代码里访问当前重试次数

在工具、输出校验器或输出函数内部，通过 **`ctx.retry`** 获取：

```python
@agent.tool(retries=2)
def get_user(ctx: RunContext[DatabaseConn], name: str) -> int:
    if ctx.retry == 0:
        print("首次调用")
    else:
        print(f"第 {ctx.retry} 次重试")
    # ...
```

### 3.4 重试预算耗尽后会怎样

**两条输出路径的不同行为**：

| 输出类型 | 预算执行方式 | 耗尽后抛出 | 典型场景 |
|---------|------------|-----------|---------|
| **文本输出** <br>(`output_type=str`) | 整个 run 共享一个全局预算 | `UnexpectedModelBehavior` <br>消息：`'Exceeded maximum output retries (N)'` | CLI 工具、简单问答 |
| **工具输出** <br>(`output_type=ToolOutput(...)`) | 每个工具独立计数 | 单个工具耗尽后继续尝试其他工具 | 结构化提取、多工具协作 |

**理解"两条路径"**：
- **文本路径**：模型直接返回文本 → 校验失败 → 消耗全局预算 → 预算用完直接抛异常
- **工具路径**：模型调用工具 → 工具返回结构化数据 → 每个工具有自己的重试预算 → 某工具耗尽不影响其他工具

**实际意义**（Smart-Data-Extractor）：
- 简历提取用 `output_type=Resume`（工具路径）→ 每个字段提取工具独立重试
- CLI 交互用 `output_type=str`（文本路径）→ 共享全局预算

## 四、官方示例（带中文注释）

### 示例场景：根据姓名查用户 ID 并发消息

```python
from pydantic import BaseModel
from pydantic_ai import Agent, RunContext, ModelRetry
from fake_database import DatabaseConn

class ChatResult(BaseModel):
    user_id: int
    message: str

# Agent 配置：输出必须包含 user_id 和 message
agent = Agent(
    'openai:gpt-4o',
    deps_type=DatabaseConn,
    output_type=ChatResult,
)

@agent.tool(retries=2)  # 这个工具最多重试 2 次
def get_user_by_name(ctx: RunContext[DatabaseConn], name: str) -> int:
    """Get a user's ID from their full name."""
    print(f"尝试查找用户：{name}")
    
    user_id = ctx.deps.users.get(name=name)
    if user_id is None:
        # 主动触发重试，并给模型明确的错误提示
        raise ModelRetry(
            f'No user found with name {name!r}, remember to provide their full name'
        )
    return user_id

# 运行
result = agent.run_sync(
    'Send a message to John Doe asking for coffee next week',
    deps=DatabaseConn()
)

print(result.output)
# 输出：user_id=123 message='Hello John, would you be free for coffee sometime next week?'
```

**执行流程分解**：

1. **首次调用**：
   - 模型理解任务："给 John Doe 发消息"
   - 模型决定先调 `get_user_by_name(name="John")` → **只用名字，没有姓**
   - 工具查数据库失败 → 抛出 `ModelRetry('No user found with name "John", remember to provide their full name')`

2. **第 1 次重试**（`ctx.retry=1`）：
   - Pydantic AI 把错误提示作为新的 prompt 回传给模型
   - 模型看到提示"要用全名" → 自我纠正
   - 模型重新调用 `get_user_by_name(name="John Doe")` → **带姓带名**
   - 工具查到 `user_id=123` → 返回成功

3. **生成最终输出**：
   - 模型用 `user_id=123` 生成消息
   - 输出符合 `ChatResult` 格式 → 校验通过
   - `run_sync()` 返回结果

**关键点**：
- 整个重试过程对调用方**透明**，你只看到最终的成功结果
- 错误提示要**具体**：不要只说"失败了"，要告诉模型怎么改（"用全名"）
- 重试预算 `retries=2` 意味着：首次 + 2 次重试 = 共 3 次机会

## 五、生产级模板代码

### 场景：简历数据提取 + 业务校验

```python
from pydantic import BaseModel, Field, field_validator
from pydantic_ai import Agent, RunContext, ModelRetry
from typing import Optional

class WorkExperience(BaseModel):
    company: str
    position: str
    start_date: str
    end_date: Optional[str] = None
    description: str

class Resume(BaseModel):
    name: str
    phone: str
    email: str
    work_experience: list[WorkExperience]
    
    @field_validator('phone')
    @classmethod
    def validate_phone(cls, v: str) -> str:
        """业务规则：手机号必须是 11 位数字"""
        if not v.isdigit() or len(v) != 11:
            # 这会触发自动重试，Pydantic 会回传校验错误
            raise ValueError('手机号必须是 11 位数字')
        return v

# Agent 配置：结构化提取任务建议 retries=2
agent = Agent(
    'openai:gpt-4o',
    output_type=Resume,
    retries={'output': 2},  # 只配置输出重试，工具用默认值
)

# 输出校验器：自定义业务规则
@agent.output_validator
async def validate_work_experience(ctx: RunContext, resume: Resume) -> Resume:
    """确保工作经历的描述不是模型编造的"""
    original_text = ctx.deps  # 假设 deps 是原始简历文本
    
    for exp in resume.work_experience:
        # 检查公司名是否在原文中
        if exp.company not in original_text:
            # 主动触发重试，给模型具体的错误提示
            raise ModelRetry(
                f'公司名 "{exp.company}" 在原文中找不到，'
                f'请仔细检查原文或标记为 null。当前重试次数：{ctx.retry}/{ctx.max_retries}'
            )
    
    return resume

# 使用：运行时覆盖预算（特别复杂的简历需要更多重试）
async def extract_resume(resume_text: str, is_complex: bool = False):
    retries = {'output': 3} if is_complex else None  # 复杂简历给 3 次重试
    
    try:
        result = await agent.run(
            f'从以下简历中提取结构化数据：\n\n{resume_text}',
            deps=resume_text,
            retries=retries
        )
        return result.output
    except UnexpectedModelBehavior as e:
        # 预算耗尽，记录日志并返回友好错误
        logger.error(f'简历提取失败，重试次数耗尽：{e}')
        return {"error": "简历格式异常，请检查后重新上传"}
```

**模板特点**：
1. **三层校验**：
   - Pydantic 自动校验（类型、必填字段）
   - `@field_validator`（字段级业务规则）
   - `@agent.output_validator`（全局业务规则）

2. **错误提示设计**：
   - 具体指出问题：`'公司名 "xxx" 在原文中找不到'`
   - 给出解决方向：`'请仔细检查原文或标记为 null'`
   - 包含调试信息：`'当前重试次数：{ctx.retry}/{ctx.max_retries}'`

3. **动态预算**：
   - 简单任务用默认值
   - 复杂任务运行时传 `retries={'output': 3}`

4. **错误处理**：
   - 捕获 `UnexpectedModelBehavior`（预算耗尽）
   - 返回友好错误而不是直接抛异常

## 六、常见坑点与解决方案

### 坑点 1：错误提示不够具体

**错误示例**：
```python
if user_id is None:
    raise ModelRetry('查询失败')  # ❌ 模型不知道怎么改
```

**正确做法**：
```python
if user_id is None:
    raise ModelRetry(
        f'未找到姓名为 {name!r} 的用户。'
        f'可能的原因：(1) 姓名拼写错误 (2) 需要提供全名而非昵称。'
        f'请重新检查输入或尝试使用其他标识符（如工号、邮箱）。'
    )  # ✅ 给出具体原因和解决方向
```

### 坑点 2：无限重试死循环

**场景**：模型始终无法生成正确输出，但你设置了 `retries=999`。

**解决方案**：
```python
# 1. 合理设置预算上限
agent = Agent(..., retries=3)  # 一般 2-3 次足够

# 2. 配合 UsageLimits 双保险
from pydantic_ai import UsageLimits

result = await agent.run(
    prompt,
    retries=3,
    usage_limits=UsageLimits(
        request_limit=10,  # 最多 10 次 API 请求（含重试）
        output_tokens_limit=5000,  # 输出总 token 上限
    )
)
```

### 坑点 3：忽略 `ctx.retry` 导致重复操作

**场景**：工具每次重试都执行昂贵的初始化操作。

**错误示例**：
```python
@agent.tool(retries=2)
def process_data(ctx: RunContext, data_id: str):
    # 每次重试都重新加载数据（浪费）
    heavy_data = load_from_database(data_id)  # ❌ 耗时操作
    # ...
```

**正确做法**：
```python
@agent.tool(retries=2)
def process_data(ctx: RunContext, data_id: str):
    # 只在首次调用时加载
    if ctx.retry == 0:
        ctx.deps.cache[data_id] = load_from_database(data_id)  # ✅ 缓存
    
    heavy_data = ctx.deps.cache[data_id]
    # ...
```

### 坑点 4：预算耗尽后未捕获异常

**错误示例**：
```python
# API 端点
@app.post("/extract")
async def extract(resume: str):
    result = await agent.run(resume)  # ❌ 可能抛 UnexpectedModelBehavior
    return result.output
```

**正确做法**：
```python
from pydantic_ai import UnexpectedModelBehavior
from fastapi import HTTPException

@app.post("/extract")
async def extract(resume: str):
    try:
        result = await agent.run(resume)
        return result.output
    except UnexpectedModelBehavior as e:
        # 预算耗尽，返回 502 让客户端重试
        raise HTTPException(
            status_code=502,
            detail=f"数据提取失败：{str(e)}，请稍后重试"
        )
```

## 七、自测问题

1. **基础理解**：哪三种情况会触发重试？哪些是自动的，哪些需要手动触发？

2. **配置优先级**：以下配置最终生效的重试次数是多少？
   ```python
   agent = Agent(..., retries=2)
   
   @agent.tool(retries=3)
   def my_tool(...): pass
   
   result = agent.run(..., retries={'tools': 1})
   ```
   
3. **两条路径**：`output_type=str` 和 `output_type=Resume` 的重试预算执行方式有何不同？

4. **实战应用**：在简历提取场景中，如何设计错误提示让模型更容易自我纠正？

5. **坑点排查**：如果你的 Agent 总是在重试 3 次后抛出 `UnexpectedModelBehavior`，可能是哪些原因？如何排查？

---

## 八、与其他笔记的关联

- **前置知识**：[03-Instructions-指令](./03-Instructions-指令.md) — 错误提示本质上也是一种动态 instruction
- **配套使用**：[10-Usage-Limits-用量限制](./10-Usage-Limits-用量限制.md) — 用 `request_limit` 防止重试死循环
- **错误处理**：[08-Model-errors-模型错误](./08-Model-errors-模型错误.md) — 捕获 `UnexpectedModelBehavior` 的完整方案
- **生产部署**：[07-Debugging-and-Monitoring-调试与监控](./07-Debugging-and-Monitoring-调试与监控.md) — 用 Logfire 追踪重试过程
