# JPA Query Methods

> **来源:** https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
> **对应计划周次:** 第 2 周 · 读 JPA Query Methods + 写笔记

## 核心理解

Spring Data JPA 给 Repository 方法对应查询逻辑有两条路：一是**方法名派生查询（Derived Query）**——你只写方法名（`findByNameContaining`），Spring 解析方法名自动生成查询；二是**声明式查询（Declared Query）**——你显式写出查询语句，又分 `@Query` 注解和 JPA 命名查询两种。决定某个方法用哪条路的机制叫 **Query Lookup Strategy（查询查找策略）**。

派生查询胜在简单，但条件一多方法名就爆炸式变长且不可读，且解析器支持的关键字有限。所以实战里：简单条件用派生查询，复杂查询直接 `@Query` 写 JPQL/原生 SQL。`@Query` 优先级高于命名查询，本笔记主线就是"能用 `@Query` 就用 `@Query`"。

在 Mini CRM 里，`UserRepository`、`CustomerRepository`、`LeadRepository` 的列表查询、关键字模糊搜索、分页列表都建立在本节能力之上：`findByXxx` 做精确匹配，`@Query` 做多条件/连表搜索，`Pageable` 做分页排序。

## 关键点

### 1. 两种查询来源：派生 vs 声明式

> The JPA module supports defining a query manually as a String or having it being derived from the method name.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // 派生查询：解析方法名 → where username = ?1
    User findByUsername(String username);

    // 声明式查询：自己写 JPQL
    @Query("select u from User u where u.username = :username")
    User queryByUsername(@Param("username") String username);
}
```

派生查询条件多了会失控，比如 `findByFirstnameAndLastnameAndAgeGreaterThanAndStatusOrderByCreatedDesc`——这种就该改用 `@Query`。

### 2. LIKE 派生查询会自动转义通配符（sanitize）

> Derived queries with the predicates ... Containing, Contains ... the respective arguments for these queries will get sanitized. ... characters recognized by LIKE as wildcards these will get escaped so they match only as literals.

`StartingWith` / `EndingWith` / `Containing`（及其同义词、`Not` 变体）这类底层走 `LIKE` 的派生查询，Spring 会自动把参数里的 `%`、`_` 转义成普通字符，避免用户输入被当成通配符。

```java
// 用户搜 "100%"，期望匹配字面量 100%，而不是把 % 当通配符
List<Customer> findByNameContaining(String keyword);
// 底层效果: where name like '%100\%%' escape '\'
```

转义字符可通过 `@EnableJpaRepositories(escapeCharacter = '\\')` 配置（默认 `\`）。注意：这只对**派生查询**自动生效，手写 `@Query` 的转义要自己负责。

| LIKE 通配符 | 含义 |
|------|------|
| `%` | 匹配任意长度任意字符 |
| `_` | 匹配任意单个字符 |

### 3. 派生查询关键字三类（StartingWith / EndingWith / Containing）

| 谓词（含同义词） | 生成的 LIKE |
|------|------|
| `StartingWith` / `StartsWith` / `IsStartingWith` | `like 'x%'` |
| `EndingWith` / `EndsWith` / `IsEndingWith` | `like '%x'` |
| `Containing` / `Contains` / `IsContaining` | `like '%x%'` |
| `NotContaining` / `NotContains` | `not like '%x%'` |

完整关键字表（`And`/`Or`/`Between`/`LessThan`/`In`/`IsNull`/`OrderBy` 等）见官方 "Supported keywords inside method names" 表，用时查阅即可，不必死记。

### 4. DISTINCT 的坑（重点）

> select distinct u from User u will produce a complete different result than select distinct u.lastname from User u.

`distinct` 作用在主键上等于没去重：

- `select distinct u from User u`：因为带了 `User.id`，每行天然唯一，等于查全表，返回 `List<User>`。
- `select distinct u.lastname from User u`：只聚焦 `lastname`，返回去重后的 `List<String>`，是完全不同的结果集。
- `countDistinctByLastname(...)` 会派生成 `select count(distinct u.id) ...`，`id` 不重复 → 等价于 `countByLastname(...)`，并非"去重的姓氏数"。

**结论：** 想要"去重统计"这类语义，派生查询往往表达不准，应手写 `@Query` 明确你到底要查什么（可能还需要投影 projection）。

### 5. @Query 优先级最高

> Queries annotated to the query method take precedence over queries defined using @NamedQuery or named queries declared in orm.xml.

同一方法上 `@Query` > `@NamedQuery` > `orm.xml` 命名查询。实战直接用 `@Query` 即可，命名查询/XML 基本不用碰。

### 6. @Query 里直接写 LIKE 通配符 %

> the LIKE delimiter character (%) is recognized, and the query is transformed into a valid JPQL query (removing the %). Upon running the query, the parameter ... gets augmented with the previously recognized LIKE pattern.

```java
@Query("select u from User u where u.firstname like %?1")
List<User> findByFirstnameEndingWith(String suffix);
```

机制：解析时 Spring 先把 `%` 从 JPQL 里摘掉（让它成为合法 JPQL），运行时再把 `%` 拼回参数值。`%?1` = "以某值结尾"。

补充 JPQL 占位符：
- 位置参数 `?1`（从 1 开始，不是 0）。
- 命名参数 `:name` 配 `@Param`，参数多时更不易错位，**推荐**。

JPQL 面向**对象与属性**（`User` 实体、`u.firstname` 字段），由 JPA 翻译成面向**表与列**的 SQL，数据库无关；关联用点号导航 `u.address.city` 而非手写 JOIN。

### 7. 原生查询 @NativeQuery 与分页 count 改写

> The @NativeQuery annotation is mostly a composed annotation for @Query(nativeQuery=true) but it also provides additional attributes such as sqlResultSetMapping ... More complex queries require either JSqlParser to be on the class path or a countQuery declared in your code.

```java
// 等价写法
@Query(value = "SELECT * FROM t_user WHERE email = ?1", nativeQuery = true)
@NativeQuery("SELECT * FROM t_user WHERE email = ?1")
```

- `@NativeQuery` 是 `@Query(nativeQuery=true)` 的组合注解，写原生 SQL。
- `sqlResultSetMapping` 把原生 SQL 查出的杂列按 `@SqlResultSetMapping` 映射成 DTO。
- 分页时 Spring 要额外跑一条 count 查询算总数。简单查询自动改写没问题；复杂查询（`GROUP BY`/子查询/`DISTINCT`/复杂 JOIN）改写会出错，需二选一：引入 `JSqlParser` 依赖，或手写 `countQuery`。**实战优先手写 `countQuery`**，更可控。

```java
@Query(
    value = "SELECT u FROM User u WHERE ...复杂查询",
    countQuery = "SELECT COUNT(u) FROM User u WHERE ...对应计数"
)
Page<Customer> findComplex(Pageable pageable);
```

### 8. Pageable / Page 分页排序

`Pageable` 封装"第几页 + 每页几条 + 排序"三件事，作为方法参数传入，Spring 自动给 SQL 加 `LIMIT/OFFSET` 和 `ORDER BY`。

```java
// 构造（页码从 0 开始！）
Pageable p = PageRequest.of(0, 10, Sort.by("createTime").descending());

