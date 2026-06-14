# 第 1 周周日实践笔记 · 完善配置（profile / 日志 / 时区）

> **类型:** 实践结论（非文档摘录）
> **对应计划周次:** 第 1 周 · 周日完善配置 + 冒烟测试
> **当日产出:** 配置分层（基础 / dev / prod）、日志级别配置、时区配置、冒烟测试（稍后追加）

---

## 1. 配置文件分层

Spring Boot 的多 profile 配置分两层，启动时**先加载基础配置，再用激活的 profile 配置叠加覆盖**。

| 文件 | 角色 | 放什么 |
|------|------|--------|
| `application.yaml` | 基础配置（所有环境共享） | 应用名、激活的 profile、端口、Jackson 时区、通用日志 |
| `application-dev.yaml` | dev 专属 | 本地 datasource、SQL 日志、自己代码的 debug |
| `application-prod.yaml` | prod 专属 | 生产 datasource（环境变量占位）、日志降到 info/warn |

关键原则：

- **环境相关的配置（如 datasource）只放 dev/prod，不放基础 `application.yaml`**——本地库和生产库不同，放基础会造成重复且语义错误。
- 用 `spring.profiles.active: dev` 指定默认激活哪个 profile。
- **prod 现在无法真填**（只能本地部署），用环境变量占位 `${DB_URL:}` + 留空即可，等 W18 真正上线再填。符合 YAGNI。

当前基础配置（已定稿）：

```yaml
spring:
  application:
    name: mini-crm
  profiles:
    active: dev
  jackson:
    time-zone: UTC
server:
  port: 8080
```

---

## 2. 数据库访问技术栈分层（概念框架）

今天答疑顺带理清了整条数据库访问链路，**全是"规范 + 实现"的分层套路**：

```
你的代码 → Spring Data JPA → JPA规范 → Hibernate → JDBC规范 → MySQL驱动 → MySQL
            (再上层封装)      (接口/注解)  (默认实现,生成SQL)  (统一接口)  (mysql-connector-j)
```

| 名称 | 是什么 | 类比 |
|------|--------|------|
| **JDBC** | Java 连数据库的**最底层规范**（`java.sql` 包接口） | 地基 |
| **Driver** | JDBC 规范的具体实现（`mysql-connector-j`） | 把标准调用翻译成 MySQL 协议 |
| **JPA** | "对象 ↔ 表"映射的 ORM **规范**（`@Entity`/`@Id` 等注解） | USB 接口标准 |
| **Hibernate** | JPA 的默认**实现**，真正生成并执行 SQL 的引擎 | 某牌子 U 盘 |
| **Spring Data JPA** | Spring 在 JPA 之上的**再封装**（方法名派生查询等） | 更省事的上层 |

- 越上层越抽象省事（只管对象）；越下层越繁琐（手写 SQL、管连接）。
- **连接池**：JDBC 管"连接"，频繁建连很贵，Spring Boot 默认用 HikariCP 复用连接。
- **地域主流**：海外主流 Hibernate（全自动 ORM，对象主导），中国主流 MyBatis（半自动 SQL 映射，SQL 可控）。根因是"全自动 ORM vs 半自动 SQL 映射"两种理念，不是地域本身。本项目面向海外，选 Spring Data JPA + Hibernate 是对的。

> 术语提醒：正式名是 **Spring Data JPA**，不是 "Spring JPA"。

---

## 3. 日志配置（核心，最容易踩坑）

### 3.1 `logging.level` 下的 key 是 logger 名字（= 包名/类名）

最关键的认知：**`logging.level` 下面的 key 不是固定关键字，而是 logger 的名字，通常就是包名或类名全路径。**

