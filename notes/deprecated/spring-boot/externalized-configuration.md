# Externalized Configuration

> **来源:** https://docs.spring.io/spring-boot/reference/features/external-config.html
>
> **对应计划周次:** 第 1 周 · Spring Boot 项目骨架（配置部分）

## 核心理解

Spring Boot 把"配置"和"代码"分离，让同一份 jar 在不同环境（本地 / 测试 / 生产）用不同配置运行。所有配置来源（命令行、环境变量、`application.yml`、`@PropertySource` 等）最终都被整合进一个统一的 `Environment` 抽象——它内部维护一个**有序的 PropertySource 列表**（`MutablePropertySources`），按优先级排好序。

理解这套机制的关键只有两点：**（1）配置来源有明确优先级，高优先级覆盖低优先级；（2）配置值在"被使用时"才解析占位符，取的是 Environment 里最终生效的值。** 掌握这两点，所有"为什么我的配置没生效"的问题都能自己定位。

在 Mini CRM 里：`application.yml` 放通用默认值，`application-dev.yml` / `application-prod.yml` 放环境差异，生产的数据库密码/JWT 密钥走**环境变量**注入（W17 Docker、W18 上线），业务配置（如 JWT 过期时间）封装成 record + `@ConfigurationProperties`（W3 用）。

## 关键点

### 1. 配置来源优先级（高 → 低，覆盖关系）

> **原文：** "Sources are considered in the following order"

所有 PropertySource 整合进 `Environment`，**靠后的被靠前的覆盖**。Mini CRM 需要记住的核心几项：

| 优先级 | 来源 | Mini CRM 用途 |
|--------|------|--------------|
| 高 | **命令行参数** `--server.port=9090` | 临时覆盖最方便，启动 jar 时指定端口/profile |
| ↑ | **OS 环境变量** `SERVER_PORT=9090` | **生产部署核心**：避免密码写进配置文件，便于 Docker 注入 |
| ↑ | **Config data**（`application.properties` / `.yml`） | 最常用，日常配置写这里 |
| ↑ | `@PropertySource`（`@Configuration` 上） | 加载自定义配置文件，**注意加载太晚** |
| 低 | 代码默认值 | 兜底 |

```bash
# 命令行覆盖（最高优先级之一）
java -jar app.jar --server.port=8082 --spring.profiles.active=prod

# 环境变量（生产推荐，密码不进文件）
export SPRING_DATASOURCE_PASSWORD=xxx
```

### 2. `@PropertySource` 加载太晚的坑

> **原文：** "such property sources are not added to the Environment until the application context is being refreshed. This is too late to configure certain properties such as logging.* and spring.main.* which are read before refresh begins."

- `@PropertySource` 引入的配置，**直到 ApplicationContext 刷新时才加入 Environment**
- 而 `logging.*`、`spring.main.*` 这些属性在**刷新之前**就被读取了
- **结论：** 这些属性必须放 `application.properties` / `application.yml`，不能靠 `@PropertySource`

```java
@Configuration
@PropertySource("classpath:custom.properties")   // 只适合普通业务配置
public class CustomConfig { }

// custom.properties 里写 logging.level.root=DEBUG → 无效！
```

### 3. 测试场景的配置覆盖

```java
// 方式一：properties 属性内联覆盖
@SpringBootTest(properties = {
    "server.port=0",        // 0 = 随机端口
    "app.test.mode=mock"
})
class MyServiceTest { }

// 方式二：@TestPropertySource 指定文件
@SpringBootTest
@TestPropertySource(locations = "classpath:test-overrides.properties")
class DatabaseTest { }
```

W14-W15 写测试时会用到，知道有这两种方式即可。

### 4. application 配置文件的查找位置与顺序

> **原文：** "Spring Boot will automatically find and load application.properties and application.yaml files from the following locations when your application starts"

查找顺序（后者覆盖前者）：
1. classpath 根目录（jar 内 / `src/main/resources`）
2. classpath 下的 `/config` 目录
3. jar 所在的文件系统当前目录
4. 当前目录下的 `config/` 子目录

