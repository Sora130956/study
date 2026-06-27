# Architecture（Servlet 应用的过滤器链与核心组件）

> **来源:** https://docs.spring.io/spring-security/reference/servlet/architecture.html
> **对应计划周次:** 第 3 周 · Spring Security + JWT 注册登录

## 核心理解

Spring Security 在 Servlet 应用里是通过**一条过滤器链（FilterChainProxy / SecurityFilterChain）**工作的。每个请求进来都会依次穿过一组 Security Filter，每个 Filter 负责一件事：加载安全上下文、认证、处理异常、做授权判断等。理解架构的关键就是搞清楚**这些 Filter 的顺序**——因为顺序决定了「某个能力在哪一步才可用」。

两个最该记牢的角色：
- **SecurityContext / SecurityContextHolder**：存「当前是谁在访问」的容器，默认基于 ThreadLocal，同一请求线程内任何地方都能拿到当前用户，请求结束时被清空。
- **ExceptionTranslationFilter**：安全异常的「翻译官」，用 try-catch 接住后续过滤链抛出的 `AuthenticationException`（没登录）和 `AccessDeniedException`（没权限），分别翻译成「引导登录（401 / 登录页）」和「拒绝访问（403）」。

在 Mini CRM 里，这套架构正是后面接入 JWT 登录的地盘：自定义的 JWT 过滤器要插在认证 Filter 的位置（认证之后才能拿到当前用户去做多租户/权限判断），而 401/403 的统一响应则由 ExceptionTranslationFilter 这一层决定。

## 关键点

### 自定义 Filter 的插入位置：先想清楚「需要哪些事情已经发生」

> Consider which events you need to have happened in order to locate your filter.
> For example, let's say that you want to add a Filter that gets a tenant id header and check if the current user has access to that tenant. Since we need to know the current user, we need to add it **after the authentication filters**.

判断自定义 Filter 该放哪，经验法则是：**先列出这个 Filter 依赖「哪些前置动作必须已经完成」**。

文档给的例子和 Mini CRM 高度相关：要加一个 Filter，从请求头读 `tenant id`，再检查当前用户是否有权访问该租户。因为它需要「知道当前用户是谁」，而当前用户是认证 Filter 才填进 SecurityContext 的，所以这个 Filter 必须放在**认证 Filter 之后**。

```java
// Mini CRM 多租户校验过滤器：必须放在认证之后
http.addFilterAfter(new TenantAccessFilter(), AuthorizationFilter.class);
// 因为 TenantAccessFilter 要用 SecurityContextHolder.getContext().getAuthentication()
// 拿到当前用户，认证没完成前拿到的是 null/匿名
```

### SecurityContext 是什么：存「当前是谁」的容器

> The SecurityContextHolder is cleared out.（请求结束 / 启动认证时会被清空）

三层嵌套结构，记住这个包裹关系：

```
SecurityContextHolder   ← 持有者（静态工具类，全局访问入口，默认基于 ThreadLocal）
      └── SecurityContext   ← 上下文容器（很薄，里面就装一个 Authentication）
              └── Authentication   ← 认证对象 = 当前用户身份
                      ├── Principal     → 用户主体（通常是 UserDetails，"你是谁"）
                      ├── Credentials   → 凭证（如密码，认证后通常被擦除为 null）
                      ├── Authorities   → 权限/角色列表（"你能干啥"，如 ROLE_ADMIN）
                      └── authenticated → 是否已认证（boolean）
```

代码里怎么拿当前用户：

```java
// 1. Holder → Context → Authentication
Authentication auth = SecurityContextHolder.getContext().getAuthentication();

String username = auth.getName();                 // 用户名
Object principal = auth.getPrincipal();           // 用户详情对象（UserDetails）
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities(); // 权限列表
boolean isAuthenticated = auth.isAuthenticated(); // 是否已登录
```

**Authentication 在两个阶段含义不同：**

| 阶段 | Authentication 的含义 |
|------|----------------------|
| 认证前 | 装着用户提交的凭证（用户名+密码），用于「请求认证」 |
| 认证后 | 装着已验证的身份 + 权限，代表「已登录用户」 |

**为什么用 ThreadLocal？** 一个 HTTP 请求通常由一个线程处理，把用户信息放 ThreadLocal 里，整个请求处理过程（Controller、Service）任何地方都能直接拿到当前用户，不用一层层传参；不同请求互不干扰。

三种存储策略（`SecurityContextHolder.setStrategyName(...)`）：

| 策略 | 说明 |
|------|------|
| `MODE_THREADLOCAL` | 默认，每个线程独立 |
| `MODE_INHERITABLETHREADLOCAL` | 子线程继承父线程的 Context（开异步线程时用） |
| `MODE_GLOBAL` | 全局共享（一般只在桌面应用用，Web 不用） |

### SecurityContext 的生命周期（结合过滤器链）

```
请求进来
   ↓
① SecurityContextHolderFilter
    从 Session/请求中加载已保存的 SecurityContext，放进 SecurityContextHolder（ThreadLocal）
   ↓
② 认证 Filter（如登录请求）验证身份成功后
    创建新的 Authentication，setAuthentication() 存进 SecurityContext
   ↓
③ 业务代码（Controller/Service）
    随时通过 SecurityContextHolder.getContext() 拿当前用户
   ↓
④ 请求结束（finally 块）
    SecurityContextHolder.clearContext() 清空 ThreadLocal（避免线程污染）
```

### ExceptionTranslationFilter：安全异常的「翻译官」

> The ExceptionTranslationFilter allows translation of `AccessDeniedException` and `AuthenticationException` into HTTP responses.
> ExceptionTranslationFilter is inserted into the FilterChainProxy as one of the Security Filters.

