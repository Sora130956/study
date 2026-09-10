# uv：Python 项目依赖管理

- 目标读者：从其他语言（npm / cargo / go mod）转来的开发者，已懂"依赖 + 锁文件 + 虚拟环境"这套概念
- 前置知识：见第 3 节（术语速查表可扫一眼直接进第 4 节）
- 学习时长：约 30 分钟 / 可跳过：第 3 节正文（你多半已懂，扫术语表即可）

---

## 第 1 节：它是什么（一句话 + 一句类比）

**一句话**：uv 是一个用 Rust 写的、把 Python 的"装依赖、管虚拟环境、管 Python 版本、管锁文件"全部打包进一个命令的工具。

**类比**：uv 之于 Python，就像 **cargo 之于 Rust**，或者 **npm + nvm + package-lock.json 三合一** 之于 Node。

- 你以前在 Node 里要 `npm install`（装依赖）+ `nvm use`（切 Node 版本）+ `package-lock.json`（锁版本），三个东西分开管。
- uv 把这套东西合成一个 `uv` 命令：`uv add` 装依赖、`uv python install` 管 Python 版本、`uv.lock` 锁版本，全在一个工具里。

它最出名的一点是**快**——因为底层用 Rust 重写，装依赖通常比 pip 快 10~100 倍。但"快"只是副产品，真正值钱的是它把散落一地的工具收拢成一个。

---

## 第 2 节：它解决什么问题 / 为什么需要它

### 反面推演：不用 uv，Python 项目长什么样

一个典型的"传统 Python 项目"要跑起来，你得手动串起至少 4 个工具：

```bash
# 1. 先装一个 Python 解释器（去官网下载，或 pyenv 装）
pyenv install 3.12

# 2. 建虚拟环境（隔离依赖，避免污染全局）
python -m venv .venv
source .venv/bin/activate   # Windows 是 .venv\Scripts\activate

# 3. 装依赖（pip 没有锁文件，版本会漂移）
pip install requests
pip freeze > requirements.txt   # 手动导出，容易漏、容易脏

# 4. 换台机器 / 换个人，再重复一遍，且版本可能对不上
```

**可感知的痛点**（不是抽象的"不好维护"，是具体到动作）：

1. **要记 4 个工具、4 套命令**：pyenv 管版本、venv 管隔离、pip 管安装、pip-tools 管锁文件。新手光"先激活哪个"就能卡半天。
2. **`requirements.txt` 锁不住版本**：它只记"直接依赖"，不记"依赖的依赖"（传递依赖）。你装 `requests`，它背后还拉 `urllib3`、`certifi`，这些版本没锁，下次装可能就变了。
3. **换机器要重演一遍**：没有 `package-lock.json` 那种"照着锁文件精确还原"的机制，环境不可复现。
4. **慢**：pip 一个个下载、一个个装，大项目装一次能等几分钟。

### 问题根源（一句话）

**根源是 Python 生态把"依赖管理"拆成了好几个互不通信的工具，每个只干一半，剩下靠人肉串起来。** uv 的解法就是把这些工具合并成一个，并且用锁文件把"装了什么、什么版本"钉死。

### 经典应用场景

- 新建一个 Python 项目，从零搭环境（`uv init` + `uv add`）
- 团队协作，保证每个人/每台 CI 机器装出来的依赖**完全一致**（靠 `uv.lock`）
- 管理多个 Python 版本（`uv python install 3.12`，不用再装 pyenv）
- 装命令行工具（`uv tool install ruff`，类似 `npm install -g`）

---

## 第 3 节：读下去之前，先搞懂这些概念（前置知识）

> 你从 npm/cargo 转来，这些概念你多半都见过，只是 Python 里叫法不同。扫一眼术语表，直接进第 4 节即可。

### 概念逐个讲

**1. 依赖（dependency）**
- 定义：你的项目要用的第三方库。
- 大白话：你 `import requests`，`requests` 就是依赖。
- 对应：npm 的 `dependencies`、cargo 的 `[dependencies]`。

**2. 传递依赖（transitive dependency）**
- 定义：你依赖的库，它自己又依赖的库。
- 大白话：你装 `requests`，它内部要用 `urllib3`，`urllib3` 就是传递依赖。你没直接写它，但它必须被装上。
- 对应：`node_modules` 里那些你没手动装、却出现在里面的包。