**含义：** jar 内 `application.yml` 提供默认值，部署到新环境时在 jar **外部**放一个同名文件即可覆盖，无需重新打包。

> **配置文件格式不要混用：** 同一位置同时有 `.properties` 和 YAML 时，`.properties` 优先。**Mini CRM 统一用 YAML。**

### 5. classpath 不是固定物理位置，而是"类加载器的搜索路径集合"

**两种打包形态：**

**普通 JAR：** jar 根目录就是 classpath 根，作为二方/三方包被依赖，通常不可直接运行。

**可执行 JAR（Spring Boot Fat JAR）：** 结构变化——
```
springboot-app.jar
├── META-INF/MANIFEST.MF          ← Main-Class = JarLauncher, Start-Class = 你的 App
├── org/springframework/boot/loader/...   ← Spring Boot 启动器（标准 AppClassLoader 加载）
└── BOOT-INF/
    ├── classes/                  ← 你的代码 + 资源（LaunchedURLClassLoader 加载）
    │   ├── com/example/App.class
    │   ├── application.properties
    │   └── static/index.html
    └── lib/                      ← 所有第三方依赖 jar（LaunchedURLClassLoader 加载）
```

**关键澄清：** 标准 JVM 不支持"嵌套 JAR"（jar 套 jar），所以 Spring Boot 用自定义的 `LaunchedURLClassLoader` 把 `BOOT-INF/classes/` 和 `BOOT-INF/lib/*.jar` 都纳入加载范围。**`BOOT-INF/lib` 确实是 classpath，但不是 JVM 原生 classpath，而是 Spring Boot 自定义类加载器实现的。** 这正是"一个 jar 即可独立运行"的核心。

> **结论：** classpath = 类加载器查找类与资源的搜索路径集合，可由命令行、环境变量、清单文件、框架自定义机制共同决定，**不是一个固定的物理位置**。

### 6. Profile-specific 文件

> **原文：** "if your application activates a profile named prod and uses YAML files, then both application.yaml and application-prod.yaml will be considered."

- 激活 `prod` profile → `application.yml`（通用）+ `application-prod.yml`（专属）**都加载**
- **profile 专属文件覆盖通用文件**
- 多 profile 时 **last-wins**：`spring.profiles.active=prod,live` → `application-live` 覆盖 `application-prod`
- 没激活任何 profile → 用 `default` profile → 加载 `application-default`

**Mini CRM W1 必用：** dev / prod 两套 profile。

### 7. Environment Variables 暂时感觉用不着，但实际上是生产部署常用解

你批注里写了"感觉暂时用不着"，对 **W1 本地开发** 来说确实如此；但对 **W17 Docker / W18 真实上线** 来说，这是核心能力。

```bash
# Linux / macOS
export SERVER_PORT=8081
export SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/minicrm
export SPRING_DATASOURCE_PASSWORD=secret
```

- 本地开发：多数配置直接写 `application-dev.yml`
- 生产部署：数据库密码、JWT 密钥、Redis 密码等**不要写进仓库**，改走环境变量
- 这也是容器部署最自然的注入方式

### 8. 占位符 `${...}` 是在"取用时"解析，不是在文件加载时立刻展开

> **原文：** "The values in application.properties and application.yaml are filtered through the existing Environment when they are used..."

这个理解你抓得很准。关键顺序是：

1. Spring Boot 在 `prepareEnvironment()` 阶段收集所有外部配置源，合并成最终 `Environment`
2. 这时配置文件里的 `${...}` 仍然只是字符串字面量
3. 到 Bean 实例化 / `@Value` 注入 / `@ConfigurationProperties` 绑定 / `Environment.getProperty()` 时，才触发占位符解析
4. 解析时取的是 **Environment 里最终生效的值**，不是当前文件里某一份原始值

```yaml
app:
  name: mini-crm
  welcome: ${app.name:default-app}
```

```java
@Value("${app.welcome}")
private String welcome;
```

这里 `welcome` 注入时，Spring 才会去 `Environment` 里拿 `app.name`。

