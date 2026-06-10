# Switch Expressions and Statements

> **来源:**
> - Java 17 Tutorial: https://docs.oracle.com/en/java/javase/17/language/switch-expressions-and-statements.html
> - JEP 361 (Switch Expressions): https://openjdk.org/jeps/361
>
> **对应计划周次:** 第 1 周 · 现代 Java 速览

## 核心理解

Java 14 起，`switch` 从只能当**语句**用，升级为**既可以当语句、又可以当表达式**。
- **switch 语句**（statement）：执行一段逻辑，不产生值，用 `break;` 跳出
- **switch 表达式**（expression）：**算出一个值**，可以直接放在 `=` 右边，必须有结果（要么 `yield` 一个值，要么抛异常）

伴随而来的还有两个新语法元素：
- **箭头规则**（arrow label, `case X -> ...`）：右侧是表达式或代码块，**无 fall-through**（不会穿透到下一个 case），更安全
- **yield 语句**：在代码块形式的 case 里**显式返回一个值**给整个 switch 表达式

在 Mini CRM 里，最典型的用法是状态机分支判断（如 Lead 阶段流转）、枚举到字符串/金额的映射、根据角色返回不同数据范围常量——这些场景下 switch 表达式比传统 if/else 链条**更清晰、更不容易漏分支**。

## 关键点

### 1. 两种语法风格：箭头规则 vs 传统冒号

> "A switch labeled rule" vs "A switch labeled statement group"

**箭头规则（推荐）：** `case X -> 表达式 或 代码块`
- 右边是单个表达式 → 该表达式即为值
- 右边是代码块 → 必须用 `yield` 给出值
- **不会 fall-through**（不会自动穿透到下一个 case）
- 一个 case 可以匹配多个标签：`case MONDAY, FRIDAY -> ...`

```java
String type = switch (stage) {
    case NEW, CONTACTED -> "early";
    case QUALIFIED       -> "mid";
    case CONVERTED, LOST -> "final";
};
```

**传统冒号（兼容老语法）：** `case X: 语句; ... break/yield;`
- 仍保留 fall-through 行为
- 在 switch **表达式**里必须用 `yield 值;` 显式产出值
- 在 switch **语句**里用 `break;` 跳出

```java
String type = switch (stage) {
    case NEW:
    case CONTACTED:
        yield "early";      // 表达式里必须 yield，不能 break
    default:
        yield "other";
};
```

### 2. switch 表达式必须"算出一个值"

> "A switch expression must either complete normally with a value or complete abruptly by throwing an exception."

两种合法结局：
- **正常完成并带一个值**（normal completion with a value）：所有路径都给出一个值
- **异常中断**（abrupt completion by throwing an exception）：抛异常打断整个表达式

**编译器强制要求：**
- 必须**穷尽**所有分支（枚举要么覆盖所有常量、要么有 `default`）
- 任何代码块路径都必须 `yield`（或抛异常）

### 3. 三种编译错误（来自官方文档反例）

#### ❌ 错误一：箭头规则的代码块缺少 yield

> "the following code doesn't compile because the switch labeled rule doesn't contain a yield statement"

```java
// ❌ 不能编译
int j = switch (day) {
    case MONDAY -> {
        System.out.println("Monday");
        // 缺 yield → 块算不出值
    }
    default -> 0;
};

// ✅ 修正
int j = switch (day) {
    case MONDAY -> {
        System.out.println("Monday");
        yield 1;
    }
    default -> 0;
};
```

#### ❌ 错误二：冒号 statement group 缺少 yield

> "the switch labeled statement group doesn't contain a yield statement"

```java
// ❌ 不能编译
int result = switch (day) {
    case MONDAY:
    case TUESDAY:
        System.out.println("Workday");
        // 缺 yield → 走完没有值可给 result
    default:
        yield 0;
};

// ✅ 修正：每个分支都要 yield
int result = switch (day) {
    case MONDAY:
    case TUESDAY:
        System.out.println("Workday");
        yield 1;
    default:
        yield 0;
};
```