Page<Customer> findByStatus(String status, Pageable pageable);
```

`Page<T>` 返回数据 + 分页元信息：`getContent()`、`getTotalElements()`（需 count 查询）、`getTotalPages()`、`hasNext()`。不需要总数时用 `Slice<T>`，省一次 count 查询。

Controller 可直接注入 `Pageable`，从 URL 参数 `?page=0&size=10&sort=createTime,desc` 自动拼装：

```java
@GetMapping("/customers")
public Page<Customer> list(Pageable pageable) {
    return customerService.findActive(pageable);
}
```

### 9. Sort 排序：字段校验与 JpaSort.unsafe

> The properties ... need to resolve to either a property or an alias used within the query ... Using any non-referenceable path expression leads to an Exception ... you can use JpaSort.unsafe to add potentially unsafe ordering.

`Sort.by("xxx")` 的字段必须能解析到**实体属性**或**查询里的别名**（JPQL 的 state field path expression），否则抛异常。这是为防止把用户输入直接拼进 `ORDER BY` 造成 SQL 注入。

```java
@Query("select u from User u where u.lastname like ?1%")
List<User> findByAndSort(String lastname, Sort sort);

@Query("select u.id, LENGTH(u.firstname) as fn_len from User u where u.lastname like ?1%")
List<Object[]> findByAsArrayAndSort(String lastname, Sort sort);

