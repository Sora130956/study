# Hibernate 核心运行机制

> **来源:** 答疑整理（W2 定义 User 实体时延伸，补 Hibernate 关键理论）
> **对应计划周次:** 第 2 周 · 周五定义 User 实体（延伸）

## 核心理解

Spring Data JPA 底层默认用 Hibernate 实现。理解 Hibernate 的运行机制（Session、持久化上下文、实体三态、脏检查、延迟加载、flush）才能看懂「为什么改对象不调 save 也能更新」「为什么关联查询会变慢」这类现象，避免踩坑。

与 MyBatis 最大的区别：MyBatis 改对象什么都不会发生，必须显式调 update；Hibernate 把查出来的实体「托管」起来，改字段后事务提交时自动比对并发 UPDATE（脏检查）。这是 ORM「用对象思维操作数据库」的核心体现，也是它的能力与陷阱所在。

## 关键点

### 1. EntityManager / Session / 持久化上下文 / 事务的关系

- **EntityManager ≈ Session**：`EntityManager` 是 JPA 规范叫法，`Session` 是 Hibernate 实现，Hibernate Session 实现了 EntityManager 接口，视为同一东西的两个名字。
- **持久化上下文**：被 EntityManager 管理的「实体缓存 + 状态追踪区」。EntityManager 是操作入口（提供 `persist`/`find`/`flush`），持久化上下文是它管理的数据。一个 EntityManager 对应一个持久化上下文。
- **Session 与事务不严格相等**：一个 Session 生命周期 ⊇ 事务生命周期。Spring 标准用法（`@Transactional`）下可近似理解为「一个事务 = 一个持久化上下文」，此心智模型够用。

```
EntityManager（操作入口，= Hibernate Session）
   └── 持久化上下文（缓存 + 状态追踪）
          └── 托管态实体（用 id 作 key 管理）
事务（@Transactional）：通常与上面同生命周期
```

### 2. 实体三态（结合"价值"理解）

```
Transient ──persist──> Managed ──事务结束/Session关──> Detached
 new出来未存            进了Session                     脱离管理
            <──merge/save── （重新托管）
```

- **Transient（瞬时）**：`new User()` 未 `persist` 的状态，与数据库无关，Session 不认识它。只在「新建还没存」时存在。
- **Managed（托管）**：`findById`/`findAll`/`save` 后、仍在事务内的实体。改字段自动同步到库（脏检查）。
- **Detached（游离）**：曾被管理、有 id、但 Session 关了脱离管理。**价值：事务结束后对象仍可读其数据**——Web 应用把查出的对象转 JSON 返回前端，用的就是游离态。改它不再自动同步。

游离态可用 `merge()`/`save()` 重新托管，但**不推荐**「游离改完再 merge」的跨事务模式（易并发覆盖）。推荐「同一事务内查→改→自动同步」。

### 3. 脏检查（Dirty Checking）

Session 记住每个托管实体「查出时的快照」，flush 时拿当前值与快照比对，有变化自动生成 UPDATE。**这是不调 save 也能更新的原理**。

```java
@Transactional
void rename(Long id) {
    User u = repo.findById(id).get(); // 托管 + 记快照
    u.setUsername("new");             // 改内存对象，未调 save
}   // 提交时脏检查发现变化 → 自动 UPDATE
```

### 4. flush 时机与 SQL 重排（重点）

**flush 时机**（默认 `FlushMode.AUTO`）：① 事务提交前 ② 执行查询前（怕查到旧数据先 flush）③ 手动 `em.flush()`。

**flush 不按代码操作顺序发 SQL**，而是按固定的 action ordering（write-behind）：

```
1. INSERT  2. UPDATE  3. 删集合元素  4. 插集合元素  5. DELETE
```

目的：同类 SQL 攒一起批量发（性能）+ 减少外键约束冲突。

**可能的坑**：唯一约束下，UPDATE 被统一重排到 INSERT 之后，可能瞬间撞唯一约束报错。需严格顺序时手动 `flush()` 打断重排。简单 CRUD 不会遇到，复杂事务（如线索转客户）再留意。

### 5. 延迟加载（Lazy Loading）+ N+1 问题（实战高频坑）

关联对象默认懒加载：Hibernate 给 `user.organization` 塞一个**代理对象（proxy）**，只揣着外键值（`organization_id` 本就在 user 表里），真正调 `getOrganization().getName()` 时才查 organization 表。没用到就不发多余 SQL。

**N+1 问题**：

```java
List<User> users = repo.findAll();    // 1 条 SQL 查 N 个 user
for (User u : users) {
    u.getOrganization().getName();    // 每个 user 各 1 条 → N 条
}                                     // 共 1 + N 条 SQL！
```

数据量大时是性能灾难。解法：`JOIN FETCH`、`@EntityGraph`、批量抓取。**W6 组织树 + 用户归属必踩，届时吃透**（计划已标记）。

### 6. 一级缓存 vs 二级缓存

- **一级缓存** = 持久化上下文，Session 级，默认开启不可关。同 Session 查同一 id 不重复打库。
- **二级缓存** = SessionFactory 级，跨 Session 共享，需额外配置（如 EhCache）。Mini CRM 用不到，知道有即可。

### 7. 主键生成策略与批量插入（IDENTITY 的代价）

`@GeneratedValue(strategy = GenerationType.IDENTITY)` 用 MySQL `AUTO_INCREMENT` 生成主键。

| 策略 | 生成方式 | 适用 |
|------|------|------|
| `IDENTITY` | 数据库自增列 | **MySQL 首选** |
| `SEQUENCE` | 数据库序列对象 | PostgreSQL / Oracle |
| `AUTO` | 按方言自动挑 | 交给框架决定 |
| `TABLE` | 单独建表模拟序列 | 兼容好但慢，基本不用 |

**IDENTITY 无法 JDBC 批量插入**：持久化上下文用 id 作 key 管理实体，实体进缓存前必须有 id；而 IDENTITY 的 id 要 INSERT 执行后数据库才生成，于是 Hibernate 被迫「插一条问一条」，无法攒批。SEQUENCE 能「提前批量取号→配好 id→批量 INSERT」，故不受限。Mini CRM 业务量无影响，YAGNI。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| SessionFactory / EntityManagerFactory | n. | 重量级线程安全工厂，应用级单例 |
| Session / EntityManager | n. | 轻量非线程安全会话，每事务一个 |
| Persistence Context | n. | 持久化上下文，缓存 + 状态追踪 |
| Transient / Managed / Detached | n. | 实体三态：瞬时 / 托管 / 游离 |
| Dirty Checking | n. | 脏检查，托管实体改动自动同步 |
| merge | v. | 把游离态实体合并回上下文重新托管 |
| flush | v. | 把内存变更生成 SQL 发往数据库 |
| action ordering / write-behind | n. | flush 按固定类型顺序重排 SQL |
| Lazy Loading | n. | 延迟加载，关联对象用到时才查 |
| proxy | n. | 代理对象，承载延迟加载的占位 |
| N+1 problem | n. | 关联遍历引发 1+N 条查询的性能坑 |
| First-Level / Second-Level Cache | n. | 一级（Session）/ 二级（Factory）缓存 |
| GenerationType.IDENTITY | n. | 主键交数据库自增列生成 |