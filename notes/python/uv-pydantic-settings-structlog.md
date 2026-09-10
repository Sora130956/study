# uv + pydantic-settings + structlog：环境、配置与日志 教程

> **来源:** [uv 官方文档](https://docs.astral.sh/uv/) · [pydantic-settings 官方文档](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) · [structlog 官方文档](https://www.structlog.org/en/stable/)
> **对应计划周次:** 第 1 周 · 周五上午「环境与配置：uv 管理依赖、pydantic-settings 读 .env、结构化日志」+ 周五下午「整理成可复用骨架 repo」
> **关键版本:** Python 3.12+ · pydantic-settings 2.x · uv 与 structlog 的命令/API 于 2026-09 对照官方文档校准
> **生成日期:** 2026-09-09

## 为什么学这个

设想三个接单时会真实遇到的场景。项目一部署到 Render，客户说「我本地跑不起来」——你发去 `requirements.txt`，客户装完发现版本和你的对不上；你的 API key 上周不小心 commit 进了 GitHub 公开仓库，被扫号脚本盗刷；LLM 接口偶发 429 限流，你想查是哪个请求触发的，日志里只有一堆 print 出来的字符串，什么都筛不出来。

这三个场景对应三个问题：**依赖能不能一键复现**、**配置（尤其密钥）怎么安全地进入代码**、**日志能不能被检索**。它们不影响功能，却直接决定「能跑的 demo」和「能交付的项目」的差距——接单场景里，客户复现不了环境、密钥泄漏、出问题查不了日志，任何一条都足以变成差评。

三个工具各管一段：**uv** 管依赖（替代 pip + venv 的组合），**pydantic-settings** 管配置（是周一 pydantic 笔记在配置场景的直接延伸），**structlog** 管日志（替代 print 和裸 logging）。学完当天下午，就能把本周 FastAPI + async + pytest 的产出组装成 `ai-service-template` 骨架——计划里项目一、二、三全部从这个骨架起步。

前置衔接：venv/pip「为什么必须隔离环境」已在 [modules-and-venv.md](modules-and-venv.md) 讲过；`.env` + `.gitignore` 的秘密管理基础已在 [dotenv-secrets-management.md](dotenv-secrets-management.md) 讲过。本篇从那两篇的结论处继续往前走，不重复。

## 重点标注（星级导航）

> 星级标准：★★★ 必须精通（学完当天能默写关键代码）｜★★☆ 需要理解（能讲清楚原理，细节可查）｜★☆☆ 了解即可（用到再回来看）

| 章节 | 星级 | 为什么（对照学习目的/计划周次） |
| ---- | ---- | ---- |
| 第 1 章 uv 项目工作流 | ★★★ | 每个项目的第一天都用它建骨架；交付给客户的 repo 靠 `uv sync` 一键复现，是接单交付的硬要求 |
| 第 2 章 pydantic-settings | ★★★ | LLM API key 管理的标配写法；W2 起所有项目读 `.env` 全靠它，配置缺漏在启动瞬间报错，好过运行到一半才炸 |
| 第 3 章 structlog | ★★☆ | 其中 3.1–3.2 单独 ★★★——W11 接 Langfuse 之前，日志是唯一排障手段，`bind` + `request_id` 必须会；processors 深度定制了解即可 |
| 综合示例：骨架 repo | ★★★ | 就是周五下午交付物本身，W2 起项目一/二/三都从这里复制起步 |

**建议学习路线（对应周五上午 3 节 + 下午组装）**：第 1 章 → 第 2 章 → 第 3 章精读 3.1/3.2 → 直接动手做综合示例；3.3 的 processor 机制扫一遍即可。

## 第 1 章 uv：项目依赖管理 ★★★

### 1.1 概念解释

在 uv 出现之前，Python 项目依赖管理是三件套：`venv` 建虚拟环境、`pip install` 装包、`pip freeze > requirements.txt` 记清单。痛点明显：`requirements.txt` 只记「你直接装了什么」，不精确锁定「依赖的依赖」的版本，换台机器装出来的结果可能不同；三个工具各管一段，命令又慢又碎。

**uv** 是 Astral 公司用 Rust 写的 Python 包管理器，把「Python 版本管理 + 虚拟环境 + 装包 + 版本锁定（lockfile，锁文件）」合并成一个命令行工具，速度比 pip 快一到两个数量级。对照你熟悉的 Java：uv ≈ Maven/Gradle + SDKMAN 的合体——`pyproject.toml` 相当于 `pom.xml`（声明直接依赖），`uv.lock` 相当于锁定后的完整依赖树（精确版本）。

项目里三个文件的分工（uv 自动维护，你只需要认识它们）：

| 文件 | 类比 | 放什么 | 提交到 git |
| ---- | ---- | ---- | ---- |
| `pyproject.toml` | pom.xml | 项目元数据 + 你**直接**声明的依赖（版本范围） | ✅ |
| `uv.lock` | 锁定后的依赖树 | 全部依赖（含传递依赖）的**精确版本**，跨平台可复现 | ✅ |
| `.python-version` | — | 本项目固定用的 Python 版本号 | ✅ |

`.venv/` 虚拟环境目录不提交——uv 随时能按锁文件重建它。

### 1.2 代码示例

Windows PowerShell 安装（官方一行命令，装完重开终端生效）：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

从零建出周五交付的骨架，这一串命令就是全天工作流：

```bash
# 1. 建项目（生成 pyproject.toml、.python-version、README、示例脚本）
uv init ai-service-template
cd ai-service-template
uv python pin 3.12        # 固定解释器版本，写入 .python-version

# 2. 装运行依赖（自动创建 .venv，并更新 pyproject.toml + uv.lock）
uv add fastapi "uvicorn[standard]" httpx pydantic-settings structlog

# 3. 装开发依赖（只在本机开发/测试用，不进生产）
uv add --dev pytest pytest-asyncio respx

# 4. 跑起来——注意是 uv run，不是裸 python
uv run uvicorn app.main:app --reload
uv run pytest
```

日常只需四个高频动作：**改依赖用 `uv add` / `uv remove`；跑任何命令用 `uv run`；换机器或部署后用 `uv sync` 按锁文件精确复原环境**。`uv sync --frozen` 表示「严格按 `uv.lock` 装、不许重新解析」，CI 和部署机上用它。

另外记住 `uvx`（等价 `uv tool run`）：在一次性临时环境里跑某个工具，不污染任何项目，例如 `uvx ruff check .`——接单时快速检查别人的仓库非常好用。

### 1.3 ⚠️ 常见坑

1. **直接 `python main.py` 报 `ModuleNotFoundError`**：系统 `python` 用的是全局解释器，没装项目依赖。要么 `uv run python main.py`，要么先激活环境（`.venv\Scripts\activate`）。养成 `uv run` 习惯后可以完全不激活。
2. **手改 `uv.lock`**：锁文件是生成物。改依赖的正确姿势永远是改 `pyproject.toml`（通常通过 `uv add`）再让 uv 重新锁。
3. **接手老项目只有 requirements.txt**：`uv add -r requirements.txt` 一次性迁入，之后统一走 uv。
4. **以为 `.venv` 要提交**：不提交。README 写一句「安装 uv 后执行 `uv sync`」，任何人都能复原你的环境——这是交付物的验收标准之一。

## 第 2 章 pydantic-settings：类型安全的配置管理 ★★★

### 2.1 概念解释

先从一个最日常的场景说起。你写了一个调用 OpenAI 的脚本，开头长这样：

```python
API_KEY = "sk-abc123..."      # 写死在代码里
TIMEOUT = 60
```

能跑，但第二天就遇到麻烦：想把代码 push 到 GitHub，key 会跟着一起公开；想给客户部署一份，人家用的是另一把 key。于是你把这类「会变、且不能进 git」的值挪出代码，存进一个 `.env` 文件（纯文本，每行一条 `键=值`），程序启动时再读进来——这些值就叫**配置**。

读配置的原始写法长这样（[dotenv-secrets-management.md](dotenv-secrets-management.md) 讲过）：

```python
import os
from dotenv import load_dotenv

load_dotenv()                          # 把 .env 的内容倒进环境变量
timeout = os.environ["LLM_TIMEOUT"]    # 取出来的是字符串 "60"，不是数字 60
```

亲手用几天，你会撞上四个坑：

1. 读出来**全是字符串**。`timeout + 1` 直接报 `TypeError`，得自己写 `int(...)` 转换。
2. 键名拼错（比如 `LLM_TIMOUT` 少个 E），程序不会立刻报错，而是运行到那一行才抛 `KeyError`——可能已经是上线后的第 200 次请求。
3. `.env` 里哪项必填、哪项可选，全靠脑子记。别人接手你的项目，根本不知道要配什么。
4. 读取代码散落在各个文件里，改一处漏一处。

**pydantic-settings** 的解决办法：不写「读取代码」，而是先写一份**配置清单**——一个类，逐项声明「我要一个叫 `llm_timeout` 的配置，它是整数，默认 60」。程序启动时，它拿着这份清单，自动去环境变量和 `.env` 文件里把值找齐、转好类型；哪项找不到或转不了，**启动瞬间就报错，并且指名道姓是哪一项**。

类比：像搬家时给搬家公司的清单——「冰箱 1 台、书 20 箱」。工人（pydantic-settings）按清单去旧房子（环境变量 / `.env`）里搬，缺了哪样当场打电话问你，而不是搬到新家才发现。（如果你写过 Java，这就是 Spring Boot 里 `application.yml` + `@ConfigurationProperties` 的角色。）

标题里「类型安全」的意思：清单上写了 `int`，到手就一定是 `int`——`.env` 里的 `"60"` 自动转成数字 60；写了 `SecretStr`，打印时自动打码。这套校验能力就是你周一学的 pydantic，只是数据来源从「函数传参」换成了「环境变量和 `.env` 文件」。

### 2.2 代码示例

#### 第一步：30 秒最小实验（强烈建议亲手跑）

新建一个空文件夹，`uv init` 后 `uv add pydantic-settings`，然后放两个文件。

`.env`（文件名以点开头，和代码放同一目录）：

```bash
OPENAI_API_KEY=sk-test-123
LLM_TIMEOUT_SECONDS=60
```

`config.py`：

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    openai_api_key: str              # 必填：没有默认值，启动时必须找得到
    llm_timeout_seconds: int = 60    # 有默认值：找不到就用 60


settings = Settings()
print(settings.llm_timeout_seconds, type(settings.llm_timeout_seconds))
```

`uv run python config.py`，输出：

```text
60 <class 'int'>
```

盯住这个输出：`.env` 里写的明明是 `60`（文本文件里一切都是文本），拿到手已经是 `int` 了——这就是「类型安全」最直观的含义。

#### 第二步：故意搞坏，看报错长什么样

学这个库最重要的一步是**亲眼看看它怎么报错**。把 `.env` 里 `OPENAI_API_KEY` 那一行删掉，再跑一次：

```text
pydantic ValidationError: 1 validation error for Settings
openai_api_key
  Field required [type=missing]
```

进程在**启动第一行**就死了，并告诉你是 `openai_api_key` 这项缺了。对比原始写法「跑到第 200 行才 KeyError」，接单时客户配漏了哪项，报错信息直接替你回答了。

再把 `LLM_TIMEOUT_SECONDS=60` 改成 `LLM_TIMEOUT_SECONDS=abc` 跑一次——同样启动即报，告诉你 `abc` 转不成整数。这就是星级导航里那句「配置缺漏在启动瞬间报错，好过运行到一半才炸」。

#### 第三步：看懂清单的每一行

回到骨架项目里那份完整的 `app/config.py`：

```python
from functools import lru_cache

from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",        # 除了环境变量，再从项目根目录的 .env 读
        env_file_encoding="utf-8",
        extra="ignore",         # .env 里多出的、清单上没登记的键，忽略而不报错
    )

    app_env: str = "dev"                     # dev / prod
    openai_api_key: SecretStr                # SecretStr：打印/日志里自动打码成 **********
    anthropic_api_key: SecretStr | None = None   # 可选，不配就是 None
    llm_provider: str = "openai"
    llm_model: str = "gpt-4o-mini"
    llm_timeout_seconds: int = 60


