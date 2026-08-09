# File I/O（文件操作：读写文本 / CSV / JSON）

> **来源:** <https://liaoxuefeng.com/books/python/io/index.html>（文件读写、操作文件和目录、序列化）+ Python 官方 csv 文档
> **对应计划周次:** 第 1 周 · 周一下午 · 文件操作

本篇覆盖：文本文件读写、`with` 上下文管理、`os` / `os.path` 操作文件与目录、JSON 序列化、CSV 读写。这是接自动化 / 数据处理单子的核心技能——爬取、清洗后的数据都要落地成文本 / JSON / CSV。

## 核心理解

读写文件本质是**请求操作系统打开一个文件对象（文件描述符）**，再通过它读数据或写数据。文件对象会占用系统资源，用完**必须关闭**，否则可能丢数据、耗尽句柄。

Python 用 `open()` 打开文件、`with` 语句自动关闭，用内置 `json` 模块和标准库 `csv` 模块做结构化数据的存取。日常数据处理里，JSON 适合嵌套结构与跨语言交换，CSV 适合表格数据（可被 Excel 直接打开）。

## 文本文件读写

### 读文件

```python
f = open('/path/to/test.txt', 'r')   # 'r' 表示读，文件不存在抛 FileNotFoundError
content = f.read()                    # 一次读全部内容，返回 str
f.close()                             # 必须关闭，释放系统资源
```

三种读取方法，按需选择：

| 方法 | 行为 | 适用 |
| ---- | ---- | ---- |
| `read()` | 一次读全部，返回 str | 小文件 |
| `read(size)` | 每次最多读 size 字节 | 大文件（避免撑爆内存） |
| `readline()` | 每次读一行 | 逐行处理 |
| `readlines()` | 读全部，按行返回 list | 配置文件 |

逐行处理常用写法：

```python
for line in f.readlines():
    print(line.strip())   # strip() 去掉行尾的 '\n'
```

### 写文件

```python
f = open('/path/to/test.txt', 'w')   # 'w' 覆盖写；'a' 追加写
f.write('Hello, world!')
f.close()                             # 不关闭可能只写了一部分到磁盘
```

写入时操作系统常先放内存缓存，`close()` 时才保证全部落盘。忘记 `close()` 可能丢数据。

### with 语句（推荐）

因为读写都可能出错，出错后 `close()` 就不执行了。`with` 语句会自动关闭文件，等价于 `try...finally`，但更简洁：

```python
with open('/path/to/file', 'r') as f:
    print(f.read())
# 离开 with 块自动 close()，不必手写
```

### 字符编码

默认按 UTF-8 读文本。读非 UTF-8 文件要传 `encoding`；遇到非法字符可用 `errors='ignore'` 跳过：

```python
with open('gbk.txt', 'r', encoding='gbk') as f:
    print(f.read())

open('gbk.txt', 'r', encoding='gbk', errors='ignore')   # 忽略编码错误
```

处理中文文件时，**统一显式指定 `encoding='utf-8'`** 是避免乱码的最佳实践。

### 二进制文件

图片、视频等用 `'rb'` / `'wb'` 模式，读到的是 `bytes`：

```python
with open('test.jpg', 'rb') as f:
    data = f.read()   # b'\xff\xd8\xff...'
```

### 打开模式速查

| 模式 | 含义 |
| ---- | ---- |
| `'r'` | 读（默认），文件不存在报错 |
| `'w'` | 写，覆盖；不存在则创建 |
| `'a'` | 追加，写到文件末尾 |
| `'x'` | 排他创建，已存在则失败 |
| `'b'` | 二进制模式，与上组合如 `'rb'` |
| `'+'` | 读写，如 `'r+'` |

## os / os.path 操作文件和目录

`os` 模块封装操作系统的目录和文件操作。**注意函数分散在 `os` 和 `os.path` 两处。**

```python
import os

os.name                         # 'posix'(Linux/Mac) 或 'nt'(Windows)
os.environ.get('PATH')          # 读环境变量
```

### 路径拼接与拆分（跨平台关键）

**不要手动拼字符串**（Windows 用 `\`、Linux 用 `/`），一律用 `os.path` 函数，自动适配当前系统分隔符：

```python
os.path.abspath('.')                          # 当前目录绝对路径
os.path.join('/Users/michael', 'testdir')     # 拼路径，跨平台安全
os.path.split('/a/b/file.txt')                # ('/a/b', 'file.txt')
os.path.splitext('/a/file.txt')               # ('/a/file', '.txt') 取扩展名
```

这些函数只对字符串操作，不要求路径真实存在。

### 增删目录与文件

```python
os.mkdir('/path/testdir')       # 创建目录
os.rmdir('/path/testdir')       # 删除空目录
os.rename('a.txt', 'b.txt')     # 重命名
os.remove('b.txt')              # 删除文件
```

**复制文件 `os` 里没有**（不是系统调用），用 `shutil.copyfile()`。

### 用列表生成式过滤文件

```python
# 列出当前目录所有子目录
[x for x in os.listdir('.') if os.path.isdir(x)]