repo.findByAndSort("lannister", Sort.by("firstname"));                // ✅ 实体属性
repo.findByAndSort("stark", Sort.by("LENGTH(firstname)"));            // ❌ 函数被拦，抛异常
repo.findByAndSort("targaryen", JpaSort.unsafe("LENGTH(firstname)")); // ✅ 显式 unsafe 放行
repo.findByAsArrayAndSort("bolton", Sort.by("fn_len"));               // ✅ 指向查询别名
```

- 默认拒绝含函数调用的排序；`JpaSort.unsafe(...)` 是逃生舱，仅配合 `@Query` 有意义。
- **绝不能把用户输入塞进 `JpaSort.unsafe`**，等于开注入后门。
- 需按函数/计算列排序，优先用**查询别名**（如 `as fn_len`），比 unsafe 更安全。
- 对外排序字段务必走白名单校验。

### 10. 命名参数优于位置参数

> Spring Data JPA uses position-based parameter binding ... This makes query methods a little error-prone when refactoring regarding the parameter position. ... use @Param annotation to give a method parameter a concrete name.

位置参数 `?1`/`?2` 靠位置对应，调换参数顺序会静默错位（编译不报错，运行才出 bug）。`@Param` 靠名字绑定，与参数位置无关：

```java
@Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
User findByLastnameOrFirstname(@Param("lastname") String lastname,
                               @Param("firstname") String firstname);
// 方法签名 lastname 在前，SQL 里 :firstname 在前，因按名字绑定照样正确
```

- 参数 3 个以上几乎必选命名参数。
- 命名参数可**复用**（同一个 `:kw` 引用多次），位置参数做不到。
- 即使开了 `-parameters` 编译参数可省略 `@Param`，仍**推荐显式写**，不依赖编译配置。

### 11. @Modifying 修改型查询

> @Modifying ... triggers the query annotated to the method as an updating query instead of a selecting one ... you can set the @Modifying annotation's clearAutomatically attribute to true. The @Modifying annotation is only relevant in combination with the @Query annotation.

手写 `UPDATE`/`DELETE` 的 `@Query` 必须加 `@Modifying`，否则按 SELECT 执行报错。返回 `int` = 影响行数，且要在事务里（Service 层 `@Transactional`），否则报 `TransactionRequiredException`。

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Customer c set c.status = :status where c.id = :id")
int updateStatus(@Param("id") Long id, @Param("status") String status);
```

不需要 `@Modifying` 的三类：派生删除 `deleteByXxx`、`JpaRepository` 自带 `save/delete`、自定义实现方法。`@Modifying` 专为「手写 @Query 的批量改」。

### 12. 持久化上下文（一级缓存）与 @Modifying 的坑（重点）

**持久化上下文 = `EntityManager` 内部的内存缓存 + 状态追踪表**，也叫一级缓存。两大职责：

1. **缓存实体（同一性）**：同一事务同一条记录，多次查返回同一个 Java 对象。
2. **脏检查（Dirty Checking）**：托管态实体改了字段不用 `save`，**flush 时自动比对并发 UPDATE**。

实体三状态：
| 状态 | 含义 | 受管理 |
|------|------|------|
| Transient 瞬时态 | `new` 出来没存过库 | 否 |
| Managed 托管态 | `find`/`save` 得到的 | **是**，改了自动同步 |
| Detached 游离态 | 事务结束或被 clear 后 | 否，改了不会同步 |

