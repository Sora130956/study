# 面向对象（类 / 继承 / 多态）· Java 视角对照

> **来源:** <https://liaoxuefeng.com/books/python/oop/index.html>（面向对象编程）
> **对应计划周次:** 第 1 周 · 周二 · 面向对象
> **阅读方式:** 已有 Java OOP 基础，本篇只讲「Python 和 Java 哪里一样、哪里不一样」，不重复 OOP 概念本身。

## 核心差异总览

| 维度 | Java | Python |
| ---- | ---- | ------ |
| 实例引用 | 隐式 `this` | 显式 `self`（每个方法第一个参数） |
| 构造器 | 与类同名的方法 | `__init__` |
| 访问控制 | `private/protected/public` 关键字，编译期强制 | 靠命名约定 `_x` / `__x`，无真正私有 |
| 多态 | 基于继承 + 编译期类型检查 | 鸭子类型，运行期只看「有没有这个方法」 |
| 方法重载 | 同名不同参可共存 | 不支持，后定义覆盖前者（用默认参数替代） |
| 类型声明 | 强制静态类型 | 动态类型，类型注解仅提示不强制 |
| 抽象类/接口 | `abstract` / `interface` | `abc` 模块，或干脆不用（鸭子类型） |

一句话：**Python 的 OOP 语法更松、更靠约定，多态靠「能调用就行」而不是「类型对不对」。**

## 1. 类与实例：self 就是 Java 的 this

```python
class Student:
    def __init__(self, name, score):   # 构造器，self 相当于 Java 的 this
        self.name = name               # 实例属性，直接赋值即声明，不用先声明字段
        self.score = score

    def print_score(self):             # 每个实例方法第一个参数必须是 self
        print(f'{self.name}: {self.score}')

bart = Student('Bart', 59)             # 不需要 new 关键字
bart.print_score()                     # self 由解释器自动传入，不用手写
```

对照 Java：

```java
class Student {
    String name; int score;            // Java 要先声明字段
    Student(String name, int score) { this.name = name; this.score = score; }
    void printScore() { System.out.println(name + ": " + score); }
}
Student bart = new Student("Bart", 59);
```

**关键差异**：
- 没有 `new`，直接 `Student(...)` 就是实例化。
- 属性不用提前声明，`self.name = ...` 第一次赋值就创建了。
- `self` 必须**显式**写在每个方法的第一个参数位，调用时不用传（`bart.print_score()` 而非 `bart.print_score(bart)`）。这点最容易让 Java 程序员别扭，记住：**定义时写 self，调用时不写**。

## 2. 访问控制：Python 没有真正的 private

Java 有 `private` 编译期强制。Python **没有关键字**，只有命名约定：

```python
class Student:
    def __init__(self, name):
        self.name = name        # public：外部可随意访问
        self._score = 0         # 单下划线：约定「内部使用」，但外部仍能访问（君子协定）
        self.__id = 1001        # 双下划线：名称改写(name mangling)，外部不能直接访问
```

- `self.name`：完全公开。
- `self._score`：**约定**私有，IDE 会提示，但 `stu._score` 照样能读能写，全靠自觉。
- `self.__id`：Python 把它改名成 `_Student__id`，所以 `stu.__id` 会报 `AttributeError`——**这不是真私有**，`stu._Student__id` 仍能访问，只是防止子类误覆盖。

**给 Java 程序员的心智模型**：Python 相信程序员，不做强制封装。想要「只读属性」用 `@property`：

```python
class Student:
    def __init__(self, score):
        self._score = score

    @property                    # 相当于 getter，外部用 stu.score 读
    def score(self):
        return self._score

    @score.setter                # 相当于 setter，带校验
    def score(self, value):
        if not 0 <= value <= 100:
            raise ValueError('score must be 0-100')
        self._score = value

stu = Student(80)
stu.score          # 80，像访问属性一样，实际调了方法
stu.score = 90     # 触发 setter 校验
stu.score = 200    # ValueError
```

`@property` 比 Java 的 `getXxx/setXxx` 更优雅：**调用方写法和普通属性一样**，不用写括号。

**`@property` 机制拆解**：它把一个方法**伪装成属性**，外部 `stu.score` 看似读字段，实际调用了被装饰的方法。三处名字有讲究：

- `@property` 下的 `def score` → 决定**对外属性名**，同时是「读」的逻辑
- `@score.setter` 下的 `def score` → 「写」的逻辑，名字必须和 getter 一致
- 内部真实数据存在**另一个名字** `_score`（不能同名）——若 getter 里写 `return self.score` 会「读 score 要先读 score」造成**无限递归爆栈**

| 名字 | 角色 | 谁用 |
| ---- | ---- | ---- |
| `score`（方法名） | 对外暴露的属性名 | 外部 `stu.score` |
| `_score`（带下划线） | 内部真正存数据的字段 | 方法内部 `self._score` |

**只读属性 = 只写 getter，不写 setter**。没有 setter，Python 不知道怎么「写」，赋值直接报错：

