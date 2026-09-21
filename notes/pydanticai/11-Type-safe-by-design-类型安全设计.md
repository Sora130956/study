# 11 Type safe by design — 为什么 Pydantic AI 能在写代码时就发现错误

> 原文档：https://pydantic.dev/docs/ai/core-concepts/agent/#static-type-checking
> 重写日期：2026-09-21
> 学习目标：面试加分项，不影响功能实现

---

## 快速理解：类型安全是什么

**一句话**：类型检查器（mypy、pyright）能在你**写代码时**（甚至运行前）就发现类型错误，而不是等到 Agent 运行时才崩溃。

**对比其他框架**：

| 框架 | 依赖传递方式 | 工具注册 | 类型检查 |
|------|-------------|---------|---------|
| LangChain | 字符串 key 从 dict 里取 | 装饰器 + 字符串名称 | ❌ 运行时才报错 |
| Pydantic AI | `RunContext[DepsType]` 泛型 | 装饰器 + 类型推导 | ✅ IDE 红线 + mypy 检查 |

---

## 场景：类型错误在写代码时就被发现

### 错误代码示例

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class User:
    name: str

agent = Agent(
    'openai:gpt-4o',
    deps_type=User,        # ← Agent 期望 User 类型的依赖
    output_type=bool,      # ← Agent 输出 bool
)

@agent.system_prompt
def add_user_name(ctx: RunContext[str]) -> str:  # ❌ 这里写错了！期望 User 却写成 str
    return f"The user's name is {ctx.deps}."

def foobar(x: bytes) -> None:
    pass

result = agent.run_sync('Does their name start with "A"?', deps=User('Anne'))
foobar(result.output)  # ❌ 又写错了！output 是 bool 却传给了期望 bytes 的函数
```

### mypy 检查结果

```bash
$ mypy type_mistakes.py
type_mistakes.py:18: error: Argument 1 to "system_prompt" has incompatible type 
  "Callable[[RunContext[str]], str]"; expected "Callable[[RunContext[User]], str]"
type_mistakes.py:28: error: Argument 1 to "foobar" has incompatible type "bool"; expected "bytes"
Found 2 errors in 1 file
```

**关键点**：

- 错误 1：`add_user_name` 函数声明的 `RunContext[str]` 与 Agent 的 `deps_type=User` 不匹配
- 错误 2：`result.output` 的类型是 `bool`（Agent 定义的 `output_type=bool`），但 `foobar` 期望 `bytes`

这两个错误在**写代码时**就被 IDE 标红了，不需要运行到这一行才发现。

---

## 真实场景：为什么类型安全有用？

### 场景 1：重构时的保护伞

你做了个数据提取 Agent，原本 `deps_type=str`（传文件路径）：

```python
agent = Agent('openai:gpt-4o', deps_type=str)

@agent.tool
def read_file(ctx: RunContext[str]) -> str:
    return Path(ctx.deps).read_text()
```

后来需求变了，要传 `Database` 连接对象：

```python
@dataclass
class Database:
    connection: Any

agent = Agent('openai:gpt-4o', deps_type=Database)  # 改了这里

@agent.tool
def read_file(ctx: RunContext[str]) -> str:  # ❌ 忘了改这里！
    return Path(ctx.deps).read_text()  # 运行时会崩溃：Database 对象没有 read_text()
```

**如果没有类型检查**：要等到 Agent 运行、调用工具时才报错（可能是生产环境）。

**有了类型检查**：保存文件时 IDE 就报错：

```
error: Argument 1 has incompatible type "Callable[[RunContext[str]], str]"; 
       expected "Callable[[RunContext[Database]], str]"
```

### 场景 2：多人协作时的契约

你写了个 Agent，同事要注册新工具：

```python
# 你的代码
agent = Agent('openai:gpt-4o', deps_type=Database)

# 同事的代码（在另一个文件）
@agent.tool
def export_data(ctx: RunContext[str]) -> dict:  # ❌ 类型写错了
    # ...