#### ❌ 错误三：不能用 break/return/continue/yield 跳出 switch 表达式

> "you can't jump through a switch expression with a break, yield, return, or continue statement"

```java
// ❌ 不能编译：在 switch 表达式里 continue 外层循环
int total = 0;
outer:
for (int i = 0; i < items.length; i++) {
    total += switch (items[i]) {
        case "A" -> 1;
        case "B" -> 2;
        default -> {
            continue outer;   // ❌ 试图跳过外层循环，但 switch 没产出值
        }
    };
}

// ✅ 修正：把控制流放到外面
int total = 0;
for (int i = 0; i < items.length; i++) {
    int v = switch (items[i]) {
        case "A" -> 1;
        case "B" -> 2;
        default  -> 0;
    };
    if (v == 0) continue;
    total += v;
}
```

**核心原因：** switch 表达式必须 `evaluate to a single value`（求值为一个值），用 break/return/continue 跳到外面就意味着"没产出值"，违背了表达式的本质。

### 4. yield 语句

> `yield` 用于在 switch 表达式的**代码块分支**里返回一个值。

```java
BigDecimal commission = switch (stage) {
    case CONVERTED -> opportunity.getAmount().multiply(new BigDecimal("0.05"));
    case LOST       -> BigDecimal.ZERO;
    default -> {
        log.warn("阶段 {} 不计算提成", stage);
        yield BigDecimal.ZERO;     // ✅ 代码块里用 yield
    }
};
```

- `yield 表达式;` 只能用在 switch 表达式内部
- 不要和 `return` 混淆：`return` 离开方法，`yield` 只离开当前 switch 表达式

### 5. 箭头规则无 fall-through

> "code following a switch labeled rule doesn't fall through to subsequent cases"

```java
// ❌ 老 switch 语句容易漏 break，意外穿透
switch (stage) {
    case NEW:
        log.info("new");
        // 漏写 break → 会继续执行 CONTACTED 分支！
    case CONTACTED:
        log.info("contacted");
        break;
}

// ✅ 箭头规则天然不穿透
switch (stage) {
    case NEW       -> log.info("new");
    case CONTACTED -> log.info("contacted");
}
```

### 6. 穷尽性检查（exhaustiveness）

switch 表达式在面对枚举时，编译器会检查是否覆盖了所有常量：

```java
public enum LeadStage { NEW, CONTACTED, QUALIFIED, CONVERTED, LOST }

// ❌ 编译错误：未覆盖 CONVERTED、LOST
String s = switch (stage) {
    case NEW       -> "n";
    case CONTACTED -> "c";
    case QUALIFIED -> "q";
};

// ✅ 加 default 或覆盖全部
String s = switch (stage) {
    case NEW, CONTACTED -> "early";
    case QUALIFIED      -> "mid";
    case CONVERTED      -> "won";
    case LOST           -> "lost";
};
```

这个特性的价值：**新增枚举常量时，编译器会立刻告诉你哪些 switch 还没更新**——比 if/else 链条安全得多。

## 正确使用例子

```java
// 1) 枚举到字符串的映射
String stageLabel = switch (lead.getStage()) {
    case NEW        -> "新线索";
    case CONTACTED  -> "已联系";
    case QUALIFIED  -> "已确认";
    case CONVERTED  -> "已转化";
    case LOST       -> "已流失";
};

// 2) 多标签合并
boolean isClosed = switch (stage) {
    case CONVERTED, LOST -> true;
    default              -> false;
};

// 3) 带副作用的代码块 + yield
BigDecimal commission = switch (stage) {
    case CONVERTED -> opportunity.getAmount().multiply(RATE);
    case LOST      -> BigDecimal.ZERO;
    default -> {
        log.debug("阶段 {} 暂不计提成", stage);
        yield BigDecimal.ZERO;
    }
};

// 4) 当语句用：箭头规则避免漏 break
switch (event) {
    case CREATED  -> activityService.recordCreate(lead);
    case UPDATED  -> activityService.recordUpdate(lead);
    case DELETED  -> activityService.recordDelete(lead);
}

// 5) 状态流转合法性校验
boolean canTransit = switch (currentStage) {
    case NEW         -> nextStage == CONTACTED || nextStage == LOST;
    case CONTACTED   -> nextStage == QUALIFIED || nextStage == LOST;
    case QUALIFIED   -> nextStage == CONVERTED || nextStage == LOST;
    case CONVERTED, LOST -> false;   // 终态不能再迁移
};
```