```python
class Student:
    def __init__(self, score):
        self._score = score

    @property
    def score(self):        # 只有 getter，故意不写 setter
        return self._score

stu.score          # ✅ 能读
stu.score = 90     # ❌ AttributeError: can't set attribute（这就是只读）
```

相当于 Java 里只给 `getScore()` 不给 `setScore()`（或字段 `final`）。

**但 `_score` 能绕过校验直接改**——因为 `@property` 只作用在 `score` 这个名字上，`_score` 只是普通属性：

```python
stu.score = 200      # ❌ ValueError，走 setter 校验
stu._score = 200     # ✅ 成功！绕过校验直接写真实数据，脏数据进去了
```

这再次印证「Python 没有真 private」：`@property` 防的是**误操作、引导正常调用走校验路径**，不是 Java `private` 那种**安全边界**。`__score`（双下划线）也挡不住，`stu._Student__score` 照样能访问。真需要强保证（安全 / 金额），得在**系统边界**（用户输入、API 入口）统一校验，不能指望 `@property`。

## 3. 继承与多态

```python
class Animal:
    def run(self):
        print('Animal is running...')

class Dog(Animal):               # 括号里写父类，相当于 Java 的 extends
    def run(self):               # 直接同名即重写(override)，不用 @Override
        print('Dog is running...')

class Cat(Animal):
    def run(self):
        print('Cat is running...')
```

多态调用：

```python
def run_twice(animal):           # 注意：参数没有类型声明
    animal.run()
    animal.run()

run_twice(Dog())   # Dog is running...
run_twice(Cat())   # Cat is running...
```

**最大差异——鸭子类型（Duck Typing）**：

Java 里 `run_twice` 必须声明参数类型为 `Animal`，只有 `Animal` 及其子类能传入。Python 不看类型，**只看这个对象有没有 `run()` 方法**：

```python
class Timer:                     # 完全没继承 Animal
    def run(self):
        print('Start...')

run_twice(Timer())   # 照样能跑！Start... Start...
```

> 「走起来像鸭子、叫起来像鸭子，那就当它是鸭子。」——Python 不要求继承关系，只要求**有对应的方法**。这让 Python 的多态比 Java 灵活得多，但也失去了编译期类型检查的保护。

调用父类方法用 `super()`：

```python
class Dog(Animal):
    def __init__(self, name):
        super().__init__()       # 相当于 Java 的 super()
        self.name = name
```

## 4. 没有方法重载，用默认参数替代

Java 可以靠参数不同定义多个同名方法。Python **不行**，后定义的会覆盖前面的：

```python
class Foo:
    def bar(self, x): ...
    def bar(self, x, y): ...     # 直接覆盖上面那个，上面的失效
```

替代方案是**默认参数 / 可变参数**（呼应你学过的函数章节）：

```python
class Foo:
    def bar(self, x, y=None):    # 一个方法搞定「一个参数」和「两个参数」
        if y is None:
            ...
        else:
            ...
```

## 5. 常用特殊方法（dunder methods）

Python 靠「双下划线方法」实现运算符和内置函数行为，类似 Java 的 `toString()`、`equals()`、`Comparable` 等，但更系统：

| 特殊方法 | 作用 | Java 类比 |
| -------- | ---- | --------- |
| `__init__` | 构造器 | 构造方法 |
| `__str__` | `print(obj)` / `str(obj)` 的显示 | `toString()` |
| `__repr__` | 调试 / 交互式回显的显示 | 无直接对应 |
| `__eq__` | `==` 比较 | `equals()` |
| `__len__` | `len(obj)` | `size()` |
| `__call__` | 让实例能像函数一样调用 `obj()` | 无（近似 `Function`） |

```python
class Student:
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return f'Student({self.name})'

print(Student('Bob'))    # Student(Bob)，否则打印出 <__main__.Student object at 0x...>
```

## 6. `__slots__`：限制实例属性（选学）

Python 实例默认能随意加属性（因为靠 `__dict__` 存）。`__slots__` 可限制只能有指定属性，省内存：

```python
class Student:
    __slots__ = ('name', 'age')   # 只允许这两个属性
```

Java 天生就是固定字段，这个特性对你来说是「Python 想变回 Java 的固定结构」时才用，日常很少用。

## 小结（Java 程序员速记）

- **self = this**，但定义时要显式写，调用时不写。
- **不用 new**，`ClassName()` 直接实例化；属性赋值即声明。
- **没有真 private**，靠 `_` / `__` 约定 + `@property` 做封装。
- **继承用 `(父类)`，重写直接同名**，`super()` 调父类。
- **多态靠鸭子类型**——只看有没有方法，不看类型，无编译期检查。
- **没有方法重载**，用默认参数 / `*args` 替代。
- **特殊方法**（`__str__`/`__eq__` 等）对应 Java 的 `toString`/`equals`，靠双下划线约定实现。

> 心态调整：Python OOP「能少写就少写、能靠约定就不强制」。从 Java 过来，最需要放下的是「编译器会帮我检查类型和访问权限」——在 Python 里这些都靠**约定和运行期**，写单元测试比在 Java 里更重要。
