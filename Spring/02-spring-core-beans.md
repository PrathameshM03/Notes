# Spring Framework Full Course — Notes 2: Spring Core, Beans & Bootstrapping
### Covers Lectures 5–9: Beans & annotation config → Circular dependency & scopes → Bean lifecycle → XML config → Spring Boot's own startup internals

---

## Lecture 5 — Spring IoC Container, Beans & Annotation-Based Configuration

### 1. Setup
Annotation-based Spring Core needs just the `spring-context` dependency, which gives you `ApplicationContext`, component scanning, and bean creation/injection.

### 2. What is a Bean?
> A **Spring Bean** is an object whose creation, dependency wiring, and lifecycle are managed by the Spring IoC container (instead of your code doing `new` and wiring by hand).

Two configuration styles exist: **annotation-based** (`@Component`, `@Configuration`, `@ComponentScan`, `@Autowired`, `@Bean` — modern, preferred) and **XML-based** (older, still found in legacy projects — covered in Lecture 8).

### 3. A quick reflection detour
`Student.class` is not a `Student` object — it's an object of type `Class<Student>`, holding *metadata* (fields, methods, constructors, annotations). Spring uses this metadata internally to inspect and build beans. This is why you can pass `AppConfig.class` to the container — you're handing it class metadata to read.