# 列出所有 .py 文件
[x for x in os.listdir('.')
 if os.path.isfile(x) and os.path.splitext(x)[1] == '.py']
```

## JSON 序列化

**序列化**：把内存变量变成可存储 / 传输的形式（Python 里 pickle 也叫 pickling）；**反序列化**是其逆过程。JSON 是跨语言标准格式，比 pickle 通用，接单处理数据首选。

JSON 与 Python 类型对应：

| JSON | Python |
| ---- | ------ |
| `{}` | dict |
| `[]` | list |
| `"string"` | str |
| 数字 | int / float |
| `true` / `false` | True / False |
| `null` | None |

### 四个核心函数

`dumps` / `loads` 处理**字符串**，`dump` / `load` 直接对接**文件对象**：

```python
import json

d = dict(name='Bob', age=20, score=88)

json.dumps(d)                   # 对象 → JSON 字符串
json.loads('{"name": "Bob"}')   # JSON 字符串 → 对象（dict）

# 直接读写文件
with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(d, f)             # 对象 → 写入文件
with open('data.json', 'r', encoding='utf-8') as f:
    d = json.load(f)           # 从文件读 → 对象
```

### 序列化自定义 class

**为什么类实例不能直接转 JSON**：JSON 只认识 dict / list / str / 数字 / 布尔 / null，自定义的 `Student` 类它不认识，直接转会报 `TypeError: Student object is not JSON serializable`。核心矛盾是「JSON 只会转 dict，但实例不是 dict」，所以思路是**先把对象变成 dict，再交给 JSON**。

**序列化（对象 → JSON）用 `default`**：告诉 json 遇到不认识的对象时，调用这个函数把它转成 dict。

```python
# 手写转换函数版（可挑字段、改名，可控）
def student2dict(std):
    return {'name': std.name, 'age': std.age, 'score': std.score}

json.dumps(s, default=student2dict)

# lambda + __dict__ 版（通用，任何类都能用）
json.dumps(s, default=lambda obj: obj.__dict__)
```

执行流程：`dumps(s)` 发现 `s` 不认识 → 调用 `default` 函数 → 返回 dict → json 转成字符串。

`__dict__` 是每个实例内置的属性，本身就是装着所有字段的 dict：

```python
s.__dict__   # {'name': 'Bob', 'age': 20, 'score': 88}
```

所以 `lambda obj: obj.__dict__` = 「不管什么对象，直接返回它的字段 dict」，类比 Java 反射自动 `toMap()`。

**反序列化（JSON → 对象）用 `object_hook`**：json 默认只还原成 dict，`object_hook` 让它把解析出的 dict 再转成你要的对象。

```python
def dict2student(d):                                   # d 是 json 解析出的 dict
    return Student(d['name'], d['age'], d['score'])    # 用 dict 的值构造对象

json.loads(json_str, object_hook=dict2student)   # 返回 Student 实例，不是 dict
```

执行流程：`loads` 先解析成 dict → 传给 `object_hook` 函数 → 函数 new 出对象 → 返回实例。

**对称记忆**（两者互逆）：

| 方向 | 函数 | 参数 | 参数作用 |
| ---- | ---- | ---- | -------- |
| 对象 → JSON（序列化） | `dumps` | `default` | 把「不认识的对象」变成 dict |
| JSON → 对象（反序列化） | `loads` | `object_hook` | 把解析出的 dict 变成「你要的对象」 |

- 存：`default` → 对象拆成 dict → JSON
- 读：`object_hook` → JSON 变 dict → 拼回对象

> **实战提醒**：大多数场景用不到这两个参数——接单处理的数据通常本来就是 dict / list（API 返回的 JSON、CSV 读出的行），直接 `dumps` / `loads` 即可。只有当你自己用 class 建模数据、又要存成 JSON 时才需要它们。

### 中文不转义

默认中文会被转成 `\uXXXX`。要输出可读中文，传 `ensure_ascii=False`：

```python
json.dumps(dict(name='小明'), ensure_ascii=False)   # {"name": "小明"}
```

## CSV 读写（标准库 csv）

CSV 是逗号分隔的表格文本，Excel 可直接打开，是数据处理交付的常见格式。廖雪峰教程未涵盖，用标准库 `csv`。

> 关键坑：Windows 上打开文件要加 `newline=''`，否则写出的 CSV 每行之间会多出空行。

### 按行读写（list）

```python
import csv

