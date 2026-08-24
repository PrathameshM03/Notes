# Scenarios & Deeper Reasoning — 2: Beans, Proxies & The Self-Invocation Trap
### Companion to `02-spring-core-beans.md`. This introduces the proxy self-invocation problem once, in depth — every later doc (AOP, `@Transactional`, `@Async`) just points back here.

---

## Story 1: The bug that "makes no sense" — self-invocation

**The situation:** A developer adds logging to `OrderService` using AOP (or later, adds `@Transactional` to a helper method), calls it from another method in the *same class*, and the advice/transaction **silently doesn't run**. No error. No warning. Just... doesn't happen.

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        // business logic...
        updateInventory(order); // calling a method on "this"
    }

    @Transactional
    public void updateInventory(Order order) {
        // expects to run in its own transaction... but doesn't, when called like this!
    }
}
```

**The naive assumption:** "I put `@Transactional` on the method, so it's transactional whenever it runs." This is *false*, and understanding why is one of the most-tested "gotcha" questions in Spring interviews.

**Why it breaks — the core mechanism:**
Recall from the AOP/bean-lifecycle notes: Spring doesn't hand you the *real* `OrderService` object. It hands you a **proxy** that wraps it. All the magic — `@Transactional`, `@Async`, `@Cacheable`, custom `@Aspect` advice — is implemented by that proxy intercepting calls **from the outside**.

```
Caller → Proxy(OrderService) → [advice runs] → real OrderService.placeOrder()
```

When `placeOrder()` calls `updateInventory(order)`, it's calling it via **`this`** — i.e. directly on the *real* object, **not through the proxy**. The proxy is never involved in that internal call, so none of its advice (transaction start/commit, logging, caching, async dispatch) fires.

```
Caller → Proxy(OrderService) → [advice runs] → real OrderService.placeOrder()
                                                       │
                                                       └── this.updateInventory() → BYPASSES the proxy entirely
```

**Why Spring designed it this way (not a bug, a trade-off):** Spring AOP is deliberately **proxy-based** rather than doing full bytecode weaving of every method call (which AspectJ's compile-time weaving *does* handle, at the cost of a separate build step and more complex tooling). Proxy-based AOP is simpler to set up (no special compiler/build plugin needed) and works purely through normal Spring bean wiring — the trade-off Spring's designers accepted is exactly this self-invocation blind spot.

**Real-world consequences of not knowing this:**
- A `@Transactional` method silently runs in the *caller's* existing transaction (if one exists) instead of getting its own — or with *no* transaction at all if the caller has none, meaning failures don't roll back what you expected.
- An `@Async` method called internally runs **synchronously**, blocking the calling thread exactly like a normal method call — defeating the entire purpose of marking it async.
- A `@Cacheable` method called internally **never uses the cache** — it just executes the real logic every time.

**The fix — three real options, with trade-offs:**

1. **Move the method into a separate bean/service**, and inject that bean, then call it from outside:
```java
@Service
public class InventoryService {
    @Transactional
    public void updateInventory(Order order) { ... }
}

@Service
public class OrderService {
    private final InventoryService inventoryService;
    public void placeOrder(Order order) {
        inventoryService.updateInventory(order); // now goes through InventoryService's proxy
    }
}
```
This is the **cleanest and most recommended fix** — it also usually improves your design, since it forces you to think about whether `updateInventory` really belongs in `OrderService` at all (often it doesn't — single responsibility).

2. **Self-inject the proxy** (works, but a well-known code smell):
```java
@Service
public class OrderService {
    @Autowired
    private OrderService self; // Spring injects the PROXY here, not "this"

    public void placeOrder(Order order) {
        self.updateInventory(order); // now goes through the proxy
    }
}
```
Most teams consider this ugly — it works but signals a design that should probably be split into two classes anyway.

3. **`AopContext.currentProxy()`** (requires exposing the proxy explicitly) — rarely used in modern code, mentioned mainly so you recognize it if you see it in a legacy codebase.

**Interview angle:** *"Why doesn't `@Transactional` work when I call the method from within the same class?"* — this is an extremely common question, and the strong answer walks through the proxy diagram above, names the term **self-invocation**, and gives the "move it to a separate bean" fix as the *recommended* solution (not just "use `AopContext`").

---

## Story 2: The singleton that silently corrupted itself

**The situation:** A developer writes a `@Component` service and, out of habit from other languages, adds a mutable instance field to hold "current request" data:
```java
@Component
public class ReportGenerator {
    private String currentUserName; // <-- danger

