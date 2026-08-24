# Scenarios & Deeper Reasoning — 1: Why Spring Exists At All
### Companion to `01-foundations-web-maven-di.md`. Same topics, but told as "here's the fork in the road, here's what we tried, here's why it lost."

---

## Story 1: Before Spring — the world of raw Servlets and J2EE/EJB

**The situation:** It's the early 2000s (conceptually — this history matters for *why* Spring's design choices exist). You're building a Java web application. Your options are:

1. **Raw Servlets + JSP** — you write everything by hand: HTTP parsing (handled by the container), but *everything else* — object creation, database connections, transaction management, security — is your own responsibility, scattered across servlet classes.
2. **J2EE / EJB (Enterprise JavaBeans)** — Sun's "enterprise" answer. EJB promised to handle transactions, security, and remote objects for you.

**The naive attempt — EJB:** You write an EJB. To use it, you look it up via **JNDI** (Java Naming and Directory Interface):
```java
Context ctx = new InitialContext();
Object ref = ctx.lookup("java:comp/env/ejb/PaymentService");
PaymentServiceHome home = (PaymentServiceHome) PortableRemoteObject.narrow(ref, PaymentServiceHome.class);
PaymentService paymentService = home.create();
```
This pattern — asking a central registry "give me the object named X" — is called the **Service Locator pattern**.

**Where it breaks:**
- Every class that needs a dependency must know *how to look it up* (the JNDI name, the casting, the narrowing) — this is boilerplate repeated everywhere, and it's tightly coupled to the container's naming scheme.
- **Testing is a nightmare.** You can't unit-test a class that does a JNDI lookup inside itself without spinning up a fake JNDI context — EJB classes were notoriously difficult to test outside a real (or mocked) application server.
- EJB required implementing verbose interfaces (`SessionBean`, `EntityBean`), writing deployment descriptors (XML), and following rigid naming/packaging conventions — heavy ceremony for simple business logic.
- The dependency is *hidden inside the method body* — you can't tell what a class needs just by reading its constructor; you have to read the whole method to find the lookup call.

