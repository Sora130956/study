# pydantic-settings 入门教程

```
- 目标读者：会用基础 Python（写过函数、类、import，但没接触过 pydantic / 配置管理库）
- 前置知识：第 3 节的"类型注解""环境变量""类与属性"（概念会铺垫，术语表可边读边回查）
- 学习时长：约 45 分钟
- 可否跳过：第 1、2 节是背景铺垫，急着上手可直接跳到第 4 节；第 6 节（坑）建议必读
```

---

## 第 1 节：它是什么

**一句话**：pydantic-settings 是一个把你程序里的"配置"统一收到一个地方、并自动帮你校验和读取的工具。

**生活类比**：想象一家餐厅的后厨。

- 不同厨师做菜要用到**油盐酱醋的用量**——这就是"配置"（数据库地址、API 密钥、端口号）。
- 以前每个厨师炒菜时都自己凭空抓调料，有的抓多有的抓少，出错还得一个个去问。
- pydantic-settings 就像后厨装了一块**固定调料墙**：所有调料配比写在墙上一处，厨师要用时直接去墙上取；用量配比写错了（比如盐写成"很多"），墙会**当场报警**，而不是等你菜做坏了才发现。

在 Python 里，"调料墙"就是你定义的那个 `Settings` 类。它读配置、做校验、给出一份类型安全的配置对象，你的代码拿着它去用，不用操心配置从哪来。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 反面推演：不用它，你通常会怎样写

很多初学者管理配置是这么写的，散乱地取环境变量：

```python
import os

DB_HOST = os.getenv("DB_HOST", "localhost")
DB_PORT = int(os.getenv("DB_PORT", "5432"))   # 环境变量是字符串，得手动转 int
API_KEY = os.getenv("API_KEY", "")
DATABASE_URL = f"postgres://{DB_HOST}:{DB_PORT}/mydb"   # 拼字符串

# 用的时候：
if not API_KEY:
    raise ValueError("API_KEY 未设置!")
```

这种写法有几个**可感知的痛点**：

1. **到处重复**：每个 .py 文件要用配置就自己 `os.getenv` 一遍，改一次环境变量名要全局搜着改。
2. **类型靠人肉转**：环境变量读出来永远是字符串，`"5432"` 要手动 `int()`，漏了转换就出 bug。
3. **非法值要等到用到才炸**：`int(os.getenv("DB_PORT"))` 如果给了个非数字，报错发生在使用那一行，你很难定位是配置写错了。
4. **没有一处是"配置的家"**：散落在各文件的变量，你没法一眼看清程序到底需要哪些配置。

**问题根源一句话**：配置的**读取、解析、校验、使用**全被写死在各处调用点，没有任何集中管理和类型约束。

### pydantic-settings 的经典应用场景

- FastAPI / 任意 Web 应用：数据库连接、JWT 密钥、CORS 白名单、第三方服务地址
- 各种 CLI / 脚本工具：运行参数、日志级别、缓存目录
- 微服务：从 `config.py` 集中管理所有环境相关配置

它做的本质上就是：**把"散落的配置"变成"一个校验过的对象"**。

---

## 第 3 节：读下去之前，先搞懂这些概念

### 概念 1：类型注解（type hint）

**定义**：在变量或函数参数后写 `: 类型`，告诉读者和工具这个值该是什么类型。它**不影响运行**，只用来提示和供工具检查（如 IDE 自动补全、pydantic 用它做校验）。

**大白话**：给变量贴个"标签"，说"这块应该是 int，别塞字符串进来"。

**最小示例**：

```python
def add(a: int, b: int) -> int:
    return a + b      # 运行时不检查，但 pydantic 会认真看这个标签
```

### 概念 2：环境变量（environment variable）

**定义**：操作系统里的一组全局"键值对"，程序可以读取，常用来存放不写进代码的敏感/环境相关配置。

**大白话**：系统级的起名规则——你给一个名字设一个值，任何程序都能按名字取到。比如设置 `DB_HOST=localhost` 后，程序里 `os.getenv("DB_HOST")` 就返回 `"localhost"`。

**最小示例**：

```python
import os
host = os.getenv("DB_HOST", "localhost")   # 取不到就用默认值 "localhost"
```

### 概念 3：.env 文件

**定义**：一个纯文本文件，把环境变量按 `KEY=VALUE` 写进去，程序启动时自动加载，方便本地开发不污染系统环境。

**大白话**：把"系统级环境变量"装进项目里的一个文件，`.env` 里的配置会被读取，等于临时给程序设置环境。

