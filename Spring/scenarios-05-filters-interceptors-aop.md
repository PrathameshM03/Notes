# Scenarios & Deeper Reasoning — 5: Cross-Cutting Concerns, When They Cross Each Other
### Companion to `05-filters-interceptors-aop.md`

---

## Story 1: One bug, three symptoms — self-invocation strikes again

**The situation:** Doc 2 introduced the self-invocation trap for a plain `@Aspect`. It's worth seeing the *exact same underlying cause* produce three completely different-looking bugs, because interviewers often ask about each one separately without naming the common root cause — recognizing "oh, this is the proxy problem again" is the real skill.

**Symptom A — logging aspect "randomly" doesn't fire:**
```java
@Service
public class StudentService {
    public Student createStudent(Student s) {
        validateAndLog(s); // internal call — bypasses the proxy
        return studentRepository.save(s);
    }
    @Before("execution(* ..validateAndLog(..))")
    // this advice is DEFINED correctly, but never triggers when called like this
    public void validateAndLog(Student s) { ... }
}
```

**Symptom B — a `@Transactional` method doesn't roll back on failure:**
```java
@Service
public class OrderService {
    public void processOrder(Order order) {
        saveOrder(order);       // internal call
        chargeCustomer(order);  // if this throws, saveOrder's changes DON'T roll back
    }
    @Transactional
    public void saveOrder(Order order) { orderRepository.save(order); }
}
```
Here it's worse than "the advice doesn't fire" — `saveOrder` still **works** (it does save the order), because a database write doesn't strictly need a transaction wrapper to execute. What's silently missing is the **rollback guarantee** — if `chargeCustomer` fails afterward, the already-saved order **stays saved**, because there was never a real transaction boundary wrapping both calls together. This is the most dangerous version of this bug because nothing *looks* broken until a failure happens in production.

**Symptom C — an `@Async` method runs synchronously and blocks:**
```java
@Service
public class NotificationService {
    public void handleSignup(User user) {
        saveUser(user);
        sendWelcomeEmail(user); // internal call
    }
    @Async
    public void sendWelcomeEmail(User user) { /* meant to run on a separate thread */ }
}
```
The signup request now **waits for the email to actually send** before responding — defeating the entire point of marking it `@Async`, and making the API slow/flaky whenever the email provider is slow.

**The one root cause behind all three:** every one of these annotations (`@Before`/`@Around` from `@Aspect`, `@Transactional`, `@Async`) is implemented via a **proxy wrapping the bean**. A call through `this` inside the same class never touches that proxy. Recognizing "this is the same mechanism as the AOP self-invocation problem" — instead of treating each symptom as a separate mystery — is exactly the kind of pattern-matching interviewers are checking for when they ask a `@Transactional`-specific or `@Async`-specific version of this question.

**The fix, consistently, across all three:** extract the internally-called method into a **separate bean**, and call it through an injected reference (Doc 2's Fix #1) — this is the uniform, recommended solution regardless of *which* annotation is involved.

**Interview angle:** if you get asked *both* "why doesn't `@Transactional` work here" and, later in the same interview, "why doesn't `@Async` work here" — naming the shared root cause (proxy-based AOP, self-invocation bypasses the proxy) instead of re-deriving it from scratch each time is a strong signal of real understanding versus memorized answers.

---

## Story 2: The filter that ran *after* the interceptor it depended on — an ordering bug

**The situation:** A team builds two pieces of infrastructure independently:
- A **Filter** that reads a custom header and sets a "tenant ID" as a **request attribute** for multi-tenant data isolation.
- An **Interceptor** that, in `preHandle()`, reads that tenant ID attribute and uses it to check authorization ("is this tenant allowed to call this endpoint?").

```java
@Component
public class TenantFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        String tenantId = ((HttpServletRequest) req).getHeader("X-Tenant-Id");
        req.setAttribute("tenantId", tenantId);
        chain.doFilter(req, res);
    }
}

@Component
public class TenantAuthInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        String tenantId = (String) req.getAttribute("tenantId"); // expects the Filter already set this
        return isAuthorized(tenantId);
    }
}
```
In isolated testing this worked fine. In production, intermittent `401`s started appearing for legitimate requests.

**Where it breaks — and why it's confusing:** Filters and Interceptors are **different mechanisms with different ordering rules**. Filters run at the **Servlet layer**, always *before* `DispatcherServlet` even starts routing to a handler — so *by construction*, every registered Filter runs before every Interceptor, provided the Filter itself doesn't have a competing registration-order problem. The actual bug in this scenario turned out to be **a second, unrelated filter** (a compression filter, registered via `@Order` with a *lower* order value than `TenantFilter`) running *before* `TenantFilter` and, in some environments, wrapping/replacing the request object in a way that dropped the custom attribute before `TenantFilter` even got to set it.

**The real lesson:** ordering bugs between Filters (and between Filters and Interceptors) are genuinely hard to spot by reading code, because **nothing in the code visually shows execution order** — it's determined by `@Order` values, bean registration order, or `FilterRegistrationBean` configuration, none of which are visible at the call site. This is *why* the course material's emphasis on `@Order` isn't pedantic — it's the actual mechanism controlling correctness here.

**The fix — make ordering explicit and centralized, don't rely on default/implicit ordering:**
```java
@Component
@Order(1) // explicitly runs first
public class TenantFilter implements Filter { ... }

@Component
@Order(2) // explicitly runs after TenantFilter
public class CompressionFilter implements Filter { ... }
```
And, as a defensive habit: **don't pass data between a Filter and an Interceptor via request attributes when you can avoid it** — if the data can instead be derived independently in each layer (or the tenant-resolution logic centralized into one single Filter that does both the header-reading *and* the auth check, if they're always used together), there's no ordering dependency left to break.

