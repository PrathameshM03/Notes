# Spring Framework Full Course — Notes 5: Cross-Cutting Concerns — Filters, Interceptors & AOP
### Covers Lectures 19–25: Servlet Filters → Filters part 2 (request/response wrapping) → Spring MVC Interceptors → AOP fundamentals → Advice types → Pointcut expressions → Custom annotations & AOP internals

---

## Lecture 19 — Filters in Spring Boot

### 1. The problem: cross-cutting concerns
Things like logging, authentication checks, and request/response manipulation are needed across *many* endpoints, not just one. Putting this logic inside every controller method causes:
1. The same infrastructure logic repeated across multiple APIs.
2. Controller methods become harder to read.
3. Developers forget to add the logic to newly created APIs.
4. Controllers stop focusing on their real responsibility (handling application logic).

> A **cross-cutting concern** is a piece of functionality needed across many unrelated parts of the application (logging, security, auditing, metrics) — it "cuts across" the normal layered structure instead of belonging to one specific layer.

### 2. What is a Filter?
> A **Filter** is a checkpoint that an HTTP request and response pass through, able to run logic **before** the request continues down the chain and **after** downstream processing returns.

Filters come from the **Servlet API** (`jakarta.servlet.Filter` in Spring Boot 3 / Servlet API namespace change from `javax.servlet` in Boot 2 to `jakarta.servlet` in Boot 3 — an important detail when following older tutorials). Because Filters sit at the raw servlet layer, they operate on `ServletRequest`/`ServletResponse` (not Spring-specific types), and they run **before the request even reaches `DispatcherServlet`**.

### 3. Where filters sit in the flow
```
Client → Tomcat → [Filter 1 → Filter 2 → ... ] → DispatcherServlet → Controller
```

### 4. Creating a custom Filter
```java
@Component
@Order(1)
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(request, response); // pass control onward
        } finally {
            long duration = System.currentTimeMillis() - start;
            System.out.println("Request took " + duration + "ms");
        }
    }
}
```
- `doFilter(request, response, chain)` is the core method every Filter implements.
- **`FilterChain.doFilter(request, response)`** is what actually passes control to the *next* filter (or, if this is the last one, to `DispatcherServlet`). **If you don't call it, the request stops here** — this is how a Filter can block a request (e.g. return a 401/403 JSON response directly without ever reaching the controller).
- Wrapping the call in `try { chain.doFilter(...) } finally { ... }` is the standard pattern for **timing/cleanup logic that must run regardless of success or failure downstream**.

### 5. Execution order (mental model)
```
Filter code before chain.doFilter()   →  (runs on the way IN)
   → next filter / DispatcherServlet / controller
Filter code after chain.doFilter()    →  (runs on the way OUT, in reverse order)
```
This is conceptually the same "wrap around" shape as a `try`/`finally` block — code before `proceed()`, code after.

### 6. Ordering multiple filters
```java
@Component
@Order(1)  // lower number = runs first
public class AuthenticationFilter implements Filter { ... }

@Component
@Order(2)
public class LoggingFilter implements Filter { ... }
```

### Common use cases of Filters
- Request/response logging.
- Blocking unauthorized requests early (before they reach any controller).
- Adding common response headers (CORS, security headers).
- Character encoding.

### Key takeaways
- Filters live at the **Servlet layer**, below Spring MVC — they see every request, regardless of whether it maps to a controller.
- `chain.doFilter()` is the point of no return control-wise: skip it and the request stops there.
- Filters are the right layer for concerns that don't need to know *which controller method* will handle the request.

---

## Lecture 20 — Filters Part 2: Wrapping Requests/Responses & Registration

### 1. Modifying the response
By default you can set status/headers on `HttpServletResponse` before it's "committed" (i.e. before the body starts being written), but you **cannot read the already-written response body** through the plain response object — Tomcat streams output; it isn't buffered for you to inspect afterward.