**flush vs commit：** flush = 把变更生成 SQL 发给数据库（还能回滚）；commit = 事务落地。flush 默认触发时机：①事务提交前 ②执行查询前（怕查到旧数据）③手动 `em.flush()`。所以「自动 UPDATE」是 flush 在做，commit 前必然 flush。

**@Modifying 的坑：** 批量 UPDATE/DELETE 直接发 SQL **绕过持久化上下文**，缓存里的托管实体不知道数据库被旁路改了 → 缓存与库脱节（脏数据）。

```java
Customer c = repo.findById(1L).get();   // 缓存 status = "NEW"
repo.batchUpdateStatus(tid, "NEW", "WON"); // 库已改 "WON"，缓存仍 "NEW"
c.getStatus();                          // 读到旧值 "NEW"！
```

- `clearAutomatically = true`：执行后 clear 缓存，**让旧实体作废**。注意 clear 只是把旧实体踢出缓存，逼**下一次查询**查库；它**不会刷新你手里已持有的旧引用**（访问 `c.getStatus()` 仍是旧值，必须重新 `findById` 拿 `fresh`）。
- `flushAutomatically = true`：执行前先 flush，避免未写出的改动被 clear 丢掉。

**避坑原则（按场景选工具，而非死记规则）：**
- 改单条/几条还要接着用 → 用 `findById` + setter 走脏检查，最自然不出错。
- 批量改很多行、改完就结束 → 用 `@Modifying`，天然没问题。
- 批量改 + 同事务还要重读 → `@Modifying(flush+clear)` + 重新查询。

### 13. 派生删除 vs 批量删除（生命周期回调差异）

> the latter method issues a single JPQL query ... currently loaded instances of User do not see lifecycle callbacks invoked ... a derived delete query is a shortcut for running the query and then calling CrudRepository.delete(...).

```java
void deleteByRoleId(long roleId);                       // 派生：先 SELECT 加载，再逐个 delete

@Modifying
@Query("delete from User u where u.role.id = ?1")
void deleteInBulkByRoleId(long roleId);                 // 批量：单条 DELETE SQL
```

| 维度 | 派生删除 `deleteByXxx` | 批量 `@Modifying delete` |
|------|------|------|
| 执行 | 先 SELECT 全加载，再逐条 DELETE | 单条 DELETE SQL |
| `@PreRemove` 回调 | ✅ 触发 | ❌ 不触发 |
| 级联删除 / orphanRemoval | ✅ 生效 | ❌ 不生效 |
| 内存 | 大数据量吃内存（全加载进 session 直到事务结束） | 几乎不占 |
| 性能 | 慢（N+1 条 SQL） | 快（1 条） |

- 有级联/回调依赖 → 派生删除，安全。
- 批量物理删大量数据、无回调依赖 → `@Modifying delete`，快且省内存。
- CRM 更常做**软删除**（`update set deleted = true`），同样走 `@Modifying`，注意缓存一致性。

### 14. 实体生命周期回调与审计（@PreRemove / Auditing）

JPA 七个生命周期回调，在实体内写无参 void 方法标注即可，特定时机自动触发：

| 注解 | 时机 | | 注解 | 时机 |
|------|------|---|------|------|
| `@PrePersist` | insert 前 | | `@PostPersist` | insert 后 |
| `@PreUpdate` | update 前 | | `@PostUpdate` | update 后 |
| `@PreRemove` | delete 前 | | `@PostRemove` | delete 后 |
| `@PostLoad` | 查出加载后 | | | |

**关键：回调只在「实体级操作」触发**（`delete(entity)`/`deleteById`/派生删除）；批量 `@Modifying` 直接发 SQL，不加载实体，**不触发回调**。

实务里维护创建/更新时间，**推荐用 Spring Data Auditing 而非手写 `@PrePersist`**：