@lru_cache                     # 缓存：整个进程只真正读取一次
def get_settings() -> Settings:
    return Settings()
```

三件事逐一说清：

- **字段名怎么和 `.env` 键名对上**：默认规则是「同名、忽略大小写」——字段 `openai_api_key` 自动匹配 `.env` 里的 `OPENAI_API_KEY`，不用写任何映射。
- **`SecretStr` 是什么**：pydantic 提供的「保密字符串」。只有调 `.get_secret_value()` 才拿得到原文；`print`、日志、`repr` 里一律显示 `**********`。API key 字段一律用它，防止哪天日志把 key 泄出去。
- **为什么不直接 `Settings()`，要包一层 `get_settings()`**：`Settings()` 每调用一次就重新读一遍环境变量和文件；加上 `@lru_cache` 后，第一次的结果被缓存，全进程复用同一份——既快，又保证所有人拿到的是同一份配置。

配套的 `.env.example`（提交到 git 当配置模板；真实 `.env` 不提交）：

```bash
# 复制为 .env 后填真实值
APP_ENV=dev
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
LLM_PROVIDER=openai           # openai | anthropic，W3 多模型适配的雏形
LLM_MODEL=gpt-4o-mini
LLM_TIMEOUT_SECONDS=60
```

#### 第四步：FastAPI 里接入（衔接周三学的 lifespan）

```python
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI

from app.config import Settings, get_settings


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    print(f"starting env={settings.app_env} provider={settings.llm_provider}")
    yield


app = FastAPI(lifespan=lifespan)


@app.get("/config-check")
async def config_check(settings: Settings = Depends(get_settings)):
    return {"provider": settings.llm_provider, "model": settings.llm_model}
```

`Depends(get_settings)` 是 FastAPI 的依赖注入写法：每个请求进来，框架自动调用 `get_settings()`，把配置对象递到接口函数手里——因为有 `lru_cache`，递来递去都是同一份。

#### 取值优先级：记一条链

同一个配置项可能同时出现在好几个地方，pydantic-settings 按固定顺序决定听谁的：

**显式传参 > 环境变量 > `.env` 文件 > 默认值**

想亲眼验证：`.env` 里写着 `LLM_MODEL=gpt-4o-mini`，然后代码里 `Settings(llm_model="claude-3-5-haiku")`——打印出来是 claude，因为显式传参优先级最高。这条链的实用价值：同一份代码三态运行——**测试**时显式传参覆盖（`Settings(openai_api_key="sk-test")`）、**本地**开发读 `.env`、**生产**用平台（Render）注入的环境变量，代码一个字不用改。

### 2.3 ⚠️ 常见坑

1. **大小写不用纠结，但 Windows 有个例外**：默认忽略大小写，字段 `openai_api_key` 和 `.env` 里的 `OPENAI_API_KEY` 天然对上。例外是 `case_sensitive=True` 这个选项在 Windows 上无效——Windows 的环境变量本身不分大小写（官方文档明确说明），别依赖它。
2. **列表等复杂类型必须写成 JSON 字符串**：字段是 `list[str]` 时，`.env` 里要写 `LLM_MODELS='["gpt-4o-mini", "claude-3-5-haiku"]'`。写成逗号分隔的 `a,b,c` 会在启动时校验失败——环境变量没有「列表」概念，pydantic-settings 只按 JSON 解析。
3. **API key 忘记用 `SecretStr`**：用 `str` 声明 key 字段，哪天 `print(settings)` 或日志带上它，原文就进了日志文件。`SecretStr` 一律显示 `**********`——W2 成本核算、W12 密钥管理都靠这个习惯打底。
4. **到处 `Settings()` 而不用 `get_settings()`**：每次实例化都重读一遍来源，浪费且可能读到不一致的状态。统一走 `get_settings()` 或 FastAPI 的 `Depends`；测试里想覆盖某一项，用 `Settings(openai_api_key="sk-test")` 显式传参——这正是「显式传参优先级最高」的设计用途。
5. **改了 `.env` 没重启进程**：`.env` 只在 `Settings()` 实例化那一刻读一次，运行中改文件不会生效；`uvicorn --reload` 只监听代码文件，改 `.env` 不会触发重启。本地调试改完配置，记得手动重启。

## 第 3 章 structlog：结构化日志 ★★☆

### 3.1 概念解释

print 的问题：只有肉眼能读。标准库 logging 的问题：`f"用户 {uid} 请求 {model} 花了 {ms}ms"` 对人可读，对机器不可检索——想在千行日志里筛出「某个 request_id 的全部日志」「所有耗时超过 5 秒的 LLM 调用」，纯文本做不到。

**结构化日志（structured logging）**：每条日志不再是一个字符串，而是一个**键值对字典**——一个 `event`（事件名）加若干字段。开发时渲染成彩色控制台，生产时渲染成 JSON；JSON 才能被日志平台或 `grep` + `jq` 按字段过滤、聚合、出看板。

**structlog** 是 Python 结构化日志的事实标准库。三个核心概念：

- **event dict（事件字典）**：一条日志就是一个字典。`log.info("chat_completed", model="gpt-4o-mini", latency_ms=812)` 里 `chat_completed` 是 event，后面全是键值对字段。
- **bind（绑定上下文）**：把一组字段预先绑在 logger 上，之后这个 logger 发出的每条日志自动携带这些字段——官方的说法是「logger 变成了你的闭包」。
- **processor（处理器）**：日志发出前依次穿过的处理函数链，链尾是渲染器（控制台彩色渲染器或 JSON 渲染器）。

对 AI 应用，结构化日志的价值 W2 就兑现：每次 LLM 调用记下 `model / tokens_in / tokens_out / latency_ms / request_id`，就是最原始的成本与延迟监控——W11 接 Langfuse 之前你只有它，接了之后 trace 用的也是同一批字段。

### 3.2 代码示例

`app/logging.py`——一份开发/生产两用的配置函数：

```python
import logging