**Interview angle:** *"How do Filters and Interceptors interact — does order matter?"* — a strong answer states the *general* rule (Filters always run before the DispatcherServlet reaches any Interceptor) but also flags that **ordering among multiple Filters, and among multiple Interceptors, is not automatic** — it's controlled by `@Order`/registration order, and cross-Filter data-passing via request attributes is a common source of hard-to-reproduce bugs precisely because the ordering isn't visible in the code.

---

## Story 3: Why AOP for logging, but Interceptors for "did this controller method get called with the right role" — choosing the right tool

**The situation:** A team needs two things: (1) log the execution time of every service-layer method, and (2) log which controller method + HTTP method + path was hit, for API usage analytics. A developer initially tries to do **both** with a single `@Aspect`.

**Where it gets awkward:** an `@Aspect` pointcut like `execution(* in.company..*Controller.*(..))` *can* technically match controller methods too — but it doesn't have easy, built-in access to the **HTTP-specific** context (the raw path, query params, response status) the way a `HandlerInterceptor` does, because AOP operates at the level of "a Java method was called," with no inherent concept of "this is happening inside an HTTP request."

**The better split (this is the actual reasoning behind "Filter vs Interceptor vs AOP," not just a memorized table):**
- **AOP** — best for concerns tied to **business/service-layer method execution**, independent of *how* that method got triggered (HTTP request, scheduled job, message queue listener — doesn't matter). Timing a service method, applying a custom `@Retryable`-style annotation, auditing a business operation — these should work the same whether the method was called from a controller or a Kafka listener.
- **Interceptor** — best for concerns that are specifically about **the HTTP request/response and which controller handler matched** — logging path + method + status code, applying a custom `@RequireRole` annotation *specifically* to controller endpoints, timing "how long did this whole HTTP request take end-to-end" (as opposed to one service method).
- **Filter** — best for concerns that don't care about Spring MVC at all — pure infrastructure (compression, CORS headers, raw request logging including static resources that never reach a controller).

**Interview angle:** *"When would you choose AOP vs. an Interceptor vs. a Filter for the same-sounding requirement (e.g. 'log every request')?"* — the sharp answer identifies *what the concern is actually coupled to*: service-layer business logic → AOP; HTTP request/controller-handler specifics → Interceptor; raw servlet-level infrastructure, independent of routing → Filter. Not "they're basically the same, pick one."

---

## Quick-fire: AOP/proxy details worth having ready

| Detail | Why it comes up |
|---|---|
| **AOP only advises Spring-managed beans** | A plain `new SomeService()` (not fetched via the container) is never proxied — a common "why isn't my aspect firing" root cause distinct from self-invocation |
| **Only public methods are advised by default** | Calling a `private`/`protected` method — even from outside the class — never goes through the proxy, because the proxy can only intercept calls to methods it actually overrides/implements |
| **`@Transactional` on a `private` method silently does nothing** | Same root cause as above — a frequently-asked "spot the bug" interview question |
| **CGLIB proxies can't advise `final` classes/methods** | Because CGLIB works by subclassing the target — a `final` class can't be subclassed, so a `final` service class can break Spring AOP/`@Transactional` entirely, not just for one method |

---
*Next companion doc: `scenarios-06-jdbc-hibernate-jpa-tx.md` — a real N+1 production slowdown story, `LazyInitializationException`, optimistic vs pessimistic locking, and the self-invocation problem's most dangerous form (transactions).*
