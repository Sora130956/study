# 异常处理 + 常用标准库（os / sys / datetime / re）· Java 视角对照

> **来源:** <https://liaoxuefeng.com/books/python/errors/index.html>（错误处理）+ Python 官方标准库文档
> **对应计划周次:** 第 1 周 · 周三上午 · 异常处理 + 常用标准库
> **阅读方式:** 已有 Java 基础，本篇只讲「Python 和 Java 哪里一样、哪里不一样」，聚焦接单常用的 os / re。

## 核心差异总览

| 维度 | Java | Python |
| ---- | ---- | ------ |
| 捕获语法 | `try/catch/finally` | `try/except/finally` |
| 多类型捕获 | `catch (A \| B e)` | `except (A, B) as e` |
| 检查异常 | 有 checked exception，强制 try 或 throws | **全是 unchecked**，不强制处理 |
| 抛出 | `throw new Ex()` | `raise Ex()` |
| 异常基类 | `Throwable` / `Exception` | `BaseException` / `Exception` |
| 自定义异常 | `extends Exception` | `class MyError(Exception)` |
| 独有语法 | 无 | `try/except/else`、`with`（上下文管理） |

## 1. 异常处理

### 基本结构

```python
try:
    r = 10 / int('a')
except ValueError as e:          # 只捕获特定异常
    print('值错误:', e)
except ZeroDivisionError as e:
    print('除零:', e)
except (TypeError, KeyError) as e:   # 一次捕获多种，注意是 tuple
    print('类型或键错误:', e)
else:                            # Java 没有：没抛异常时才执行
    print('一切正常')
finally:                         # 无论如何都执行，通常用于释放资源
    print('结束')
```

和 Java 最大的三点不同：

1. **没有 checked exception**。Java 里 `IOException` 必须 try 或 `throws` 声明，编译器强制。Python **完全不强制**——不捕获就一路往上抛，最终打印 traceback 退出。灵活但也意味着**得靠自觉处理**。
2. **`else` 子句**：只有 `try` 块**没抛异常**时才执行。用来把「正常逻辑」和「可能出错的逻辑」分开，比全塞进 try 更清晰。
3. **多类型捕获传 tuple**：`except (TypeError, KeyError)`——又是 tuple，和 `isinstance(x, (int, float))` 同一个套路（底层就是判断异常对象 `isinstance` 于这几种类型），表示「是其中任意一种就捕获」，用于多种异常处理逻辑相同时合并。**括号必需**，漏了在 Python 3 是 `SyntaxError`。相当于 Java 的 `catch (A | B e)`。

### 抛出与自定义异常

```python
raise ValueError('bad input')        # 相当于 throw new

class PaymentError(Exception):       # 自定义：继承 Exception 即可
    pass

raise PaymentError('余额不足')
```

- 继承 `Exception`（不要直接继承 `BaseException`，那是给 `KeyboardInterrupt` 等系统级用的）。
- **异常链**：`raise NewError() from e`，相当于 Java 的 `new Ex(cause)`，保留原始异常上下文。

### 常见坑

- **别裸写 `except:`**（不指定类型）——它会连 `KeyboardInterrupt`（Ctrl+C）都吞掉，程序按不了停。要么写 `except Exception`，要么指定具体类型。
- Python 的 traceback 从**外层调用往内层**打印，最底下那行才是真正出错的位置——和 Java 堆栈阅读顺序相反，别看错。

## 2. os / os.path / sys

接单里文件自动化的核心。os 已在文件操作笔记出现过，这里补齐重点。

```python
import os

os.getcwd()                       # 当前工作目录，相当于 System.getProperty("user.dir")
os.listdir('.')                   # 列目录（不递归）
os.makedirs(os.path.join('a', 'b', 'c'), exist_ok=True)  # 递归建目录（用 join 拼路径），exist_ok 避免已存在报错
os.remove('x.txt')                # 删文件
os.rename('old.txt', 'new.txt')   # 重命名 / 移动
os.environ.get('PATH')            # 环境变量
```

**路径操作优先用 `os.path`（或更现代的 `pathlib`）**，别手拼字符串——跨平台分隔符不同（Windows `\`、Linux `/`）：

```python
os.path.join('data', 'sub', 'f.txt')   # 自动用当前系统分隔符
os.path.exists(p)                       # 是否存在
os.path.isfile(p) / os.path.isdir(p)    # 是文件 / 目录
os.path.splitext('a.txt')               # ('a', '.txt')  分离扩展名，重命名脚本常用
os.path.basename('/x/y/a.txt')          # 'a.txt'
os.path.dirname('/x/y/a.txt')           # '/x/y'
```

`sys` 主要用两个：

```python
import sys
sys.argv                          # 命令行参数列表，argv[0] 是脚本名，相当于 main(String[] args)
sys.exit(0)                       # 退出程序，非 0 表示异常退出
```

## 3. datetime

Java 8 后用 `LocalDateTime` / `DateTimeFormatter`，Python 用 `datetime`。概念一一对应：

```python
from datetime import datetime, timedelta, date