**`ContentCachingResponseWrapper`** — wraps the response so its body is *cached in memory*, letting a Filter inspect (or even modify) the body after downstream processing, as long as you remember to call **`copyBodyToResponse()`** at the end — otherwise the actual client never receives the body (it stays trapped in the wrapper's buffer).

### 2. Modifying the request
The raw `HttpServletRequest` provides no setters for headers or query parameters — it's designed to be read-only from the servlet's perspective. To let downstream code see *different* values than what the client actually sent, you wrap the request in a custom subclass (a **Request Wrapper**) that overrides the relevant getter methods (e.g. override `getParameter()` to inject/modify a value).

### 3. Why the request body normally can't be re-read
An `HttpServletRequest`'s input stream can typically be **read only once** — after `getInputStream()` (or an equivalent) has been consumed once, downstream code can't read it again. Reading the body in a Filter can therefore accidentally starve the controller of the body it needs. **Fix:** cache/buffer the body in a wrapper that overrides `getInputStream()` to return a re-readable copy.

> These wrapper techniques (buffering request/response bodies) have real costs: memory usage, performance overhead, and potential privacy concerns (caching sensitive payloads) — use them narrowly, only where genuinely needed.

### 4. Registering Filters in Spring Boot
Two main approaches:
- **`@Component` on the Filter class** — auto-registered by Spring Boot, applied to *all* URLs by default.
- **`FilterRegistrationBean`** — explicit registration giving you control over URL patterns and order:
```java
@Bean
public FilterRegistrationBean<LoggingFilter> loggingFilter() {
    FilterRegistrationBean<LoggingFilter> bean = new FilterRegistrationBean<>(new LoggingFilter());
    bean.addUrlPatterns("/api/*");
    bean.setOrder(1);
    return bean;
}
```
**Avoid combining** `@Component` registration *and* manual `FilterRegistrationBean` registration for the **same** filter class — it can cause the filter to run twice.

### 5. `OncePerRequestFilter`
A convenient Spring-provided base class that guarantees the filter runs **exactly once per request**, even in scenarios (like internal forwards/includes) where a raw `Filter` might otherwise be invoked multiple times for one logical request. It also exposes a clean **`shouldNotFilter(request)`** method to exclude specific URLs without manually branching inside `doFilter`:
```java
@Component
public class AuthFilter extends OncePerRequestFilter {
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return request.getRequestURI().startsWith("/public/");
    }
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        chain.doFilter(req, res);
    }
}
```
`OncePerRequestFilter` is generally more convenient for Spring Boot applications than implementing the raw `Filter` interface directly.

### Key takeaways
- A Filter can stop a request simply by not calling `chain.doFilter()`.
- Headers must be added **before** the response is committed.
- `ContentCachingResponseWrapper` lets you inspect a response body after the fact — remember `copyBodyToResponse()`.
- Request bodies are single-read streams by default — re-reading requires a caching wrapper.
- Prefer `OncePerRequestFilter` + `shouldNotFilter()` for clean, Spring-idiomatic filter code.
- Use `FilterRegistrationBean` when you need explicit URL-pattern/order control; skip mixing it with `@Component` on the same class.

---

## Lecture 21 — Spring MVC Interceptors

### 1. The problem Interceptors solve
Similar motivation to Filters — avoid cluttering controllers and duplicating cross-cutting logic — but Interceptors operate **inside Spring MVC**, *after* the `DispatcherServlet` has identified which controller/handler will process the request, so an Interceptor **knows the target controller method** (Filters generally do not).

### 2. Where Interceptors fit
```
DispatcherServlet receives request
   → HandlerMapping identifies the matching handler
   → Spring builds a handler execution chain
   → registered Interceptors execute around the handler
   → HandlerAdapter invokes the controller method
   → Spring processes the return value
   → response sent to client
```

### 3. The three interceptor lifecycle methods (`HandlerInterceptor`)
| Method | Runs | Typical use |
|---|---|---|
| `preHandle(request, response, handler)` | **Before** the controller method | Auth checks, logging start time, blocking a request (return `false` to stop the chain) |
| `postHandle(request, response, handler, modelAndView)` | **After** the controller method, **before** the view is rendered | Adding common model attributes (traditional MVC), post-processing |
| `afterCompletion(request, response, handler, ex)` | **After** the complete request lifecycle, including view rendering | Cleanup, logging total time, logging exceptions |

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("startTime", System.currentTimeMillis());
        return true; // false would block the request here
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
        long duration = System.currentTimeMillis() - (long) req.getAttribute("startTime");
        System.out.println("Handled in " + duration + "ms");
    }
}
```
**Registering** it (via `WebMvcConfigurer`):
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    private final LoggingInterceptor loggingInterceptor;
    public WebConfig(LoggingInterceptor loggingInterceptor) { this.loggingInterceptor = loggingInterceptor; }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```
