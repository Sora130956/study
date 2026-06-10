# Local Variable Type Inference (`var`)

> **来源:**
> - JEP 286: https://openjdk.org/jeps/286
> - Java 17 Tutorial: https://docs.oracle.com/en/java/javase/17/language/local-variable-type-inference.html
>
> **对应计划周次:** 第 1 周 · 现代 Java 速览

## 核心理解

`var` 是 Java 10 引入的**局部变量类型推断**关键字（注意：不是真正的关键字，而是"保留类型名"——你仍然可以用 `var` 作变量名、方法名）。它让编译器根据**初始化表达式的静态类型**自动推断变量类型，从而省去重复写一遍类型声明的样板代码。

它的设计动机非常克制：**只在局部变量**这一最噪音、最安全的位置降低样板，类型信息仍然是完全静态的（不是动态类型），编译完成后字节码里有明确类型，运行时没有额外开销。

在 Mini CRM 里，最常见的使用场景是 Service 层里写 `var customer = customerRepository.findById(id).orElseThrow(...);`，或者循环遍历 `var entry : map.entrySet()`，避免把又长又啰嗦的泛型类型再写一遍。

## 关键点

### 1. 只能用于"有初始值的局部变量"

> "Allow var to be used to declare a local variable... with an initializer."（JEP 286）

`var` **必须有初始化表达式**，且只能用在：
- 方法/构造器/初始化块内的局部变量
- 增强 for 循环的索引变量
- 传统 for 循环的索引变量
- try-with-resources 的资源变量

**不能**用于：成员字段、方法参数、方法返回类型、catch 参数、lambda 形参。

```java
// ✅ 局部变量
var name = "Alice";          // 推断为 String

// ✅ 增强 for
for (var lead : leads) { ... }   // 推断为 Lead

// ✅ 传统 for
for (var i = 0; i < 10; i++) { ... }   // 推断为 int

// ✅ try-with-resources
try (var reader = Files.newBufferedReader(path)) { ... }
```

### 2. 类型由初始化表达式静态决定，且声明后不可变

> "The type of the variable is inferred from the type of the initializer."

推断结果是**编译期就确定的静态类型**，不是运行时类型，也不是"动态类型"。后续赋值必须兼容这个类型。

```java
var count = 0;            // 推断为 int
count = "hello";          // ❌ 编译错误：不能把 String 赋给 int
```

### 3. 推断的是"声明类型"，不是值的运行时类型

```java
// ✅ map 的声明类型是 HashMap<String, Customer>
var map = new HashMap<String, Customer>();

// ⚠️ 与 Map<String, Customer> m = new HashMap<>(); 不等价
//   前者声明类型是 HashMap，暴露更多实现细节；
//   接口编程的场景下还是显式写接口类型更稳。
```

### 4. 与菱形 `<>` 一起用时，泛型参数会被推断为 `Object`

> "Inference for var of generic types... defaults to Object."

```java
var list = new ArrayList<>();    // 推断为 ArrayList<Object>，几乎没用
List<Customer> list2 = new ArrayList<>();   // 这才是想要的

// 用 var 时显式给泛型
var list3 = new ArrayList<Customer>();   // ArrayList<Customer>
```

### 5. 推荐的使用风格（来自 OpenJDK 官方风格指南）

- 优先用于**初始化表达式右侧类型信息足够明显**的场合
- 变量名要更有信息量，弥补类型不可见
- 避免在右侧是 `null`、lambda、方法引用、数组初始化器等无法推断的位置使用

## 正确使用例子

```java
// 1) 类型从右侧一眼可见
var customer = new Customer("张三", "13800000000");

// 2) 避免重复啰嗦的泛型
var leadsByStage = new HashMap<LeadStage, List<Lead>>();
//  ↑ 比 HashMap<LeadStage, List<Lead>> leadsByStage = new HashMap<>(); 短

// 3) 增强 for + Stream 中间结果
var opportunities = opportunityRepository.findByCustomerId(id);
for (var opp : opportunities) {
    log.info("商机: {}", opp.getName());
}

// 4) try-with-resources，类型从工厂方法一目了然
try (var in = Files.newInputStream(path)) {
    return in.readAllBytes();
}

// 5) 一目了然的链式调用结果
var totalAmount = opportunities.stream()
        .map(Opportunity::getAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
```