- logger 名默认就是**产生日志的那个类的全限定名**（如 `com.minicrm.common.controller.PingController`）。
- logger 名按 `.` 分层，有父子继承关系：`...PingController` → `com.minicrm.common.controller` → `com.minicrm` → `root`。
- 一条日志输出时，先找最具体的匹配级别，找不到就向上继承，直到 `root` 兜底。
- **唯一特殊 key 是 `root`**——根记录器，所有 logger 的兜底父级。
- **`sql`、`web` 这种简写名是无效的**——没有框架用这名字写日志。要写真实包名：SQL 用 `org.hibernate.SQL`，Web 用 `org.springframework.web`。

### 3.2 日志级别（从低到高）

```
TRACE  <  DEBUG  <  INFO  <  WARN  <  ERROR
```

核心规则：**设一个级别 = 放行该级别及更高的，过滤更低的。** 级别设得越低输出越多越吵。所以 `root` 通常保持 `info`，只对关心的具体包单独调高。

| 级别 | 用途 |
|------|------|
| TRACE | 最细追踪（如 SQL 参数绑定值） |
| DEBUG | 调试细节（入参、分支、生成的 SQL） |
| INFO | 正常运行的关键里程碑（启动完成、关键业务动作） |
| WARN | 不正常但能继续（重试、降级、废弃 API） |
| ERROR | 出错且需人关注（异常、操作失败） |

### 3.3 配置只是"开闸"，还要代码"放水"

`com.minicrm: debug` 只是把闸门降到 debug，**日志不会自己产生**，必须在代码里主动写：

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger log = LoggerFactory.getLogger(PingController.class);
// ...
log.debug("收到 ping 请求");   // ← 这行才真正写日志
```

- 用 SLF4J 的 `Logger`/`LoggerFactory`（`org.slf4j` 包），底层由 Logback 实现，Spring Boot 默认这套。
- `getLogger(PingController.class)` 用类对象取 logger，logger 名自动 = 类全名，所以按包名配置能精准命中。
- 框架日志（Spring/Hibernate）不用你写——框架内部早写满了 `log.debug(...)`，你只要调高对应包的级别就能看到。

### 3.4 dev 日志定稿配置

```yaml
logging:
  level:
    root: info                          # 兜底
    org.hibernate.SQL: debug            # Hibernate 生成的 SQL（W2 连库后才有输出）
    org.hibernate.orm.jdbc.bind: trace  # SQL 参数实际值（必须 trace，debug 打不出）
    com.minicrm: debug                  # 自己代码（需代码里写 log 才有输出）
```

⚠️ 易错点：`bind` 必须用 `trace` 而非 `debug`——参数绑定值记录在 trace 级，门槛不够低就打不出参数实际值。
风格建议：统一用扁平点号写法（`org.hibernate.SQL`），别和 YAML 嵌套写法混用。

---

## 4. 时区配置

时区有多层，今天只配了 JSON 序列化这层：

- `spring.jackson.time-zone: UTC` —— 控制 JSON 序列化输出的时区。
- 数据库连接时区（datasource url 的 `serverTimezone`）—— 留到 W2 连库时配。
- 项目面向海外，统一用 **UTC** 存储、展示层再转，是国际化项目常见做法。

---

## 5. 什么情况加日志、加什么级别

### 按级别

- **ERROR**：预期外异常、操作彻底失败，**必带异常堆栈**（`log.error("...", e)`）。可预期的业务校验失败不用 ERROR。
- **WARN**：不正常但能继续（重试、降级、走兜底、限流、非法请求被拦）。
- **INFO**（生产默认）：关键业务里程碑（启动完成、注册/登录成功、订单创建、租户切换）。原则：只看 INFO 能还原"系统发生了什么大事"，但不淹没在细节里。
- **DEBUG**：开发/排查用细节（入参、中间结果、分支走向、缓存命中），生产默认关。
- **TRACE**：最细追踪（循环内部、SQL 参数绑定）。

### 按位置

适合：系统边界（Controller 入口、调外部 API）、关键业务节点（状态流转、涉及钱/权限）、异常捕获处、重要分支决策点。

不该：getter/setter 等简单方法、循环体内高频打 INFO、每个方法进出都打、**打印敏感信息（密码/token/身份证，绝对禁止）**。

### 工程习惯

1. **用占位符 `{}` 不用字符串拼接**：`log.info("登录成功, userId={}", userId)`。级别不放行时连参数都不拼，省性能。
2. **异常对象放最后一个参数**：`log.error("失败, id={}", id, e)`，不要 `e.getMessage()` 拼进字符串（会丢堆栈）。
3. **日志带上下文 ID**：userId、orderId、tenantId。多租户项目里带 `tenantId` 对排查极有帮助。

### Mini CRM 具体建议

| 场景 | 级别 |
|------|------|
| 用户注册/登录成功（W3） | INFO |
| 登录失败、token 无效（W3） | WARN |
| 租户上下文切换（W4-5） | DEBUG |
| 业务流转 Lead→Opportunity | INFO |
| 全局异常处理器捕获未知异常（W3） | ERROR + 堆栈 |
| 参数校验失败 | WARN 或不记 |

---

## 6. 优雅地声明 logger：Lombok `@Slf4j`

### 先澄清：手写那行不是 new

```java
private static final Logger log = LoggerFactory.getLogger(PingController.class);
```

`LoggerFactory.getLogger(...)` 是工厂方法，内部按 logger 名**复用实例**，不是每次创建新对象。所以没性能问题，只是写起来啰嗦。

### 更优雅：Lombok 的 `@Slf4j`

类上加一个注解，Lombok 在编译期自动生成名为 `log` 的 logger：

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RestController
@RequestMapping("/api")
public class PingController {
    @GetMapping("/ping")
    public ApiResponse<Long> ping() {
        log.debug("收到 ping 请求");   // 直接用 log，无需声明
        return ApiResponse.success(System.currentTimeMillis());
    }
}
```