`addPathPatterns` / `excludePathPatterns` scope which URLs the interceptor applies to. Multiple interceptors run in the order they're registered (their `preHandle`s run in registration order; `postHandle`/`afterCompletion` run in reverse).

### 4. Filter vs Interceptor — direct comparison
| Aspect | Filter | Interceptor |
|---|---|---|
| Layer | Servlet layer (below Spring MVC) | Spring MVC layer |
| Operates on | Raw `ServletRequest`/`ServletResponse` | The selected MVC handler |
| Knows the controller method? | Normally no | Yes, via `HandlerMethod` |
| Before-callback | Code before `chain.doFilter()` | `preHandle()` |
| After-callback | Code after `chain.doFilter()` | `postHandle()` + `afterCompletion()` |
| Can wrap request/response bodies | Yes | Not in the same chain-wide manner |
| Can block processing | Yes (skip `chain.doFilter()`) | Yes (`preHandle()` returns `false`) |

**When to use a Filter:** request/response body wrapping & caching, character encoding, compression, CORS, logging *all* servlet traffic (including static resources), security processing that must happen before Spring MVC even runs.

**When to use an Interceptor:** logging controller/method names specifically, reading custom annotations on controller methods, MVC-specific timing/metrics, adding common model attributes for traditional (view-based) MVC, applying tenant/organization context based on which handler matched.

### Final mental model
```
Filter    → guards/observes the whole servlet pipeline, controller-agnostic
Interceptor → guards/observes specifically around Spring MVC's chosen handler, controller-aware
```
Both can coexist in the same application; each is the right tool for a different layer of concern.

---

## Lecture 22 — Introduction to Spring AOP

### 1. Business logic vs supporting logic
A "simple" business method (e.g. `createStudent()`) often accumulates *supporting* logic that isn't really about the business rule itself: logging, timing, transaction management, security checks, auditing, exception translation. This supporting logic is a **concern**; when the same concern needs to be applied consistently *across many otherwise-unrelated methods*, it's a **cross-cutting concern** — the same category discussed for Filters/Interceptors, but now at the level of individual **methods**, not just HTTP requests.

### 2. Two core problems: Scattering and Tangling
- **Scattering** — the same logging/timing/security code is duplicated (scattered) across many methods.
- **Tangling** — a single method's code mixes business logic *and* supporting logic together, making the business intent harder to see.

Consequences: harder to see the real business intent, inconsistent application of supporting behavior, more expensive changes, developers can forget to add mandatory behavior to new methods, harder testing and maintenance.

### 3. Earlier (inadequate) attempts to solve this
1. **Utility methods** — reduces duplication of *how* logging happens, but you still have to remember to *call* the utility everywhere — scattering persists.
2. **Inheritance** — put shared logic in a base class — inflexible, forces an artificial class hierarchy, doesn't compose well with multiple unrelated concerns.
3. **Manual wrappers** — write a wrapper method/class around the target that adds behavior before/after calling it — works, but requires **manually wrapping every single method you care about**, and the wrapping code itself is now scattered across wrapper classes.

### 4. Aspect-Oriented Programming (AOP)
> AOP lets you define a cross-cutting concern **once**, in one place, and have it applied **automatically** to every method that matches a declarative rule — without manually editing or wrapping each target method.

**OOP and AOP solve different dimensions:** OOP organizes code around **objects/domain entities** (a `Student`, an `Order`). AOP organizes code around **concerns that cut across multiple objects** (logging, security, transactions) — the two are complementary, not competing.