它处理两种异常，核心区分是「有没有登录」vs「登录了但有没有权限」：

| 异常 | 含义 | 通俗理解 |
|------|------|---------|
| `AuthenticationException` | 认证异常 | "你是谁？我不认识你"（没登录 / 登录失败） |
| `AccessDeniedException` | 授权异常 | "我知道你是谁，但你没权限干这事"（登录了但权限不够） |

**工作原理：它自己不抛异常、也不主动检查权限，而是先放行再用 try-catch 兜底：**

```java
// 伪代码，体现核心思想
try {
    filterChain.doFilter(request, response);   // 先放行，让后面的过滤器/业务执行
} catch (AuthenticationException ex) {
    startAuthentication(...);                  // 没登录 → 启动认证流程
} catch (AccessDeniedException ex) {
    accessDeniedHandler.handle(...);           // 没权限 → 拒绝访问
}
```

正因为靠 try-catch 兜后续异常，它必须放在**授权 Filter（AuthorizationFilter）之前**——只有它先执行，才能接住后面授权 Filter 冒泡上来的异常。

**① 认证异常 → 启动认证（Start Authentication），三个动作：**
1. **清空 SecurityContextHolder** —— 把残留的、无效的认证信息清掉，保证干净状态。
2. **保存当前 HttpServletRequest**（用 `RequestCache`）—— 登录成功后能「重放」原始请求，自动跳回用户原本想访问的页面（如访问 `/admin` 被拦去登录，登录后直接回 `/admin` 而非首页）。
3. **调用 `AuthenticationEntryPoint`** 向客户端「要凭证」—— 表单登录场景重定向到登录页；HTTP Basic 场景返回 `401 + WWW-Authenticate` 响应头。

**② 授权异常 → 拒绝访问（Access Denied）：**
- 调用 `AccessDeniedHandler` 处理，默认返回 **HTTP 403 Forbidden**。

完整流程：

```
请求进来
   ↓
ExceptionTranslationFilter（doFilter 先放行）
   ↓
后面的过滤器 / 业务逻辑执行
   ├── 一切正常 → 返回正常响应
   ├── 抛 AuthenticationException（没登录/登录失败）
   │      → 1.清空 SecurityContext  2.保存原始请求  3.EntryPoint → 跳登录页/返回 401
   └── 抛 AccessDeniedException（登录了但没权限）
          → AccessDeniedHandler → 返回 403
```

> Mini CRM 用 JWT 时通常关闭 Session 重放那套（无状态），EntryPoint 直接返回 JSON 格式的 401，AccessDeniedHandler 返回 JSON 格式的 403，而不是重定向到登录页。

### 不把未认证请求存进 Session（先了解，按需启用）

> There are a number of reasons you may want to not store the user's unauthenticated request in the session. You may want to offload that storage onto the user's browser or store it in a database. Or you may want to shut off this feature since you always want to redirect the user to the home page instead of the page they tried to visit before login.

「登录后跳回原页面」这个能力依赖把未认证请求存进 Session。可以选择不存：放到浏览器、放数据库，或干脆关掉（登录后一律跳首页）。Mini CRM 走无状态 JWT，这块基本用不上，**先跳过，需要时回读**。

### 调试 40x：打开 DEBUG 日志

> Spring Security provides comprehensive logging of all security related events at the DEBUG and TRACE level... for security measures Spring Security does not add any detail of why a request has been rejected to the response body. If you come across a 401 or 403 error, it is very likely that you will find a log message that will help you understand what is going on.

**重要经验：** 出于安全考虑，Spring Security **不会**在响应体里写明请求为何被拒。遇到 401/403 排查不出原因时，打开 DEBUG/TRACE 级别日志，几乎一定能在日志里找到被拒的真正原因。

```yaml
# application.yml —— 调试 Mini CRM 的 401/403 时临时开启
logging:
  level:
    org.springframework.security: DEBUG
```

## 术语表

| 英文 | 词性 | 释义 |
|------|------|------|
| FilterChainProxy | n. | Spring Security 的核心过滤器代理，串联所有 Security Filter |
| SecurityFilterChain | n. | 一条匹配特定请求的安全过滤器链 |
| SecurityContextHolder | n. | 安全上下文持有者，静态工具类，默认基于 ThreadLocal，全局访问入口 |
| SecurityContext | n. | 安全上下文容器，内部仅持有一个 Authentication |
| Authentication | n. | 认证对象，代表当前用户身份（含 Principal/Credentials/Authorities） |
| Principal | n. | 主体，「用户是谁」，通常是 UserDetails 对象 |
| Credentials | n. | 凭证，如密码，认证后通常被擦除为 null |
| GrantedAuthority | n. | 授予的权限/角色（如 ROLE_ADMIN） |
| SecurityContextHolderFilter | n. | 从 Session/请求加载已保存 SecurityContext 的过滤器 |
| ExceptionTranslationFilter | n. | 异常翻译过滤器，将安全异常翻译成 HTTP 响应 |
| AuthenticationException | n. | 认证异常，表示未登录/登录失败 |
| AccessDeniedException | n. | 授权异常，表示已登录但权限不足 |
| AuthenticationEntryPoint | n. | 认证入口点，引导客户端去认证（跳登录页 / 返回 401） |
| AccessDeniedHandler | n. | 拒绝访问处理器，处理权限不足（默认返回 403） |
| RequestCache | n. | 请求缓存，保存原始请求以便登录后重放 |
| AuthorizationFilter | n. | 授权过滤器，做访问控制判断，位于 ExceptionTranslationFilter 之后 |
| ThreadLocal | n. | 线程本地存储，使同一请求线程内共享数据、不同请求互不干扰 |