**3. 锁文件（lockfile）**
- 定义：把"最终装了哪些包、每个包精确到哪个版本"完整记录下来的文件。
- 大白话：一张"这次装了什么、什么版本"的**快照**，下次照着它原样还原。
- 对应：`package-lock.json` / `Cargo.lock`。uv 里叫 `uv.lock`。

**4. 虚拟环境（virtual environment）**
- 定义：给单个项目单独划一块装依赖的隔离空间，互不干扰。
- 大白话：项目 A 用 `requests 2.0`，项目 B 用 `requests 3.0`，各装各的，不打架。
- 对应：`node_modules`（每个项目一份）。uv 里默认建在 `.venv/` 目录。

**5. pyproject.toml**
- 定义：Python 项目的"清单文件"，声明项目名、Python 版本要求、依赖列表。
- 大白话：项目的"身份证 + 购物清单"。
- 对应：`package.json` / `Cargo.toml`。

**6. 依赖组（dependency group）**
- 定义：把依赖分组，比如"开发时用的"和"运行时用的"分开。
- 大白话：`pytest` 只在开发/测试时用，不该跟着项目一起发布，就放进 `dev` 组。
- 对应：npm 的 `devDependencies`。

### 术语速查表

| 术语 | 大白话解释 | 对应到正文哪节 |
|------|-----------|--------------|
| 依赖 | 你要用的第三方库 | 第 4 节示例 1 |
| 传递依赖 | 依赖的依赖，你没写但必须装 | 第 2 节痛点 2 |
| 锁文件 `uv.lock` | 装了什么、什么版本的快照 | 第 4 节示例 2 |
| 虚拟环境 `.venv` | 项目专属的隔离安装区 | 第 4 节示例 1 |
| `pyproject.toml` | 项目清单文件 | 第 4 节示例 1 |
| 依赖组 | 开发依赖 vs 运行依赖分开 | 第 5 节 |

---

## 第 4 节：从最简单的代码示例开始（循序渐进）

> 下面三个示例是**递进**的：示例 1 只让 uv "活着"，示例 2 加锁文件，示例 3 贴近真实项目结构。跟着敲一遍即可。

### 示例 1：最小可运行版本 —— 建一个项目、装一个依赖、跑起来

```bash
# 1. 初始化一个项目（uv 会生成 pyproject.toml、.python-version、main.py）
uv init my-app          # 建一个叫 my-app 的项目目录，并生成基础文件

# 2. 进入项目目录
cd my-app

# 3. 装一个依赖（uv 会自动建虚拟环境、下载包、更新 pyproject.toml）
uv add requests         # 等价于 npm install requests / cargo add requests

# 4. 在项目环境里跑代码（uv run 会自动用 .venv 里的环境，不用手动 activate）
uv run python main.py   # 等价于 npm run / cargo run
```

`uv init` 生成的 `pyproject.toml` 长这样（这是项目的"清单文件"）：

```toml
[project]
name = "my-app"              # 项目名
version = "0.1.0"            # 版本号
description = "Add your description here"   # 项目描述
requires-python = ">=3.12"   # 要求 Python 版本 >= 3.12
dependencies = []            # 依赖列表，uv add 会往这里写
```

**逐段解读**：

- **`uv init`**：相当于 `npm init` / `cargo new`。它生成 `pyproject.toml`（清单）、`.python-version`（指定用哪个 Python）、`main.py`（入口文件）。这一步只建骨架，不装任何东西。
- **`uv add requests`**：这是核心动作。它一次干三件事——① 建虚拟环境 `.venv`（如果还没有）；② 下载 `requests` 及其传递依赖；③ 把 `requests` 写进 `pyproject.toml` 的 `dependencies`。你不需要手动 `venv` + `pip install` 两步。
- **`uv run python main.py`**：`uv run` 会**自动激活**项目虚拟环境再执行后面的命令。你不需要先 `source .venv/bin/activate`。这是 uv 最省心的地方之一。

**这一步你得到了什么**：一个能跑的项目骨架 + 一个装好依赖的隔离环境，全程只用了 3 个命令，没有手动建 venv、没有手动 activate。

---

### 示例 2：加一个真实小需求 —— 锁文件让环境可复现

真实项目里，你装完依赖后，会生成一个 `uv.lock` 文件。它记录"最终装了哪些包、精确到哪个版本"，保证换台机器装出来**一模一样**。