## 错误使用例子（编译错误）

```java
// ❌ 1) 代码块缺 yield
int x = switch (n) {
    case 1 -> { System.out.println("one"); }     // 缺 yield
    default -> 0;
};

// ❌ 2) 冒号 statement group 缺 yield
int y = switch (n) {
    case 1:
        System.out.println("one");               // 走完没值
    default:
        yield 0;
};

// ❌ 3) 用 break 跳出 switch 表达式
String s = switch (n) {
    case 1 -> { break; }                          // 不允许：要用 yield
    default -> "x";
};

// ❌ 4) continue 跳到外层循环
for (var x : list) {
    var v = switch (x) {
        default -> { continue; }                 // 不允许
    };
}

// ❌ 5) 枚举未穷尽且无 default
LeadStage s = ...;
String label = switch (s) {
    case NEW -> "n";
    case CONTACTED -> "c";
    // 缺 QUALIFIED/CONVERTED/LOST 也没 default → 编译错误
};

// ❌ 6) 在 switch 语句的箭头规则后写 break（多余且报错）
switch (stage) {
    case NEW -> { log.info("n"); break; }        // 箭头规则不需要 break
}
```

## 句子解析

### 原文: "a switch expression must either complete normally with a value or complete abruptly by throwing an exception."

- **翻译:** switch 表达式必须要么"正常完成并带一个值"，要么"通过抛异常异常中断"。
- **解析:**
  - `complete normally` vs `complete abruptly` 是 JLS（Java 语言规范）里的一对反义术语：前者指走完正常路径，后者指被异常/跳转打断
  - `with a value` 强调正常完成时**必须有值**
  - `either ... or ...` 二选一，没有"什么都不返回"这种结局
  - 这是 switch 表达式所有规则的根本依据

### 原文: "Because a switch expression must evaluate to a single value (or throw an exception), you can't jump through a switch expression with a break, yield, return, or continue statement"

- **翻译:** 因为 switch 表达式必须求值为单一的一个值（或抛异常），所以不能用 break、yield、return、continue 语句从 switch 表达式中"跳出去"。
- **解析:**
  - `evaluate to a single value` = "求值为一个值" → 表达式语义
  - `jump through` 这里表示"从内部跳到外面"
  - 列出的四个语句都会让表达式"没机会产出值"，所以被禁止
  - 注意 `yield` 本身是给出值用的，没值的 `yield;` 不允许；`yield 值;` 才是正确写法

### 原文: "code following a switch labeled rule doesn't fall through to subsequent cases"

- **翻译:** 箭头规则后面的代码不会"穿透"到后面的 case。
- **解析:**
  - `fall through` 是 C/Java 老 switch 的经典坑：漏写 break 会顺着执行下一个 case
  - 箭头规则从设计上根除这个坑，更安全
  - 想要"多 label 走同一逻辑"，用 `case A, B, C -> ...` 显式写出

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| switch expression | n. | switch 表达式（产生值的 switch） |
| switch statement | n. | switch 语句（不产生值的 switch） |
| switch labeled rule | n. | 箭头规则 `case X -> ...` |
| switch labeled statement group | n. | 冒号语句组 `case X: ...` |
| arrow label | n. | 箭头标签（`->` 形式） |
| yield | v./stmt. | 在 switch 表达式代码块内返回一个值 |
| fall through | v. | 穿透（漏 break 时自动执行下一个 case） |
| complete normally | v. | 正常完成（走完代码并产出值） |
| complete abruptly | v. | 异常中断（被异常/跳转打断） |
| exhaustiveness | n. | 穷尽性（覆盖所有可能分支） |
| evaluate to | v. | 求值为 |