## 错误使用例子

```java
// ❌ 1) 没有初始化表达式
var x;                        // 编译错误：cannot infer type
x = 10;

// ❌ 2) 用 null 初始化
var name = null;              // 编译错误：variable initializer is 'null'

// ❌ 3) 作为字段
public class Customer {
    var name = "张三";        // 编译错误：'var' is not allowed here
}

// ❌ 4) 作为方法参数 / 返回类型
public var save(var customer) { ... }   // 编译错误

// ❌ 5) 作为 catch 形参
try { ... } catch (var e) { ... }       // 编译错误

// ❌ 6) 作为 lambda 显式形参（不带类型）会冲突
BiFunction<Integer, Integer, Integer> add = (var a, var b) -> a + b;
//                                            ↑ Java 11 才允许，但要求全部参数都用 var

// ❌ 7) 数组初始化器没有目标类型
var arr = { 1, 2, 3 };        // 编译错误：array initializer needs explicit target type
var arr2 = new int[]{1,2,3};  // ✅ 这样可以

// ❌ 8) 让代码可读性变差的远距离推断
var result = service.process(input);   // result 到底是什么类型？读者要跳转
//  → 此时显式声明 Result<List<Customer>> result = ... 反而更友好

// ❌ 9) 误以为是动态类型
var v = "hello";
v = 123;                      // 编译错误：String 不能装 int
```

## 句子解析

### 原文: "We seek to improve the developer experience by reducing the ceremony associated with writing Java code, while maintaining Java's commitment to static type safety."

- **翻译:** 我们的目标是通过降低 Java 代码书写的"仪式感"来改进开发体验，同时保持 Java 对静态类型安全的承诺。
- **解析:**
  - `seek to improve` = 致力于改进
  - `ceremony` 在编程语境里特指"样板代码 / 冗余写法"
  - `while maintaining ...` 同时保持……，强调引入 `var` **不是**牺牲类型安全
  - 这句话直接点明 `var` 的设计哲学：减少噪音，但仍是静态类型

### 原文: "The type of the variable is inferred from the type of the initializer."

- **翻译:** 变量的类型从其初始化表达式的类型推断而来。
- **解析:**
  - `is inferred from` 是被动语态："从……被推断"
  - 关键词 `initializer`：必须有初始化表达式，编译器才能推断
  - 没有初始化器就没有类型来源 → 编译错误

### 原文: "`var` is not a keyword; instead, it is a reserved type name."

- **翻译:** `var` 不是关键字，而是一个保留的类型名。
- **解析:**
  - `keyword` vs `reserved type name`：区别在于"保留类型名"只在类型位置被特殊处理
  - 含义：你现有代码里如果有变量叫 `var`、方法叫 `var()`、包名叫 `var`，升级 Java 10+ **不会**被破坏
  - 但不能再有类名叫 `var`（类名属于类型位置）

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| local variable | n. | 局部变量（方法/块内声明的变量） |
| type inference | n. | 类型推断（编译器根据上下文推导类型） |
| initializer | n. | 初始化表达式（声明时右侧的赋值表达式） |
| reserved type name | n. | 保留类型名（不是关键字但在类型位置受限的标识符） |
| static type safety | n. | 静态类型安全（编译期类型检查保证） |
| ceremony | n. | （此处）样板、仪式性代码 |
| diamond operator | n. | 菱形运算符 `<>`，用于泛型推断 |
| enhanced for | n. | 增强 for 循环（`for (T x : iterable)`） |
| try-with-resources | n. | 自动资源管理的 try 语法 |
