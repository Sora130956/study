# 模块与包 + 虚拟环境（venv）+ pip · Java 视角对照

> **来源:** <https://liaoxuefeng.com/books/python/module/index.html>（模块）+ Python 官方 venv / pip 文档
> **对应计划周次:** 第 1 周 · 周四上午 · 模块与包、虚拟环境（venv）、pip
> **阅读方式:** 已有 Java 基础，本篇只讲「Python 和 Java 哪里对得上、哪里不一样」，聚焦接单时真正要用的依赖管理。

## 核心差异总览

| 维度 | Java | Python |
| ---- | ---- | ------ |
| 代码文件 | 一个 `.java`（一个 public class） | 一个 `.py` = 一个**模块**（module） |
| 目录组织 | package，靠 `package a.b.c;` 声明 | **包**（package）= 含 `__init__.py` 的目录 |
| 引入 | `import a.b.C;` | `import a.b.c` / `from a.b import c` |
| 依赖清单 | `pom.xml` / `build.gradle` | `requirements.txt` / `pyproject.toml` |
| 依赖工具 | Maven / Gradle | pip |
| 依赖仓库 | Maven Central | PyPI（<https://pypi.org>） |
| 环境隔离 | 每个项目独立 classpath，天然隔离 | **默认全局共享**，要靠 venv 手动隔离 |
| 隔离粒度 | 项目级（jar 不污染系统） | 需显式建虚拟环境 |

一句话抓重点：**Java 的依赖天生按项目隔离，Python 默认全装到一个全局环境里**，所以 venv 不是可选项而是刚需，否则多个项目的依赖会互相打架。

## 1. 模块（module）

**一个 `.py` 文件就是一个模块**，文件名（去掉 `.py`）就是模块名。这跟 Java「一个文件一个 public class」类似，但 Python 没有「文件名必须等于类名」的强制——一个模块里可以放任意多个函数、类、变量。

```python
# mytool.py 里定义
def clean(text): ...
class Parser: ...
VERSION = '1.0'
```

引入方式几种

```python
import mytool                 # 整个模块，用时 mytool.clean(...)
from mytool import clean      # 只引入 clean，直接 clean(...)
from mytool import clean, Parser
from mytool import clean as c # 起别名，相当于 Java 没有的能力
import mytool as mt           # 给模块起别名，mt.clean(...)
```

对照 Java：`import mytool` ≈ 引入整个类后带前缀用；`from mytool import clean` ≈ `import static`。区别是 Python 的 `import` 会**执行整个被引入的文件一遍**（所以上次讲的 `if __name__ == '__main__'` 才重要），Java 的 import 只是编译期符号引用，不执行代码。

### 关键心智模型：import 得到的是「变量」，不是类型

这是和 Java 最本质的区别，务必记住：

- **Java 的 import 是编译期符号引用**，只是告诉编译器去哪找这个类，本身**不产生任何运行时对象**。`import java.util.List;` 之后并没有多出一个叫 `List` 的变量。
- **Python 的 import 是运行时动作**，执行后会在当前作用域**产生一个变量，指向模块对象**。`import sys` 之后，`sys` 就是个实实在在的变量（类型是 `module`），`sys.argv`、`sys.exit()` 本质就是「变量.成员」，和访问任何对象的属性没区别。

```python
import sys
print(type(sys))     # <class 'module'> —— 它就是个对象
s = sys              # 既然是变量，就能赋给别人
s.exit(0)            # 通过 s 一样能用
```

正因为「import 得到的是变量」，各种写法就统一了：

- `import sys` → 产生变量 `sys` 指向模块，用 `sys.xxx`
- `import sys as s` → 同一个模块对象，变量改名叫 `s`
- `from sys import argv` → 不要整个模块变量，只把模块里的 `argv` 成员拎出来绑成本地变量 `argv`

**标准库模块**（之前用过的 `os`/`sys`/`re`/`json`/`datetime`）就是 Python 自带的一批模块，`import` 即用，不用装。

### 访问控制：`_` 前缀 —— 约定，不是强制

Python 的访问控制**基于命名约定**，作用范围分两个边界——**模块级**和**类级**，用的是同一套 `_` 约定。但和 Java 最本质的区别：**Python 没有真正的强制访问控制，`_` 只是「君子协定」。**