**The Spring answer:** Rod Johnson (Spring's creator) argued: *don't make classes go and find their dependencies — hand the dependencies to the class instead.* This is **Dependency Injection**, and it flips Service Locator on its head:

| | Service Locator | Dependency Injection |
|---|---|---|
| Who initiates the lookup? | The class itself, at runtime | The container, before the class is even used |
| Is the dependency visible from outside? | No — hidden inside method bodies | Yes — visible in the constructor signature |
| Can you unit-test without the container? | Hard — often need a fake registry | Easy — just call `new MyClass(fakeDependency)` |
| Coupling | Tightly coupled to the lookup mechanism (JNDI) | Coupled only to the dependency's *type* |

**Why this matters today:** every time you write a constructor like `public OrderService(PaymentService p)` instead of `PaymentService p = ServiceLocator.lookup("PaymentService")`, you're benefiting from a fight Spring won 20 years ago. It's also *why* constructor injection specifically is preferred — it makes the dependency **visible and mandatory**, exactly the property Service Locator lacked.

**Interview angle:** *"What problem does Dependency Injection actually solve, historically?"* — most candidates say "loose coupling," which is true but shallow. The sharper answer: it moves the *responsibility* of finding a dependency from the consuming class to an external container, which is what makes the consuming class testable in isolation and keeps its dependencies declared up front rather than buried in method bodies.

---

## Story 2: Why not build your own DI container?

**The situation:** You understand DI now. Why not just write:
```java
public class SimpleContainer {
    private Map<Class<?>, Object> objects = new HashMap<>();
    public void register(Class<?> type, Object obj) { objects.put(type, obj); }
    public <T> T get(Class<?> type) { return (T) objects.get(type); }
}
```

**Where it breaks fast:**
- What if `OrderService` needs `PaymentService`, which needs `Logger`, which needs `Config`? You now need to figure out **creation order** yourself — this is exactly the dependency-graph resolution problem Spring's `BeanDefinition` mechanism (Lecture 5) solves automatically.
- What about lifecycle — who calls `init()`/`destroy()` methods? Who decides singleton vs. a-new-object-every-time? You'd be re-inventing bean scopes (Lecture 6) and the bean lifecycle (Lecture 7) from scratch.
- What about cross-cutting concerns like transactions or security applied automatically to matching methods? You'd be re-inventing AOP (Lectures 22–25).
- What about configuration from multiple sources (files, env vars, CLI args) with sensible precedence? You'd be re-inventing `Environment`/`@ConfigurationProperties` (Lecture 10).

**The real answer:** a hand-rolled container is a fun weekend project but a multi-year engineering investment to get *production-grade* — correct concurrent bean creation, circular dependency detection, AOP proxying, transaction synchronization, event publishing, internationalization... Spring isn't "a DI container," it's a DI container **plus 20+ years of solving every edge case that DI containers run into in real applications.** This is also why "just use `new` everywhere for a small app" is a legitimate answer in an interview for small/simple projects — Spring's overhead only pays for itself once the object graph and cross-cutting concerns grow past a certain size.

---

## Story 3: Why Maven (and not Ant, or nothing)?

**The situation:** Before Maven, Java projects used **Ant** — a build tool where you wrote an XML file (`build.xml`) describing *exact steps*: compile these files, copy these resources, zip into a JAR. Ant is essentially a **scripting language for builds** — extremely flexible, but that flexibility is the problem.

**Where Ant breaks down:**
- Every project's `build.xml` looks different — there's no shared convention, so a new developer has to *read the whole script* to understand how a project builds.
- Dependency management is entirely manual — you download JARs yourself and reference them by path. No concept of "transitive dependencies," no repository, no version resolution.

**The Spring/Maven answer:** Maven trades flexibility for **convention** — `src/main/java`, `src/test/java`, a lifecycle every project follows the same way (`validate → compile → test → package → install → deploy`), and a **dependency + repository model** that automatically resolves transitive dependencies. You lose some flexibility (customizing the build process is more awkward than in Ant) but gain the ability to open *any* Maven project and immediately know its shape.

**Where does Gradle fit?** Gradle (not covered in the course, worth knowing) is the newer alternative — it keeps Maven's convention-over-configuration *and* Ant's scripting flexibility, using a Groovy/Kotlin DSL instead of XML. Many newer Spring Boot projects (and virtually all Android projects) use Gradle instead of Maven today. **Why would a team still pick Maven?** XML is more rigid but also more predictable and less "programmable" — some teams deliberately want a build file that can't accidentally become a mini-program, for reliability and easier auditing in regulated environments.

**Interview angle:** *"Maven vs Gradle — which do you prefer and why?"* — a good answer isn't "Gradle is faster" (though it often is, via incremental builds and caching) — it's understanding the trade-off: Maven's rigidity is a *feature* for large, many-team organizations that want predictable builds; Gradle's flexibility is a feature for projects that need custom build logic (e.g. Android's resource compilation pipeline).

---

## Story 4: The tight-coupling production disaster (a realistic scenario)

**The situation:** A team ships `OrderService` with `PaymentService paymentService = new RazorpayService();` hardcoded inside it — no interface, no injection. It works fine in production for a year.

**Where it breaks:** The company needs to add Stripe support for international customers, and *also* needs to run automated tests in CI without hitting Razorpay's real (rate-limited, costs-money-per-call) API.

**What tight coupling costs them, concretely:**
1. **Can't swap providers per region** without an `if/else` sprinkled through `OrderService`, mixing business logic with provider selection logic.
2. **Can't unit test `OrderService`** without either hitting the real Razorpay API (slow, flaky, costs money) or wrapping the whole class in complicated mocking-the-unmockable workarounds.
3. **Every new provider means editing and redeploying `OrderService`** — a class that should only care about "orders," not "which payment vendor is currently in fashion."

**The fix (this is Lecture 4/5's actual payoff, made concrete):**
```java
public interface PaymentService { PaymentResult pay(Order order); }

@Component
@Profile("razorpay")
public class RazorpayService implements PaymentService { ... }

@Component
@Profile("stripe")
public class StripeService implements PaymentService { ... }

@Service
public class OrderService {
    private final PaymentService paymentService; // interface, injected — don't care which one
    public OrderService(PaymentService paymentService) { this.paymentService = paymentService; }
}
```
Now: swapping providers is a **config change** (`@Profile`), not a code change. Testing `OrderService` means injecting a `MockPaymentService` — zero real API calls. This is the concrete, "why does anyone care" payoff of everything in Lecture 4.

**Interview angle:** interviewers sometimes phrase this as *"walk me through refactoring tightly-coupled code to use DI"* — the strong answer isn't reciting the DI definition, it's telling exactly this kind of story: what breaks in production/testing *because* of tight coupling, and how the interface + injection fixes each specific pain point, not just "it's a best practice."

---

## Quick-fire: concepts worth knowing exist, even if you'll rarely touch them

| Concept | What it is | Why it's not what you'll use today |
|---|---|---|
| **EJB / J2EE** | Sun's enterprise Java framework, predates Spring | Too heavyweight, too much ceremony (XML descriptors, remote interfaces) for most applications — Spring won this fight in the mid-2000s |
| **JNDI** | Java's naming/directory lookup API | Still used under the hood for some resources (like JDBC `DataSource` lookups in application servers), but application code almost never calls it directly anymore |
| **Ant** | XML-based Java build scripting tool | No shared convention across projects, no built-in dependency management — superseded by Maven/Gradle |
| **Gradle** | Groovy/Kotlin DSL build tool | Not "wrong," just a different trade-off (flexibility over rigidity) — very much still in active use, especially Android and newer Spring projects |
| **`BeanFactory`** (raw) | The most basic Spring container interface | `ApplicationContext` is used almost everywhere in practice — `BeanFactory` is the foundation `ApplicationContext` builds on, rarely instantiated directly |

---
*Next companion doc: `scenarios-02-spring-core-beans.md` — the self-invocation/proxy trap (referenced by AOP, `@Transactional`, and `@Async` later), singleton misuse stories, and why prototype beans are rarer in practice than tutorials suggest.*