**最小示例**（`.env` 文件内容）：

```
DB_HOST=localhost
DB_PORT=5432
API_KEY=my-secret-123
```

> 注意：`.env` 常含密钥，务必加进 `.gitignore` 不提交。

### 概念 4：类与属性（class & attribute）

**定义**：`class` 是模板，`属性`是模板里的变量；实例化后每个对象持有自己的属性值。

**大白话**：类是一张"表格模板"，属性是表头列；你 `Settings()` 生成一张填好的表。

**最小示例**：

```python
class Settings:
    api_key: str = ""     # 类属性，带默认值

s = Settings()
print(s.api_key)          # 输出空字符串
```

### 概念 5：pydantic 的 BaseModel（前置概念）

**定义**：pydantic 是一个数据校验库，`BaseModel` 是它的基类。继承它后，你在类里声明的字段会**自动做类型校验**。

**大白话**：你声明"这个字段是 int"，pydantic 遇到字符串 `"123"` 会自动帮你转成 `123`，转不了就抛错。

**最小示例**：

```python
from pydantic import BaseModel

class User(BaseModel):
    age: int

u = User(age="30")     # pydantic 自动把字符串 "30" 校验并转成 30
print(u.age, type(u.age))   # 30 <class 'int'>
```

> pydantic-settings 就是在 BaseModel 之上，额外加上了**从环境变量 / .env 读取**的能力。

### 术语速查表

| 术语 | 大白话 | 对应正文 |
|---|---|---|
| 类型注解 | 给值贴"标签"说明类型 | 第 4 节所有 `fields` |
| 环境变量 | 系统级键值对 | 第 4、5 节 |
| .env 文件 | 项目里的环境变量记事本 | 第 5 节 |
| BaseModel | 帮你自动校验类型的基类 | 第 4 节 |
| Settings | 你定义的"调料墙"类 | 第 4 节 |

> 若这些概念你多半已懂，跳过本节的逐段阅读，直接进第 4 节，遇到不懂再回查术语表。

---

## 第 4 节：从最简单的代码示例开始

### 示例 1：最小可运行版本 —— 一个能用的 Settings

只保留"让 pydantic-settings 活着"的最少代码：定义类、读环境变量、打印。

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

# Settings 类 = 我们的"调料墙"，继承 BaseSettings 才有"从环境变量读"的能力
class Settings(BaseSettings):
    app_name: str = "my-app"        # 字段 + 默认值
    debug: bool = False             # 布尔字段，默认 False

    # 告诉 pydantic-settings 去哪里读配置：当前工作目录下的 .env（存在就加载）
    model_config = SettingsConfigDict(env_file=".env")

# 实例化：它会自动读环境变量 / .env，并把值填进字段
settings = Settings()

print(settings.app_name)    # 若 .env 里有 app_name=xxx 则输出 xxx，否则默认 "my-app"
print(settings.debug)       # 同理，输出 True/False
```

**逐段解读**：

- `from pydantic_settings import BaseSettings, SettingsConfigDict`：引入基类和配置类。
- `class Settings(BaseSettings)`：继承 `BaseSettings`，这是关键——正是它让类拥有"自动读环境变量/文件"的能力，普通 `BaseModel` 没有。
- `app_name: str = "my-app"`：声明一个字段，类型 `str`，默认 `"my-app"`。装饰等于"配置项的名字"。
- `bool = False`：注意这里类型是 `bool`。pydantic-settings 会把字符串 `"true"`、`"1"`、`"yes"` 自动解析成 `True`，这是它帮你做的事。
- `model_config = SettingsConfigDict(env_file=".env")`：告诉它去读 `.env` 文件。这行是可选的（不写就从纯环境变量读），但项目里几乎都会写。
- `settings = Settings()`：实例化时，pydantic-settings 同步去找环境变量和 `.env`，填进字段并**校验类型**。

**语法补充：`model_config` 是什么**

这是 pydantic v2 的写法。以前版本用 `class Config`，现在统一用 `model_config = SettingsConfigDict(...)` 声明"这个类的行为配置"。记住：它管的是"怎么读配置"（读哪个文件、是否大小写敏感、前缀等），和你业务字段分开。

**这一步你得到了什么**：一个声明式、带类型、可自动读取外部的配置对象。以后代码里用 `settings.app_name` 就能拿配置，不再到处 `os.getenv`。

---

### 示例 2：加一个真实需求 —— 数据库连接 + 必填校验

现在加一个真实场景：连数据库。配置里有必填的敏感项，也有需要二次处理的组合项。

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    # --- 基础信息 ---
    app_name: str = "my-app"

    # --- 数据库配置 ---
    db_host: str = "localhost"
    db_port: int = 5432                              # 类型 int，字符串会自动转成整数
    db_user: str = "admin"
    db_password: str = Field(..., description="必填")  # ... 表示"无默认值，必须从环境变量拿到"

    # 组合成完整连接串（@property 即"计算属性"，用法同普通属性）
    @property
    def database_url(self) -> str:
        return f"postgres://{self.db_user}:{self.db_password}@{self.db_host}:{self.db_port}/mydb"

    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
print(settings.database_url)
```