import structlog


def configure_logging(env: str = "dev") -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,   # 合并每请求上下文（见下）
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,      # 异常信息变成字段
            # dev：彩色控制台；prod：JSON（一行一条，可按字段过滤）
            structlog.dev.ConsoleRenderer()
            if env == "dev"
            else structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        cache_logger_on_first_use=True,
    )
```

业务代码里的用法（AI 场景长这样）：

```python
import time

import structlog

logger = structlog.get_logger()


async def chat(model: str, prompt: str) -> str:
    log = logger.bind(model=model)          # 绑定公共字段
    t0 = time.perf_counter()
    log.info("llm_call_started")
    resp = await call_llm(model, prompt)
    log.info(
        "chat_completed",
        latency_ms=round((time.perf_counter() - t0) * 1000),
        tokens_in=resp.usage.prompt_tokens,
        tokens_out=resp.usage.completion_tokens,
    )
    return resp.text
```

FastAPI 中间件给每个请求绑 `request_id`（配合 `merge_contextvars`，该请求后续所有日志自动携带它）：

```python
import uuid

import structlog
from fastapi import Request


@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    structlog.contextvars.clear_contextvars()   # 清掉上一个请求的上下文
    rid = request.headers.get("x-request-id", str(uuid.uuid4()))
    structlog.contextvars.bind_contextvars(request_id=rid, path=request.url.path)
    response = await call_next(request)
    response.headers["x-request-id"] = rid
    return response