```java
@EntityListeners(AuditingEntityListener.class)
@Entity
public class Customer {
    @CreatedDate      private LocalDateTime createdAt;
    @LastModifiedDate private LocalDateTime updatedAt;
}
// 启动类/配置类加 @EnableJpaAuditing
```

> 学习心法：**记「能力」不记「拼写」**。脑子里留「JPA 能自动填时间戳/有删除前钩子」的模糊印象即可，具体注解名靠 IDE 补全、官方文档目录、看现成项目、搜索 反查。笔记突出"解决什么问题"就是为了用时 30 秒定位回来。

### 15. Scrolling 大结果集（YAGNI，先标记）

处理超大数据集、避免一次性全加载内存。三种消费方式：
- **Paging**：`Pageable` + `Page`，会额外查 count 算总页数，能跳到第 N 页。
- **Offset scrolling**：比分页轻，不查总数，本质 `LIMIT/OFFSET`，适合顺序往下刷。
- **Keyset scrolling**（键集/seek method）：靠「记住上一批末尾位置 + 索引」往后翻，避免 `OFFSET` 越深越慢的问题。返回 `Window<T>`，游标用 `ScrollPosition`。代价：只能顺序翻不能跳页，排序字段需有索引且唯一定位。

```java
Window<Customer> findByTenantId(Long tenantId, ScrollPosition position);
Window<Customer> w = repo.findByTenantId(tid, ScrollPosition.keyset());
ScrollPosition next = w.positionAt(w.size() - 1);  // 下一批从此位置接着取
```

> Mini CRM 作品阶段数据量小，常规列表用 `Page` 足够，Keyset **按 YAGNI 标 TODO 回读**，仅超大数据量/无限滚动才考虑。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| Query Lookup Strategy | n. | 查询查找策略，决定方法用派生/声明式查询 |
| Derived Query | n. | 派生查询，解析方法名自动生成 |
| Declared Query | n. | 声明式查询，显式写出查询语句 |
| predicate | n. | 谓词，方法名里的条件关键字（如 `Containing`） |
| sanitize | v. | 净化/转义，处理参数中的特殊字符 |
| escapeCharacter | n. | 转义字符，`@EnableJpaRepositories` 可配置 |
| JPQL | n. | Java 持久化查询语言，面向对象的查询语言 |
| Positional Parameter | n. | 位置参数，`?1`（从 1 开始） |
| Named Parameter | n. | 命名参数，`:name` 配 `@Param` |
| composed annotation | n. | 组合注解，封装其他注解（如 `@NativeQuery`） |
| sqlResultSetMapping | n. | SQL 结果集映射，把原生查询列映射成对象 |
| count query | n. | 计数查询，分页时算总数用 |
| Pageable | n. | 分页请求抽象（页码/大小/排序） |
| Slice | n. | 分页结果，只判断是否有下一页，不查总数 |
| Sort / JpaSort.unsafe | n. | 排序对象；unsafe 放行含函数的排序（防注入） |
| state field path expression | n. | 状态字段路径表达式，Sort 字段须可解析到属性/别名 |
| @Modifying | n. | 标注修改型 @Query（UPDATE/DELETE） |
| Persistence Context | n. | 持久化上下文，一级缓存 + 实体状态追踪 |
| Dirty Checking | n. | 脏检查，托管实体改动自动同步 |
| flush / commit | v. | flush 发 SQL 到库（可回滚）；commit 事务落地 |
| Managed / Detached / Transient | n. | 实体三态：托管/游离/瞬时 |
| clearAutomatically / flushAutomatically | n. | @Modifying 属性：执行后清缓存 / 执行前先 flush |
| lifecycle callback | n. | 生命周期回调（@PrePersist/@PreRemove 等） |
| Auditing | n. | 审计，自动填充创建/修改时间（@CreatedDate 等） |
| Keyset scrolling | n. | 键集滚动，避免深分页性能问题 |
| ScrollPosition / Window | n. | 滚动游标位置 / 滚动结果窗口 |