**逐段解读**：

- `db_port: int = 5432`：环境变量读完是字符串 `"5432"`，pydantic 自动转成 `5432`。**不用你手动 int()**——这正是解决第 2 节痛点 2。
- `db_password: str = Field(..., ...)`：`...`（省略号）是 pydantic 的"必填标记"。意味着如果环境变量和 .env 里都**没有** `db_password`，实例化 `Settings()` 会立刻抛错。**配置缺失在启动时就暴露**，不是等到查询数据库失败才炸——解决痛点 3。
- `@property def database_url`：一个计算属性。它不存值，而是每次访问时现算。这里把分散的字段拼成完整连接串。用 `settings.database_url` 和用普通属性一样。
- 字段名会自动映射到同名环境变量：`db_host` → 读 `DB_HOST`（pydantic-settings 默认大小写不敏感），`db_password` → 读 `DB_PASSWORD`。

**语法补充：为什么需要 `Field`**

`Field(...)` 是 pydantic 给字段加额外约束/元数据的工具。`...` 表示必填；你也可以 `Field(default, ge=0)` 表示默认值+最小值。普通 `type = 默认值` 只够表达"有默认值"，表达"必须提供"就得用 `Field(...)`。

**这一步你得到了什么**：配置能表达"哪些是必填的""哪些要类型转换"，还学会了把多个字段组合成 `database_url` 这种派生配置——这正是真实项目里最常见的用法。

---

### 示例 3：贴近项目的样子 —— 环境区分 + 别名 + 前缀

真实项目通常分开发/生产/测试环境，配置项很多，还要防止环境变量名撞车。这个示例展示常用的进阶组织方式（仍是玩具规模）。

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

# 读环境变量前先区分环境：PYTHON_ENV 或 APP_ENV 决定加载哪个 .env
import os
ENV = os.getenv("APP_ENV", "dev")

class Settings(BaseSettings):
    app_name: str = "my-app"
    api_key: str = Field(..., alias="MY_API_KEY")   # 环境变量名可以不同：用 MY_API_KEY

    db_host: str = "localhost"
    db_port: int = 5432

    db_password: str = Field(..., validate_default=True)

    model_config = SettingsConfigDict(
        env_file=f".env.{ENV}",          # 按环境加载：.env.dev / .env.prod
        env_prefix="",                   # 字段不加前缀直接对应环境变量
        extra="ignore",                  # 配置里多出的无关键直接忽略，不报错
    )

settings = Settings()
print(settings.api_key)
```

**逐段解读**：

- `import os; ENV = os.getenv("APP_ENV", "dev")`：先读当前环境，决定用哪个 .env 文件。这是项目里"多环境"的常规做法。
- `env_file=f".env.{ENV}"`：把字符串拼成 `.env.dev` / `.env.prod`，一套代码按环境读不同配置。
- `alias="MY_API_KEY"`：字段名叫 `api_key`，但它从环境变量 `MY_API_KEY` 读。当你不想让环境变量名和字段名一致时用。
- `extra="ignore"`：如果 .env 里有多余的键（比如其他工具写的），默认 pydantic-settings 会**报错**，`ignore` 让它安静忽略。项目里常备用，避免被无关键炸到。
- `Field(..., validate_default=True)`：对默认值也做校验（复杂场景需要时才用，可先不深究）。

**这一步你得到了什么**：一套能按环境加载、支持别名、能容忍多余配置项的结构，已经很接近真实 FastAPI 项目的 `config.py`。

---

## 第 5 节：最小可用的实际用法（生产场景模板）

这是直接可搬进真实项目的最小用法，复制后改字段即可。**注释标出哪些行是必须的、哪些可删。**

```python
# config.py —— 项目里通常单独一个文件，其他模块 from config import settings

