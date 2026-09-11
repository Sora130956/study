# Managing Secrets with .env & python-dotenv

> **来源:** 实践总结（requests 调 marketstack API 时遇到的 API key 管理问题）
> **对应计划周次:** 第 2 周 · 周一 requests 实践（调公开 API）

## 核心理解

代码要 push 到 GitHub 时，API key、密码、token 这类秘密**绝不能明文写进代码文件**——因为代码文件本身要公开。`.gitignore` 只能让 git 不追踪某个文件，但无法「保护」已经写在待提交代码里的秘密，你也不可能把整个脚本加进 `.gitignore`。

正确思路是**把秘密从代码里挪出去**：秘密单独放进一个本地文件（`.env`），用 `.gitignore` 让它不被 push；代码运行时通过 `python-dotenv` 把 `.env` 里的键值对加载进「环境变量」，再用 `os.environ` 按键名取值。这样代码可以公开，秘密只留在本地。这套模式是行业通用做法，Spring Boot 等后端框架也是同一套思路（外部化配置 + 环境变量），学的就是以后 Mini CRM 会一直用的。

## 关键点

### 四个文件的分工

| 文件 | 放什么 | 是否 push | 作用 |
|------|--------|-----------|------|
| `.env` | 真实秘密（`KEY=真实值`） | ❌ 不 push | 本地存放秘密 |
| `.env.example` | 只有键名（`KEY=`） | ✅ push | 模板，告诉别人要配哪些变量 |
| `.gitignore` | 加一行 `.env` | ✅ push | 让 git 忽略 `.env` |
| 代码文件 | 从 `os.environ` 读 | ✅ push | 不出现明文秘密 |

⚠️ 只忽略 `.env`，**不要**忽略 `.env.example`——一个藏秘密，一个当模板，两者都要提交（除 .env 外）。

### .env 文件格式

放在项目根目录，每行一个 `键=值`：

```
MARKETSTACK_ACCESS_KEY=真实key
```

### .gitignore 配置

在 `.gitignore` 末尾加：

```
# Secrets
.env
```

### 代码里读取秘密的链路

```python
import os
from pathlib import Path

import requests
from dotenv import load_dotenv

load_dotenv(Path(__file__).resolve().parents[2] / ".env")

payload = {
    "symbol": "AAPL",
    "access_key": os.environ["MARKETSTACK_ACCESS_KEY"],
}
requests.get("https://api.marketstack.com/v1/eod", params=payload)
```

完整链路分三步：

1. `load_dotenv(...)` —— 读 `.env` 文件，把每一行 `键=值` 塞进「环境变量」这个系统级临时字典里。
2. `os.environ["MARKETSTACK_ACCESS_KEY"]` —— 从环境变量按键名取出值。
3. 代码从头到尾没有真实 key，key 只存在于本地 `.env`，而 `.env` 被 git 忽略。

### .env 路径的坑：默认从「当前工作目录」向上找

`load_dotenv()` 不传参时，默认从**当前工作目录**开始往上层找 `.env`。如果脚本在子目录、又从别处运行，可能找不到。稳妥做法是写死路径：

```python
load_dotenv(Path(__file__).resolve().parents[2] / ".env")
```

- `__file__`：当前脚本路径。
- `.resolve()`：转成绝对路径。
- `.parents[2]`：从脚本往上跳 3 层。例：`scripts/request/stock.py` → request → scripts → 项目根，正好定位到根目录的 `.env`。这样无论从哪个目录运行都能找到。

### 前置：安装依赖

`python-dotenv` 是第三方库，要在项目虚拟环境里装：

```
pip install python-dotenv
```

装完可以用 `pip freeze > requirements.txt` 把依赖记进清单。

### push 前的安全确认

```
git status --ignored
```

`.env` 应出现在 Ignored files 里、不在待提交列表中，才算安全。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| environment variable | n. | 环境变量，进程运行时可读的系统级键值对 |
| .env | n. | 存放环境变量的本地文件，不提交到 git |
| .env.example | n. | 只含键名的模板文件，提交到 git 供他人参考 |
| python-dotenv | n. | 把 `.env` 加载进环境变量的第三方库 |
| load_dotenv | v. | dotenv 提供的加载函数，读 `.env` 写入环境变量 |
| os.environ | n. | Python 访问环境变量的字典对象 |
| .gitignore | n. | 声明 git 不追踪哪些文件的配置文件 |
| secret | n. | 秘密（key/密码/token），不可明文进代码库 |
