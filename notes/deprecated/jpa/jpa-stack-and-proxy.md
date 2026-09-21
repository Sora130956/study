# JPA 技术栈分层：JDBC / JPA / Hibernate / 方言 / 代理

> **来源:** 答疑整理（基于 Spring Data JPA Core Concepts 延伸）
> **对应计划周次:** 第 2 周 · 周一读 JPA Core Concepts（延伸）

## 核心理解

用 Spring Data JPA 访问数据库时，背后是一条分层的调用链：**你的代码 → JPA 规范 → Hibernate 实现 → JDBC → 数据库驱动 → 数据库**。理解每一层的职责，才能看懂"为什么只写接口就能查库""为什么换数据库基本不用改代码"。

一句话区分：**JDBC 是最底层的连接标准，JPA 是对象映射规范，Hibernate 是这套规范的实现，方言负责吸收各数据库的 SQL 差异，代理则是 Spring Data 为你的接口自动生成的实现对象。** Spring Boot 的角色只是把这一整套自动配置好。

## 关键点

### 分层全景

```
你的代码（操作对象）
   ↓
JPA 规范（@Entity / EntityManager 接口）   ← 规范，只定义"该有什么"
   ↓
Hibernate（JPA 的实现，生成 SQL）          ← 实现，真正干活
   ↓
JDBC（发 SQL、连数据库的标准 API）          ← Java 连库的底层标准
   ↓
数据库驱动（MySQL Driver…）                ← 各厂商实现 JDBC
   ↓
数据库（MySQL / Oracle / PG）
```

### JDBC：最底层的数据库连接标准

JDBC（Java Database Connectivity）是 Java 官方定义的**连接关系型数据库的 API 标准**，规定"怎么连库、怎么发 SQL、怎么读结果"，各数据库厂商提供各自的**驱动（Driver）**来实现。

```java
Connection conn = DriverManager.getConnection(url, user, pwd);
PreparedStatement ps = conn.prepareStatement("SELECT * FROM user WHERE username = ?");
ps.setString(1, "alice");
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    Long id = rs.getLong("id");          // 手动把列搬进对象
    String name = rs.getString("username");
}
rs.close(); ps.close(); conn.close();    // 手动管理资源
```

痛点：手写 SQL、手动行列映射、手动管连接，重复且易错。它是地基，但太原始。

### JPA：对象 ↔ 表的映射规范

JPA（Jakarta Persistence API）是 Java 官方的 **ORM 规范**（Object-Relational Mapping，对象关系映射）。目标：**用操作 Java 对象的方式操作数据库**。

要点：**JPA 只是规范（接口/注解标准），不是实现**。它定义 `@Entity`、`@Id`、`EntityManager` 等，本身不干活，需要具体框架去实现。

```java
User user = new User();
user.setUsername("alice");
entityManager.persist(user);              // 相当于 INSERT，但没写 SQL
User u = entityManager.find(User.class, 1L); // 相当于 SELECT
```

### Hibernate：JPA 最主流的实现

Hibernate 是实现 JPA 规范的具体框架（Spring Boot 默认使用）。类比：**JPA 像 USB 标准，Hibernate 像某个厂的 USB 设备**。

职责：把对象操作（`persist` / `find` / `findByUsername`）翻译成 SQL，通过 JDBC 发给数据库，再把结果组装回 Java 对象。它是 **JDBC 之上的一层封装**，免去手写样板代码。

### Hibernate 方言（Dialect）：吸收各数据库的 SQL 差异

各数据库 SQL 语法有差异，最典型是分页：

| 数据库 | 分页语法 |
|--------|---------|
| MySQL | `LIMIT 10 OFFSET 20` |
| Oracle | `ROWNUM` 或 `FETCH FIRST 10 ROWS ONLY` |
| SQL Server | `OFFSET ... FETCH NEXT` |

Hibernate 先生成**数据库无关**的中间查询，最后由**方言（Dialect）**翻译成当前数据库的具体 SQL。方言就是封装某数据库语法差异的**适配器**。

换数据库 ≈ 换方言（`MySQLDialect` → `OracleDialect`）。Spring Boot 通常能根据连接 URL **自动探测**方言，一般不用手动配。这就是"同一套 Repository 代码能跨数据库"的原理——差异被方言吸收。

> 边界：MongoDB、Redis 等**非关系型**数据库不走 `JpaRepository`，各有 `MongoRepository`、`RedisRepository`。

### Spring Data JPA 的"代理实现类"

只写接口、没写实现类，却能被注入并跑通：

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}

@Service
public class UserService {
    private final UserRepository userRepository; // 注入的是什么？
}
```

没有任何类 `implements UserRepository`，注入进来的是 **Spring Data JPA 在启动时动态创建的代理对象**。

**代理（Proxy）**：框架在运行时用动态字节码技术（JDK 动态代理 / CGLIB）**凭空生成一个实现了该接口的实例**，注册成 Spring Bean。代理收到调用后：

- **继承自 `JpaRepository` 的方法**（`findById` / `save`）→ 转发给 Spring Data 内置通用实现（底层调 Hibernate 的 `EntityManager`）。
- **自定义派生方法**（`findByUsername`）→ 代理**解析方法名**（`findBy` + `Username`），推导查询意图，生成 JPQL，交给 Hibernate 执行。

完整理解：**你声明接口表达"要查什么"，Spring Data 启动时扫描接口，为每个接口在内存里自动造一个实现类的代理实例，包办"方法名 → 查询逻辑"的翻译。** 这正是"接口即契约，实现由框架代理生成"的具体机制。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| JDBC (Java Database Connectivity) | n. | Java 连接关系型数据库的底层 API 标准 |
| Driver | n. | 数据库厂商提供的 JDBC 实现 |
| JPA (Jakarta Persistence API) | n. | Java 官方 ORM 规范，只定义接口/注解不含实现 |
| ORM (Object-Relational Mapping) | n. | 对象关系映射，用对象方式操作数据库 |
| Hibernate | n. | JPA 规范最主流的实现框架 |
| EntityManager | n. | JPA 操作实体的核心 API（persist/find 等） |
| Dialect | n. | 方言，Hibernate 中封装各数据库 SQL 差异的适配器 |
| Proxy | n. | 代理，运行时动态生成的接口实现对象 |
| JPQL | n. | JPA 查询语言，数据库无关，由方言翻译成具体 SQL |