```

生产环境的输出（JSON，`jq '.latency_ms'` 直接可查）：

```json
{"request_id": "6f9c2e...", "path": "/chat", "model": "gpt-4o-mini",
 "event": "chat_completed", "level": "info",
 "timestamp": "2026-09-09T14:03:22.114402Z", "latency_ms": 812,
 "tokens_in": 1520, "tokens_out": 233}
```

### 3.3 ⚠️ 常见坑

1. **`configure()` 必须放在进程最早处**（`main.py` 顶部、任何 `get_logger()` 使用之前）；`cache_logger_on_first_use=True` 生效后再改配置不会反映到已缓存的 logger。
2. **忘了 `clear_contextvars()`**：上一个请求的 `request_id` 会串进下一个请求的日志——async 并发下这是经典事故，中间件开头必须清。
3. **Windows 彩色输出**需额外装 colorama（`uv add colorama`）；不装也不报错，只是没颜色。
4. **事件名用稳定字符串、数据放键值对**：不要 `log.info(f"completed {rid}")`，要 `log.info("chat_completed", request_id=rid)`——前者无法按字段检索，等于白结构化。

## 综合示例：ai-service-template 骨架（周五下午交付物）

把上午三件事和本周全部产出组装成一个 repo，之后三个项目都从这里复制起步。目录结构：

```text
ai-service-template/
├── pyproject.toml            # uv 维护：fastapi / uvicorn / httpx / pydantic-settings / structlog + dev 组
├── uv.lock                   # 提交；uv sync 靠它复现
├── .python-version           # 3.12
├── .env.example              # 第 2 章那份，提交
├── .env                      # 本地真实值，不提交（.gitignore 已含）
├── app/
│   ├── __init__.py
│   ├── config.py             # Settings + get_settings()（第 2 章）
│   ├── logging.py            # configure_logging()（第 3 章）
│   └── main.py               # FastAPI：/health、SSE 流式接口、request_id 中间件
└── tests/
    ├── conftest.py           # 周四的 fixture：mock 外部 HTTP、注入测试用 Settings
    └── test_health.py
