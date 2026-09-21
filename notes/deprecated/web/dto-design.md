# DTO 设计原则（为什么不直接暴露实体）

> **来源:** 答疑整理（W2 周六 User CRUD 前，DTO 必要性讨论）
> **对应计划周次:** 第 2 周 · 周六 User CRUD（DTO 用 record）

## 核心理解

实体（Entity）是给数据库用的，DTO 是给接口用的，两者关注点不同。Controller 边界用 DTO 做「翻译」：进来 Request → 实体，出去 实体 → Response，**实体永远不越过 Service 层暴露给前端**。直接用实体收发请求会带来安全、耦合、灵活性三方面问题。

DTO 不是机械地每个实体凑齐三个，而是**按需建、能合则合**：安全相关的（隐藏敏感字段、防过度提交）是红线不能省；纯增删改查的字段差异，能用一个 Request 覆盖就不拆。配合 `record` + 映射工具（MapStruct）+ AI 生成样板，成本很低。

## 关键点

### 1. 安全红线一：响应过滤敏感字段

`User` 含 `password`（即便是 BCrypt 哈希）。直接返回实体会把密码哈希、`tenant_id`、内部状态全吐给前端。

```java
// ❌ 直接返回实体 → JSON 含 password
// ✅ 用 Response 只挑安全字段
public record UserResponse(Long id, String username, LocalDateTime createdAt) {}
```

### 2. 安全红线二：防过度提交（Mass Assignment）

用实体接收请求，攻击者可在 JSON 里塞 `id`/`role`/`tenantId` 等本不该客户端设置的字段，可能被 Hibernate 吃进去。

```java
// ✅ CreateRequest 只声明允许传的字段，塞别的直接被忽略
public record UserCreateRequest(String username, String password) {}
```

### 3. 解耦：API 契约独立于数据库结构

实体跟数据库走（W4 加 `tenant_id`、W6 加 `org_id`）。若接口直接用实体，每次改表都改变 API 格式，前端/对接方被迫跟改。DTO 这层缓冲让实体内部怎么变都不影响对外契约，反之 API 改字段名也不动数据库。

### 4. 灵活：增删改查字段本就不同

| DTO | 用途 | 字段 |
|------|------|------|
| `UserCreateRequest` | 创建入参 | username, password（不要 id/时间戳） |
| `UserUpdateRequest` | 更新入参 | 可改字段（不含 password，改密走单独接口） |
| `UserResponse` | 返回出参 | id, username, createdAt（无 password） |

一个实体无法同时满足三种形态。

### 5. 校验注解归位

请求 DTO 挂校验注解，实体保持干净（校验是接口层的事）：

```java
public record UserCreateRequest(
    @NotBlank @Size(min = 3, max = 50) String username,
    @NotBlank @Size(min = 6, max = 100) String password
) {}
```

### 6. 是否每个实体都要三个 DTO：务实决策

```
返回给前端？ → 含敏感/内部字段吗？
   ├─ 含 → 必须建 Response（红线）
   └─ 无 → 建议仍建 Response 保契约稳定
接收创建？ → 有客户端不该设的字段吗（基本都有 id/时间戳）？
   └─ 有 → 必须建 CreateRequest（红线）
接收更新？ → 字段/规则和创建不同吗？
   ├─ 不同 → 单独建 UpdateRequest
   └─ 相同 → 与 Create 合并成一个 Request
```

Mini CRM 四实体务实建议：

| 实体 | CreateRequest | UpdateRequest | Response |
|------|------|------|------|
| User | ✅ | ✅（不含 password） | ✅（无 password） |
| Customer | ✅ | 可与 Create 合并 | ✅（过滤 tenantId） |
| Lead | ✅ | 可与 Create 合并 | ✅ |
| Organization | ✅ | 可与 Create 合并 | ✅（树形可能含子节点） |

**结论：Response 和 CreateRequest 基本都要（安全红线），UpdateRequest 能合则合。**

### 7. 为什么用 record

`record`（Java 16+）天生适合 DTO：不可变、自动生成构造器/getter/equals/toString，一行声明无样板。DTO 是一次性数据载体，不可变正合适。

## 分层图

```
前端 ──CreateRequest──> Controller ──> Service ──> 实体 ──> 数据库
前端 <──Response─────── Controller <── Service <── 实体 <── 数据库
       （只暴露安全字段）                （全字段，内部用）
```

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| DTO (Data Transfer Object) | n. | 数据传输对象，接口层数据载体 |
| Entity | n. | 实体，映射数据库表，内部使用 |
| Mass Assignment | n. | 过度提交攻击，客户端塞不该设的字段 |
| Request / Response DTO | n. | 入参 / 出参 DTO |
| record | n. | Java 不可变数据载体，适合 DTO |
| MapStruct | n. | 实体↔DTO 自动映射框架 |
| API contract | n. | API 契约，对外请求/响应格式约定 |