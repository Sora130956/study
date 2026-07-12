# Python Basics

> **来源:** https://docs.python.org/3/tutorial/introduction.html
> **对应计划周次:** Python 入门学习

## 核心理解

Python 变量是"标签"（name binding），不是"盒子"。`=` 操作是将名称绑定到对象，而不是复制值。这意味着多个变量可以同时指向同一个对象，重新赋值只是把标签贴到新对象上，不影响其他指向旧对象的变量。

## 关键点

### 变量赋值与引用（Name Binding）

```python
a = 'A'
b = a       # b 与 a 指向同一个 'A' 对象
a = 'B'     # a 改为指向新对象 'B'，b 不受影响
print(b)    # A

# 用 id() 验证
a = 'A'
b = a
print(id(a) == id(b))  # True，同一个对象
a = 'B'
print(id(a) == id(b))  # False，a 已指向新对象
```

- `=` 是名称绑定（name binding），不是值拷贝
- 官方术语：variable 是 name，`=` 是 bind 操作

---

### 除法：`/` 与 `//`

| 运算符 | 名称 | 行为 |
|--------|------|------|
| `/` | 真除法 | 永远返回 float，不截断 |
| `//` | 地板除 | 截断小数，返回整数 |

```python
10 / 3    # 3.3333333333333335
10 // 3   # 3
```

- Python 3 的 `/` 是"精确"的，相对 Python 2 而言：Python 2 中 `10/3` 直接返回 `3`（截断整数），Python 3 返回真实浮点结果
- `3.3333...35` 末尾的 `5` 是 IEEE 754 浮点精度限制，所有语言都有，不是 Python 的问题

---

### 字符串类型

#### Raw String（原始字符串）

前缀 `r`，让 `\` 不被当作转义符，作为普通字符处理。

```python
print('C:\new_folder')   # C:（换行）ew_folder  ← \n 被解释为换行
print(r'C:\new_folder')  # C:\new_folder        ← \n 就是字面字符
```

常见用途：Windows 文件路径、正则表达式。

#### 三引号字符串（Triple-quoted String）

`'''...'''` 或 `"""..."""`，支持跨多行，换行会被保留。

```python
s = '''Hello,
Bob!'''
print(s)
# Hello,
# Bob!
```

可以组合使用：`r'''...'''` 同时具备 raw + 多行特性。

---

### 一切皆对象（Everything is an Object）

Python 中所有数据都是对象，包括整数、字符串、布尔值，甚至函数和类本身。`int` 类型的值就是 `int` 类的实例。

```python
x = 42
print(type(x))            # <class 'int'>
print(isinstance(x, int)) # True
print(x.bit_length())     # 6，整数对象有方法
print(id(x))              # 内存地址，说明它是对象
```

**与 Java 的对比：**

| | Python | Java |
|---|---|---|
| 整数 | `int` 对象（唯一形态） | `int`（primitive）+ `Integer`（包装类）|
| 是否需要装箱 | 不需要，天生就是对象 | 需要显式或自动装箱 |
| 不可变性 | `int` 对象不可变 | `Integer` 对象不可变 |

Python 的 `42` 直接等价于 Java 的 `Integer` 实例，但 Python 没有对应的 primitive 版本，语言层面更统一。

**小整数缓存（CPython 实现细节）：**

CPython 对 `-5` 到 `256` 的整数有缓存，同一个值共用同一个对象：

```python
a = 100
b = 100
print(a is b)  # True，同一个对象（缓存范围内）

a = 1000
b = 1000
print(a is b)  # False，超出缓存范围，两个不同对象
```

---

### 文件头部约定

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
```

- **shebang（`#!/usr/bin/env python3`）**：告诉 Unix/Linux/macOS 用哪个解释器运行，Windows 上无效。有了它可以直接 `./hello.py` 执行，不用写 `python3 hello.py`
- **编码声明（`# -*- coding: utf-8 -*-`）**：Python 3 默认已是 UTF-8，这行可以不写；历史遗留写法，Python 2 必须写否则中文报错