```

`app/main.py` 把三块拼起来（骨架的核心部分）：

```python
import uuid
from contextlib import asynccontextmanager

import structlog
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

from app.config import get_settings
from app.logging import configure_logging


@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    configure_logging(settings.app_env)          # 日志配置越早越好
    logger = structlog.get_logger()
    logger.info("app_started", env=settings.app_env,
                provider=settings.llm_provider)  # key 不会泄漏（SecretStr）
    yield
    logger.info("app_stopped")


app = FastAPI(lifespan=lifespan)


@app.middleware("http")
async def request_id_middleware(request: Request, call_next):
    structlog.contextvars.clear_contextvars()
    rid = request.headers.get("x-request-id", str(uuid.uuid4()))
    structlog.contextvars.bind_contextvars(request_id=rid)
    response = await call_next(request)
    response.headers["x-request-id"] = rid
    return response


@app.get("/health")
async def health() -> dict:
    return {"status": "ok"}


async def echo_stream(prompt: str):
    logger = structlog.get_logger()
    for word in f"echo: {prompt}".split():
        yield word + " "
    logger.info("stream_finished")


@app.get("/stream")
async def stream(q: str) -> StreamingResponse:
    return StreamingResponse(echo_stream(q), media_type="text/event-stream")
```

（SSE 端点套周三的 `StreamingResponse` 写法，测试套周四的 respx mock，此处骨架只演示拼装。）

验收清单（对照计划 W1 交付物标准）：

```bash
uv sync                                  # 干净机器上复原依赖
cp .env.example .env                     # 填入真实 key
uv run pytest                            # 全绿
uv run uvicorn app.main:app --reload     # /health 返回 ok，日志带 request_id
```

README 写清这四步，客户五分钟跑起来——「任何人可复现」本身就是接单竞争力。

## 附录 A：速查表

| 命令 / API | 作用 | 关键参数 / 注意点 |
| ---- | ---- | ---- |
| `uv init` | 建项目 | 生成 pyproject.toml、.python-version、示例脚本 |
| `uv add PKG` | 加依赖 | 自动建 `.venv` 并更新 lock；`--dev` 进开发依赖组 |
| `uv remove PKG` | 删依赖 | 同步修改 pyproject + lock |
| `uv sync` | 按锁文件复原环境 | 部署/CI 加 `--frozen`，严格按 lock 装 |
| `uv run CMD` | 在项目环境里执行 | 替代「激活再跑」的首选方式 |
| `uvx TOOL` | 临时环境跑工具 | 不污染任何项目环境 |
| `uv python pin 3.12` | 固定解释器版本 | 写入 `.python-version` |
| `uv add -r requirements.txt` | 迁移老项目 | 一次性导 |
| `SettingsConfigDict(env_file=...)` | 声明配置来源 | 常用项：`env_prefix` / `extra` / `case_sensitive` |
| `SecretStr` | 密钥字段类型 | 日志与 repr 自动打码 |
| `get_settings()` + `lru_cache` | 配置单例 | 测试用 `Settings(k=v)` 显式覆盖 |
| `structlog.get_logger()` | 取 logger | 任何模块直接调用，无需传递 |
| `logger.bind(k=v)` | 绑定上下文 | 不可变，返回携带新字段的 logger |
| `clear_contextvars()` / `bind_contextvars()` | 每请求上下文 | 中间件开头必须先 clear |
| `ConsoleRenderer` / `JSONRenderer` | 渲染器 | dev / prod 二选一 |
| `make_filtering_bound_logger(INFO)` | 级别过滤 | 替代 `logging.basicConfig` 的作用 |

## 附录 B：官方文档链接

- uv 功能总览（2026-09 校准）：<https://docs.astral.sh/uv/getting-started/features/>
- uv 项目工作流指南：<https://docs.astral.sh/uv/guides/projects/>
- pydantic-settings 官方文档：<https://docs.pydantic.dev/latest/concepts/pydantic_settings/>
- structlog 入门（2026-09 校准）：<https://www.structlog.org/en/stable/getting-started.html>
- structlog 配置详解：<https://www.structlog.org/en/stable/configuration.html>
