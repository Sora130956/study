# Functional Programming（函数式编程）

> **来源:** <https://liaoxuefeng.com/books/python/functional/index.html>
> **对应计划周次:** Python 入门学习

本篇覆盖：高阶函数、map/reduce、返回函数与闭包、匿名函数（lambda）、装饰器、偏函数。

## 高阶函数（Higher-order Function）

### 变量可以指向函数

函数本身也是对象，可以赋值给变量。`abs` 是函数本身，`abs(-10)` 才是调用：

```python
f = abs        # f 指向 abs 函数对象
f(-10)         # 10，通过变量调用完全等价于 abs(-10)
```

函数名其实就是指向函数的变量。若把 `abs = 10`，`abs` 就不再指向函数，`abs(-10)` 会报 `TypeError: 'int' object is not callable`。

### 传入函数

函数参数能接收函数，接收函数作为参数的函数就叫**高阶函数**：

```python
def add(x, y, f):
    return f(x) + f(y)

add(-5, 6, abs)   # abs(-5) + abs(6) = 11
```

一句话：把函数作为参数传入，就是高阶函数；函数式编程就是这种高度抽象的范式。

## map / reduce

`map()` 和 `reduce()` 是 Python 的高阶函数，核心思想是把「运算规则」抽象成函数参数传入，而不是手写循环去描述执行步骤。

- `map(f, iterable)`：把函数 `f` 依次作用到序列每个元素，返回一个新的 `Iterator`（惰性序列）。
- `reduce(f, iterable)`：把接收两个参数的函数 `f` 依次作用于「上次计算结果」和「序列下一个元素」，做累积计算，最终归约成单个值。

高阶函数的价值在于把运算规则抽象出来：一行 `map(f, list)` 就表达了「把 f 作用到每个元素并生成新序列」，比 `for` 循环 + `append` 更直接地体现意图。

## 关键点

### map()

`map()` 接收两个参数：一个函数、一个 `Iterable`。它把函数作用到序列每个元素，结果作为新的 `Iterator` 返回。

```python
def f(x):
    return x * x

r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
list(r)   # [1, 4, 9, 16, 25, 36, 49, 64, 81]
```

要点：

- 第一个参数传的是**函数对象本身**（`f`，不加括号）。
- 返回值 `r` 是 `Iterator`，惰性求值，用 `list(r)` 才会把整个序列计算出来。
- 运算规则被抽象，可传任意函数。例如把数字全转字符串：

```python
list(map(str, [1, 2, 3, 4, 5]))   # ['1', '2', '3', '4', '5']
```

等价的命令式写法（更啰嗦，意图不够直观）：

```python
L = []
for n in [1, 2, 3, 4, 5]:
    L.append(f(n))
```

### reduce()

`reduce()` 把一个接收两个参数的函数，累积作用到序列上。效果等价于：

```python
reduce(f, [x1, x2, x3, x4]) == f(f(f(x1, x2), x3), x4)
```

需要从 `functools` 导入：

```python
from functools import reduce

def add(x, y):
    return x + y

reduce(add, [1, 3, 5, 7, 9])   # 25
```

求和用内置 `sum()` 即可，`reduce` 的价值在于表达累积变换。例如把序列拼成整数：

```python
from functools import reduce

def fn(x, y):
    return x * 10 + y

reduce(fn, [1, 3, 5, 7, 9])   # 13579
```

### map + reduce 组合：手写 str2int

字符串 `str` 也是序列，`map` 先把每个字符转数字，`reduce` 再累积拼成整数：

```python
from functools import reduce

DIGITS = {'0': 0, '1': 1, '2': 2, '3': 3, '4': 4,
          '5': 5, '6': 6, '7': 7, '8': 8, '9': 9}

def char2num(s):
    return DIGITS[s]

def str2int(s):
    return reduce(lambda x, y: x * 10 + y, map(char2num, s))

str2int('13579')   # 13579
```

`map(char2num, s)` 把 `'13579'` 转成数字序列 `[1,3,5,7,9]`，`reduce` 再归约成整数 `13579`。

### 与列表生成式的关系

`map` 的很多场景可用列表生成式替代，且社区更推崇生成式（更易读）：

```python
list(map(lambda x: x * x, range(10)))   # map + lambda
[x * x for x in range(10)]              # 列表生成式，更地道
```

`reduce` 在 Python 3 中已从内置函数移到 `functools`，多数累积场景用循环或 `sum()` / `max()` 等更清晰。

## 返回函数与闭包（Closure）

高阶函数不仅能接收函数，还能把函数作为返回值。

### 惰性求和示例

```python
def lazy_sum(*args):
    def sum():
        ax = 0
        for n in args:
            ax = ax + n
        return ax
    return sum        # 返回函数本身，不是求和结果

f = lazy_sum(1, 3, 5, 7, 9)   # f 是函数，尚未计算
f()                           # 25，调用时才真正求和
```

内部函数 `sum` 引用了外部函数的参数 `args`，当 `lazy_sum` 返回 `sum` 时，相关变量都被保存在返回的函数里，这种结构称为**闭包**。

每次调用 `lazy_sum()` 都返回**全新的函数**，即使参数相同：`lazy_sum(1,3) == lazy_sum(1,3)` 为 `False`。

### 闭包陷阱：不要引用循环变量

返回的函数不会立刻执行，等到执行时循环变量已经变化：

```python
def count():
    fs = []
    for i in range(1, 4):
        def f():
            return i * i
        fs.append(f)
    return fs

f1, f2, f3 = count()
f1(), f2(), f3()   # 9, 9, 9 ← 都是 9，不是 1,4,9！
```