---

### 字符串格式化

三种写法，推荐 f-string：

```python
name = 'Bob'
age = 25

# 方式一：% 格式化（老写法，来自 C 的 printf，了解即可）
'Age: %s. Gender: %s' % (25, True)   # 'Age: 25. Gender: True'

# 方式二：format()，Python 2.6+
'Hello, {}. Age: {}'.format(name, age)

# 方式三：f-string，Python 3.6+（推荐）
f'Hello, {name}. Age: {age}'
```

常用 `%` 占位符（老代码里会见到）：

| 占位符 | 含义 |
|--------|------|
| `%s` | 字符串（任何类型自动转字符串） |
| `%d` | 整数 |
| `%f` | 浮点数 |

---

### list 与负数索引

`list` 是有序集合，等价于 Java 的 `ArrayList`，支持正向和负向索引：

```python
classmates = ['Michael', 'Bob', 'Tracy']

# 正向索引（从0开始）
classmates[0]   # 'Michael'
classmates[1]   # 'Bob'

# 负数索引（从末尾倒数）
classmates[-1]  # 'Tracy'，最后一个
classmates[-2]  # 'Bob'，倒数第二
classmates[-3]  # 'Michael'，倒数第三
```

负数索引规律：`-1` 等价于 `len(list) - 1`，避免手动计算末尾位置。

#### list 常用操作

```python
classmates = ['Michael', 'Bob', 'Tracy']

# 追加到末尾
classmates.append('Adam')       # ['Michael', 'Bob', 'Tracy', 'Adam']

# 插入到指定位置
classmates.insert(1, 'Jack')    # ['Michael', 'Jack', 'Bob', 'Tracy', 'Adam']

# 删除末尾元素（返回被删除的值）
classmates.pop()                # 返回 'Adam'

# 删除指定位置元素
classmates.pop(1)               # 返回 'Jack'

# 替换元素
classmates[1] = 'Sarah'         # ['Michael', 'Sarah', 'Tracy']
```

#### list 的灵活性

- 元素类型可以不同（不像 Java 泛型要求统一类型）：
```python
L = ['Apple', 123, True]
```

- 元素可以是另一个 list（嵌套 list，等价于多维数组）：
```python
p = ['asp', 'php']
s = ['python', 'java', p, 'scheme']
len(s)      # 4，p 整体算一个元素
s[2][1]     # 'php'，二维索引
```

- 空 list：
```python
L = []
len(L)  # 0
```

---

### tuple（元组）

tuple 是不可变的有序序列，用 `()` 定义，等价于"冻结的 list"：

```python
classmates = ('Michael', 'Bob', 'Tracy')
classmates[0]   # 'Michael'，支持索引
classmates[-1]  # 'Tracy'，支持负数索引
# classmates[0] = 'X'  ← 报错，不可修改
# classmates.append('X')  ← 没有这个方法
```

**单元素 tuple 的陷阱：**

```python
t = (1)   # 不是 tuple！括号被当成数学括号，t 是整数 1
t = (1,)  # 才是 tuple，必须加逗号消除歧义
```

**"可变的" tuple 的本质：**

tuple 的"不变"是指每个元素的**指向**不变，不是元素本身不变：

```python
t = ('a', 'b', ['A', 'B'])
t[2][0] = 'X'  # 合法！改的是 list 内部，不是 tuple 的指向
t[2][1] = 'Y'
# t → ('a', 'b', ['X', 'Y'])
# tuple 仍然指向同一个 list 对象，只是 list 内容变了
```

**tuple vs list：**

| | list | tuple |
|---|---|---|
| 符号 | `[]` | `()` |
| 可变 | 是 | 否 |
| 用途 | 动态集合 | 固定数据组合（坐标、记录等）|

