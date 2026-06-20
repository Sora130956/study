# Flyway Database Migration

> **来源:** https://documentation.red-gate.com/fd/quickstart-how-flyway-works （原链接已失效，内容据 Flyway 官方机制整理）
> **对应计划周次:** 第 2 周 · 读 Flyway 机制 + 写第一个迁移脚本

## 核心理解

Flyway 是数据库结构的「版本控制」工具——代码用 Git 管版本，数据库表结构变更用 Flyway 管版本。把每次结构变更写成一个 `.sql` 脚本放进项目，应用启动时 Flyway 自动按顺序执行没跑过的脚本，保证任何环境（本机、新同事、生产）的数据库结构完全一致、可追溯、可重放。

它解决的痛点是：手动执行建表/改表 SQL 极易遗漏、顺序错乱、环境不一致。在 Mini CRM 第 2 周，我们**不靠 JPA 的 `ddl-auto` 建表**，而是把建表权交给 Flyway（`ddl-auto: validate`，JPA 只校验实体与表是否对得上），由 `V1__init_user.sql` 在启动时自动建出 `user` 表。

## 关键点

### 1. 脚本命名规则

脚本固定放在 `src/main/resources/db/migration/`，文件名严格遵守：

```
V{版本号}__{描述}.sql
```

| 部分 | 说明 | 例子 |
|------|------|------|
| `V` | 前缀，大写 V（Versioned 版本迁移） | `V` |
| `{版本号}` | 决定执行顺序 | `1`、`2`、`1.1`、`20260619` |
| `__` | **两个下划线**（关键，不是一个） | `__` |
| `{描述}` | 描述，单词间用下划线 | `init_user` |
| `.sql` | 后缀 | `.sql` |

合法示例：

```
V1__init_user.sql
V2__add_customer_table.sql
V3__add_index_on_username.sql
V1.1__alter_user_add_email.sql
```

易错点：
- **必须两个下划线 `__`**，一个会报错。
- 版本号决定**执行顺序**（1→2→3），不是按文件名字母序。
- 执行过且被记录的脚本**内容不能再改**（见校验机制），改表要新建下一个版本。

### 2. flyway_schema_history「账本」表

Flyway 首次运行会在库里自动建一张 `flyway_schema_history`，每跑一个脚本记一行：版本号、描述、**checksum（内容指纹）**、成功与否、执行时间。

| version | description | checksum | success | installed_on |
|---------|-------------|----------|---------|------|
| 1 | init user | -1234567 | 1 | 2026-06-19 ... |

### 3. 启动时执行流程

应用启动 → 扫描 `db/migration/` 所有脚本 → 与账本比对：

```
对每个脚本：
  ├─ 账本里没有这个版本 → 执行它，并记一行到账本
  └─ 账本里已有这个版本 → 跳过（已跑过）
```

这就是「**重启应用不重复执行迁移**」的原理：第二次启动 V1 已在账本里，直接跳过，只执行新增的 V2、V3……

### 4. checksum 校验（防篡改）

启动时 Flyway 重新计算已执行脚本的 checksum 与账本比对：
- 一致 → 正常跳过。
- **不一致**（偷偷改了已执行的 V1）→ 启动直接报错 `Migration checksum mismatch`。

核心约束：**已执行的脚本是「历史」，不可修改**。要改表就新建下一个版本脚本，保证所有环境变更历史一致。

### 5. 第 2 周接入步骤

```sql
-- src/main/resources/db/migration/V1__init_user.sql
CREATE TABLE `users` (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    username    VARCHAR(50)  NOT NULL,
    password    VARCHAR(100) NOT NULL,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

> **表名约定（影响所有实体）：** 表名统一用**复数英文**（`users`/`customers`/`leads`/`organizations`）。原因：① `user` 是 MySQL 保留字，裸写 SQL 要加反引号易踩坑，复数 `users` 避开；② 复数是英文技术圈主流约定，更适合海外项目。JPA 实体类名用单数 + `@Table(name="users")` 显式声明。

> **YAGNI：** V1 只建登录注册必需字段。`tenant_id`/`user_tenant` 关联表留给 W4-5（V2），`org_id` 留给 W6，角色权限留给 W7-8。每个里程碑新建一个版本脚本，不在 V1 预埋空字段。

1. 加依赖（Boot 4 默认不自动引）：`flyway-core`、`flyway-mysql`。
2. 关 JPA 自动建表：`spring.jpa.hibernate.ddl-auto: validate`（建表交 Flyway，JPA 只校验）。
3. 写 `V1__init_user.sql`。
4. 启动看日志：

```
Successfully validated 1 migration
Creating Schema History table `minicrm`.`flyway_schema_history`
Migrating schema `minicrm` to version "1 - init user"
Successfully applied 1 migration
```

5. 重启验证：第二次显示 `Schema is up to date. No migration necessary`，不重复执行。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| migration | n. | 迁移，一次数据库结构变更（一个脚本） |
| Versioned Migration | n. | 版本迁移，`V` 前缀脚本，按版本号顺序执行一次 |
| flyway_schema_history | n. | 迁移历史表（账本），记录已执行脚本 |
| checksum | n. | 校验和/内容指纹，防止已执行脚本被篡改 |
| ddl-auto: validate | n. | JPA 只校验实体与表结构是否匹配，不建表 |
| flyway-core / flyway-mysql | n. | Flyway 核心依赖 / MySQL 适配依赖 |