生成的 logger 名仍是当前类全名，行为一致，但省去 import 和声明。

### 使用前提

1. 需引入 Lombok 依赖（计划 W1 依赖清单写的"Lombok 可选"，允许引入）：
   ```xml
   <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
   </dependency>
   ```
2. **IDE 要装 Lombok 插件**，否则编辑器标红找不到 `log`（编译本身没问题）。
3. Lombok 是"编译期生成代码"工具，后续写实体/DTO 时 `@Data`/`@Getter`/`@Builder` 也能省大量样板代码，长期收益高。
4. 权衡：Lombok 是"魔法"，部分海外团队不喜欢（隐藏代码、调试有坑）。个人作品提效角度利大于弊。

---

## 7. Java 日志框架生态（门面 + 实现）

又是"规范 + 实现"的分层套路。日志世界分两类角色：

```
你的代码 → SLF4J（门面/规范） → Logback（实现/引擎） → 控制台/文件
                ↑ 类比 JPA          ↑ 类比 Hibernate
```

| | SLF4J | Logback | Log4j2 |
|---|---|---|---|
| 角色 | 门面（规范），只定义接口 | 实现，真正写日志 | 实现，真正写日志 |
| 你代码 import 谁 | **SLF4J**（`org.slf4j`） | 不直接碰 | 不直接碰 |
| Spring Boot 默认 | 是门面 | **默认实现** | 需手动切换 |

- **Log4j 1.x**：元老，已**停止维护，不要再用**。
- **Log4j 2（Log4j2）**：重写版，异步日志性能强，仍是主流实现之一。（2021 年 Log4Shell 远程代码执行漏洞即出自 Log4j2，已修复；做海外项目知道这段背景有助于回应依赖安全问题。）
- **Logback**：Log4j 1.x 原作者重新设计的进化版，与 SLF4J 同作者、无缝配合，是 **Spring Boot 默认实现**。

### 为什么要有门面（SLF4J）这层

跟 JPA 一样：**解耦、可替换**。代码永远只 import SLF4J，不直接依赖 Logback。将来想换 Log4j2 只换底层依赖，业务代码一行不改。这就是"面向门面编程"。

### 落到项目