    public void setUser(String name) { this.currentUserName = name; }
    public String generateReport() { return "Report for " + currentUserName; }
}
```

**Where it breaks:** Spring beans are **singleton by default** (Lecture 6) — there is exactly **one instance** of `ReportGenerator` shared across **every concurrent HTTP request** in the whole application. Two users hitting the app at the same time:
```
Thread A: setUser("Alice")
Thread B: setUser("Bob")      <-- overwrites Alice's value on the SAME shared object
Thread A: generateReport()    <-- returns "Report for Bob" !!
```
This is a genuine, silent data-leak bug — Alice sees Bob's report, or vice versa, depending on timing. No exception, no crash — just wrong data, intermittently, under load (which makes it brutal to reproduce and debug).

**Why this "shouldn't" surprise anyone (but does):** singleton scope was covered as "one object per bean definition" — but the *practical implication* (shared mutable state = race condition) is easy to forget when you're used to writing simple, single-threaded scripts.

**The fix — never store per-request/per-user mutable state in a singleton field.** Options, in order of preference:
1. **Pass the data as a method parameter** instead of a field — the cleanest fix, and usually all you need:
```java
public String generateReport(String userName) { return "Report for " + userName; }
```
2. **Use a request-scoped bean** (Lecture 6's `@Scope("request")`) if you genuinely need bean-like injection of per-request state.
3. **`ThreadLocal`**, for advanced cases (e.g. security context storage — which is literally how `SecurityContextHolder` works internally) — but this needs careful cleanup (a `ThreadLocal` not cleared can leak into a *different* request if the thread is reused from a pool, which is exactly how web servers work).

**Interview angle:** *"Are Spring singleton beans thread-safe?"* — the honest, precise answer: **Spring doesn't make them thread-safe — it just guarantees there's one instance.** Thread-safety is *your* responsibility: keep singleton beans **stateless** (no mutable instance fields that vary per request), and any state that does vary per request should be a method parameter, a request-scoped bean, or `ThreadLocal`.

---

## Story 3: Why prototype scope is rarer in real projects than the tutorials suggest

**The situation:** A tutorial shows `@Scope("prototype")` and it seems like a natural default for "objects that hold some state per use" (which, per Story 2, is most non-trivial classes). So why doesn't everyone just use prototype scope everywhere to sidestep the singleton-mutable-state problem?

**Where the "just use prototype everywhere" idea breaks down:**
1. **Performance** — creating a new object (and re-resolving/injecting *its* dependencies) on every single `getBean()` call is real, avoidable overhead for objects that don't actually need per-use state.
2. **The prototype-in-singleton trap** (from Lecture 6) — if you inject a prototype bean into a singleton via a normal field, it's only created **once**, at the singleton's construction time, and then reused forever — completely defeating the purpose of "prototype." Getting a genuinely fresh prototype instance on every *use* (not just every *injection*) requires extra machinery (`ObjectFactory<T>`, `Provider<T>`, or `@Lookup` method injection) that most developers don't bother learning, which is exactly why prototype scope is underused relative to how often the "I need per-use state" problem actually comes up.
3. **In practice, the Story 2 fixes are simpler** — pass data as a method parameter, or use a plain (non-Spring-managed) object with `new`, instead of asking Spring to manage a short-lived object's lifecycle at all. Prototype scope earns its keep mainly when the object genuinely needs **dependency injection** *and* per-use freshness simultaneously (e.g. a stateful builder-like helper that itself depends on other Spring beans).

**Interview angle:** *"When would you actually use prototype scope?"* — a thoughtful answer names the narrow case (needs both DI *and* fresh state per use) rather than reciting the definition, and mentions the singleton-injecting-a-prototype trap unprompted — that's the detail that separates "read the docs" from "debugged this once."

---

## Story 4: Circular dependency — why Spring "fails loudly" is actually the right call

**The situation:** Two services end up needing each other:
```java
@Service
public class UserService {
    public UserService(OrderService orderService) { ... }
}
@Service
public class OrderService {
    public OrderService(UserService userService) { ... }
}
```
Spring throws `BeanCurrentlyInCreationException` at startup with constructor injection.

**The tempting "fix":** switch to field injection (`@Autowired` on the field instead of the constructor) — this often *does* make the error go away, because Spring can create both raw objects first (via a no-arg constructor) and inject fields afterward.

**Why "making the error go away" is the wrong instinct:** a circular dependency between two services is almost always a sign that they're **too tightly entangled** — they should either be merged into one cohesive service, or the shared behavior should be extracted into a **third service** that both depend on (breaking the cycle structurally, not just papering over it with a different injection style):
```java
@Service
public class OrderUserSharedLogic { ... } // extracted shared behavior

@Service
public class UserService {
    public UserService(OrderUserSharedLogic shared) { ... }
}
@Service
public class OrderService {
    public OrderService(OrderUserSharedLogic shared) { ... }
}
```
**Why Spring Boot actively disables the field-injection workaround by default** (as noted in Lecture 6): the framework's own designers decided that letting circular dependencies "quietly work" via field injection does more long-term harm (entangled, hard-to-test, hard-to-reason-about services) than the short-term pain of a loud startup failure that forces a redesign.

**Interview angle:** *"How do you fix a circular dependency?"* — the weak answer is "use `@Lazy` or field injection." The strong answer leads with "redesign to remove the cycle — usually by extracting shared logic into a third class," and only *then* mentions `@Lazy`/setter injection as an escape hatch for legacy code you can't immediately refactor.

---

## Quick-fire: concepts worth knowing exist, even if rarely used directly

| Concept | What it is | Why it's rarely reached for |
|---|---|---|
| `@Lookup` method injection | Lets a singleton fetch a fresh prototype bean on each call | Verbose, non-obvious syntax (abstract method Spring implements at runtime) — `ObjectFactory<T>`/`Provider<T>` injection is usually clearer when this pattern is genuinely needed |
| `BeanFactoryAware` / `ApplicationContextAware` | Aware interfaces that hand a bean a reference to the container itself | Using these tightly couples your business class to the Spring framework API — almost always a sign you should be using normal DI instead |
| XML bean configuration | The pre-annotation way to wire beans | Verbose, no compile-time checking, no IDE refactoring support — kept alive today mainly by legacy codebases, not new projects |
| `@Scope("prototype")` | New instance per request-for-bean | Real use is narrower than tutorials imply — see Story 3 |

---
*Next companion doc: `scenarios-03-boot-config-crud.md` — idempotency in practice, PUT vs PATCH real-world confusion, and a soft-delete production incident.*
