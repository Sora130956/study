# 使用 uv 创建并运行 FastAPI 工程

## 1. 初始化项目

使用 `uv init` 创建项目：

```bash
uv init my-fastapi-app --app
cd my-fastapi-app
```

`--app` 会创建一个应用项目，通常包含 `main.py`。
## 1.1. 实际使用：创建不包含默认src目录的工程

```bash
uv init ai-service-template --no-package
cd ai-service-template
```

## 2. 添加 FastAPI 依赖

使用 `uv add` 添加 FastAPI 标准依赖：

```bash
uv add "fastapi[standard]"
```

该命令会：

- 创建虚拟环境 `.venv`
- 将依赖写入 `pyproject.toml`
- 生成 `uv.lock` 锁文件
## 2.1. 这个标准依赖包含了：

| 包名                      | 作用                         | 是否核心依赖 |
| ----------------------- | -------------------------- | ------ |
| **`fastapi`**           | Web 框架本体                   | 是      |
| **`pydantic`**          | 数据验证与序列化                   | **是**  |
| **`starlette`**         | FastAPI 的底层 Web 框架基础       | 是      |
| **`uvicorn[standard]`** | 运行应用的 ASGI 服务器             | 否，标准依赖 |
| **`fastapi-cli`**       | 提供 `fastapi dev` 等命令行工具    | 否，标准依赖 |
| **`httpx`**             | 用于 `TestClient` 的 HTTP 客户端 | 否，标准依赖 |
| **`python-multipart`**  | 解析表单数据和文件上传                | 否，标准依赖 |
| **`jinja2`**            | 模板渲染支持                     | 否，标准依赖 |
| **`email-validator`**   | 邮箱字段验证                     | 否，标准依赖 |
| **`pydantic-settings`** | 管理应用配置                     | 否，标准依赖 |

## 3. 编写应用代码

编辑 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello, FastAPI with uv!"}
```

## 4. 运行开发服务器

```bash
uv run uvicorn app.main:app --reload
```

默认运行在 `http://127.0.0.1:8000`，支持代码热重载。

- 应用地址：http://127.0.0.1:8000
- 交互式文档：http://127.0.0.1:8000/docs

## 4.1 命令解释
uv run        fastapi dev        app/main.py
│              │                         │
│              │                         └─ 指定 FastAPI 应用入口文件
│              └─ 启动 FastAPI 开发服务器
└─ 在 uv 管理的虚拟环境中执行后面的命令

## 5. 运行生产服务器

```bash
uv run fastapi run main.py
```

## 常用命令速查

| 目的              | 命令                             |
| --------------- | ------------------------------ |
| 初始化项目           | `uv init my-fastapi-app --app` |
| 添加 FastAPI 标准依赖 | `uv add "fastapi[standard]"`   |
| 运行开发服务器（热重载）    | `uv run fastapi dev main.py`   |
| 运行生产服务器         | `uv run fastapi run main.py`   |

# 6. 为啥装了依赖Pycharm还是报红 依赖不能解析
**需要先通过 `uv` 安装依赖**，同时还要让 PyCharm 使用 `uv` 创建的那个虚拟环境（`.venv`），否则 PyCharm 就会显示“无法解析”。

## 6.1. 让 PyCharm 使用这个 `.venv`

PyCharm 默认可能用的是系统 Python 或另一个解释器，所以找不到这些包。需要手动切换：

1. 打开 `File` → `Settings`（Windows）或 `PyCharm` → `Settings`（macOS）。
    
2. 进入 `Project: <你的项目名>` → `Python Interpreter`。
    
3. 点击右上角齿轮 → `Add Interpreter` → `Add Local Interpreter`。
    
4. 选择 `Existing`，然后找到项目下的：
    
    - Windows：`.venv\Scripts\python.exe`
        
    - macOS / Linux：`.venv/bin/python`
        
5. 确定后，等待 PyCharm 重新索引（右下角进度条走完）。
    

之后 `import httpx`、`from fastapi import ...` 就不会再报红了。

## 6.2. 快速切换解释器的小技巧

PyCharm 右下角状态栏会显示当前解释器名称。  
直接点它 → `Interpreter Settings` → 选择 `.venv` 里的 Python 即可。
![[Pasted image 20260911080454.png]]![[Pasted image 20260911080835.png]]