### 9. YAML 本质上会被拍平成 Spring Environment 可识别的扁平 key

> **原文：** "YAML documents need to be converted from their hierarchical format to a flat structure that can be used with the Spring Environment."

```yaml
my:
  service:
    remote-address: 192.168.1.1
    security:
      username: admin
      password: secret
      roles:
        - USER
        - ADMIN
```

会被理解为近似这样的扁平 key：

```properties
my.service.remote-address=192.168.1.1
my.service.security.username=admin
my.service.security.password=secret
my.service.security.roles[0]=USER
my.service.security.roles[1]=ADMIN
```

这个心智模型很重要，因为它解释了：
- 为什么 `@Value("${my.service.remote-address}")` 能读到 YAML
- 为什么 `@ConfigurationProperties(prefix = "my.service")` 能绑定嵌套对象
- 为什么环境变量 `MY_SERVICE_REMOTEADDRESS` 也能绑定到同一属性

### 10. `@ConfigurationProperties` 绑定流程

> **原文：** "It is possible to bind a bean declaring standard JavaBean properties as shown in the following example"

你总结的流程基本就是正确的运行时顺序：

1. 应用启动，读取 `application.yml`，构建 `Environment`
2. Spring 创建 `MyProperties` 实例
3. Binder 从 `Environment` 中找出 `my.service.*` 的配置项
4. 按前缀 + 字段路径把值绑定到对象
5. 若加了 `@Validated`，执行校验
6. 绑定完成后的对象注册进容器，供其他 Bean 注入使用

**关键结论：** 当其他组件拿到 `MyProperties` Bean 时，值已经填好了。

### 11. `@ConfigurationProperties` 类必须先注册成 Bean

光写 `@ConfigurationProperties` 不会自动进容器，至少要满足下面一种方式：

```java
// 方式一：显式启用（推荐，职责清晰）
@Configuration
@EnableConfigurationProperties(MyProperties.class)
public class MyConfig {
}
```

```java
// 方式二：交给组件扫描
@Component
@ConfigurationProperties("my.service")
public class MyProperties {
}
```

或者使用 `@ConfigurationPropertiesScan` 统一扫描。

### 12. JavaBean 绑定：为什么通常要求无参构造 + getter/setter

> **原文：** "Such arrangement relies on a default empty constructor and getters and setters are usually mandatory, since binding is through standard Java Beans property descriptors"

这里的底层机制就是你总结的 **Java Bean PropertyDescriptor**：

- Spring 不是直接看字段名做绑定
- 它先通过 Java 内省机制分析 getter/setter
- 再推断出有哪些可写属性
- 最后调用 setter 把配置值塞进去

```java
@ConfigurationProperties("my.service")
public class MyProperties {

    private String remoteAddress;

    public String getRemoteAddress() {
        return remoteAddress;
    }

    public void setRemoteAddress(String remoteAddress) {
        this.remoteAddress = remoteAddress;
    }
}
```

**所以会得到几个重要规则：**

| 现象 | 原因 |
|------|------|
| 字段通常要有 setter | JavaBean 绑定靠 setter 写值 |
| 只有 getter 没 setter 常常绑不进去 | 没有 write method |
| boolean 的 `isXxx()` 也能识别 | JavaBean 命名约定支持 |
| 嵌套对象通常要有 getter | Spring 先拿到内部对象，再往里继续绑定 |

### 13. 构造器绑定：不可变配置类的更现代方案

```java
@ConfigurationProperties("my.service")
public class MyProperties {

    private final String remoteAddress;

    public MyProperties(String remoteAddress) {
        this.remoteAddress = remoteAddress;
    }

    public String getRemoteAddress() {
        return remoteAddress;
    }
}
```

**适合场景：**
- 配置对象希望不可变
- 字段用 `final`
- 不想写 setter
- 尤其适合 record

> **原文：** "Constructor binding can be used with records"

这也是为什么你在 W1 学了 `record` 之后，后面写配置类时完全可以这样：

```java
@ConfigurationProperties("minicrm.jwt")
public record JwtProperties(String secret, Duration ttl) {
}
```