- 什么都不用配，Spring Boot 已装好 `SLF4J + Logback`。
- 写日志：用 SLF4J（`org.slf4j.Logger` 或 Lombok `@Slf4j`）。
- 调级别：`application.yaml` 的 `logging.level`，由 Logback 执行。
- 无特殊性能需求**不必换 Log4j2**，默认 Logback 足够（YAGNI）。

---

## 8. 最小冒烟测试

### 8.1 什么是冒烟测试

名字来自硬件：新电路板通电，**冒烟就说明短路，不用往下测了**。软件借用：**冒烟测试 = 最基本的"系统还能不能正常启动/运行"的验证**，只回答"这东西没彻底坏吧、能跑起来吧"。

特点：范围最浅最快、是第一道关卡（冒烟不过后面测试没意义）、数量少（一两个即可，不追覆盖率）。

本项目的两个冒烟测试：

1. **`MiniCrmApplicationTests.contextLoads()`**：应用级——验证整个 Spring 上下文能否成功装配。
2. **`PingControllerTest`**：Web 层——验证 `/api/ping` 接口请求响应正确。

### 8.2 空的 `contextLoads()` 为何能验证上下文启动

关键全在 `@SpringBootTest` 注解上，**验证发生在方法体运行之前**：

1. JUnit 看到类上 `@SpringBootTest`，在跑任何 `@Test` 前先**启动一整个 Spring 应用上下文**（加载配置、扫描创建所有 Bean、依赖注入、自动装配）。
2. 上下文成功起来后，才执行空的 `contextLoads()`，直接通过。
3. **若上下文启动失败（Bean 缺依赖、循环依赖、配置错、Bean 冲突、Security 配置错、连不上库……）会直接抛异常，测试在进入方法体前就 FAIL。**

所以：**方法体为空却能绿过 → 证明上下文从头到尾干净启动了。** 它是"以启动类为入口的应用级冒烟测试"，测的是整个上下文，不是启动类那三行本身。覆盖范围随项目长大自动扩大（每加一个 Bean/配置都顺带被它验证）。

### 8.3 MockMvc：测"接口"而非"方法"

**MockMvc = Spring 提供的"假装发 HTTP 请求"的测试工具**，走完整 Spring MVC 流程（路由匹配、参数绑定、JSON 序列化、Security 过滤器），但**不启动真实 Tomcat、不占端口**，全程内存模拟。

为什么不用"直接调 Controller 方法"：直接调方法会**绕过** HTTP 层（路由、序列化、过滤器链），等于没测 Web 层。MockMvc 既真实（走完整链路）又快（不起服务器）。

`PingControllerTest` 最终版：

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
public class PingControllerTest {
    @Autowired
    MockMvc mockMvc;

