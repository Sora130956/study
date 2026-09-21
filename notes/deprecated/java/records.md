# Records

> **来源:**
> - JEP 395: https://openjdk.org/jeps/395
> - Java 17 Tutorial: https://docs.oracle.com/en/java/javase/17/language/records.html
>
> **对应计划周次:** 第 1 周 · 现代 Java 速览

## 核心理解

`record` 是 Java 16 正式引入的一种**特殊的类**，专门用来声明**不可变的数据载体**（plain data carrier）。一行 `record Point(int x, int y) {}` 就等价于一个完整的不可变类：自动生成 `private final` 字段、规范构造器、访问器（`x()`、`y()`，注意没有 `get` 前缀）、`equals` / `hashCode` / `toString`。

它的设计动机非常明确：**消除"普通 Java 类承载数据时的样板噪音"**。在 Java 14 之前，写一个简单的"姓名+电话"数据类要写 50 行（字段、构造器、getter、equals、hashCode、toString），现在一行搞定，而且语义更清晰——`record` 在语法层面就告诉读者"这是一个不可变的数据组合，不要给它加可变状态"。

在 Mini CRM 里，**最适合的场景就是 DTO（数据传输对象）**：Controller 接收请求体、向前端返回响应、Service 之间传递参数。这些对象天生不需要可变、不需要继承、只需要把几个字段打包传输，正是 record 的甜区。

## 关键点

### 1. 一行声明，自动生成所有样板

> "A record class declares its state—the group of variables—and commits to an API that matches that state."（JEP 395）

```java
public record CustomerDto(Long id, String name, String phone) {}
```

编译器自动生成：
- `private final Long id; private final String name; private final String phone;`
- **规范构造器**（canonical constructor）：`CustomerDto(Long id, String name, String phone)`
- **访问器方法**：`id()`、`name()`、`phone()`（**注意：没有 `get` 前缀**）
- `equals(Object)`：所有字段相等才相等
- `hashCode()`：基于所有字段
- `toString()`：`CustomerDto[id=1, name=张三, phone=138...]`

### 2. record 是隐式 final 且不能继承类

> "A record class is implicitly final, and cannot be abstract."

- `record` 类是隐式 `final`，**不能被继承**
- `record` **不能 extends** 任何类（已隐式 extends `java.lang.Record`）
- 但 **可以 implements 接口**
- 所有字段都是 `private final`，**不可变**

```java
public record LeadDto(Long id, String name) implements Serializable {}  // ✅
public record LeadDto(...) extends BaseDto {}                           // ❌
```

### 3. 紧凑构造器：在不重写参数列表的情况下做校验/规范化

> "A compact constructor lets you put validation and normalization logic at the top of the canonical constructor."

```java
public record CustomerDto(Long id, String name, String phone) {
    // 紧凑构造器：不写参数列表
    public CustomerDto {
        Objects.requireNonNull(name, "name 不能为 null");
        if (phone != null && !phone.matches("\\d{11}")) {
            throw new IllegalArgumentException("手机号格式错误");
        }
        // 不需要写 this.id = id 之类——编译器在末尾自动赋值
    }
}
```

### 4. 可以加静态字段、静态方法、实例方法

> "Records can declare static fields, static methods, and instance methods."

```java
public record Money(BigDecimal amount, String currency) {
    public static final Money ZERO_CNY = new Money(BigDecimal.ZERO, "CNY");

    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("币种不一致");
        }
        return new Money(amount.add(other.amount), currency);
    }
}
```

但**不能声明额外的实例字段**——所有实例状态必须在 header 的"组件"里声明。

### 5. 可以重写访问器和 equals/hashCode/toString（但通常不需要）

```java
public record Customer(Long id, String name) {
    // 重写访问器：返回防御性拷贝、做格式化等
    @Override
    public String name() {
        return name == null ? "" : name.trim();
    }
}
```

### 6. 局部 record：方法内部声明临时数据结构

> "Records can be declared locally inside a method."

```java
public List<TopCustomer> top3Customers(List<Customer> all) {
    // 方法内的临时数据结构
    record Pair(Customer c, int orderCount) {}

    return all.stream()
            .map(c -> new Pair(c, countOrders(c)))
            .sorted(Comparator.comparingInt(Pair::orderCount).reversed())
            .limit(3)
            .map(p -> new TopCustomer(p.c().id(), p.orderCount()))
            .toList();
}
```

这是 Stream 链路中临时聚合的利器，避免污染顶层类型。

## 为什么 record 适合做 DTO

### DTO 是什么

**DTO（Data Transfer Object，数据传输对象）** 是软件分层架构里专门用于**跨层/跨进程传递数据**的对象。它不包含业务逻辑，只是一个"字段集合"，目的是把数据从一个边界搬到另一个边界。

**典型使用场景：**

| 场景 | 角色 | 例子 |
|------|------|------|
| HTTP 请求体 | Request DTO | `CreateLeadRequest(String name, String phone, String source)` |
| HTTP 响应体 | Response DTO | `LeadResponse(Long id, String name, String stage, Instant createdAt)` |
| Service 间参数 | Command/Query DTO | `ConvertLeadCommand(Long leadId, Long ownerId)` |
| 列表分页结果 | Page DTO | `PageResult<LeadResponse>(List<LeadResponse> items, long total)` |