```

**如果没有类型检查**：同事提交代码后，CI 测试可能都过了（如果没覆盖到这个工具），直到生产环境用户触发这个工具才崩溃。

**有了类型检查**：CI 跑 mypy 时就失败，拒绝合并：

```bash
$ mypy .
export.py:42: error: Incompatible types in assignment
```

---

## 进阶：Agent 的泛型是如何传播的

Pydantic AI 的 `Agent` 是一个泛型类：

```python
class Agent[DepsType, OutputType]:
    def __init__(
        self,
        model: str,
        deps_type: type[DepsType] = ...,
        output_type: type[OutputType] = ...,
    ):
        ...
    
    def run_sync(
        self,
        user_prompt: str,
        deps: DepsType,  # ← 必须传 DepsType 类型
    ) -> RunResult[OutputType]:  # ← 返回值的 output 字段是 OutputType
        ...
```

**类型传播链条**：

1. `Agent(deps_type=User, output_type=bool)` → `Agent[User, bool]`
2. 装饰器注册的工具必须是 `Callable[[RunContext[User]], ...]`
3. `result.output` 的类型被推导为 `bool`

所有类型检查器（mypy、pyright、Pylance）都能跟踪这个链条。

---

## 生产环境最佳实践

### 示例 1：在 CI 里强制类型检查

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install mypy pydantic-ai
      - run: mypy src/  # ← 强制类型检查
```

**效果**：所有 PR 必须通过类型检查才能合并。

### 示例 2：在 IDE 里启用严格模式

VS Code 的 Pylance 设置：

```json
{
  "python.analysis.typeCheckingMode": "strict"
}
```

PyCharm 的设置：启用 "Type Checker" 插件。

**效果**：写代码时实时看到类型错误（红色波浪线）。

### 示例 3：给复杂依赖加类型别名

```python
from typing import TypeAlias
from pydantic_ai import Agent, RunContext

# 定义类型别名，增强可读性
AppDeps: TypeAlias = tuple[Database, Logger, Cache]

agent = Agent('openai:gpt-4o', deps_type=AppDeps)

@agent.tool
def complex_operation(ctx: RunContext[AppDeps]) -> dict:
    db, logger, cache = ctx.deps
    # 类型检查器知道 db 是 Database、logger 是 Logger、cache 是 Cache
    logger.info('Starting operation')
    return db.query('SELECT ...')
```

---

## 常见问题 Q&A

### Q1: 类型检查是运行时还是静态的？

**静态的**。mypy/pyright 只分析代码，不运行代码。运行时依然由 Pydantic 校验（见笔记 05 的输出重试）。

**两层保障**：

- 编译时（静态）：mypy 检查类型是否匹配
- 运行时（动态）：Pydantic 校验实际数据是否符合 schema

### Q2: 如果我不用类型检查，Pydantic AI 还能用吗？

**能用**。类型标注是可选的（Python 的设计），但你会失去：

- IDE 自动补全（不知道 `ctx.deps` 有哪些字段）
- 重构保护（改了 `deps_type` 不知道哪些工具要改）
- 团队协作契约（别人不知道该传什么类型）

### Q3: 面试时怎么说这个特性？

**话术模板**：

> Pydantic AI 是 generic over deps and output types，所以工具函数里的 `RunContext[DepsType]` 如果写错类型，mypy 能在 CI 阶段就抓住。这是 type-safe by design 的体现，区别于 LangChain 那种运行时才报错的字符串拼装风格。

---

## 两个必记要点

| 要点 | 说明 |
|------|------|
| `Agent[DepsType, OutputType]` | Agent 是泛型类，类型会传播到工具、结果 |
| mypy/pyright 能检查 | 工具签名、`result.output` 类型错误在写代码时就能发现 |

---

## 自测问题

1. Agent 的两个泛型参数是什么？它们分别约束什么？
2. 如果 `deps_type=User` 但工具函数写成 `RunContext[str]`，什么时候会报错？
3. 类型检查和运行时校验有什么区别？Pydantic AI 用了哪两层保障？
