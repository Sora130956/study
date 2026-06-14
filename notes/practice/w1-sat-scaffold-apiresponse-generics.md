# 第 1 周周六实践笔记 · 搭骨架（ApiResponse / 泛型 / Security）

> **类型:** 实践结论（非文档摘录）
> **对应计划周次:** 第 1 周 · 周六搭骨架
> **当日产出:** 包结构 + `/api/ping` + 统一响应 `ApiResponse<T>` + 临时放行 Security，链路打通

---

## 1. 为什么需要 `/api/ping` 接口

`/api/ping` 是一个**健康检查接口（health check）**：客户端 `GET /api/ping`，服务端回固定成功响应，不查库、不依赖业务逻辑，只回答一个问题——**"服务还活着、能正常响应 HTTP 吗？"**

它不是用完即删的临时测试代码，有长期价值：

- **冒烟验证**：搭完骨架后 `curl /api/ping` 通了，说明"路由 → Controller → 统一响应"整条链路通了。这是周六的验证点。
- **部署探活**：上线后（W18），Docker / 负载均衡 / 监控会定时打这个接口，判断容器是否要重启、流量是否转发。生产真的会用。
- **需放行鉴权**：W3 引入 JWT 后大部分接口要认证，但 `/api/ping` 必须放行——它得在"没登录"时也能访问，否则探活失效。

> 命名建议：叫 `PingController` / `HealthController`，别叫 `TestController`，避免被误当临时代码。

---

## 2. ApiResponse 怎么设计

核心目标：所有接口返回**统一结构**，前端用同一套逻辑解析（先看成功与否，再取 data 或读 message）。

### 最小够用的字段

```java
public record ApiResponse<T>(Integer code, String message, T data) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(SUCCESS, "success", data);
    }
    public static <T> ApiResponse<T> success() {          // 成功但无数据（如删除）
        return new ApiResponse<>(SUCCESS, "success", null);
    }
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(ERROR, message, null);
    }
}
```

- **code**：业务状态码，和 HTTP 状态码是两回事。本项目采用**业务码派**：`SUCCESS = 0`，`ERROR = 1000`（非 0 即失败）。HTTP 码那层交给全局异常处理器管，各司其职。
- **message**：给人看的提示，成功 `"success"`，失败是错误描述。
- **data**：业务数据，失败时通常 `null`。

### 关键设计决策

1. **用泛型 `ApiResponse<T>`**：`data` 类型千变万化，泛型让编译器做类型检查，而非 `Object`。
2. **静态工厂方法，不让调用方 new**：`ApiResponse.success(data)` 比手动塞三个字段干净，也统一了"成功"定义。
3. **用 record**：纯数据载体、字段不可变，正是 record 的典型场景。
4. **code 暂用常量**：当前只有成功/失败两种；等 W3 有了多种错误码（401/403/校验失败…）再抽 `ResultCode` 枚举，现在抽属于过早设计（YAGNI）。
5. **不加 timestamp/traceId**：等真有日志排查需求（多半上线后）再补。

### code 两派之争（已选业务码派）

- **业务码派（本项目选用）**：`code` 与 HTTP 码解耦，`0` 成功、非 `0` 失败。业务码和 HTTP 码是两个维度——一个"用户名重复"，HTTP 是 400，业务码可细分到 1003。职责清晰。
- **HTTP 码派**：`code` 直接对齐 HTTP 状态码（200/500…）。问题是把所有失败标成 500（服务器错）不准确，多数失败其实是客户端错（400/401/403/404）。

---

## 3. 全局异常处理器是做什么的

一句话：**把代码里抛出的各种异常，统一拦截并转换成规范的响应（含正确的 HTTP 状态码 + `ApiResponse` 业务体）**。

- 没有它：每个 Controller 都要写 try-catch 包错误，重复且容易漏。
- 有了它：业务代码该抛异常就抛（如参数非法抛 `IllegalArgumentException`、找不到抛自定义 `NotFoundException`），由全局处理器集中捕获，按异常类型映射成对应 HTTP 码 + 统一的 `ApiResponse.error(...)`。

Spring 里用 `@RestControllerAdvice` + `@ExceptionHandler` 实现。**这是 W3 的任务**，今天只走成功路径，先了解它的职责即可。它正是"业务码派"能成立的配套——HTTP 码这层交给它管。