import os
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    # ==== 必须项（项目运行离不开的配置，缺了要在启动时立刻报错） ====
    database_url: str = Field(...)          # 必填设置：没配就抛错
    secret_key: str = Field(...)            # JWT 密钥，必须从环境变量取

    # ==== 带默认值的项（可删：省略默认值字段不是必须的） ====
    app_name: str = "my-app"
    debug: bool = False
    log_level: str = "INFO"

    # ==== 可选：多环境配置 ====
    env: str = os.getenv("APP_ENV", "dev")

    # ==== 配置读取行为（这行是必须的：告诉它去读哪个 .env） ====
    model_config = SettingsConfigDict(
        env_file=f".env.{os.getenv('APP_ENV', 'dev')}",   # 按环境加载
        env_file_encoding="utf-8",                        # 保证中文/编码正确（可选）
        extra="ignore",                                   # 容忍多余键（可选）
    )

# 实例化一次，全项目共享这一个配置对象（必须：项目里就这一份 settings）
settings = Settings()
```

**哪几行是必须的 / 可删的**

| 部分 | 必须？ | 说明 |
|---|---|---|
| `class Settings(BaseSettings)` | 必须 | 只有继承 `BaseSettings` 才有读环境变量的能力 |
| `database_url: str = Field(...)` 等必填字段 | 必须 | 项目离不开的配置，用 `...` 强制启动时报错 |
| `model_config = SettingsConfigDict(env_file=...)` | 必须 | 决定从哪读配置；不写则只读纯环境变量 |
| `app_name`、`debug`、`log_level` 等带默认值字段 | 可删 | 有默认值即可用，删了会缺，但通常保留 |
| `env`、`env_file_encoding`、`extra` | 可选 | 按环境加载和容错，复杂项目才需要 |

**实际项目中通常会再配什么（进阶，先跳过）**

- **按环境分文件**：`.env.dev` / `.env.prod`，配合 `gitignore` 只提交 `.env.example` 模板。
- **字段校验约束**：`Field(gt=0)`、`Field(pattern=r"^...")` 限制端口号、密钥格式。
- **组合派生属性**：用 `@property` 把多个字段拼成 `database_url` 这类派生值。
- **用到即用**：项目里 `from config import settings` 然后 `settings.database_url`，不要每处重新实例化 `Settings()`。

---

## 第 6 节：常见坑与易错点

| 报错 / 现象 | 原因 | 解决 |
|---|---|---|
| `ValidationError: db_password Field required` | 用 `Field(...)` 标了必填，但环境变量和 .env 里都没这个键 | 在 .env 或环境变量里设置该键（或在 `Settings()` 里传默认值） |
| `Output of parsing is not a valid bool` | 把 `debug` 字段赋成了字符串但解析不了（如 `"maybe"`） | 环境变量写 `true`/`false`/`1`/`0`/`yes`/`no` |
| `.env` 里写了键却读不到 | `model_config` 没写 `env_file=".env"`，或文件名/路径不对 | 确认 `model_config` 配了正确路径；`.env` 要从项目根目录读 |
| `ValueError: extra fields not permitted` | 默认 `extra` 是 `forbid`，.env 里多的键会被报错 | 加 `extra="ignore"`（或 `extra="allow"`） |
| 字段值不被解析（还是字符串） | 忘了给字段标类型，或类型标错 | 声明 `db_port: int`，让 pydantic 帮你转类型 |

### 和我直觉相反的一点

**pydantic-settings 默认大小写不敏感**：字段 `db_host` 会去读环境变量 `DB_HOST`，也可能读 `db_host`。这让"字段名"和"环境变量名"可以不完全一致，但也意味着同一配置出现在两处（大小写不同）时，取值优先级可能出乎意料。**建议养成约定：字段用小写下划线，环境变量用全大写，避免歧义。**

---

## 第 7 节：验证你学会了（自测题）

1. **复述**：pydantic-settings 解决的核心问题是什么？它和普通 `os.getenv` 相比，多替你做哪两件事？（答案见第 2 节第 1 小点、第 4 节）
2. **默写**：不看文档，写一个 `Settings` 类，包含必填的 `secret_key` 和带默认值的 `debug`，并配置从 `.env` 读取。（答案对照第 5 节）
3. **变体**：你的项目需要同时支持 `DB_PORT` 这个环境变量，但字段名想叫 `db_port`，且 `.env` 里可能有多余键不能让程序崩溃——怎么做？（答案见第 4 节示例 3 的 `alias` 和 `extra="ignore"`）