**函数（方法）和变量都可以是私有的**——`_` 约定对两者一视同仁，只要名字加 `_` 前缀即可，没有 Java 那种「字段和方法各写一遍 private」的区别。

| 边界 | 写法 | 含义 |
| ---- | ---- | ---- |
| 模块级 | 模块里的 `_func()` / `_VAR` | 「模块内部用，别 import 我」（函数、变量都适用） |
| 类级 | 类里的 `self._x` / `def _method(self)` | 「对象内部状态 / 内部方法，别外部直接碰」 |

**Java 是编译器强制**：`private` 了外部编译期就报错。**Python 全靠自觉**：硬要 `module._func()` 或 `obj._x` 语法上照样跑通，解释器不拦。

```python
# mymod.py
def public_api(): ...
def _internal(): ...      # 单下划线：约定「内部用」

import mymod
mymod._internal()         # 照样能调！只是不该这么做
```

三个层次分清：

1. **单下划线 `_name`**：弱约定「内部用」，能访问只是不建议；`from module import *` 默认不导出 `_` 开头的名字。
2. **双下划线 `__name`（类内、结尾无双下划线）**：触发**名称改写**（name mangling），`self.__x` 实际改名为 `_类名__x`，主要用来**防子类意外覆盖**，不是真私有——知道规则照样能从外部 `obj._类名__x` 访问。
3. **前后双下划线 `__name__`**：dunder，Python 保留的特殊变量/方法（`__name__`、`__init__`），别拿来当私有用。

一句话：**访问控制可模块为界、也可类为界，同一套 `_` 命名约定；但全靠自觉，不像 Java 编译器强制——Python 哲学是「我们都是成年人，说好别碰就别碰」。**

## 2. 包（package）

**包 = 一个目录，Python 靠目录里有没有 `__init__.py` 来认它是不是包**（Python 3.3+ 其实不强制，但规范项目都会放）。这对应 Java 的 package = 目录结构。

```
myproject/
├── __init__.py           # 有它，myproject 才是一个包
├── tools/
│   ├── __init__.py
│   └── rename.py         # 模块 myproject.tools.rename
└── main.py
```

引入子包里的模块

```python
from myproject.tools.rename import batchRename
import myproject.tools.rename as r
```

`__init__.py` 的作用：目录被 import 时它会先执行，常用来做包级初始化、或把子模块的东西「提上来」方便引用。空着也完全可以，就当一个「这是包」的标记。对照 Java：Java 没有对应文件，包全靠 `package` 声明 + 目录约定；Python 用 `__init__.py` 这个实体文件标记。

## 3. 虚拟环境（venv）—— 重点

**为什么需要**：Python 默认把 `pip install` 的库全装到系统全局。项目 A 要 `requests==2.28`、项目 B 要 `requests==2.31`，全局只能装一个版本，必冲突。venv 就是给每个项目开一个**独立的、隔离的 Python 环境**，各装各的，互不影响。Java 里 Maven 把依赖放在项目本地 + 本地仓库按版本共存，天然没这问题——venv 就是 Python 补上「项目级隔离」的机制。

**创建 + 激活 + 退出**（Windows PowerShell）

```powershell
# 1. 在项目目录下创建虚拟环境，生成一个 venv 文件夹
python -m venv venv

# 2. 激活（激活后命令行前面会出现 (venv) 前缀）
.\venv\Scripts\Activate.ps1

# 3. 激活状态下，pip install 只装进这个环境，不污染全局
pip install requests

# 4. 退出虚拟环境
deactivate
```

> macOS / Linux 激活是 `source venv/bin/activate`，其余一样。

激活后 `python`、`pip` 都指向 venv 内部那份，`where python` 能看到指向的是 `venv\Scripts\python.exe`。**venv 文件夹不要提交到 Git**（`.gitignore` 里加 `venv/`），它是本地环境，别人拉代码后自己重建。

## 4. pip —— 依赖管理

pip 相当于 Python 的 Maven/Gradle 命令行，从 PyPI 下载安装库。

