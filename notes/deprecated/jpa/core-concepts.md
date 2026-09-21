# Core concepts（Spring Data Repository 核心概念）

> **来源:** https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html
> **对应计划周次:** 第 2 周 · 周一读 JPA Core Concepts

## 核心理解

Spring Data 的核心是一套 **Repository 抽象**：你只需声明一个**接口**（不写实现），Spring Data 在运行时为它生成代理实现，帮你完成数据访问。这套抽象的中心接口是 `Repository<T, ID>`，两个泛型参数分别是"要管理的领域类型（实体）"和"该实体的主键类型"。`Repository` 本身几乎是个**标记接口（marker interface）**，不带方法，只用来捕获 `T` 和 `ID` 两个类型，并方便你发现它的子接口。

往上一层，`CrudRepository`（以及返回 `List` 的 `ListCrudRepository`）提供了开箱即用的增删改查方法；`PagingAndSortingRepository` 在此基础上加了分页与排序。再具体到某种存储技术，还有 `JpaRepository`、`MongoRepository` 这类**技术专属接口**，它们继承自 `CrudRepository`，额外暴露该技术的能力。这条继承链就是后面 Mini CRM 里 `UserRepository extends JpaRepository<User, Long>` 能直接拥有大量现成方法的原因。

文档还点出一个 DDD 视角：Spring Data 把领域类型视为**实体 / 聚合（aggregate）**，实体一定有标识符（identifier），否则就只是无身份的值对象。这正是 `Repository<T, ID>` 里要单独传一个 `ID` 类型、以及 `findById` 这类"保留方法"存在的根本原因——数据访问模式总要靠标识符来定位对象。在 Mini CRM 中，本周的 `User` 就是第一个这样的实体（`ID` 为 `Long`），它也是后续 IAM、多租户、四大业务实体复用同一套 Repository 模式的起点。

## 关键点

### Repository 是标记接口，只负责捕获类型

> The central interface in the Spring Data repository abstraction is `Repository`. It takes the domain class to manage as well as the identifier type of the domain class as type arguments. This interface acts primarily as a marker interface to capture the types to work with and to help you to discover interfaces that extend this one.

`Repository<T, ID>` 自身不声明任何方法，作用是"记住"你要操作的实体类型 `T` 和主键类型 `ID`。真正能用的方法来自它的子接口。Mini CRM 不会直接继承它，而是继承下游的 `JpaRepository`。

### CrudRepository 提供标准 CRUD 方法

> The `CrudRepository` and `ListCrudRepository` interfaces provide sophisticated CRUD functionality for the entity class that is being managed.

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S entity);      // 1 保存（新增或更新）
    Optional<T> findById(ID primaryKey); // 2 按主键查，返回 Optional
    Iterable<T> findAll();               // 3 查全部
    long count();                        // 4 计数
    void delete(T entity);               // 5 删除
    boolean existsById(ID primaryKey);   // 6 是否存在
}
```

这些就是常说的 CRUD 方法。`ListCrudRepository` 方法一样，只是把 `Iterable` 换成更好用的 `List`。

> 对应 Mini CRM 周六：`UserService` 的增删改查会直接调用这些方法，比如 `save` 既做新增也做更新，`findById` 配合 `Optional` 处理"查不到"的情况。

### findById 是"保留方法"，固定指向主键

> The repository interface implies a few reserved methods like `findById(ID identifier)` that target the domain type identifier property regardless of its property name.

`findById` 等保留方法**永远指向实体的主键属性**，不管这个属性叫 `id` 还是别的名字。只有当主键属性名不是 `Id`、又想沿用 `Id` 语义时才需要 `@Query` 自定义——但文档明确不推荐，因为一旦 `ID` 泛型类型和实体里 `Id` 属性类型不一致，很快就会撞上类型限制。

### JpaRepository 是技术专属接口

> We also provide persistence technology-specific abstractions, such as `JpaRepository` or `MongoRepository`. Those interfaces extend `CrudRepository` and expose the capabilities of the underlying persistence technology...

`JpaRepository` 继承 `CrudRepository`，在通用 CRUD 之上额外暴露 JPA 特有能力（如 `flush`、批量删除、`getReferenceById` 等）。

```java
// Mini CRM 本周实际写法
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username); // 派生查询，留待周二细看
}
```

### PagingAndSortingRepository 加分页与排序

> In addition to `CrudRepository`, there are `PagingAndSortingRepository` and `ListPagingAndSortingRepository` which add additional methods to ease paginated access to entities.

```java
interface PagingAndSortingRepository<T, ID> extends Repository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}