### 14. 构造器绑定的自动判断规则与歧义

> **原文：** "the presence of a single parameterized constructor implies that constructor binding should be used"

规则很简单：
- 一个 `@ConfigurationProperties` 类如果**只有一个带参构造器**
- Spring Boot 默认就认为你要用**构造器绑定**

这在大多数情况下是方便的，但会产生一个歧义：

```java
@ConfigurationProperties("my")
public class MyProperties {

    private final MyBean myBean;
    private String name;

    @Autowired
    public MyProperties(MyBean myBean) {
        this.myBean = myBean;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

这里构造器参数 `MyBean` 明明是想做 **依赖注入**，不是从配置文件里绑定值。  
如果不声明意图，Spring 可能会误以为你要做构造器绑定，去找 `my.my-bean` 这样的配置，结果出错。

**退出构造器绑定（opt-out）的两种方式：**
- 给带参构造器加 `@Autowired`
- 或把该构造器设为 `private`

这等于明确告诉 Spring：这个构造器是给 DI 用的，不是给配置绑定用的。

### 15. 构造器绑定这一堆约束，只记最重要的 5 条

你问"这一大堆约束，总结一下吧"，这里直接给你最实用版：

1. **推荐优先级**
   - 简单、传统、可变配置类：JavaBean 绑定
   - 现代、不可变、想少写样板：构造器绑定 / record

2. **要启用**
   - 构造器绑定的类要通过 `@EnableConfigurationProperties` 或 `@ConfigurationPropertiesScan` 注册
   - 不要把它同时当普通 `@Component` / `@Bean` 方式创建

3. **record 天然适合**
   - `record` + `@ConfigurationProperties` 是非常顺手的组合

4. **不要用 `Optional` 当配置字段**
   - 官方不推荐
   - 没值时绑定出来常常是 `null`，不是你以为的 `Optional.empty()`

5. **保留字属性名**
   - 像 `my.service.import` 这种保留字场景很少见
   - 以后真遇到再回文档查 `@Name`

> **W1 只要记一句：** 配置类优先用 `record` + `@ConfigurationProperties`，不要上来研究全部边角规则。

### 16. Relaxed Binding：同一个属性支持多种命名形式

这部分你整理得已经很完整了。核心就是：

Java 属性：

```java
private String firstName;
```

下面这些都能绑定到它：

```properties
my.main-project.person.first-name=Rod
my.main-project.person.firstName=Rod
my.main-project.person.first_name=Rod
```

环境变量里则通常写成：

```bash
MY_MAINPROJECT_PERSON_FIRSTNAME=Rod
```

### 17. prefix 必须用 kebab case

> **原文：** "The prefix value for the annotation must be in kebab case."

```java
// ✅ 推荐
@ConfigurationProperties("my.main-project.person")

// ❌ 不推荐 / 不应使用
@ConfigurationProperties("my.mainProject.person")
@ConfigurationProperties("my.main_project.person")
```

**规则：** 注解上的 prefix 用小写 + `-` 分词；类里的字段名继续用 Java 驼峰。

### 18. 不同配置来源下的 Relaxed Binding 规则

**properties 文件：**

```properties
my.person.first-name=Rod
my.person.firstName=Rod
my.person.first_name=Rod
my.person.roles[0]=USER
my.person.roles[1]=ADMIN
```

**YAML：**

```yaml
my:
  person:
    first-name: Rod
    roles:
      - USER
      - ADMIN
```

**环境变量：**

```bash
MY_PERSON_FIRSTNAME=Rod
SERVER_PORT=8080
MY_SERVICE_0_OTHER=value
```

其中：
- `MY_SERVICE_0_OTHER=value` 对应 `my.service[0].other=value`
- YAML 虽然也支持驼峰/下划线写法，但**推荐统一 kebab case**

### 19. Map 绑定的特殊规则：有特殊字符时用 `[]` 保 key

```properties
my.map[/key1]=value1
my.map[/key2]=value2
my.map./key3=value3
```

绑定到：

```java
private Map<String, String> map;
```

结果近似：

```java
{
  "/key1": "value1",
  "/key2": "value2",
  "key3": "value3"
}
```

原因：
- key 没有被 `[]` 包起来时
- 除了字母、数字、`-`、`.` 之外的字符会被清理
- 所以 `/key3` 最终变成 `key3`

**YAML 里如果 key 带 `[]`，要加引号：**

```yaml
my:
  map:
    "[/key1]": value1
    "[/key2]": value2
    /key3: value3
