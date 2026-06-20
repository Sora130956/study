# @Transactional 声明式事务

> **来源:** 答疑整理（W2 延伸，Service 层事务边界）
> **对应计划周次:** 第 2 周延伸（事务为独立主题，后续周次深化）

## 核心理解

`@Transactional` 是 Spring 声明式事务的核心注解，加在方法/类上，让方法内所有数据库操作「要么全成功，要么全回滚」，无需手写 `begin/commit/rollback`。它靠 Spring AOP 代理实现：代理拦截方法 → 开启事务 → 执行方法体 → 正常提交 / 异常回滚。

惯例加在 **Service 层**方法上——Service 才是「一个完整业务操作」的边界。一个 Service 方法 = 一个事务 = 一个持久化上下文，方法结束提交时触发 flush + 脏检查。CRM 典型场景：线索转客户（新建 customer + 更新 lead 状态 + 写转换记录，必须捆成一个事务）。

## 关键点

### 1. 回滚触发的真正条件（重点：吞掉异常 = 不回滚）

回滚不是「异常 throw 到代理那层」就回滚，而是：**异常从方法抛出、穿过代理层时，代理捕获并判断是否该回滚的类型**。

> ⚠️ **常见坑：如果方法内 `try-catch` 把异常吞了、没往外抛，代理感知不到，不会回滚。** 关键是「异常有没有真的抛出方法外」。

```java
@Transactional
public void doSomething() {
    repo.save(x);
    try {
        riskyOp();
    } catch (Exception e) {
        log.error("出错", e);   // ❌ 吞掉异常 → 代理感知不到 → x 照样提交，不回滚
    }
}
```

需要回滚又想 catch 时，处理后要重新抛出，或手动标记回滚 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。

### 2. 默认只回滚未检查异常（受检异常不回滚）

默认仅对 `RuntimeException` 及子类、`Error` 回滚；**受检异常（checked exception）默认不回滚**。

```java
@Transactional
public void doIt() throws IOException {
    repo.save(x);
    throw new IOException();   // ❌ 受检异常，默认不回滚，x 被提交
}
```

让所有异常都回滚需显式指定（实务常用）：

```java
@Transactional(rollbackFor = Exception.class)
```

### 3. AOP 代理机制带来的两个坑

**坑 1：同类内部调用失效**

```java
public void a() {
    b();   // ❌ this.b() 绕过代理，@Transactional 不生效
}
@Transactional
public void b() { ... }
```

`this` 调用不经过代理，注解形同虚设。解法：拆到另一个 Bean，或注入自身代理。`REQUIRES_NEW` 等也受此坑影响——必须跨 Bean 调用才有效。

**坑 2：只对 public 方法生效**，private/protected 上无效。

### 4. 传播行为（Propagation）

解决「事务方法 A 调用事务方法 B 时，B 用 A 的事务还是自己开」。7 种按重要性分档：

**第一档（必须掌握）**

`REQUIRED`（默认）——加入或新建：
```
当前有事务 → 加入它（共用）；当前没事务 → 新建
```
A 和 B **同生共死**：任一抛异常整个事务回滚。绝大多数业务用它。

`REQUIRES_NEW`——永远新建独立事务：
```
当前有事务 → 挂起它，开全新独立事务；没事务 → 新建
```
B **独立于** A。典型场景：记日志/审计——不管主流程成败，日志都要留下，B 提交后 A 回滚也带不走它。有挂起+新连接开销，别滥用。

**第二档（知道含义，用到再查）**

`NESTED`——嵌套事务（savepoint 保存点）：
```
当前有事务 → 内部建保存点，B 失败只回滚到保存点；没事务 → 等同 REQUIRED
```

REQUIRES_NEW vs NESTED 关键区别：

| | REQUIRES_NEW | NESTED |
|---|------|------|
| 是否独立 | 完全独立新事务 | 外层事务的子部分（保存点） |
| B 回滚 | 不影响外层 | 只回滚到保存点，外层可继续 |
| **外层回滚** | **不影响 B（已独立提交）** | **B 跟着一起回滚** |

NESTED 是「外层能甩掉 B，但 B 甩不掉外层」的单向关系。场景：批处理单条失败只回滚那条。依赖数据库 savepoint。

**第三档（了解即可）**

| 值 | 含义 |
|------|------|
| `SUPPORTS` | 有事务就加入，没有就非事务运行 |
| `NOT_SUPPORTED` | 总非事务运行，有事务就挂起 |
| `MANDATORY` | 必须已存在事务，否则抛异常 |
| `NEVER` | 必须没有事务，有就抛异常 |

**传播行为决策表：**

| 我想要 | 用 |
|------|------|
| 普通业务，几步捆成一个事务 | `REQUIRED`（默认） |
| 不管主流程成败都要独立保存（日志/审计） | `REQUIRES_NEW` |
| 批处理中单条失败不连累其他 | `NESTED` |
| 其余 | 用到再查 |

### 5. 其他常用属性

```java
@Transactional(readOnly = true)   // 查询方法建议加，跳过脏检查，性能优化
@Transactional(timeout = 5)       // 超时秒数，超时回滚
@Transactional(isolation = ...)   // 隔离级别，默认跟数据库（MySQL 默认 RR），一般不动
```

### 6. 与 Hibernate 知识的串联

- `@Transactional` 方法 = 一个持久化上下文的生命周期，方法结束（提交）触发 flush + 脏检查。
- `@Modifying` 批量更新必须在事务里，配合 `@Transactional`。
- 方法内查出的实体是托管态，方法结束变游离态。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| declarative transaction | n. | 声明式事务，注解驱动无需手写事务代码 |
| rollback | v. | 回滚，撤销事务内所有未提交改动 |
| checked / unchecked exception | n. | 受检 / 非受检异常（默认只回滚后者） |
| rollbackFor | n. | 指定额外触发回滚的异常类型 |
| propagation | n. | 传播行为，控制嵌套调用的事务边界 |
| REQUIRED / REQUIRES_NEW / NESTED | n. | 加入或新建 / 独立新建 / 嵌套保存点 |
| savepoint | n. | 保存点，NESTED 回滚的目标位置 |
| readOnly | n. | 只读事务，跳过脏检查优化查询 |
| self-invocation | n. | 同类内部调用，绕过代理致注解失效 |
| setRollbackOnly | v. | 手动标记事务回滚 |