```bash
# 1. 继续用示例 1 的项目，再装一个依赖
uv add httpx            # 装 httpx（一个 HTTP 客户端库）

# 2. 看看项目目录现在多了什么
ls                      # 你会看到：pyproject.toml、uv.lock、.venv、main.py

# 3. 换台机器 / 删掉 .venv 后，用锁文件精确还原环境
uv sync                 # 照着 uv.lock 把依赖原样装回来（等价于 npm ci / cargo build）
```

`uv add httpx` 之后，`pyproject.toml` 的 `dependencies` 变成了：

```toml
dependencies = [
    "requests>=2.0",   # 你声明的依赖，带版本范围（>= 表示"至少 2.0"）
    "httpx>=0.27",      # 新加的依赖
]
```

而 `uv.lock` 里记录的是**精确版本**（示意，非真实内容）：

```toml
# uv.lock 片段：把每个包钉死到具体版本，包括传递依赖
[[package]]
name = "requests"
version = "2.32.3"        # 精确版本，不是 ">=2.0" 这种范围

[[package]]
name = "urllib3"          # 这是 requests 的传递依赖，也被锁住了
version = "2.2.3"
```

**逐段解读**：

- **`pyproject.toml` 记"意图"，`uv.lock` 记"结果"**：前者写"我要 requests 2.0 以上"，后者写"这次实际装的是 2.32.3，连它带的 urllib3 2.2.3 也钉死"。这跟 `package.json`（意图）vs `package-lock.json`（结果）完全对应。
- **`uv sync`**：这是"照着锁文件还原"的命令。团队里别人 `git clone` 你的项目后，跑 `uv sync`，装出来的依赖和你**逐字节一致**。它等价于 npm 的 `npm ci`（严格按锁文件装）。
- **为什么锁文件重要**：没有它，`requests>=2.0` 今天装 2.32，下个月可能装 2.33，行为悄悄变了，bug 难查。锁文件把这种"漂移"钉死。

**这一步你得到了什么**：你的项目现在有了"可复现"能力——任何人、任何机器，`uv sync` 一下就能得到和你完全相同的环境。这是团队协作和 CI 的基石。

---

### 示例 3：贴近项目的样子 —— 开发依赖分组 + 项目结构

真实项目里，测试工具（如 `pytest`）只在开发时用，不该跟着项目一起发布。uv 用**依赖组**来区分。

```bash
# 1. 装一个"开发时用"的依赖，放进 dev 组
uv add --dev pytest     # 等价于 npm install -D pytest / cargo add --dev

# 2. 跑测试（uv run 会自动用项目环境，pytest 在 dev 组里也能用）
uv run pytest           # 运行测试

# 3. 看 pyproject.toml 现在长什么样
```

`pyproject.toml` 现在多了 `[dependency-groups]` 段：

```toml
[project]
name = "my-app"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.0",     # 运行时依赖：项目跑起来必须要有
    "httpx>=0.27",
]

[dependency-groups]      # 开发依赖：只在开发/测试时用
dev = [
    "pytest>=8.0",       # 测试框架，发布时不需要
]
```

**逐段解读**：

- **`uv add --dev pytest`**：`--dev` 标志把依赖放进 `dev` 组，而不是 `dependencies`。这对应 npm 的 `devDependencies`——"开发时用，发布时排除"。
- **`uv run pytest`**：`uv run` 后面跟任何命令，都会在项目环境里执行。所以 `uv run pytest` 就是"在项目环境里跑 pytest"，不用先 activate。
- **为什么分组**：如果 `pytest` 混进 `dependencies`，别人装你的项目时会把测试框架也装上，白白多装一堆东西。分组让"运行时依赖"和"开发依赖"各归各位。

**这一步你得到了什么**：你的项目现在有了真实项目的完整形态——运行时依赖、开发依赖、锁文件、虚拟环境，全部由 uv 统一管理，结构清晰、可复现。

---

## 第 5 节：最小可用的实际用法（生产场景模板）

> 下面是一套**可直接复制进真实项目**的完整流程。注释标了哪些是必须、哪些可选。

```bash
# ============ 一、初始化项目（必须） ============
uv init my-project            # 建项目骨架
cd my-project

# ============ 二、指定 Python 版本（可选，但推荐） ============
uv python install 3.12        # 装一个 Python 3.12（uv 自带版本管理，不用 pyenv）
uv python pin 3.12            # 把项目钉在 3.12，写进 .python-version

# ============ 三、装依赖（必须） ============
uv add fastapi uvicorn        # 运行时依赖：Web 框架 + 服务器
uv add --dev pytest httpx     # 开发依赖：测试框架 + 测试用 HTTP 客户端

# ============ 四、同步环境（必须，团队协作/CI 用） ============
uv sync                       # 照着 uv.lock 精确还原环境

# ============ 五、日常运行（必须） ============
uv run uvicorn main:app --reload   # 在项目环境里启动服务
uv run pytest                      # 在项目环境里跑测试
```