```

### 20. Mini CRM 里真正要落地的配置实践

```yaml
# application.yml
spring:
  application:
    name: mini-crm

minicrm:
  jwt:
    ttl: 30m

---
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/minicrm_dev
    username: root
    password: root
```

```java
@ConfigurationProperties("minicrm.jwt")
public record JwtProperties(Duration ttl) {
}
```

生产环境：

```bash
export SPRING_DATASOURCE_PASSWORD=real-secret
export SPRING_PROFILES_ACTIVE=prod
```

**这就是 W1 真正该会的外部配置能力。**

## 句子解析

### 原文: "This is too late to configure certain properties such as logging.* and spring.main.* which are read before refresh begins."

- **翻译:** 这已经太晚了，无法再配置某些属性，例如 `logging.*` 和 `spring.main.*`，因为这些属性会在刷新开始之前就被读取。
- **解析:**  
  `too late to configure` = 已经过了配置时机；  
  `before refresh begins` 指的是 ApplicationContext `refresh()` 之前；  
  结论就是 `@PropertySource` 不适合配置早期启动参数。

### 原文: "Sources are considered in the following order."

- **翻译:** 配置来源会按如下顺序被考虑。
- **解析:**  
  这里的 `order` 就是优先级顺序；  
  Spring 最终把它们组织进 `Environment` 的有序 `PropertySource` 列表中。

### 原文: "By default, SpringApplication converts any command line option arguments ... to a property and adds them to the Spring Environment."

- **翻译:** 默认情况下，`SpringApplication` 会把所有命令行选项参数转换成属性并加入 Spring 的 `Environment`。
- **解析:**  
  这就是为什么 `--server.port=9000` 能像普通配置一样生效；  
  命令行参数会覆盖文件配置，是临时覆盖最方便的方式。

### 原文: "The values in application.properties and application.yaml are filtered through the existing Environment when they are used..."

- **翻译:** `application.properties` 和 `application.yaml` 中的值在被使用时，会经过现有 `Environment` 的过滤处理。
- **解析:**  
  `when they are used` 很关键，不是文件一加载就立刻展开占位符；  
  占位符解析基于最终合并后的 `Environment`。

### 原文: "The presence of a single parameterized constructor implies that constructor binding should be used."

- **翻译:** 当类中存在唯一一个带参数的构造器时，就意味着应当使用构造器绑定。
- **解析:**  
  `implies` 这里是"推定/意味着"；  
  它说的是 Spring Boot 的默认判断规则，不代表你不能通过 `@Autowired` 等方式退出这种推断。

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| externalized configuration | n. | 外部化配置 |
| PropertySource | n. | 配置来源抽象 |
| Environment | n. | Spring 的环境抽象，统一读取配置 |
| MutablePropertySources | n. | 有序的 PropertySource 列表容器 |
| classpath | n. | 类加载器搜索类与资源的路径集合 |
| config data | n. | 配置数据文件，如 `application.yml` |
| property placeholder | n. | 属性占位符，如 `${name:default}` |
| JavaBean binding | n. | 基于 getter/setter 的配置绑定方式 |
| constructor binding | n. | 基于构造器参数的配置绑定方式 |
| relaxed binding | n. | 宽松绑定，支持多种命名形式映射到同一属性 |
| kebab case | n. | 小写短横线命名，如 `first-name` |
| property descriptor | n. | 属性描述符，描述属性名、类型、读写方法 |
| introspection | n. | 内省，通过反射和命名规则分析 Bean 属性 |
| opt-out | v. | 主动退出某种默认行为 |
| profile-specific file | n. | profile 专属配置文件，如 `application-prod.yml` |
