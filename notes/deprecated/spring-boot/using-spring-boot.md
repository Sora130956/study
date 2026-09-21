# Spring Boot Getting Started（Using Spring Boot）

> **来源:** https://docs.spring.io/spring-boot/reference/using/index.html
>
> 子页面：
> - Build Systems: https://docs.spring.io/spring-boot/reference/using/build-systems.html
> - Structuring Your Code: https://docs.spring.io/spring-boot/reference/using/structuring-your-code.html
> - Configuration Classes: https://docs.spring.io/spring-boot/reference/using/configuration-classes.html
> - Auto-configuration: https://docs.spring.io/spring-boot/reference/using/auto-configuration.html
> - Spring Beans and DI: https://docs.spring.io/spring-boot/reference/using/spring-beans-and-dependency-injection.html
> - @SpringBootApplication: https://docs.spring.io/spring-boot/reference/using/using-the-springbootapplication-annotation.html
> - Running Your Application: https://docs.spring.io/spring-boot/reference/using/running-your-application.html
> - DevTools: https://docs.spring.io/spring-boot/reference/using/devtools.html
> - Packaging for Production: https://docs.spring.io/spring-boot/reference/using/packaging-for-production.html
>
> **对应计划周次:** 第 1 周 · Spring Boot 项目骨架

## 核心理解

Spring Boot 的 "Using" 章节回答了一个最实际的问题：**拿到 Spring Boot 之后，怎么把一个项目正确地搭起来、跑起来、改起来。** 它不教 Spring 框架本身，而是讲项目骨架的"约定"——build 工具用哪些 starter、代码包怎么摆、主类怎么写、应用怎么启动、开发期怎么提效、上生产前还要做什么。

这一章的核心理念可以总结为四个关键词：**Starter（依赖打包）、Auto-configuration（按依赖自动装配）、Component Scan（按包扫描 Bean）、Convention over Configuration（约定优于配置）**。理解这四个词，就理解了 Spring Boot 为什么"零配置就能跑起来"。

在 Mini CRM 里，本章对应 W1 周末搭骨架的全部动作：用 Initializer 拉 starter 依赖、确定包结构、写带 `@SpringBootApplication` 的主类、`mvn spring-boot:run` 跑起来、开发期可选加 devtools 提速。

## 关键点

### 1. Starter：依赖的"成品包"

Starter 是 Spring Boot 把"做某件事所需的所有 jar"打成的依赖集合，让你**一行依赖就完成一类能力的引入**。

| Starter | 作用 |
|---------|------|
| `spring-boot-starter-web` | 构建 Web/RESTful 应用，**默认内嵌 Tomcat** |
| `spring-boot-starter-data-jpa` | JPA + Hibernate + 数据源 |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5、Mockito、AssertJ |
| `spring-boot-starter-validation` | Bean Validation |

> **原文：** "Starter for building web, including RESTful, applications using Spring MVC. Uses Tomcat as the default embedded container."

**Mini CRM W1 的依赖清单（用 Initializer 勾选）：**
- Web、JPA、MySQL Driver、Security、Validation（可选 Lombok）