### 5. The three fundamental questions AOP answers
1. **What** should run (the extra behavior)? → the **Advice**.
2. **Where/when** should it run (which methods, and at what point relative to them)? → the **Pointcut** (which methods) + **Advice type** (before/after/around, etc.).
3. **How** does it actually get woven in without touching the original code? → a **Proxy**, generated by Spring at runtime, that wraps the real target object.

### 6. Key AOP vocabulary
| Term | Meaning |
|---|---|
| **Aspect** | A module encapsulating a cross-cutting concern (a class annotated `@Aspect`) |
| **Advice** | The actual code that runs (the "what") — `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around` |
| **Join point** | A point in program execution where advice *could* apply (in Spring AOP, effectively: a method execution) |
| **Pointcut** | An expression that selects *which* join points (methods) the advice applies to |
| **Target object** | The real object being advised |
| **Proxy** | The object Spring actually gives you instead of the raw target — it wraps the target and applies advice around calls to it |
| **Weaving** | The process of linking aspects into the target's execution — Spring AOP does this via runtime proxying |

### Manual wrapper vs Spring AOP proxy
A manual wrapper is code **you** write and maintain per-target. A Spring AOP proxy is **generated automatically** by the framework based on your pointcut expression — write the concern once as an aspect, and Spring applies it to every currently- and future-matching method without you touching those methods at all.

### Key takeaways
- AOP exists specifically to solve **scattering** and **tangling** of cross-cutting concerns.
- The three AOP questions: *what* (advice), *where* (pointcut), *how* (proxy).
- Spring AOP is proxy-based — this has real implications covered in Lecture 24/25 (self-invocation, only public method interception, etc.).

---

## Lecture 23 — Implementing Spring AOP: Advice Types

### 1. Setup
Add `spring-boot-starter-aop`. Recommended package structure separates `aspect/` from `service/`/`controller/`.

### 2. A target bean and an aspect
```java
@Service
public class StudentService {
    public Student createStudent(Student s) { ... }
}
```
```java
@Component
@Aspect
public class LoggingAspect {
    @Before("execution(* in.strikes.aopDemo.service.StudentService.createStudent(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Student is going to be saved");
    }
}
```
- `@Aspect` marks the class as containing advice.
- `@Component` is still needed — the aspect itself must be a Spring-managed bean.
- The String inside `@Before(...)` is a **pointcut expression** (deep dive in Lecture 24): `execution(* <fully.qualified.Class>.method(..))` matches a specific method execution (`*` = any return type, `..` = any arguments).

### 3. Target object vs Proxy object
Spring does **not** give the caller a reference to the raw `StudentService` object — it gives a **proxy** that wraps it. When you call a method on the injected `StudentService`, you're really calling the proxy, which runs the matched advice around a call to the real target.

### 4. Advice types — the `try/catch/finally` mental model
Think of AOP advice types as different points in this structure around the target method call:
```
@Before        → runs right before the call
try {
    result = targetMethod();
    @AfterReturning → runs after a normal (non-exception) return, has access to the result
}
catch (Exception e) {
    @AfterThrowing   → runs only if an exception was thrown
}
finally {
    @After           → runs regardless of outcome (success or exception)
}
@Around            → wraps the ENTIRE try/catch/finally block — most powerful, full control
```