**哪几行是必须的、哪几行可删**：

- **必须**：`uv init`、`uv add`、`uv sync`、`uv run`——这是最小闭环，缺一个项目跑不起来。
- **可选**：`uv python install` / `uv python pin`——如果你机器上已有合适 Python 版本，可跳过；但团队协作时**强烈建议 pin**，保证大家用同一个 Python 版本。
- **可删**：`--dev` 那行——如果项目暂时没测试，可以先不装开发依赖。

**实际项目里通常还会配什么（进阶，可先跳过）**：

- **CI 集成**：在 GitHub Actions 里，`uv sync` 之后跑 `uv run pytest`，保证每次提交都测一遍。uv 官方有现成的 `astral-sh/setup-uv` action。
- **锁文件提交进 git**：`uv.lock` 要提交进版本库（和 `package-lock.json` 一样），`.venv/` 要加进 `.gitignore`（和 `node_modules` 一样）。
- **配置项**：`pyproject.toml` 里可加 `[tool.uv]` 段配置 uv 行为，比如指定镜像源、指定 Python 版本策略。初学先不用管。

---

## 第 6 节：常见坑与易错点

**坑 1：忘了 `uv run`，直接 `python main.py` 报 `ModuleNotFoundError`**

- 报错：`ModuleNotFoundError: No module named 'requests'`
- 现象：你明明 `uv add requests` 了，但直接 `python main.py` 还是找不到。
- 原因：`uv add` 把包装进了 `.venv`，但直接 `python` 用的是**全局** Python，没进虚拟环境。
- 解决：用 `uv run python main.py`（uv 会自动进 `.venv`），或者手动 `source .venv/bin/activate` 后再 `python main.py`。

**坑 2：改了 `pyproject.toml` 但没跑 `uv sync`，环境没更新**

- 现象：你手动在 `pyproject.toml` 里加了个依赖，但代码里 `import` 还是报错。
- 原因：改 `pyproject.toml` 只是改了"清单"，没真正装。`uv add` 会自动装，但**手动改文件不会**。
- 解决：手动改完 `pyproject.toml` 后，跑 `uv sync` 让环境跟上。或者干脆用 `uv add` 来加依赖，别手改。

**坑 3：把 `.venv` 提交进了 git**

- 现象：仓库里出现一个巨大的 `.venv/` 目录，别人 clone 慢、还容易冲突。
- 原因：`.venv` 是本地环境，不该进版本库（和 `node_modules` 一个道理）。
- 解决：在 `.gitignore` 里加一行 `.venv/`。`uv init` 生成的 `.gitignore` 通常已包含，但老项目要自己加。

**坑 4：`uv.lock` 没提交，团队环境不一致**

- 现象：同事 clone 后 `uv sync`，装出来的版本和你不一样，出现"我这边能跑你那边报错"。
- 原因：`uv.lock` 是"精确版本快照"，不提交它，别人只能按 `pyproject.toml` 的版本范围装，会漂移。
- 解决：**`uv.lock` 必须提交进 git**（和 `package-lock.json` 一样），`.venv/` 才该忽略。

**和我直觉相反的一点**：`uv add` 会**同时**改 `pyproject.toml` 和 `uv.lock` 两个文件，还会自动建 `.venv`。很多人以为"装依赖"只是下载包，其实 uv 把"声明依赖 + 锁版本 + 建环境"三件事一起做了。所以别惊讶于 `uv add` 之后目录里突然多了 `.venv` 和 `uv.lock`。

---

## 第 7 节：验证你学会了（自测题）

**① 复述**：用一句话说清 uv 是什么、解决什么问题。（答案在第 1、2 节）

**② 默写**：不看文档，写出"新建项目 → 装一个依赖 → 跑起来"的最小三条命令。（答案在第 4 节示例 1）

**③ 变体应用**：你要新建一个 Web 项目，运行时用 `fastapi`，开发时用 `pytest`，并且要保证团队里每个人装出来的环境一致。写出完整的命令序列，并说明哪个文件要提交进 git、哪个要忽略。（答案在第 5 节 + 第 6 节坑 3、坑 4）