now = datetime.now()                        # 当前时间，相当于 LocalDateTime.now()
dt = datetime(2026, 8, 1, 13, 30, 0)        # 指定时间

# 格式化：datetime → 字符串（strftime，f=format）
now.strftime('%Y-%m-%d %H:%M:%S')           # '2026-08-01 13:30:00'

# 解析：字符串 → datetime（strptime，p=parse）
datetime.strptime('2026-08-01', '%Y-%m-%d') # 相当于 DateTimeFormatter.parse

# 时间加减：用 timedelta，相当于 plusDays/minusHours
now + timedelta(days=7, hours=3)
(date(2026, 8, 8) - date(2026, 8, 1)).days  # 7，两个日期差

# 时间戳互转
now.timestamp()                             # datetime → Unix 秒
datetime.fromtimestamp(1754024400)          # 秒 → datetime
```

记忆点：**`strftime` = string from time（输出）、`strptime` = string parse time（输入）**，f/p 区分方向。格式符 `%Y`（4位年）`%m`（月）`%d`（日）`%H`（24时）`%M`（分）`%S`（秒），大小写敏感（`%m` 月 vs `%M` 分）。

## 4. re（正则）

语法和 Java `java.util.regex` 基本一致（都源自 PCRE 风格），主要差异在 API 和「原始字符串」。

```python
import re

# 关键：正则字符串前面加 r，表示 raw string，反斜杠不转义
p = r'(\d{4})-(\d{2})-(\d{2})'      # Java 里要写 "\\d{4}"，Python 用 r'' 省掉双反斜杠

# 匹配
m = re.match(p, '2026-08-01')       # 从开头匹配，相当于 Matcher.lookingAt
if m:
    m.group(0)                       # '2026-08-01' 整个匹配
    m.group(1)                       # '2026'，第 1 个捕获组
    m.groups()                       # ('2026', '08', '01') 所有组

re.search(p, 'date: 2026-08-01')    # 在整个串里找第一个（match 只从开头）
re.findall(r'\d+', 'a1b22c333')     # ['1', '22', '333'] 找全部
re.sub(r'\s+', '_', 'a  b   c')     # 'a_b_c' 替换，相当于 replaceAll
re.split(r'[,;]', 'a,b;c')          # ['a', 'b', 'c'] 按正则分割
```

和 Java 的关键区别：

1. **`r''` 原始字符串**：Python 正则必备。不加 `r`，`\d` 会先被字符串层解释一次，得写 `'\\d'` 很难看。养成习惯——**正则一律用 `r''`**。
2. **`match` 只从开头匹配**，不是「整串匹配」。要整串匹配得自己加 `$` 或用 `re.fullmatch`。Java 的 `matches()` 才是整串——别搞混。
3. **性能**：同一正则反复用，先 `p = re.compile(r'...')` 预编译，再 `p.findall(...)`，相当于 Java 的 `Pattern.compile`。

## 5. 周三上午实战衔接：文件批量重命名脚本

下午要写的重命名脚本，正好把这几块串起来：

```python
import os
import re

def batch_rename(folder, pattern, replacement):
    for name in os.listdir(folder):                    # os：列目录
        old = os.path.join(folder, name)               # os.path：拼路径
        if not os.path.isfile(old):
            continue
        new_name = re.sub(pattern, replacement, name)  # re：改名
        new = os.path.join(folder, new_name)
        if new_name != name:
            try:
                os.rename(old, new)                    # os：重命名
                print(f'{name} -> {new_name}')
            except OSError as e:                       # 异常：单个失败不中断整体
                print(f'跳过 {name}: {e}')
```

这一个函数用到了本篇全部内容：`os` 遍历 / 重命名、`os.path` 拼路径、`re` 做替换、`try/except` 保证批量操作里单个失败不拖垮整体。下午照着扩展（加命令行参数 `sys.argv`、加时间戳前缀 `datetime`）就是一个能交付的小工具。

## 小结 · Java 程序员速记

- 异常：`try/except/finally` 同 Java，多了 `else`；**无 checked exception**，不强制处理；多类型捕获传 **tuple**。
- 别裸 `except:`，会吞掉 Ctrl+C。
- 路径**永远用 `os.path.join`**，别手拼，跨平台。
- 日期：`strftime` 输出 / `strptime` 解析，`timedelta` 做加减。
- 正则：**一律 `r''`**；`match` 只匹配开头（不是整串）；重复使用先 `re.compile`。
- **别直接调 dunder 方法**：比较写 `a == b` / `a != b`，不要写 `a.__eq__(b)`。`__eq__`、`__len__` 这类双下划线方法是给解释器内部调的，`==`/`len()` 就是它们的语法糖。这跟 Java 不同——Java 必须 `a.equals(b)`（`==` 比引用），Python 的 `==` 对字符串就是值比较，直接用即可。
