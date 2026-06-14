# Spring Boot 项目中 build / compile / run 的区别

> **类型:** 实践结论（非文档摘录）
> **对应计划周次:** 第 1 周 · 周六搭骨架
> **场景:** 搭项目骨架时，搞清 IDE 里 build、compile、run 三个动作分别做了什么

## 一句话结论

- **compile**：把 `.java` 编译成 `.class` 字节码，只管语法/类型对不对。
- **build**：一条完整流水线（编译 + 拷资源 + 跑测试 + 打包），compile 只是其中一步。
- **run**：启动 JVM 跑 `main()`，加载 Bean、监听端口，把程序真正跑起来。

关系：`compile ⊂ build`；`run` 依赖 compile 的结果，但不依赖完整 build。

```
compile  ⊂  build  →  产物(jar/war)
   │
   └──→ run（启动 JVM 跑 main，加载 Bean，监听端口）
```

## Compile（编译）

- 输入：`src/main/java/**/*.java`
- 输出：`target/classes/**/*.class`
- 只做语法/类型/符号引用检查，不碰资源文件、不打包、不跑程序。
- Maven 命令：`mvn compile`
- IDE 保存文件时的"增量编译"就是这个层级，只重编改动的类，所以快。

## Build（构建）

Maven 标准生命周期顺序（大致）：

```
validate → compile → test → package → verify → install
```

`mvn package`（最常说的 build）依次做：

1. **compile**：编译主代码
2. **process-resources**：把 `src/main/resources` 下的 `application.yaml`、Flyway 脚本等**复制**到 `target/classes`（配置文件是拷贝，不是编译）
3. **test-compile + test**：编译并运行 `src/test` 下测试，测试不过则 build 失败
4. **package**：把 `target/classes` + 依赖打成可交付产物（jar/war）

> 注意：本项目 `pom.xml` 当前是 `<packaging>war</packaging>` 且 tomcat 为 `provided`，build 产物是 war 包（需丢进外部 Tomcat）。多数 Spring Boot 作品用默认 `jar` + 内嵌 Tomcat 更省事，war 是给传统容器部署用的。上线时留意。

## Run（运行）

两种方式：

1. **IDE 点 Run `MiniCrmApplication`**：先确保主代码已编译，再启动 JVM 执行 `main()` → `SpringApplication.run(...)` → 启动内嵌容器、扫描 `@SpringBootApplication`、装配 Bean、监听端口。
2. **`mvn spring-boot:run`**：更完整，先 compile + process-resources 再跑 `main()`，能保证配置文件已就位，改了 `application.yaml` 后能生效。

## 踩坑提醒（"我明明改了怎么没生效"）

- **改 Java 代码** → 必须重新 compile（IDE 自动或手动 build）才能在 run 时生效。
- **改 `application.yaml`** → 需要 process-resources 把它拷到 `target/classes`。IDE 的 Run 有时不自动重拷资源，导致改了配置没生效。遇到时用 `mvn spring-boot:run`，或先 build 再 run 最稳。
- 验证接口 404 时，先排查是不是改了代码没重新 compile/重启。