**tuple 不是枚举。** 枚举用 `enum.Enum`，有具名常量；tuple 只是不可变的有序序列，顺序有意义但元素无名字。

---

### 条件判断：if / elif / else

Python 用缩进代替 `{}`，冒号 `:` 表示代码块开始：

```python
age = 20

# 完整形式
if age >= 18:
    print('adult')
elif age >= 6:
    print('teenager')
else:
    print('kid')
```

与 Java 对比：

```java
// Java
if (age >= 18) {
    System.out.println("adult");
}
```

```python
# Python
if age >= 18:
    print('adult')
```

**缩进规则：**

- 同一代码块内缩进必须完全一致，不一致报 `IndentationError`
- Python 3 禁止 tab 和空格混用，会直接报 `TabError`
- 社区规范（PEP 8）：统一用 **4 个空格**，PyCharm 默认已是此设置

```python
if age >= 18:
    print('adult')   # 4个空格
  print('hello')     # 2个空格 ← IndentationError！

if age >= 18:
    print('adult')   # 空格
	print('hello')   # tab ← TabError！
```

---

### input() 与类型转换

`input()` 永远返回 `str`，不会自动转换类型：

```python
s = input('birth: ')  # 用户输入 1982，s 是字符串 '1982'
birth = int(s)         # 必须手动转换为整数
if birth < 2000:
    print('00前')
else:
    print('00后')
```

`int()` 转换失败时直接抛异常：

```python
int('abc')  # ValueError: invalid literal for int() with base 10: 'abc'
```

**核心理解：** Python 没有基础数据类型，也没有隐式类型转换。`input()` 返回的永远是 `str` 实例，要做数值运算必须显式转换。Java 的 `Integer.parseInt(s)` 与 Python 的 `int(s)` 是同一回事，只是 Java 还有数值类型间的自动提升（`int` → `long` → `double`），Python 完全没有。

常用类型转换函数：

| 函数 | 作用 |
|------|------|
| `int(x)` | 转整数 |
| `float(x)` | 转浮点数 |
| `str(x)` | 转字符串 |
| `bool(x)` | 转布尔值 |

---

### match / case（Python 3.10+）

Python 的模式匹配，比 Java `switch` 更强大：

```python
age = 15

match age:
    case x if x < 10:           # 捕获变量 + 守卫条件（guard）
        print(f'< 10: {x}')
    case 10:                     # 精确匹配
        print('10 years old.')
    case 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18:  # OR 模式
        print('11~18 years old.')
    case _:                      # 通配符，等价于 default
        print('not sure.')
```

与 Java `switch` 对比：

| | Java switch | Python match |
|---|---|---|
| 匹配值 | 是 | 是 |
| 守卫条件 | 否（Java 14+ switch expression 有限支持）| `case x if 条件:` |
| OR 模式 | 多个 `case` 并排 | `case a \| b \| c:` |
| 默认分支 | `default:` | `case _:` |
| 需要 break | 是（传统 switch）| 否，自动停止 |
| 结构匹配 | 否 | 是（可匹配 list、tuple、类实例）|

#### match 序列模式

直接匹配 list 的结构：

```python
args = ['gcc', 'hello.c', 'world.c']

match args:
    case ['gcc']:                        # 精确匹配只有 'gcc' 的 list
        print('gcc: missing source file(s).')
    case ['gcc', file1, *files]:         # 'gcc' + 至少一个文件，*files 捕获剩余
        print('gcc compile: ' + file1 + ', ' + ', '.join(files))
        # file1 = 'hello.c'，files = ['world.c']
    case ['clean']:
        print('clean')
    case _:
        print('invalid command.')
```

`*files` 捕获剩余所有元素，打包成 list，类似解构赋值。

---

### 循环

#### for-each

```python
names = ['Michael', 'Bob', 'Tracy']
for name in names:
    print(name)
```

Python 没有 C 风格的 `for (int i=0; i<100; i++)`，用 `range()` 代替：