# 写
with open('out.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['name', 'age'])        # 写一行
    writer.writerows([['Bob', 20], ['Amy', 22]])   # 写多行

# 读
with open('out.csv', 'r', newline='', encoding='utf-8') as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)   # 每行是一个 list，如 ['name', 'age']
```

### 按字典读写（DictReader / DictWriter，推荐）

带表头的数据用字典形式更直观，字段名即 key：

```python
# 写
with open('out.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age'])
    writer.writeheader()                          # 写表头
    writer.writerow({'name': 'Bob', 'age': 20})

# 读
with open('out.csv', 'r', newline='', encoding='utf-8') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row['name'], row['age'])   # 每行是 dict
```

### 实战坑与进阶

**1. Excel 打开中文 CSV 乱码 → 用 `utf-8-sig`**

Windows 上 Excel 默认按 GBK 解析，用 `utf-8` 存的中文 CSV 双击打开会乱码。改用 `utf-8-sig`（带 BOM 头），Excel 就能正确识别：

```python
with open('out.csv', 'w', newline='', encoding='utf-8-sig') as f:
    writer = csv.writer(f)
    writer.writerow(['姓名', '年龄'])   # Excel 双击打开中文不乱码
```

交付给客户的 CSV 几乎都该用 `utf-8-sig`，这是最高频的坑。

**2. 自定义分隔符（TSV / 分号）**

不是所有「CSV」都用逗号：欧洲地区常用分号，TSV 用制表符。用 `delimiter` 指定：

```python
csv.reader(f, delimiter='\t')     # 读 TSV（制表符分隔）
csv.writer(f, delimiter=';')      # 写分号分隔
```

**3. 字段含逗号 / 换行 / 引号 → 别自己 split**

这是**必须用 `csv` 模块、不能手写 `line.split(',')`** 的根本原因。当字段值本身含逗号时，标准 CSV 会用引号包裹：

```
"Smith, John",30,"say ""hi"""
```

`csv` 模块会自动处理这种加引号、转义双引号、跨行字段的情况；手写 `split(',')` 遇到 `"Smith, John"` 会被错误拆成两列。

**4. 大文件逐行读，不撑内存**

`reader` / `DictReader` 本身就是迭代器，`for row in reader` 天然逐行读取，处理几十万行也不会把整个文件塞进内存：

```python
with open('big.csv', 'r', newline='', encoding='utf-8') as f:
    for row in csv.DictReader(f):   # 一次只驻留一行
        process(row)
```

**5. 和 pandas 的分工**

做数据分析 / 清洗时，更常用 `pandas`，一行读写还能顺带处理：

```python
import pandas as pd
df = pd.read_csv('in.csv')
df.to_csv('out.csv', index=False, encoding='utf-8-sig')
```

分工：标准库 `csv` 适合轻量、无第三方依赖的场景；`pandas` 适合数据分析（计划第 2-3 周数据处理会重点用）。

## 小结

- **文本读写**：`open()` 选模式（r/w/a/b/+），优先用 `with` 自动关闭；中文统一 `encoding='utf-8'`。
- **os / os.path**：路径拼接拆分用 `os.path.join/split/splitext`（跨平台），增删用 `os.mkdir/remove/rename`，复制用 `shutil`。
- **JSON**：`dumps/loads` 走字符串、`dump/load` 走文件；自定义类用 `default` / `object_hook`；中文加 `ensure_ascii=False`。
- **CSV**：`csv.writer/reader` 按 list，`DictWriter/DictReader` 按 dict（推荐）；Windows 记得 `newline=''`。

## 术语表

| 英文 | 词性 | 释义 |
| ---- | ---- | ---- |
| file descriptor | 名词 | 文件描述符，操作系统打开文件后返回的文件对象 |
| file-like object | 名词 | 类文件对象，只要有 `read()` 方法即可，如内存流、网络流 |
| context manager | 名词 | 上下文管理器，`with` 语句依赖它自动获取/释放资源 |
| serialization | 名词 | 序列化，把内存变量转成可存储/传输的形式 |
| deserialization | 名词 | 反序列化，从序列化数据还原成内存对象 |
| encoding | 名词 | 字符编码，如 UTF-8、GBK |
| delimiter | 名词 | 分隔符，CSV 中默认是逗号 |