---

## 4. 泛型讲解（结合今天踩的坑）

### 今天的 bug：声明了 `<T>` 却返回裸类型

最初写法：

```java
public static <T> ApiResponse success(T data) {   // ❌ 返回类型是裸 ApiResponse
    return new ApiResponse<>(SUCCESS, "success", data);
}
```

声明了类型参数 `<T>`，但返回类型写成 `ApiResponse` 而非 `ApiResponse<T>`，等于用了**裸类型（raw type）**。后果：

- 编译器报 unchecked 警告；
- 调用方拿到的是 raw type，`.data()` 取出来是 `Object` 而非真实类型，泛型白声明，类型安全丢失。

修正：返回类型补上 `<T>`：

```java
public static <T> ApiResponse<T> success(T data) {   // ✅
    return new ApiResponse<>(SUCCESS, "success", data);
}
```

### 菱形操作符 `<>` 的用法

`new ApiResponse<>(...)` 里的 `<>` 就是菱形操作符（Java 7+）：**右边 new 的类型参数让编译器自动推断，不用重复写**。

```java
// 啰嗦写法
ApiResponse<String> r = new ApiResponse<String>(...);
// 菱形写法：右边 <> 由左边/上下文推断
ApiResponse<String> r = new ApiResponse<>(...);
```

在静态工厂里，`return new ApiResponse<>(...)` 的 `<>` 会根据方法返回类型 `ApiResponse<T>` 推断为 `T`。所以方法签名的 `<T>` 必须写对，菱形才能推断出正确类型。

> 注意区分：方法签名前的 `<T>` 是**声明**类型参数；`new ...<>()` 的 `<>` 是**使用/推断**。两者不是一回事。

---

## 5. 泛型类中，为什么静态方法必须是泛型方法

`ApiResponse<T>` 的 `T` 是**实例级别的类型参数**——它属于"某个 ApiResponse 对象"，在 new 出对象时才确定。

而 `static` 方法**不属于任何实例**，它在类加载时就存在，调用时（`ApiResponse.success(...)`）根本没有具体对象，也就拿不到类上的 `T`。

所以：

- **静态方法不能使用类声明的 `T`**（编译报错：`Cannot make a static reference to the non-static type T`）。
- 如果静态方法自己需要泛型，必须**自带一个独立的类型参数**，即在方法签名上重新声明：`public static <T> ApiResponse<T> success(...)`。这个 `<T>` 和类的 `<T>` 只是恰好同名，是**完全独立的两个东西**。

一句话总结：**类的 `T` 跟着对象走，静态方法没有对象，所以要泛型就得自己声明一个方法级的 `T`。**

---

## 6. 今天犯的错误

1. **泛型返回裸类型**：声明了 `<T>` 却把返回类型写成 `ApiResponse` 而非 `ApiResponse<T>`，丢失类型安全（已修正，并补了菱形操作符）。
2. **Controller 方法返回裸类型**：`public ApiResponse ping()` 应为 `ApiResponse<Long>`（已修正）。
3. **改配置/代码后忘记重新 compile/重启**导致"改了没生效"——对应另一篇 build/compile/run 笔记的踩坑点。
4. **（助手侧）未核实代码就下结论**：助手凭对话记忆说"code 还是 200"，实际用户已改成 `0`。已在 CLAUDE.md 加规则：回答代码库相关结论前必须先读最新代码。

---

## 7. 做对的地方

1. **包结构清晰**：`common.controller` / `common.dto` / `common.constant` / `common.config` 按职责分层，为后续 16 周模块预留空间。
2. **用 record 做响应载体**：不可变数据载体，正确选型，也练上了 W1 刚学的 record。
3. **静态工厂方法**：`success`/`error` 统一了响应构造，调用干净。
4. **`/api/ping` 作为 `ApiResponse` 第一个使用者**：一次验证了链路 + 响应包装设计。
5. **业务码与 HTTP 码解耦**：选业务码派（0/1000），方向正确，HTTP 码留给 W3 全局异常处理器。
6. **遇到 "Please sign in" 能定位到 Security 默认配置**，并用临时 `permitAll()` 放行过渡，没有粗暴删依赖。
7. **YAGNI 克制**：prod profile 暂不硬编假配置、code 暂不抽枚举、不加 timestamp，都留到真正需要时。