```powershell
pip install requests            # 装最新版
pip install requests==2.31.0    # 装指定版本（相当于 pom 里锁版本）
pip install -U requests         # 升级
pip uninstall requests          # 卸载
pip list                        # 列出当前环境装了哪些
pip show requests               # 看某个库的详情
```

**requirements.txt —— 相当于 pom.xml 的依赖清单**

团队协作 / 交付项目时，用它记录项目依赖，别人一条命令就能复现整个环境

```powershell
# 导出当前环境所有依赖到文件（在激活的 venv 里执行）
pip freeze > requirements.txt

# 别人拿到项目后，在自己的 venv 里一键安装全部依赖
pip install -r requirements.txt
```

`requirements.txt` 内容长这样，每行一个「库==版本」

```
requests==2.31.0
beautifulsoup4==4.12.2
```

对照 Java：`pip freeze > requirements.txt` ≈ 把当前依赖固化成 `pom.xml`；`pip install -r requirements.txt` ≈ `mvn install` 拉齐依赖。区别是 pip 默认不管**传递依赖的锁定**（不像 Maven 有完整依赖树解析），所以严谨项目会用 `pip freeze` 把连带装上的库也全列进去。

## 5. 标准工作流（接单必备套路）

一个 Python 工程从建立到协作，分**作者**和**协作者**两个视角。核心思想：**本地环境（venv）不进 Git，只把「依赖清单」（requirements.txt）进 Git**，让任何人都能凭清单重建出一模一样的环境。

### 阶段 A：作者从零建工程

```powershell
# 1. 建项目目录并进入
mkdir myproject; cd myproject

# 2. 建虚拟环境。目的：让本工程的依赖只装到本工程环境下，
#    不影响系统全局、也不影响其它 Python 工程
python -m venv venv

# 3. 激活虚拟环境（之后 pip 只往这个环境里装）
.\venv\Scripts\Activate.ps1

# 4. 把 venv 排除出 Git —— 里面是本地环境信息和第三方库，
#    不该 push 到仓库（别人拉下来自己重建）
echo "venv/" >> .gitignore

# 5. 通过 pip 安装本工程需要的库
pip install requests beautifulsoup4

# 6. 写代码……

# 7. 用 pip freeze 自动把当前依赖的第三方库汇总进 requirements.txt
pip freeze > requirements.txt

# 8. push 到 Git：提交代码 + .gitignore + requirements.txt，
#    但不含 venv/（已被忽略）
git add . ; git commit -m "chore: init project with deps" ; git push
```

### 阶段 B：协作者拉取后跑起来

别人 clone 下来的工程里**没有 venv**（被 Git 排除了），但有 `requirements.txt`，所以照着清单重建环境即可：

```powershell
# 1. 拉取工程
git clone <repo> ; cd myproject

# 2. 在本地建自己的虚拟环境
python -m venv venv

# 3. 激活
.\venv\Scripts\Activate.ps1

# 4. 根据依赖清单一键装齐所有第三方库
pip install -r requirements.txt

# 5. 环境就绪，跑项目
python main.py
```

一句话串起来：**venv 保证依赖隔离且不进 Git，`pip freeze` 把依赖固化成清单进 Git，协作者用 `pip install -r` 凭清单还原环境**——这样每个人本地环境独立，却又都能装出一致的依赖版本。对照 Java：等价于「`target/` 和本地仓库不进 Git，只提交 `pom.xml`，别人 `mvn install` 拉齐依赖」。

## 小结 · Java 程序员速记

- 一个 `.py` = 一个**模块**；含 `__init__.py` 的目录 = **包**（对应 Java package）。
- `import m` 会**执行整个文件**（不同于 Java 的编译期 import），所以入口逻辑要包进 `if __name__ == '__main__'`。
- Python 默认**全局装依赖**，多项目必冲突 → 每个项目用 **venv 隔离**（Java 靠 Maven 天然隔离，不用管这层）。
- venv：`python -m venv venv` 建 → `.\venv\Scripts\Activate.ps1` 激活 → `deactivate` 退出；`venv/` 别提交 Git。
- pip = Maven/Gradle 命令行；`requirements.txt` = pom.xml 的依赖清单。
- 固化依赖 `pip freeze > requirements.txt`，复现环境 `pip install -r requirements.txt`。