> **IDE 提示：** 文档说在 Eclipse / Spring Tools 里 `ctrl-space` 可以补全 starter，**IDEA 里不行**——直接用 [start.spring.io](https://start.spring.io) 生成更省事。

### 2. 包结构：主类放根包，其他子包在它下面

> **原文：** "We generally recommend that you locate your main application class in a root package above other classes. The @SpringBootApplication annotation... implicitly defines a base 'search package' for certain items."

**原因：** `@SpringBootApplication` 隐含 `@ComponentScan`，默认扫描"主类所在包及其子包"。主类放根包，所有组件都会被扫到。如果主类放在子包里，**外层包的 `@Component` / `@Entity` 会被漏扫**。

**两种推荐布局：**

**A. 按业务模块分包（Mini CRM 采用）**
```
com.minicrm/
├── MiniCrmApplication.java    ← 主类放根包
├── common/
├── security/
├── tenant/
├── iam/
├── lead/          ← 每个模块自带 controller/service/repository/domain
│   ├── LeadController.java
│   ├── LeadService.java
│   ├── LeadRepository.java
│   └── Lead.java
├── customer/
├── opportunity/
└── activity/
```

**B. 按技术分层分包（传统）**
```
com.minicrm/
├── controller/
├── service/
├── repository/
└── domain/
```

**结论：** 项目变大后**按业务模块分包**更易维护，是 Spring 官方指南和社区推荐方式。Mini CRM 选 A。

### 3. `@SpringBootApplication` = 三个注解的合体

> **原文：** "A single @SpringBootApplication annotation can be used to enable those three features:
> - `@EnableAutoConfiguration`
> - `@ComponentScan`
> - `@SpringBootConfiguration`"

| 子注解 | 作用 |
|--------|------|
| `@EnableAutoConfiguration` | 启用**自动配置**（按 classpath 上的依赖自动装配 Bean） |
| `@ComponentScan` | 扫描当前包及子包下的 `@Component` / `@Service` / `@Repository` / `@Controller` |
| `@SpringBootConfiguration` | 等同于 `@Configuration`，**但被 `@SpringBootTest` 特殊识别为"主配置入口"** |

**`@SpringBootConfiguration` 为什么不直接用 `@Configuration`？**
- 在测试场景，项目里可能有多个 `@Configuration`（生产配置、测试配置、临时配置）
- `@SpringBootTest` 启动时需要明确**哪个是应用的主配置入口**，于是约定：找带 `@SpringBootConfiguration` 的类
- 实际开发中你**不用手动写 `@SpringBootConfiguration`**，因为 `@SpringBootApplication` 已经包含它

**Mini CRM 主类：**
```java
@SpringBootApplication
public class MiniCrmApplication {
    public static void main(String[] args) {
        SpringApplication.run(MiniCrmApplication.class, args);
    }
}
```

### 4. 自动配置（Auto-configuration）：按 classpath 推断该装配什么

> **原文：** "Spring Boot auto-configuration attempts to automatically configure your Spring application based on the jar dependencies that you have added."

**工作机制：**
- 加了 `spring-boot-starter-data-jpa` → 自动配置 `DataSource`、`EntityManagerFactory`、`PlatformTransactionManager`
- 加了 `spring-boot-starter-web` → 自动配置 `DispatcherServlet`、内嵌 Tomcat、Jackson
- 没加对应依赖 → 不会装配，节省启动开销

**两条核心规则：**

1. **必须 opt-in**：通过 `@EnableAutoConfiguration` 或 `@SpringBootApplication` 激活
2. **只能加一个**：整个项目只在主类上加一个 `@SpringBootApplication` 或 `@EnableAutoConfiguration`

> **原文：** "Auto-configuration is non-invasive. At any point, you can start to define your own configuration to replace specific parts of the auto-configuration. For example, if you add your own DataSource bean, the default embedded database support backs away."

**关键特性：自动配置是"非侵入式"的**——你随时可以用自己的 Bean 替换默认配置。例如你声明了自己的 `DataSource` Bean，默认的内嵌 H2 就会自动让位。**这是 Mini CRM W2 接 MySQL 时直接覆盖默认数据源的依据**。

### 5. 配置类（@Configuration）建议

> **原文：** "Spring Boot favors Java-based configuration... we generally recommend that your primary source be a single @Configuration class. Usually the class that defines the main method is a good candidate as the primary @Configuration."

**三条建议：**
1. **首选 Java 配置**，不推荐 XML
2. **主类天然是主 `@Configuration`**（被 `@SpringBootApplication` 包含）
3. **不必把所有配置塞进一个类**：
   - 用 `@Import` 引入额外的配置类
   - 用 `@ComponentScan` 自动扫描（默认就开了）
   - 不得已要用 XML 时，用 `@ImportResource` 加载

### 6. Bean 与依赖注入：优先构造器注入

> **原文：** "We generally recommend using constructor injection to wire up dependencies and @ComponentScan to find beans."

**三种注入方式对比：**

| 方式 | 推荐度 | 原因 |
|------|--------|------|
| 构造器注入 | ✅ **首选** | 依赖显式、字段可声明 `final`、便于单测、循环依赖编译期暴露 |
| Setter 注入 | ⚠️ 可选 | 用于可变的可选依赖 |
| 字段注入（`@Autowired` 在字段上） | ❌ 不推荐 | 难单测、隐藏依赖、`final` 不可用 |

**单构造器无需 `@Autowired`**（Spring 4.3+），多构造器才需要：

```java
// ✅ 推荐：单构造器，不写 @Autowired
@Service
public class LeadService {
    private final LeadRepository leadRepository;
    private final ActivityService activityService;

    public LeadService(LeadRepository leadRepository, ActivityService activityService) {
        this.leadRepository = leadRepository;
        this.activityService = activityService;
    }
}

// 多构造器时显式标注
@Service
public class CustomerService {
    @Autowired
    public CustomerService(CustomerRepository repo) { ... }

    public CustomerService(CustomerRepository repo, CacheManager cache) { ... }
}
```

> Lombok `@RequiredArgsConstructor` 也是一行搞定构造器注入的常见写法。

### 7. 运行应用

> **原文：** "The Spring Boot Maven plugin includes a run goal that can be used to quickly compile and run your application."

**三种启动方式：**

```bash
# 1) Maven 插件（开发常用）
mvn spring-boot:run

# 2) IDE 直接运行 main 方法（断点调试方便）

# 3) 打包后 java -jar（生产部署）
mvn clean package
java -jar target/mini-crm-0.0.1-SNAPSHOT.jar
```

> **原文：** "you can run your application as you would any other. The same applies to debugging Spring Boot applications. You do not need any special IDE plugins or extensions."

**为什么这么方便？** 因为 `spring-boot-starter-web` 内嵌了 Tomcat，应用**自包含一个 HTTP 服务器**，不需要部署到外部 servlet 容器。这就是 Spring Boot 区别于传统 Spring 的最大体验提升。

### 8. DevTools：开发期热重启 + LiveReload

> **依赖：** `spring-boot-devtools`（建议 `<optional>true</optional>` 或 `developmentOnly`）

**核心能力：**

#### a. 自动重启（Restart）
- **机制：两个 ClassLoader 分工**
  - **Base ClassLoader**：加载不会变的第三方 jar（启动好就常驻）
  - **Restart ClassLoader**：加载你正在开发的代码
- **重启时**：丢弃旧的 Restart ClassLoader，建一个新的——比"冷启动"快得多
- **触发条件**：classpath 上的文件变动（IDE 编译后自动触发）

#### b. 属性默认值覆盖（Property Defaults）
- 自动把开发期不利的缓存配置设为 `false`，例如 `spring.thymeleaf.cache=false`
- 不用你手动改 `application.properties`，引入 devtools 就生效

#### c. LiveReload
- 内嵌 LiveReload 服务器，资源变更时触发浏览器自动刷新
- 需要在浏览器装 LiveReload 扩展
- 开启：`spring.devtools.livereload.enabled=true`

#### d. 打包后自动禁用
> **原文：** "Developer tools are automatically disabled when running a fully packaged application. If your application is launched from java -jar... then it is considered a 'production application'."

**用 `java -jar` 启动时 devtools 自动关闭**，不会污染生产。

#### e. 为什么标 `<optional>true</optional>`？

> **原文：** "Flagging the dependency as optional in Maven or using the developmentOnly configuration in Gradle prevents devtools from being transitively applied to other modules that use your project."

防止 devtools **依赖传递**：如果你的项目是一个公共库，别人引入你的库时不应该被迫引入 devtools。`<optional>true</optional>` 告诉 Maven"这个依赖不传递给下游"。

**Mini CRM 配置示例（pom.xml）：**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

> Mini CRM W1 周末搭骨架时建议加上，开发反馈快很多。

### 9. 上生产前的打包

> **原文：** "For additional 'production ready' features, such as health, auditing, and metric REST or JMX end-points, consider adding spring-boot-actuator."

- 默认 `mvn package` 会用 Spring Boot Maven 插件打成**可执行 jar**（fat jar / uber jar），所有依赖都在里面
- 上生产推荐加 **spring-boot-actuator**：提供健康检查 `/actuator/health`、指标 `/actuator/metrics`、审计等运维端点
- **Mini CRM W17 容器化** 时再来处理；**W13 上线** 前考虑接入 actuator

## 句子解析

### 原文: "Spring Boot auto-configuration attempts to automatically configure your Spring application based on the jar dependencies that you have added."

- **翻译:** Spring Boot 自动配置会尝试根据你已经添加的 jar 依赖来自动配置你的 Spring 应用。
- **解析:**
  - `attempts to` = 尝试去做（不是"一定能做到"，留有余地，因为可能有缺失依赖或冲突）
  - `based on the jar dependencies` 强调**依据是 classpath**——这是自动配置的核心条件
  - 暗含的设计哲学：**约定优于配置**（Convention over Configuration）

### 原文: "Auto-configuration is non-invasive. At any point, you can start to define your own configuration to replace specific parts of the auto-configuration."

- **翻译:** 自动配置是非侵入式的。任何时候你都可以开始定义自己的配置来替换自动配置的特定部分。
- **解析:**
  - `non-invasive` = 非侵入式：不会强加在你头上，可以被覆盖
  - `at any point` = 任何时间点（开发任何阶段都可介入）
  - 这是 Spring Boot 哲学的"逃生口"：默认值好用，但你想换就换

### 原文: "We generally recommend that you locate your main application class in a root package above other classes."

- **翻译:** 我们通常建议将你的主应用类放在其他类之上的根包中。
- **解析:**
  - `locate ... in a root package` = 放在根包里
  - `above other classes` 这里 above 不是物理位置，而是**包层级上的"上层"**
  - 原因（紧接后面）：`@SpringBootApplication` 隐含 `@ComponentScan`，扫描范围是"主类所在包及子包"

### 原文: "Spring Boot favors Java-based configuration."

- **翻译:** Spring Boot 偏好基于 Java 的配置。
- **解析:**
  - `favors` = 偏爱、倾向于
  - 隐含对比对象：XML 配置（旧 Spring 风格）
  - 含义：用 `@Configuration` + `@Bean` 写配置，比 XML 类型安全、可重构、可调试

### 原文: "Developer tools are automatically disabled when running a fully packaged application."

- **翻译:** 当运行一个完整打包后的应用时，开发者工具会自动禁用。
- **解析:**
  - `fully packaged application` = 已经被打成可执行 jar 的应用
  - `automatically disabled` = 自动禁用——你不需要手动关
  - 设计意图：让 devtools 在开发期生效、在生产期"消失"，零迁移成本

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| starter | n. | 启动器（Spring Boot 的依赖打包） |
| auto-configuration | n. | 自动配置（根据 classpath 装配 Bean） |
| component scan | n. | 组件扫描（自动发现 `@Component` 等注解类） |
| non-invasive | adj. | 非侵入式的 |
| opt-in | v. | 选择加入（需主动启用） |
| primary configuration | n. | 主配置类 |
| embedded container | n. | 内嵌容器（如内嵌 Tomcat） |
| fat jar / uber jar | n. | 包含所有依赖的可执行 jar |
| classpath | n. | 类路径（JVM 查找类的位置集合） |
| transitively applied | adv. | 传递性地（依赖被传递引入） |
| mandatory | adj. | 强制的 / 必须的 |
| convention over configuration | n. | 约定优于配置 |
| constructor injection | n. | 构造器注入 |
| LiveReload | n. | 浏览器自动刷新机制 |
| Actuator | n. | Spring Boot 运维端点模块 |

## 给 Mini CRM W1 的可操作 checklist

- [ ] 用 start.spring.io 生成项目，勾选 Web / JPA / MySQL / Security / Validation
- [ ] 主类 `MiniCrmApplication` 放 `com.minicrm` 根包，加 `@SpringBootApplication`
- [ ] 子包按业务模块分（lead / customer / opportunity / activity / common / security / tenant / iam）
- [ ] Service 用**构造器注入**，字段加 `final`
- [ ] 加 `spring-boot-devtools` 依赖，`<optional>true</optional>`
- [ ] 验证 `mvn spring-boot:run` 启动成功，`/api/ping` 返回 200
- [ ] **暂不引入** Actuator（W13/W17 再说）