// 取第 2 页、每页 20 条（页码从 0 开始，所以 1 是第二页）
Page<User> users = repository.findAll(PageRequest.of(1, 20));
```

> Mini CRM 的用户/客户列表接口后续做分页时会用到，本周 CRUD 可先不引入。

### 派生 count / delete 查询

> In addition to query methods, query derivation for both count and delete queries is available.

不止"查询"能按方法名派生，**计数和删除**也能：

```java
interface UserRepository extends CrudRepository<User, Long> {
    long countByLastname(String lastname);   // 派生计数
    long deleteByLastname(String lastname);  // 派生删除，返回受影响行数
    List<User> removeByLastname(String lastname); // 删除并返回被删实体
}
```

### 实体注解：必须 vs 可选

定义 JPA 实体时，注解分两类——必须的和按需的：

| 注解 | 是否必须 | 作用 / 何时需要 |
|------|---------|----------------|
| `@Entity` | **必须** | 声明"我是 JPA 实体"，不写则不被 JPA 管理 |
| `@Id` | **必须** | 标记主键；实体必有标识符（呼应"实体必有 ID"） |
| `@Table(name=...)` | 可选 | 仅当类名与表名不一致时需要；不写则按类名推导（`User`→`user`，受命名策略影响）。推荐显式写更清晰 |
| `@GeneratedValue` | 可选 | 仅当主键自动生成时需要；手动赋值（如自定义编号/UUID）则不写。MySQL 自增用 `strategy = IDENTITY` |
| `@Column(...)` | 可选 | 仅当列名不一致或要加约束（`nullable`、`length`）时需要；不写则按字段名映射 |

最小可用实体（主键手动赋值）：

```java
@Entity
public class User {
    @Id
    private Long id;
    private String username;
}
```

> Mini CRM 的 `User` 会写全 `@Entity + @Table + @Id + @GeneratedValue`——因为表名要显式、主键要自增。但这是"我们的选择"，不是 JPA 的强制要求。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| repository abstraction | n. | 仓库抽象：声明接口即可获得数据访问实现的机制 |
| marker interface | n. | 标记接口：不含方法、仅用于捕获类型信息的接口 |
| domain type / entity / aggregate | n. | 领域类型 / 实体 / 聚合，文档中可互换使用 |
| identifier | n. | 标识符（主键），实体区别于值对象的关键 |
| value object | n. | 值对象：没有标识符、只看属性值的对象（DDD 术语） |
| CRUD methods | n. | 增删改查方法（Create/Read/Update/Delete） |
| reserved methods | n. | 保留方法，如 `findById`，固定指向主键属性 |
| persistence technology-agnostic | adj. | 与持久化技术无关的（通用接口） |
| query derivation | n. | 查询派生：由方法名推导出查询逻辑 |
| paging and sorting | n. | 分页与排序 |

---

## 成体系总结

### 一、Repository 接口继承体系（一张图记住）

```
Repository<T, ID>                      标记接口，只捕获 实体类型T + 主键类型ID，无方法
   └── CrudRepository<T, ID>           标准 CRUD：save / findById / findAll / count / delete / existsById（返回 Iterable）
         │     └── ListCrudRepository  同上，但返回 List（更好用）
         └── PagingAndSortingRepository 加 findAll(Sort) / findAll(Pageable)，支持分页排序
               └── ListPagingAndSortingRepository  同上，返回 List
   └── JpaRepository<T, ID>            技术专属，继承 CrudRepository + 分页排序，额外暴露 JPA 能力
```

选型口诀：
- **只要 CRUD** → 继承 `CrudRepository` / `ListCrudRepository`。
- **要分页排序** → 继承 `PagingAndSortingRepository` 一族。
- **用 JPA 且想要全套** → 直接继承 `JpaRepository`（Mini CRM 默认选它）。

### 二、三条最该记住的设计思想

1. **接口即契约，实现由框架代理生成。** 你写接口，Spring Data 运行时给实现。这是整套抽象的根。
2. **实体必有标识符（ID）。** 这是 `Repository<T, ID>` 要两个泛型、`findById` 作为保留方法存在的根本原因——数据访问总要靠主键定位对象。
3. **方法名能派生逻辑，不止于查询。** 查询、计数（`countBy...`）、删除（`deleteBy...` / `removeBy...`）都能按方法名派生，省去手写 SQL。（命名规则细节是周二的 Query Methods 任务。）

### 三、落到 Mini CRM 第 2 周

```java
@Entity
@Table(name = "user")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;          // 主键 → 对应 Repository<User, Long> 的 ID
    private String username;
    // ...
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

- `JpaRepository<User, Long>` 一行，就免费拿到 `save / findById / findAll / count / delete / existsById` 等全套方法 → 直接支撑周六 `UserService` 的 CRUD。
- `findById` 返回 `Optional<User>`，天然提示"可能查不到"，配合统一 `ApiResponse` 处理 404 场景。
- `findByUsername` 是派生查询的预热，为 W3 登录（按用户名查用户）打基础。

### 四、与后续主线的衔接

`User` 是第一个跑通"实体 → Repository → CRUD"链路的实体。这条链路一旦定型，后面四大业务实体（Lead / Customer / Opportunity / Activity）只是把 `User` 换成各自类型、复用同一套继承与方法派生模式。本节是整个数据访问层的"地基认知"。