**为什么需要 DTO 而不直接用 JPA 实体（Entity）？**

1. **隔离持久层与表现层**：Entity 暴露给前端会泄露数据库结构、可能触发懒加载、可能反序列化失败
2. **裁剪字段**：Entity 有 20 个字段，接口只需要 5 个——DTO 精确控制对外形状
3. **安全**：避免前端直接提交 Entity 时绕过校验、篡改 `id`/`tenantId` 等敏感字段
4. **演化解耦**：数据库表改字段不影响 API 契约；API 改字段不影响数据库

### record 为什么是 DTO 的天然形态

| DTO 的诉求 | record 的特性 | 命中 |
|-----------|---------------|------|
| 只是数据载体，无业务逻辑 | record 强制把状态写在 header，禁止额外实例字段 | ✅ |
| 不可变（避免传输途中被改） | record 字段隐式 `final` | ✅ |
| 字段相等即对象相等（便于比较、缓存 key） | record 自动生成基于字段的 `equals`/`hashCode` | ✅ |
| 调试/日志友好 | record 自动生成 `toString` | ✅ |
| 写起来简洁，专注于"有哪些字段" | 一行声明，无样板 | ✅ |
| 与 JSON 序列化（Jackson）兼容 | Jackson 2.12+ 原生支持 record | ✅ |
| 输入校验 | 紧凑构造器 + Bean Validation 注解 | ✅ |

**对比传统 POJO DTO：**

```java
// 传统写法：50+ 行
public class CustomerDto {
    private Long id;
    private String name;
    private String phone;
    public CustomerDto() {}
    public CustomerDto(Long id, String name, String phone) { ... }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    // ... 6 个 getter/setter ...
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// record 写法：1 行
public record CustomerDto(Long id, String name, String phone) {}
```

### Mini CRM 里的实际用法示例

```java
// 1) 请求 DTO + 校验
public record CreateLeadRequest(
        @NotBlank String name,
        @Pattern(regexp = "\\d{11}") String phone,
        @NotNull LeadSource source
) {}

// 2) 响应 DTO
public record LeadResponse(
        Long id,
        String name,
        String phone,
        LeadStage stage,
        Instant createdAt
) {
    // 静态工厂：从 Entity 转 DTO
    public static LeadResponse from(Lead lead) {
        return new LeadResponse(
                lead.getId(),
                lead.getName(),
                lead.getPhone(),
                lead.getStage(),
                lead.getCreatedAt()
        );
    }
}

// 3) 分页结果
public record PageResult<T>(List<T> items, long total, int page, int size) {}

// 4) Service 间命令
public record ConvertLeadCommand(Long leadId, Long ownerOrgId) {}
```

### record 不适合做什么

- ❌ **JPA Entity**：Hibernate 要求无参构造器 + 可变字段（脏检查）→ Entity 仍用传统类
- ❌ **有可变状态的领域对象**：购物车、订单聚合根 → 用普通类
- ❌ **需要继承的对象**：record 隐式 final，不能被继承
- ❌ **需要懒加载/代理**：JPA/CGLIB 代理依赖可继承类

## 句子解析

### 原文: "A record class is a shallowly immutable, transparent carrier for a fixed set of values."

- **翻译:** record 类是一种"浅层不可变"的、对一组固定值的"透明载体"。
- **解析:**
  - `shallowly immutable` = **浅层不可变**：字段引用不可变，但如果字段本身是可变对象（如 `List`），其内容仍可能被改。完整不可变需要自己加防御
  - `transparent carrier` = 透明载体：所有状态都通过 header 暴露，没有隐藏字段
  - `fixed set of values` = 固定的一组值：组件数量、顺序、类型在编译期固定

### 原文: "A record class declares its state—the group of variables—and commits to an API that matches that state."

- **翻译:** record 类声明它的状态（这一组变量），并承诺一套与该状态一一对应的 API。
- **解析:**
  - `commits to` 这里是"承诺/绑定"的意思——你声明了字段，编译器就承诺给你对应的访问器、构造器、equals 等
  - 这句话点出 record 的设计哲学：**状态即 API**，写了字段就等于定义了对外接口

### 原文: "A record class is implicitly final, and cannot be abstract."

- **翻译:** record 类是隐式 final 的，不能是抽象类。
- **解析:**
  - `implicitly final` = 隐式 final：不用写 `final` 关键字，编译器自动加
  - `cannot be abstract` 与 final 呼应——既然不能被继承，也就不能是 abstract
  - 含义：record 是"叶子类型"，专为终态的数据形状而设

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| record | n. | 记录类（Java 16+ 的不可变数据载体） |
| header / components | n. | record 头部声明的字段列表 |
| canonical constructor | n. | 规范构造器（参数与 header 一一对应） |
| compact constructor | n. | 紧凑构造器（省略参数列表，用于校验/规范化） |
| accessor | n. | 访问器方法（无 `get` 前缀，如 `name()`） |
| data carrier | n. | 数据载体 |
| shallowly immutable | adj. | 浅层不可变 |
| DTO (Data Transfer Object) | n. | 数据传输对象 |
| POJO (Plain Old Java Object) | n. | 普通 Java 对象 |
| boilerplate | n. | 样板代码 |