    @Test
    void ping() throws Exception {
        mockMvc.perform(get("/api/ping"))
                .andExpect(status().isOk())                       // HTTP 200
                .andExpect(jsonPath("$.code").value(SUCCESS))     // code=0
                .andExpect(jsonPath("$.message").value("success"))
                .andExpect(jsonPath("$.data").isNumber());        // data 是数字
    }
}
```

要点：

- `@AutoConfigureMockMvc`：让 Spring 自动配置 MockMvc Bean，才能 `@Autowired` 注入。
- `get(...)`、`status()`、`jsonPath(...)` 都是**静态方法，必须 `import static`**，否则报 `Cannot resolve method 'get'`（Java 以为是当前类的方法）。
  ```java
  import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
  import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
  ```
- `jsonPath("$.code")`：JsonPath 语法，`$` 是 JSON 根，从响应体按路径取字段。

### 8.4 测试 = 执行 + 断言（踩坑）

最初写法只有 `pingController.ping();` 没有任何断言——**只验证了"不抛异常"，没检查返回对不对**。没有断言的测试只要不崩就永远绿，毫无价值。

测试核心：**做一件事 → 断言结果符合预期**。引入了 `Assertions.*` 就要真的用。

### 8.5 测试 profile 与数据库隔离（重要踩坑）

**问题现象**：跑测试报错 `Access denied for user 'Lawliet'@'localhost' (using password: NO)` + `Unable to determine Dialect`。

**根因链**：

1. `@SpringBootTest` 默认读 `spring.profiles.active`（即 dev），dev 的 datasource 指向本地 MySQL。
2. dev 里 `username`/`password` 都是空占位，MySQL 驱动在用户名为空时**回退用操作系统登录用户名**（`Lawliet`）、无密码去连 → 被拒。
3. 连库失败 → Hibernate 无法探测方言 → `EntityManagerFactory` 创建失败 → 上下文起不来 → 测试 FAIL。

> 这条错误反而**证明 dev profile 确实生效了**（真去连了 MySQL）。"感觉没走 dev"是错觉——因为没有 `@Entity`、没执行 SQL，所以 `org.hibernate.SQL: debug` 没输出，让人误判。确认 profile 是否激活应看启动日志的 `The following 1 profile is active: "dev"`。

**解决：测试用独立 test profile + H2 内存库**（业界标准，符合冒烟测试"随处可跑、不依赖外部环境"原则）：

1. pom 加 H2 依赖 `<scope>test</scope>`。
2. **配置文件必须放 `src/test/resources/application-test.yaml`，不能放 `src/main/resources/`**：
   - H2 是 test 范围依赖，主程序 classpath 没有 H2。
   - 若把 `application-test.yaml` 放进 main，IDEA 报 `Cannot resolve class or package 'h2'`（主程序配置引用了它没有的依赖）。
   - **通用原则：配置和依赖的"作用范围"要一致**——test 依赖配 test 资源目录。
3. 测试类加 `@ActiveProfiles("test")`。

H2 配置示例（`src/test/resources/application-test.yaml`）：

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=MySQL
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
```

### 8.6 补回遗漏的 SecurityConfig

周六漏建了 SecurityConfig（当时用控制台默认密码绕过了），导致测试请求 `/api/ping` 被 Security 默认规则拦截。

补上后 `/api/ping` 在**所有环境**（含测试）都公开放行——健康检查接口本就该公开，不是为测试妥协：

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity httpSecurity) {
        return httpSecurity.csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/ping").permitAll()
                        .anyRequest().authenticated()).build();
    }
}
```

**关键点：返回类型是 `SecurityFilterChain`，但 `http` 是 `HttpSecurity`（建造者），必须 `.build()` 才能得到产品。** 直接 `return http` 类型对不上会编译报错。这是建造者模式：链式 `.csrf().authorizeHttpRequests()` 一路返回 `HttpSecurity` 本身，最后 `.build()` 造出 `SecurityFilterChain`。

> SecurityFilterChain 本质 = 一串"安检关卡"，每个 HTTP 请求依次过这些关卡决定放行/拦截。`permitAll()` = 免检放行，`authenticated()` = 必须先认证。深入留到 W3（认证、授权、JWT 过滤器）。

测试中关闭鉴权的其他备选（本次未用，留备 W3）：

- `@AutoConfigureMockMvc(addFilters = false)`：关掉所有过滤器，最快但绕过全部过滤器。
- `@WithMockUser`：模拟已登录用户（来自 `spring-boot-starter-security-test`），适合测需鉴权的接口。

### 8.7 自动化测试 vs 手动 curl

手动 `curl /api/ping` 是人工冒烟；写进 `src/test` 的是自动化冒烟——每次 build 自动跑，CI 能拦截"连启动都失败"的版本。对 Upwork 作品而言，**有测试本身就是专业度的体现**。

### 8.8 IDEA 快捷生成测试类

在被测类名上按 `Ctrl + Shift + T` → "Create New Test..."，选 JUnit5，IDEA 自动在 `src/test` 下镜像包路径生成测试类（只生成空壳，注解和逻辑仍需自己写）。`Ctrl + Shift + T` 也用于"被测类 ↔ 测试类"双向跳转。

> 测试类约定：放 `src/test`，**包路径与被测类完全一致**（同包可见性、Maven 标准约定、可读性）。