```python
# 等价于 for (int i = 0; i < 100; i++)
for i in range(100):
    print(i)

# 指定起止和步长：range(start, stop, step)
for i in range(0, 100, 2):   # 0, 2, 4, ... 98
    print(i)
```

#### while

```python
sum = 0
n = 99
while n > 0:
    sum = sum + n
    n = n - 2
print(sum)
```

#### break 与 continue

```python
# break：退出当前循环
n = 1
while n <= 100:
    if n > 10:
        break       # n=11 时跳出循环
    print(n)
    n = n + 1

# continue：跳过本轮，继续下一轮
n = 0
while n < 10:
    n = n + 1
    if n % 2 == 0:
        continue    # 偶数跳过，只打印奇数
    print(n)
```

`break`/`continue` 的缩进决定它们属于哪一层循环块，和 Java 一致。

---

### dict（字典）

等价于 Java 的 `HashMap`，用 `{}` 字面量创建，无需 `new`：

```python
d = {'Michael': 95, 'Bob': 75, 'Tracy': 85}

# 查
d['Michael']          # 95
d.get('Michael')      # 95，推荐：key不存在返回None而不是报错
d.get('Alice', 0)     # 0，key不存在时返回默认值
'Michael' in d        # True，判断key是否存在

# 增 / 改
d['Alice'] = 67       # key不存在则新增，存在则覆盖

# 删
d.pop('Bob')          # 删除并返回value，key不存在报错

# 遍历
for k, v in d.items():
    print(k, v)

for k in d.keys():    # 只遍历key
    print(k)

for v in d.values():  # 只遍历value
    print(v)

# 其他
len(d)                # 元素个数
```

**与 JSON 的关系：**

`dict` 结构与 JSON 几乎相同，互转非常自然：

```python
import json

d = {'name': 'Michael', 'score': 95}
json_str = json.dumps(d)    # dict → JSON 字符串
d2 = json.loads(json_str)   # JSON 字符串 → dict
```

**注意：** dict 的 key 必须是不可变类型（str、int、tuple），list 不能做 key。

---

### set（集合）

无序、不重复的元素集合，等价于 Java 的 `HashSet`：

```python
# 创建
s = {1, 2, 3}
s = set([1, 2, 2, 3, 3])  # 从 list 创建，自动去重 → {1, 2, 3}
s = set()                  # 空 set，不能用 {}（那是空 dict）

# 增 / 删
s.add(4)         # 添加元素，已存在则忽略
s.remove(4)      # 删除，不存在报 KeyError
s.discard(4)     # 删除，不存在不报错（推荐）

# 查
3 in s           # True，判断元素是否存在
len(s)           # 元素个数

# 集合运算
a = {1, 2, 3}
b = {2, 3, 4}
a & b            # 交集 → {2, 3}
a | b            # 并集 → {1, 2, 3, 4}
a - b            # 差集 → {1}
a ^ b            # 对称差集（只在其中一个里）→ {1, 4}

# 遍历（无序，不保证顺序）
for x in s:
    print(x)
```

**常见用途：去重**

```python
names = ['Alice', 'Bob', 'Alice', 'Tracy', 'Bob']
unique = list(set(names))   # 去重后转回 list
```

**注意：** set 的元素也必须是不可变类型，list 不能放入 set。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| name binding | 名词 | Python 变量赋值的本质，名称绑定到对象 |
| raw string | 名词 | 前缀 `r`，反斜杠不转义的字符串 |
| floor division | 名词 | `//` 运算符，向下取整除法 |
| IEEE 754 | 名词 | 浮点数存储标准，决定浮点精度上限 |
| primitive | 名词 | Java 基础数据类型，不是对象，Python 中没有此概念 |
| immutable | 形容词 | 不可变，对象创建后值无法修改 |
| CPython | 名词 | Python 的官方 C 实现，有小整数缓存优化 |