**`@Before`**
```java
@Before("execution(* ...createStudent(..))")
public void logBefore(JoinPoint joinPoint) {
    Object[] args = joinPoint.getArgs();
    System.out.println("About to save: " + args[0]);
}
```
- Can inspect arguments via `JoinPoint`, but **cannot prevent** the target method from running just by "not proceeding" (there's no `proceed()` in `@Before`) — to actually block execution you'd need to throw an exception from within the advice.
- Cannot directly replace the method's arguments (it can only observe them here, though real argument mutation for mutable objects is possible by mutating the object itself, not by reassigning the reference).

**`@AfterReturning`**
```java
@AfterReturning(value = "execution(* ...createStudent(..))", returning = "result")
public void logAfterReturning(Student result) {
    System.out.println("Intercepted createStudent(), result: " + result);
    // result.setName("Rohit"); -- can mutate the OBJECT (if mutable), but reassigning the
    // reference itself does not change what the caller receives (return value is fixed by then)
}
```
Runs only on **normal completion** (method returned without throwing) — "normal completion" specifically means the method exited via `return`, not via an exception.

**`@AfterThrowing`**
```java
@AfterThrowing(value = "execution(* ...createStudent(..))", throwing = "exception")
public void logAfterThrowing(RuntimeException exception) {
    System.out.println("Exception type: " + exception.getClass().getName());
    System.out.println("Exception message: " + exception.getMessage());
}
```
- Runs **only** when the target method throws — can be scoped to a specific exception type via the parameter type.
- **Does not "handle" or swallow the exception** — the exception still propagates after this advice runs (this advice type cannot return a fallback value).
- Good for: logging failures, alerting, auditing errors — not for actually recovering from them.

**`@After`** — runs regardless of outcome (like `finally`), no access to the return value or exception. Good for: unconditional cleanup/logging.

**`@Around`** — the most powerful advice; wraps the entire invocation:
```java
@Around("execution(* in.strikes.aopDemo.service.StudentService.dummyMethod(..))")
public Object logAroundMethod(ProceedingJoinPoint joinPoint) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        Object result = joinPoint.proceed();  // actually invokes the target method
        System.out.println("Execution Successful");
        return result;   // MUST return this (or a replacement) — the advice IS the new call
    } catch (Exception e) {
        System.out.println("Execution Failed: " + e.getMessage());
        throw e;
    } finally {
        System.out.println("Took " + (System.currentTimeMillis() - start) + "ms");
    }
}
```
- **`ProceedingJoinPoint`** — the only join-point type that exposes `proceed()`, letting `@Around` advice *choose whether and when* to actually invoke the target method.
- **If `proceed()` is never called**, the target method never runs at all — `@Around` can fully replace the target's behavior.
- **Must return the result of `proceed()`** (or a substitute) — because `@Around` advice's return value *becomes the method call's return value* to the original caller; forgetting to `return` effectively swallows the result.
- Return type is typically `Object` because the advice is generic across many possible target method signatures.
- Can inspect/replace arguments before calling `proceed(modifiedArgs)`, catch and convert an exception into a fallback return value, or even call `proceed()` more than once (e.g. simple retry logic) — real example seen in the course's own demo code:
```java
@Around("execution(* in.strikes.aopDemo.service.StudentService.dummyMethod(..))")
public Object logAroundMethod(ProceedingJoinPoint joinPoint) throws Throwable {
    Object return1 = joinPoint.proceed();
    System.out.println("Intercepted request calling again");
    Object return2 = joinPoint.proceed(); // calls the target a SECOND time
    return return2;
}
```

### Advice comparison at a glance
| Advice | Runs when | Can block target? | Can see result? | Can see exception? | Can change return value? |
|---|---|---|---|---|---|
| `@Before` | Before call | No (only by throwing) | No | No | No |
| `@AfterReturning` | After normal return | N/A | Yes | No | Not the reference itself |
| `@AfterThrowing` | After exception | N/A | No | Yes | No (doesn't swallow) |
| `@After` | Always (finally-like) | N/A | No | No | No |
| `@Around` | Wraps everything | Yes (skip `proceed()`) | Yes | Yes | Yes |

**Guideline:** prefer the **narrowest** advice type that expresses your actual requirement — reach for `@Around` only when you truly need full control (timing + conditional execution + result transformation); simpler concerns (pure logging, pure cleanup) read more clearly with `@Before`/`@After`/`@AfterReturning`/`@AfterThrowing`.

---

## Lecture 24 — Pointcut Expressions

### 1. What is a Pointcut Expression?
The String passed to `@Before`/`@Around`/etc. (or a named `@Pointcut`) that **decides which join points (method executions) the advice applies to**.

### 2. The `execution()` designator — most common
```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? method-name-pattern(param-pattern) throws-pattern?)
```
```java
execution(* in.strikes.service.StudentService.createStudent(..))
```
- `*` (first position) — matches **any return type**.
- `..` (inside parentheses) — matches **any number/type of arguments**.
- `*` can also wildcard parts of a name: `execution(* in.strikes.service.*.get*(..))` matches any method starting with `get` on any class in `in.strikes.service`.
- `..` in a package position matches the package **and all sub-packages**: `execution(* in.strikes..*.create*(..))`.

### 3. `within()` — matches by type/package, not method signature
```java
within(in.strikes.service.*)        // any method in any class directly in this package
within(in.strikes.service..*)       // any method in this package AND sub-packages
```
Use `within()` when the requirement applies to a whole class/package/architectural layer, rather than depending on a specific method name/signature.

### 4. `@annotation()` — matches methods carrying a specific annotation
```java
@annotation(in.strikes.annotation.LogExecutionTime)
```
Matches any method (regardless of its class or package) that is annotated with `@LogExecutionTime` — this is the mechanism behind **custom annotation-driven AOP**, covered fully in Lecture 25.

### 5. `bean()` — matches by Spring bean name
```java
bean(studentService)     // matches methods on the bean named "studentService"
bean(*ServiceImpl)       // wildcard bean-name matching
```

### 6. Combining pointcut expressions
Standard boolean operators combine expressions:
```java
execution(* in.strikes.service.*.*(..)) && @annotation(LogExecutionTime)
```

### 7. Named pointcuts — `@Pointcut`
Avoid repeating long expressions across multiple advice methods by centralizing them:
```java
@Aspect
@Component
public class CommonPointcuts {
    @Pointcut("execution(* in.strikes.service..*(..))")
    public void serviceLayer() {}
}

@Aspect
@Component
public class LoggingAspect {
    @Before("in.strikes.aspect.CommonPointcuts.serviceLayer()")
    public void logBefore() { ... }
}
```

### 8. JDK Dynamic Proxy vs CGLIB Proxy
Spring AOP creates proxies one of two ways:
- **JDK Dynamic Proxy** — used when the target class implements at least one interface; the proxy implements that same interface.
- **CGLIB Proxy** — used when the target class has **no interface** (or is explicitly configured); Spring generates a runtime subclass of the target class instead.

This choice happens automatically, but it's configurable (`proxyTargetClass=true` forces CGLIB even when an interface exists). **Important boundary either way:** Spring AOP only intercepts calls that go **through the proxy** — a method calling another method **on `this`** inside the *same* class bypasses the proxy entirely (self-invocation problem — advice on the second method won't fire). Also, Spring AOP typically only advises **public** methods called from outside the class.

### Practical guidelines
1. Prefer pointcuts that represent clear architectural boundaries (`within(...service..*)`) over overly specific method-by-method matching.
2. Use `execution()` when matching truly depends on method name/return type/parameters.
3. Use `within()` when the concern applies to a whole class/package/layer.
4. Use `@annotation()` when methods should **explicitly opt in** to the aspect (very common for things like custom `@LogExecutionTime`, `@Auditable`, `@RateLimited`).
5. Use named pointcuts (`@Pointcut`) instead of repeating long expressions.
6. Keep pointcuts narrow enough to actually understand and test — overly broad pointcuts make debugging "why did this advice run" much harder.
7. Remember Spring AOP works through **proxies** — self-invocation and non-public methods are blind spots.

---

## Lecture 25 — Custom Annotations, AOP Internals & `BeanPostProcessor`

### 1. Separating "what" from "how"
Two example concerns — generating a report, measuring execution time — both illustrate the same idea: **the annotation declares intent ("this method needs X"), the aspect implements the mechanism.** A custom annotation by itself **does nothing** — Spring only acts on it because an aspect's pointcut is written to detect it, read its data, and perform the actual behavior.

### 2. Creating a custom annotation
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface LogExecutionTime { }
```
- **`@Target`** — where the annotation is legal to place (method, type, field, etc.).
- **`@Retention(RUNTIME)`** — **critical for AOP**: the annotation must still be present at runtime (not just discarded after compilation) for Spring to read it via reflection when deciding whether a pointcut matches.
- **`@Documented`** — cosmetic; includes the annotation in generated Javadoc.

A **marker annotation** carries no data (like the example above); a **configured annotation** carries attributes:
```java
public @interface LogExecutionTime {
    String value() default "";
}
```

### 3. The `@annotation()` pointcut, in depth
```java
@Around("@annotation(logExecutionTime)")
public Object measure(ProceedingJoinPoint joinPoint, LogExecutionTime logExecutionTime) throws Throwable {
    long start = System.currentTimeMillis();
    Object result = joinPoint.proceed();
    System.out.println("Took " + (System.currentTimeMillis() - start) + "ms, label=" + logExecutionTime.value());
    return result;
}
```
Two things happen here: (1) **type matching** — the pointcut matches any method carrying the annotation; (2) **annotation binding** — the actual annotation instance is passed into the advice method as a parameter (its name must match the pointcut's binding variable), letting the advice read annotation attributes (like `value()`).

### 4. Where AOP fits into the bean lifecycle
Recall the bean lifecycle from Lecture 7. AOP proxies are created via **`BeanPostProcessor`** — a Spring extension point that runs *around* the initialization step:
```
Read bean definition → Instantiate bean → Inject dependencies → Run Aware callbacks
   → Run BeanPostProcessors BEFORE initialization
   → Run initialization callbacks (@PostConstruct, etc.)
   → Run BeanPostProcessors AFTER initialization   ← Spring AOP wraps the bean in a proxy HERE
   → Store and expose the final (proxied) bean reference
```

> **`BeanPostProcessor`** is a Spring interface that lets you hook into **every** bean's creation, both just before and just after initialization, and optionally **return a different object** than the one that was passed in — Spring AOP's proxy machinery is itself implemented as a `BeanPostProcessor` that detects matching beans and substitutes a proxy for the raw object.

### 5. How Spring decides which beans need proxying
At startup, Spring checks each candidate bean's methods against **all registered pointcut expressions** across all `@Aspect` beans. If any method matches, that bean is wrapped in a proxy (JDK or CGLIB, per Lecture 24); if nothing matches, the bean is left as-is — no unnecessary proxying overhead.

### 6. The method interceptor chain & `proceed()`
When multiple aspects/advices match the *same* method, Spring builds an internal **chain of interceptors**. Calling `proceed()` inside `@Around` advice doesn't necessarily call the real target directly — it calls the **next interceptor in the chain**, which may be another piece of advice, until finally the real target method is invoked at the end of the chain. This is why multiple `@Around` advices on the same method compose correctly (each one's `proceed()` triggers the next).

### 7. Controlling aspect execution order
```java
@Aspect
@Order(1)   // lower value = higher precedence, runs "outermost"
@Component
public class SecurityAspect { ... }

@Aspect
@Order(2)
@Component
public class LoggingAspect { ... }
```

### Important proxy-based AOP boundaries (recap + consolidation)
- Only **proxy-visible** calls are advised — calls made *from outside* the bean, through the injected reference.
- **Self-invocation** (a method calling another method on `this` within the same class) bypasses the proxy — advice on the second method silently does not fire.
- Only **public** methods are advised by default (private/protected calls aren't visible to the proxy).
- Spring AOP proxies only intercept **Spring-managed beans** — plain `new SomeClass()` objects are never proxied.

### Complete mental model
```
@Aspect class (marked @Component)
   ├── @Pointcut expressions define WHICH methods
   ├── @Before / @After / @AfterReturning / @AfterThrowing / @Around define WHAT runs and WHEN
   └── Spring's BeanPostProcessor machinery detects matches at bean-creation time
         and substitutes a PROXY for the real bean
Caller → Proxy → [interceptor chain: matched advices, in @Order] → real target method
```

### Key takeaways
- A custom annotation is inert on its own — it only "does something" because an aspect's pointcut is written to look for it.
- `@Retention(RUNTIME)` is mandatory for any annotation an AOP pointcut needs to detect.
- AOP proxies are wired in via `BeanPostProcessor`, right after normal bean initialization.
- Multiple matching advices form an interceptor chain; `proceed()` moves to the next link, ending at the real method.
- `@Order` controls precedence when multiple aspects match the same join point.

---
*Next file: `06-jdbc-hibernate-jpa-transactions.md` — Raw JDBC → Spring JDBC (`JdbcTemplate`) → Hibernate internals & persistence context → JPA relationships & cascading/fetching → Spring Data JPA → Transactions & propagation.*