原因：三个函数都引用同一个变量 `i`，返回时 `i` 已变成 3。**返回闭包时牢记：不要引用任何会变化的循环变量。**

若必须引用，再套一层函数，用参数把当前值绑定住：

```python
def count():
    def f(j):
        def g():
            return j * j
        return g
    fs = []
    for i in range(1, 4):
        fs.append(f(i))   # f(i) 立刻执行，i 的当前值被传入
    return fs
# 结果 1, 4, 9
```

### nonlocal

闭包内**只读**外层变量正常；但若要**赋值**外层变量，Python 会把它当成内层函数的局部变量而报错，需用 `nonlocal` 声明：

```python
def inc():
    x = 0
    def fn():
        nonlocal x        # 声明 x 不是局部变量，而是外层的
        x = x + 1
        return x
    return fn

f = inc()
f()   # 1
f()   # 2
```

## 匿名函数（lambda）

不需要显式定义函数时，可直接用 `lambda` 传入匿名函数：

```python
list(map(lambda x: x * x, [1, 2, 3]))   # [1, 4, 9]

# 等价于
def f(x):
    return x * x
```

- 语法：`lambda 参数: 表达式`，冒号前是参数，冒号后是返回的表达式。
- **限制**：只能有一个表达式，不写 `return`，返回值就是表达式结果。
- 匿名函数也是函数对象，可赋值给变量、可作为返回值：

```python
f = lambda x: x * x
f(5)   # 25

def build(x, y):
    return lambda: x * x + y * y   # 返回匿名函数
```

## 装饰器（Decorator）

**本质：一个返回函数的高阶函数**，用于在不修改原函数定义的前提下，动态增加功能（如打印日志）。借助 `@` 语法糖使用。

### 基本装饰器

```python
def log(func):
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper

@log
def now():
    print('2024-6-1')

now()
# call now():
# 2024-6-1
```

`@log` 等价于执行 `now = log(now)`：`now` 变量被重新指向 `log` 返回的 `wrapper` 函数。`wrapper` 用 `(*args, **kw)` 接收任意参数。

### 带参数的装饰器（三层嵌套）

装饰器本身需要参数时，再包一层「返回装饰器的函数」：

```python
def log(text):
    def decorator(func):
        def wrapper(*args, **kw):
            print('%s %s():' % (text, func.__name__))
            return func(*args, **kw)
        return wrapper
    return decorator

@log('execute')
def now():
    print('2024-6-1')
# 等价于 now = log('execute')(now)
```

### functools.wraps 保留元信息

装饰后函数的 `__name__` 会变成 `'wrapper'`，破坏依赖函数签名的代码。用 `@functools.wraps(func)` 把原函数的 `__name__` 等属性复制过来：

```python
import functools

def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper
```

固定写法：在定义 `wrapper()` 前加 `@functools.wraps(func)` 即可。

## 偏函数（Partial Function）

`functools.partial` 把一个函数的**某些参数固定住**（设默认值），返回一个调用更简单的新函数。注意与数学上的偏函数无关。

```python
int('12345')            # 12345，默认十进制
int('12345', base=8)    # 5349，八进制

# 转大量二进制时，固定 base=2
import functools
int2 = functools.partial(int, base=2)
int2('1000000')         # 64
int2('1010101')         # 85
int2('1000000', base=10)  # 1000000，固定的默认值仍可被覆盖
```

`functools.partial` 接收「函数对象 + `*args` + `**kw`」三部分：

- 固定关键字参数：`partial(int, base=2)` → 调用 `int2('10010')` 相当于 `int('10010', base=2)`。
- 固定位置参数：`partial(max, 10)` 把 `10` 加到 `*args` 最左边 → `max2(5,6,7)` 相当于 `max(10,5,6,7)`。

用途：函数参数太多需要简化时，用偏函数固定部分参数，调用更简单。

## 小结

- **高阶函数**：把函数作为参数传入，或作为返回值返回。
- `map`：将一个函数作用于一个序列，得到另一个序列（返回惰性 `Iterator`）。
- `reduce`：将一个函数依次作用于「上次结果」和「下一个元素」，归约得到最终结果（需 `from functools import reduce`）。
- **返回函数/闭包**：内层函数引用外层变量并被返回；牢记不要引用会变化的循环变量；对外层变量赋值需 `nonlocal`。
- **lambda**：单表达式匿名函数，无需 `return`。
- **装饰器**：返回函数的高阶函数，用 `@` 动态增强功能，配 `functools.wraps` 保留元信息。
- **偏函数**：`functools.partial` 固定部分参数，返回更易调用的新函数。

## 术语表

| 英文 | 词性 | 释义 |
| ---- | --- | ---- |
| higher-order function | 名词 | 高阶函数，接收函数作为参数或返回函数 |
| map | 动词/名词 | 映射，把函数作用到序列每个元素得到新序列 |
| reduce | 动词/名词 | 归约，把函数累积作用到序列得到单个结果 |
| Iterator | 名词 | 迭代器，惰性序列，可被 `next()` 驱动 |
| Iterable | 名词 | 可迭代对象，可被 `for` 遍历 |
| closure | 名词 | 闭包，内层函数引用外层变量并被返回，变量随函数保存 |
| nonlocal | 关键字 | 声明变量属于外层函数作用域，允许闭包内对其赋值 |
| lambda | 关键字 | 匿名函数，单表达式，无需 `return` |
| decorator | 名词 | 装饰器，返回函数的高阶函数，用 `@` 动态增强函数功能 |
| partial function | 名词 | 偏函数，固定原函数部分参数得到的新函数 |