### 4. `ApplicationContext` — the IoC container
```java
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
```
`ApplicationContext` is an **interface** representing the Spring IoC container (you can't `new` it directly — you instantiate an implementation). `AnnotationConfigApplicationContext` is the implementation used for annotation-driven config. It:
- Reads configuration
- Creates beans
- Resolves dependencies
- Manages bean lifecycle
- Hands beans back to you on request

### 5. The configuration class
```java
@Configuration
@ComponentScan("in.coderarmy")
public class AppConfig { }
```
- **`@Configuration`** — marks this class as a *source of bean definitions* (Spring's "startup instruction file").
- **`@ComponentScan("in.coderarmy")`** — tells Spring where to look for `@Component`-annotated classes (scans that package + sub-packages). If you omit the package argument, Spring scans the package the config class itself lives in (and below).

| Annotation | Meaning |
|---|---|
| `@Component` | Marks a class as eligible to become a bean |
| `@ComponentScan` | Tells Spring *where* to search for such classes |

### 6. Fetching a bean
```java
OrderService orderService = context.getBean(OrderService.class);
```
If Spring can't find a matching bean you get an error (`No qualifying bean of type ...`) — usually because the class isn't `@Component`-annotated, its package isn't covered by `@ComponentScan`, or it's not registered via `@Bean`/XML.

### 7. Three types of Dependency Injection (in practice)

**Constructor injection (preferred):**
```java
@Component
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) { // @Autowired optional with a single constructor
        this.paymentService = paymentService;
    }
}
```
Why preferred: the dependency is *mandatory* (object can't exist without it), the field can be `final` (immutable, safe), and the class is trivially testable *without Spring* (`new OrderService(new PaymentService())`).

**Field injection (avoid for important dependencies):**
```java
@Component
public class OrderService {
    @Autowired
    private PaymentService paymentService;
}
```
Downsides: dependency is hidden, can't be `final`, hard to unit test without a Spring context, object can exist in an incomplete state before injection happens.

**Setter injection (good for optional/replaceable dependencies):**
```java
@Component
public class OrderService {
    private PaymentService paymentService;
    @Autowired
    public void setPaymentService(PaymentService paymentService) { this.paymentService = paymentService; }
}
```

### 8. What Spring does internally on startup
1. Start the container (`ApplicationContext`).
2. Read the configuration class (`AppConfig`).
3. Process `@ComponentScan` → scan the package(s).
4. Find classes annotated `@Component`/`@Service`/`@Repository`/`@Controller`.
5. Build a **`BeanDefinition`** for each — *metadata* about the bean (name, class, scope, dependencies, creation strategy) — **not the object itself**.
6. Resolve dependencies between bean definitions.
7. Actually instantiate bean objects (dependency-free beans first, then beans that need them).
8. Inject dependencies (via constructor/setter/field).
9. Store finished beans in the container.
10. Return a bean whenever `getBean()` is called.

### 9. Resolving ambiguity: multiple beans of the same type
If two `@Component` classes both implement `PaymentService`, injecting `PaymentService` by type is ambiguous. Two fixes:

**`@Primary`** — marks one implementation as the default:
```java
@Primary
@Component
public class UPIPaymentService implements PaymentService { ... }
```

**`@Qualifier`** — explicitly names the exact bean to inject (default bean name = class name with a lowercase first letter, e.g. `CardPaymentService` → `cardPaymentService`):
```java
public OrderService(@Qualifier("cardPaymentService") PaymentService paymentService) { ... }
```
If both `@Primary` and `@Qualifier` are present somewhere in the graph, `@Qualifier` wins at the specific injection point where it's used.

### 10. `@Bean` — manual bean creation
Use `@Bean` inside a `@Configuration` class when you need to create an object yourself — typically for **third-party classes you don't own** (so you can't add `@Component` to them) or when construction needs custom logic:
```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentService paymentService() { return new UPIPaymentService(); }

    @Bean
    public OrderService orderService(PaymentService paymentService) {
        return new OrderService(paymentService); // Spring passes the paymentService bean automatically
    }
}
```

| | `@Component` | `@Bean` |
|---|---|---|
| Applied on | Class | Method |
| Style | Automatic detection via scanning | Manual registration |
| Best for | Your own classes | External library classes / custom creation logic |
| Needs component scanning? | Yes | No, but the `@Configuration` class must be loaded |
| Default bean name | From class name | From method name |

**Don't double-register** the same logical bean via both `@Component` and `@Bean` unless you deliberately want two separate beans — it causes ambiguity or bean-overriding surprises.

### 11. `BeanFactory` vs `ApplicationContext`
`BeanFactory` is the basic, low-level container interface. `ApplicationContext` extends it and adds real-world features: event publishing, internationalization, easier integration with the rest of Spring. **Use `ApplicationContext` in normal applications.**

### 12. Why not just wire everything in `main()`?
Because then `main()` becomes a manual container again — defeating the purpose. Keep `main()` minimal; let a `@Configuration` class + component scanning own the wiring.

---

## Lecture 6 — Circular Dependency, Bean Scope & Bean Initialization

### 1. Circular Dependency
> Two or more beans depend on each other, directly or indirectly (A needs B, B needs A — or A→B→C→A).

- **Constructor injection + circular dependency = fails.** Spring can't fully construct A without B, and can't fully construct B without A — there's no valid creation order, so Spring throws `BeanCurrentlyInCreationException`.
- **Constructor injection is still the recommended default** — this "failure" is actually useful: it forces you to notice and fix a design smell early instead of silently limping along.
- **Setter/field injection can sometimes work around it**, because Spring can create the raw object first (via a no-arg constructor) and inject the dependency *afterward*, using an early/partial object reference — but Spring Boot **disables circular references by default**, so relying on this is discouraged.
- **Best fix for circular dependency:** redesign — extract shared logic into a third service that both depend on, or use `@Lazy` to break the cycle (see below) as a last resort, not a habit.

### 2. Bean Scope
> **Bean scope** defines how many instances of a bean the container creates and how they're shared.

| Scope | Meaning |
|---|---|
| **Singleton** (default) | Exactly **one object per bean definition** in the container — the same instance is returned every time |
| **Prototype** | A **new object every time** the bean is requested (`@Scope("prototype")`) |
| **Request** | One bean instance per **HTTP request** (web apps only) |
| **Session** | One bean instance per **user session** (web apps only) |
| **Application** | One bean instance per **ServletContext** (whole web app) |

```java
@Component
@Scope("prototype")
public class ReportGenerator { ... }
```

Important nuance: "singleton" means one instance **per bean definition**, not per class — if you register the same class twice under different bean definitions (e.g. via two `@Bean` methods), you get two singleton instances, one per definition.

**Gotcha — prototype bean injected into a singleton bean:** the prototype is only created once, at the moment the singleton is wired, and then reused forever inside that singleton (because the singleton itself is only constructed once). To get a fresh prototype on every use inside a singleton you need extra patterns (e.g. `ObjectFactory`/`Provider`/`@Lookup` — mentioned here as a known caveat).

### 3. Eager vs Lazy Initialization
- **Eager initialization (default)** — all singleton beans are created **at container startup**, not on first use. This is the default because it surfaces configuration/wiring errors immediately at boot time rather than mysteriously at runtime.
- **Lazy initialization** — the bean is created only when it's actually first requested:
```java
@Component
@Lazy
public class ReportService { ... }
```
- **`@Lazy` can be applied at the bean definition** (the bean itself is created lazily wherever it's used) **or at a specific injection point** (only *that* particular dependency reference is lazy — implemented as a proxy that defers real creation).
- Global lazy init can be turned on for the whole app (`spring.main.lazy-initialization=true` in Boot) and selectively opted out per bean if needed.
- **Using `@Lazy` to break a constructor-based circular dependency:** if A needs B and B needs A, marking one side `@Lazy` lets Spring inject a proxy for that side first and resolve the real object later — this unblocks the circular chain, but it's a workaround, not a design fix; prefer restructuring the dependency graph.

### 4. Bean Lifecycle (preview — detailed in Lecture 7)
The full journey: bean definition read → object instantiated → dependencies injected → Aware callbacks → initialization callbacks → bean ready to use → destruction callbacks (on container shutdown, for singleton beans).

### Key takeaways
- Circular dependency = a design smell; constructor injection surfaces it loudly (which is a feature, not a bug).
- Singleton is Spring's default scope; prototype creates a fresh object per request-for-bean.
- Web scopes (request/session/application) only make sense inside a web application context.
- Eager init catches problems at startup; lazy init defers bean creation until actually needed.

---

## Lecture 7 — Spring Bean Lifecycle

### Complete lifecycle flow
```
Spring container starts
   → Spring reads configuration/annotations
   → Spring creates BeanDefinition
   → Spring instantiates the bean object (constructor called)
   → Spring injects dependencies
   → Spring calls Aware interfaces (e.g. BeanNameAware, ApplicationContextAware)
   → Spring runs initialization callbacks
   → Bean is ready to use
   → Application uses the bean
   → Spring runs destruction callbacks (container shutdown)
   → Bean is removed
```

**Step-by-step:**
1. **Bean definition is read** — from annotations, `@Bean` Java config, or XML.
2. **Bean object is instantiated** — the constructor runs.
3. **Dependencies are injected** — constructor args already set; setter/field injection happens now.
4. **Aware interfaces are called** — e.g. `BeanNameAware.setBeanName()`, `ApplicationContextAware.setApplicationContext()` — lets a bean "become aware of" container-provided info about itself.
5. **Initialization callbacks run** — e.g. `@PostConstruct`-annotated methods, or `InitializingBean.afterPropertiesSet()`, or a custom `init-method`.
6. **Bean is ready to use** — fully constructed, injected, and initialized.
7. **Destruction callbacks run** — e.g. `@PreDestroy`, or `DisposableBean.destroy()`, or a custom `destroy-method`, when the container shuts down.

### Singleton vs Prototype lifecycle
- **Singleton:** the container manages the *entire* lifecycle, including destruction — it calls destroy callbacks when the container itself shuts down.
- **Prototype:** the container hands you the fully-initialized object and then **stops tracking it**. Destruction callbacks are **not called automatically** for prototype beans — the container doesn't know when you're done with it, so cleanup (if needed) becomes the application's responsibility.

### Important case: Singleton depends on Prototype
As noted in Lecture 6, a prototype bean injected into a singleton is only created once (at singleton construction time) unless special patterns are used to fetch a new prototype instance on each use.

### One-line takeaway
> A Spring bean's life = **defined → instantiated → dependencies injected → made aware → initialized → used → destroyed**, and the container automates every one of those steps for singleton beans.

---

## Lecture 8 — XML-Based Configuration

### Why learn XML config today?
Modern projects use annotations, but many **legacy** enterprise Spring applications still use XML — understanding it helps you work with (or migrate away from) older codebases. Whatever the configuration style, Spring ultimately needs the same information: which classes become beans, what to name them, singleton or prototype, which dependencies to inject, which methods to run at init/destroy time.

### Basic setup
```xml
<!-- beans.xml -->
<beans xmlns="http://www.springframework.org/schema/beans" ...>
    <bean id="paymentService" class="in.coderarmy.service.PaymentService" />
    <bean id="orderService" class="in.coderarmy.service.OrderService">
        <constructor-arg ref="paymentService" />
    </bean>
</beans>
```
Loading it in Java:
```java
ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
OrderService orderService = context.getBean("orderService", OrderService.class);
```
A key gotcha beginners hit: the constructor runs **immediately when the context is created**, not when `getBean()` is called (for eager singletons) — same eager-init behavior as annotation config.

### Bean identity
- `id` — the primary bean identifier; if omitted, Spring generates one; duplicate `id`s cause an error.
- `name` — lets you assign **multiple names/aliases** to the same bean directly in the `<bean>` tag.
- `<alias>` — defines an **external alias** for a bean elsewhere in the XML.

### Dependency Injection in XML
**Constructor injection**, several equivalent styles:
```xml
<!-- by bean reference -->
<constructor-arg ref="paymentService" />
<!-- by simple literal value -->
<constructor-arg value="Standard Plan" />
<!-- by index -->
<constructor-arg index="0" ref="paymentService" />
<!-- by name -->
<constructor-arg name="paymentService" ref="paymentService" />
<!-- by type -->
<constructor-arg type="in.coderarmy.service.PaymentService" ref="paymentService" />
```
**Setter injection:**
```xml
<bean id="orderService" class="in.coderarmy.service.OrderService">
    <property name="paymentService" ref="paymentService" />
</bean>
```

### Resolving multiple beans of the same type in XML
- Explicit wiring with `ref` — just point the reference directly at the exact bean id you want.
- `primary="true"` on a `<bean>` — the XML equivalent of `@Primary`.
- `autowire-candidate="false"` on a `<bean>` — excludes that bean from autowiring resolution altogether.

### XML autowiring modes
XML supports declaring `autowire="byName"`, `byType`, `constructor`, etc. on a `<bean>` tag, letting Spring auto-wire without explicit `ref`s — useful historically, but explicit wiring is generally clearer.

### Scopes and lifecycle callbacks in XML
```xml
<bean id="reportGenerator" class="in.coderarmy.ReportGenerator" scope="prototype"
      init-method="init" destroy-method="cleanup" />
```
- `scope="singleton"` (default) / `scope="prototype"`.
- `init-method` / `destroy-method` — name the methods to call after construction / before destruction. **Note:** destroy-method is only reliably called for singleton beans when the context is explicitly closed (`context.close()`), consistent with the singleton-vs-prototype lifecycle rule from Lecture 7.

### Injecting collections
```xml
<property name="cities">
    <list>
        <value>Delhi</value>
        <value>Mumbai</value>
        <value>Bangalore</value>
    </list>
</property>
<property name="config">
    <map>
        <entry key="env" value="prod" />
    </map>
</property>
```

### Splitting XML files
Large XML configs are commonly split by concern (e.g. `orderContext.xml`, `userContext.xml`) and combined in one root file (`applicationContext.xml`) via `<import resource="..."/>`, keeping configuration organized as the project grows.

### XML + annotations together
Yes — you can enable `<context:component-scan base-package="..."/>` inside an XML file, letting XML-declared beans and annotation-scanned beans coexist. Recommended learning path: understand pure annotation config first, then pure XML, then know that hybrid configs exist mainly in legacy/migration scenarios — don't mix the two by default in new projects.

---

## Lecture 9 — Spring Boot Startup Internals & Auto-Configuration

### 1. Spring Core startup, the manual way
Without Boot, starting a Spring app means explicitly:
1. Creating the `ApplicationContext` yourself
2. Providing a configuration class manually
3. Fetching the bean manually via `getBean()`
4. Calling application logic manually

### 2. Same application, the Spring Boot way
```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
One call — `SpringApplication.run(...)` — replaces all the manual container bootstrapping above.

### 3. `spring-boot-starter-parent`
Most Spring Boot projects inherit from `spring-boot-starter-parent` in `pom.xml`. It centrally manages compatible dependency **versions** and sensible default plugin configuration, so you (mostly) don't need to specify a `<version>` for each Spring Boot-related dependency yourself.

### 4. What is a "starter"?
A **starter** dependency (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, etc.) is a curated bundle of related dependencies for one feature area — adding one starter pulls in everything typically needed for that capability, instead of hand-picking many individual JARs.

### 5. `@SpringBootApplication` — what it actually is
`@SpringBootApplication` is a convenience meta-annotation combining three annotations:
- **`@SpringBootConfiguration`** — effectively a specialized `@Configuration` (marks this class as a bean-definition source).
- **`@EnableAutoConfiguration`** — turns on Spring Boot's auto-configuration mechanism (see below).
- **`@ComponentScan`** — scans the package the annotated class lives in, and sub-packages (same behavior as annotation config in Lecture 5, applied automatically).

You *can* customize the component-scan base package(s) manually if you need to scan somewhere else, but the default (scan from the main class's own package downward) is why Spring Boot apps conventionally put the main class in the **root** package of the project.

### 6. Auto-configuration — how it decides what to create
Spring Boot's auto-configuration inspects your **classpath** and makes decisions like "if `spring-boot-starter-web` is present, auto-configure an embedded Tomcat + a `DispatcherServlet`." Roughly, for each candidate auto-configuration, Spring Boot checks (via conditional annotations like `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`):
1. **Is the required class present on the classpath?** (e.g. is Tomcat's class available?)
2. **Is the required bean missing?** (don't auto-configure something you already defined yourself — your own bean wins)
3. **Is the required property enabled?** (some auto-configuration is gated behind a config property)

This is why adding the web starter "magically" gives you an embedded server — the auto-configuration classes see the relevant classes on the classpath and register the beans for you.

### 7. Your beans vs auto-configured beans
| Source | Example |
|---|---|
| **Your code** | `@Component`/`@Service`/`@Repository`/`@Controller` classes you write, `@Bean` methods in your own `@Configuration` classes |
| **Spring Boot auto-configuration** | Embedded Tomcat, `DataSource` (when a driver + starter is present), Jackson `ObjectMapper`, `DispatcherServlet`, etc. |

If you define your **own** bean of a type Boot would otherwise auto-configure, Boot's `@ConditionalOnMissingBean` checks generally back off and let yours win.

### 8. `@ComponentScan` vs `@EnableAutoConfiguration`
Don't conflate them: `@ComponentScan` finds **your own** annotated classes; `@EnableAutoConfiguration` inspects the classpath and configures **framework-provided** beans (servers, data sources, converters, etc.) that you didn't write yourself.

### Complete Spring Boot startup flow
```
main() → SpringApplication.run(DemoApplication.class, args)
   → Spring creates the ApplicationContext
   → @SpringBootConfiguration processed (this is a configuration source)
   → @ComponentScan finds your @Component/@Service/@Repository/@Controller classes
   → @EnableAutoConfiguration inspects the classpath and conditionally registers framework beans
   → BeanDefinitions built, dependencies resolved, beans instantiated & wired (same flow as Lecture 5)
   → Embedded Tomcat starts and begins listening
   → Application is ready to serve requests
```

### Final takeaways
- Spring Boot's magic is really just: **your prior manual `ApplicationContext` bootstrapping**, automated, plus **classpath-driven conditional auto-configuration**.
- `@SpringBootApplication` = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`, all bundled together.
- Everything from Lectures 5–8 (bean definitions, dependency resolution, singleton/prototype, lifecycle callbacks) still happens under the hood — Boot doesn't replace Spring Core's IoC container, it just configures and launches it for you.

---
*Next file: `03-spring-boot-crud-config.md` — Configuration (`@Value`, `@ConfigurationProperties`, runners), and building your first CRUD REST APIs with Spring Boot.*
