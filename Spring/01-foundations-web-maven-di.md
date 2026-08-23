# Spring Framework Full Course — Notes 1: Foundations
### Covers Lectures 1–4: How the web works → Spring's place in it → First Spring Boot app → Maven → Dependency Injection

---

## Lecture 1 — Introduction to Spring & Spring Boot

### 1. Client–Server Architecture
Every web interaction starts with the same question: *how does code on one machine talk to code on another machine?* The answer is **client–server architecture**.

- **Client** — the side that asks for something (browser, mobile app, Postman, a React frontend).
- **Server** — the side that receives the request, processes it (checks logins, queries a DB, applies business rules) and sends back a response.

Rule of thumb: **Client asks → Server responds.**

### 2. HTTP — the common language
For a client and server to understand each other they need a shared protocol: **HTTP (HyperText Transfer Protocol)**. HTTP is an *application-layer protocol* built on top of TCP/IP, and it defines how a request/response should look, which method is used, which URL is called, what data is sent, and what status code comes back.

**Request–response cycle:**
```
Client sends request → Server processes it → Server sends response → Client uses/displays it
```

**Anatomy of an HTTP request** — Method, URL/path, Headers, Body.

| Method | Meaning        | Example              |
|--------|----------------|-----------------------|
| GET    | Read data      | Fetch list of orders |
| POST   | Create data    | Place a new order     |
| PUT    | Replace fully  | Overwrite user profile|
| PATCH  | Partial update | Change phone number   |
| DELETE | Remove data    | Cancel an order       |

Headers are key-value metadata (`Content-Type`, `Authorization`, `Accept`, `Host`). The Body carries the actual payload — used with POST/PUT/PATCH, usually JSON. GET normally has no body.

**Anatomy of an HTTP response** — Status code, Headers, Body. Example: `HTTP/1.1 200 OK` with a JSON body.

### 3. Why Core Java alone is not enough for the web
A plain Java program (JVM) runs locally and stops after `main()` finishes — it doesn't listen for network requests. To build a web application in raw Java you would manually need to:

1. Open a port with `ServerSocket`
2. Read raw input streams
3. Parse the HTTP request text yourself (method, URL, headers, body)
4. Route the request to the right logic
5. Build the HTTP response text yourself, with correct status/headers
6. Manage multiple concurrent users using threads
7. Handle malformed requests / errors / connection edge cases

This is exactly the repetitive, error-prone, low-level work that later gets automated away — first by **Servlets**, then by **Spring**.

### 4. Servlets — the first abstraction
A **Servlet** is a Java class that knows how to handle an HTTP request/response without you writing raw socket code. A **Servlet Container** (like **Tomcat**) does the socket work, thread management, and request/response object creation, and simply *calls your servlet method* when a matching request arrives (this is the **Inversion of Control** idea in embryonic form — the container calls you, you don't call it).

Even with Servlets, real applications still needed a lot of repeated boilerplate (mapping URLs, converting objects, wiring dependencies) — which is why **Spring** was created.

### 5. What is the Spring Framework?
Spring is not one single library — it's an **ecosystem** of modules that solve different concerns of enterprise Java development:

| Module | Purpose |
|---|---|
| **Spring Core** | IoC container, Dependency Injection, Bean management |
| **Spring MVC** | Web layer — routing HTTP requests to controller methods |
| **Spring Data** | Simplifies data access (JPA, JDBC, Mongo, etc.) |
| **Spring Security** | Authentication & authorization |
| **Spring AOP** | Cross-cutting concerns (logging, transactions, etc.) via proxies |
| **Spring AI** | Integration with AI/LLM tooling |

### 6. Spring Boot vs Spring Framework
**Spring Boot** sits on top of the Spring Framework. It is **opinionated** — it makes sensible default decisions for you (embedded server, auto-configuration, starter dependencies) so you can go from zero to a running app in minutes, instead of hand-wiring XML configuration. Spring Boot doesn't replace Spring — it *configures Spring for you*.

Microservices architecture is built using Spring Boot because Boot apps are self-contained, independently runnable (embedded Tomcat + executable JAR), which fits the "one small deployable service" model well.

### Complete flow: Browser → Spring
```
Browser --HTTP request--> Embedded Tomcat --> DispatcherServlet (Spring MVC)
   --> HandlerMapping finds controller --> Controller method executes
   --> Response built --> back through Tomcat --> Browser
```

**Key takeaway:** Learning the internals (HTTP, Servlets, Tomcat, IoC, DI) matters because Spring Boot gives *speed*, but understanding the fundamentals gives *control* — essential for debugging real production issues (404s, slow startup, requests not reaching a controller, etc.).

---

## Lecture 2 — Writing Our First Spring Boot Application

### 1. Ports and localhost
- An **IP address** identifies a *machine* on a network.
- A **port number** identifies a specific *application* running on that machine (many apps can run on one IP, each listening on its own port).
- `localhost` = your own machine = `127.0.0.1`. So `localhost:8080` means "send this request to my own machine, to whatever app is listening on port 8080."
- Browsers assume default ports when none is given: `http` → port 80, `https` → port 443. That's why you rarely type a port for normal websites but do for a local Spring Boot app (`http://localhost:8080`).

### 2. Spring Initializr
Every Spring Boot project needs similar boilerplate (folder structure, Maven config, chosen Spring Boot version, dependencies). **Spring Initializr** (start.spring.io) is a project generator that produces this skeleton so you can jump straight to business logic. It lets you pick: project type, language, Spring Boot version, metadata, packaging (JAR/WAR), Java version, and dependencies.

A **dependency** is simply an external library your project needs (e.g. Spring Web, MySQL Connector, Lombok). In Maven projects these are declared in `pom.xml`.

Version labels you'll see: **SNAPSHOT** (work-in-progress, may contain bugs), **RC** (Release Candidate — nearly final), **Stable release** (prefer this for real projects).

### 3. Project anatomy
| Path/File | Purpose |
|---|---|
| `src/main/java` | Your Java source code |
| `src/main/resources` | Config files & static resources |
| `pom.xml` | Maven build file — dependencies, plugins, metadata |
| `application.properties` | Spring Boot app configuration (e.g. server port) |
| Main class (`@SpringBootApplication`) | Entry point with `main()` that boots the app |

```java
@SpringBootApplication
public class FirstSpringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(FirstSpringBootApplication.class, args);
    }
}
```

### 4. Controllers and your first endpoint
A **Controller** is the entry point for incoming web requests — think of it as the application's "receptionist." `@RestController` tells Spring "this class can handle HTTP requests and return data directly as the response body."

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String sayHello() {
        return "Hello World";
    }
}
```
- `@RestController` → eligible to receive web requests.
- `@GetMapping("/hello")` → run this method for GET `/hello`.
- The returned String becomes the HTTP response body.

### 5. What actually happens when you run it
When the app starts you'll see a log line like `Tomcat started on port 8080`. You never installed Tomcat or wrote socket code — **Spring Boot bundles and auto-configures an embedded Tomcat server** whenever the `web` starter is on the classpath. So:

- Spring Boot starts and configures the application context.
- Embedded Tomcat listens for HTTP requests on the configured port.
- Spring MVC maps the incoming request to the matching controller method.

Change the port via `application.properties`:
```properties
server.port=9090
```

### Full request flow
```
Browser → localhost:8080/hello
   → request reaches embedded Tomcat
   → Spring MVC checks registered mappings
   → /hello matched to sayHello()
   → sayHello() returns "Hello World"
   → response goes back to the browser
```

### Why learn internals if Boot does it all?
Writing `@RestController` + `@GetMapping` is easy — many tools can generate that. Real engineering value comes from being able to answer questions like: *Why is this returning 404? Why is startup slow? Why isn't my controller receiving the request?* Those require understanding component scanning, bean creation, MVC registration, and the Tomcat request lifecycle — which is exactly what the rest of this course builds toward.

---

## Lecture 3 — Maven Deep Dive

### 1. From `.java` to `.class` to JAR
```
Hello.java --(javac)--> Hello.class --(JVM)--> output
```
Real projects have many `.java`/`.class` files. Sharing loose `.class` files is impractical (files get misplaced, package structure breaks, resources get left out). Java's answer is the **JAR (Java Archive)** — a ZIP-like bundle of compiled classes + resources + metadata.

- **Library** — reusable code meant to be used *inside* other projects (no `main()` needed to run standalone).
- **Application** — a runnable program, usually with a `main()` entry point.
- A **normal library JAR** typically does *not* bundle its own dependencies inside it — those are resolved separately via Maven.
- A **Spring Boot executable ("fat") JAR** *does* bundle the application code **and** its dependencies, so it can run standalone with `java -jar app.jar`.

### 2. Classpath
The **classpath** is where Java looks to find classes — your own compiled classes plus any external JARs. Manually tracking classpaths for many third-party JARs (and their own sub-dependencies, and version conflicts) doesn't scale — this is the exact problem Maven solves.

### 3. What is Maven?
Maven is a **project management and build automation tool** for Java — independent of Spring, usable with any Java project. It provides:
1. A standard project structure
2. Java compilation
3. Test execution
4. JAR/WAR packaging
5. Dependency downloading
6. Dependency version management
7. Plugin-based build tasks

Guiding principle: **Convention over Configuration** — Maven assumes a standard layout (`src/main/java`, `src/test/java`, etc.) so you don't need to configure paths manually.

### 4. Standard Maven project structure
```
my-maven-project/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/          # application source code
│   │   └── resources/     # application.properties, static files, etc.
│   └── test/
│       ├── java/          # test code
│       └── resources/     # test-only resources
└── target/                # generated output (compiled classes, final JAR/WAR)
```
`target/` is *generated*, not source — safe to delete; Maven recreates it.

### 5. `pom.xml` — Project Object Model
The heart of a Maven project. Declares project identity, dependencies, plugins, and build config.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.coderarmy</groupId>
    <artifactId>calculator-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
</project>
```

**Maven coordinates** uniquely identify any project/dependency: `groupId` (organization/domain), `artifactId` (project/module name), `version` (`1.0.0` = stable, `1.0.0-SNAPSHOT` = in development).

**Packaging types:** `jar` (default), `war` (traditional web app deployable to an external servlet container), `pom` (parent/aggregator project).

### 6. Dependencies vs starters vs transitive dependencies
A single **starter** dependency (e.g. `spring-boot-starter-web`) pulls in a *group* of related dependencies needed for that feature — you don't add each JAR one by one.

**Transitive dependencies** — if your project needs A, and A needs B, and B needs C, Maven automatically resolves and downloads B and C too. This is one of Maven's biggest wins over manual JAR management.

**Dependency vs Plugin:**
- **Dependency** → library your *application code* uses at runtime/compile-time.
- **Plugin** → tool Maven itself uses *during the build* (e.g. Spring Boot Maven Plugin packages/runs the app).

### 7. Repositories
| Type | What it is |
|---|---|
| **Local repository** | `~/.m2/repository` (Mac/Linux) or `C:\Users\<user>\.m2\repository` (Windows) — caches downloaded JARs and stores your own `mvn install`-ed artifacts |
| **Maven Central** | Huge public repository for open-source Java libraries (Spring, Hibernate, JUnit, etc.) |
| **Remote repository** | Any non-local repo — Maven Central, or a company's private Nexus/Artifactory/GitHub Packages server |

**Resolution order:** Maven reads `pom.xml` → checks local repo first → if missing, checks remote repos → downloads JAR + POM → caches locally → uses it in the build. This is why the *first* build is slower than subsequent ones.

`settings.xml` (inside `.m2/`) is **user/machine-level** config (credentials, proxies, mirrors) — distinct from `pom.xml`, which is **project-level**.

> Deleting a corrupted dependency folder (or the whole `.m2/repository`) forces Maven to re-download it — a common fix for broken/partial downloads.

### 8. Maven Lifecycle
**Golden rule:** running a phase automatically runs *every earlier phase* in that lifecycle first.

Three lifecycles: **Clean**, **Default**, **Site**.

**Default lifecycle phases (in order):**
| Phase | Meaning |
|---|---|
| `validate` | Checks the project/POM is structurally valid |
| `compile` | `src/main/java` → `target/classes` |
| `test` | Runs `src/test/java` |
| `package` | Produces JAR/WAR in `target/` |
| `verify` | Extra checks on the packaged output (integration tests, quality checks) |
| `install` | Copies the artifact into your **local** `.m2` repo |
| `deploy` | Uploads the artifact to a **remote** repo |

`mvn clean` removes `target/`. `mvn clean install` = wipe old output, then run the default lifecycle up through `install`.

A **Maven archetype** is a project template with predefined structure/config — the ancestor of what Spring Initializr does more conveniently today.

---

## Lecture 4 — Why Spring Core? Dependency Injection & IoC

### 1. The problem: tight coupling
Consider a class that creates its own dependency internally:
```java
class OrderService {
    private EmailService emailService = new EmailService(); // created internally
}
```
This is **tight coupling** — `OrderService` is hard-wired to one specific implementation. Problems:
- Can't swap `EmailService` for `SmsService` without editing `OrderService`.
- Hard to unit test (can't inject a mock).
- Violates separation of concerns — object creation logic is mixed with business logic.

### 2. First improvement: program to an interface
```java
interface NotificationService { void send(String msg); }
class EmailService implements NotificationService { ... }
class SmsService implements NotificationService { ... }

class OrderService {
    private NotificationService notificationService = new EmailService(); // still tightly coupled!
}
```
An interface alone isn't enough — `OrderService` still *decides and creates* the concrete implementation itself.

### 3. The real solution: Dependency Injection (DI)
Instead of a class creating its own dependency, the dependency is **created outside and handed (injected) in**:
```java
class OrderService {
    private final NotificationService notificationService;

    OrderService(NotificationService notificationService) { // injected via constructor
        this.notificationService = notificationService;
    }
}
```
**Dependency Injection** = supplying an object's dependencies from the outside rather than letting the object construct them itself.

**Benefits of DI:**
- Loose coupling — depend on abstractions, not concrete classes.
- Easier testing (inject mocks/stubs).
- Flexible — swap implementations without touching consumer code.
- Centralized object creation and wiring.

### 4. Types of Dependency Injection
| Type | How |
|---|---|
| **Constructor Injection** | Dependency passed through the constructor (recommended — enables immutability & guarantees the object is fully formed) |
| **Setter Injection** | Dependency passed through a setter method after construction |
| **Field Injection** | Dependency injected directly into a field (e.g. via `@Autowired` on the field) — least recommended, harder to test |

### 5. Inversion of Control (IoC)
If DI is *how* dependencies get supplied, **IoC** is the broader principle behind it: **the control of creating and managing objects is taken away from your classes and given to a container/framework.**

> Simple definition: normally *your code* controls object creation. With IoC, *control is inverted* — a framework creates, wires, and manages objects for you.

**Relationship:** IoC is the principle; DI is one practical technique that implements it.

### 6. Where Spring fits in
The **Spring IoC Container** is the framework component responsible for creating objects, managing their lifecycle, and injecting dependencies between them, so your classes don't have to do it manually.

**What Spring Core does:**
1. Creates objects (**beans**)
2. Manages those objects' lifecycle
3. Connects (wires/injects) objects together

### 7. What is a Bean?
> A **Bean** is simply an object that is created and managed by the Spring IoC Container (instead of being created manually with `new` by your own code).

### 8. Manual Java flow vs Spring flow
**Manual:**
```java
NotificationService ns = new EmailService();
OrderService os = new OrderService(ns); // you wire everything by hand
```
**Spring:**
```java
// You just declare the classes and their dependencies (via annotations/config).
// The Spring IoC container creates the objects, resolves dependencies,
// and wires them together automatically.
OrderService os = context.getBean(OrderService.class);
```

### Summary
- Web apps work via client–server + HTTP; Servlets and Tomcat handle the low-level plumbing; Spring builds a rich ecosystem on top.
- Spring Boot is Spring, pre-configured with sensible opinionated defaults + embedded server + starters.
- Maven manages project structure, dependencies (including transitive ones), and the build lifecycle so JAR management doesn't have to be manual.
- Spring Core's whole reason to exist is solving **tight coupling** through **Dependency Injection**, implementing the broader principle of **Inversion of Control** — with the **IoC Container** creating and wiring **Beans** on your behalf.

---
*Next file: `02-spring-core-beans.md` — Beans, XML config, circular dependency, scopes, bean lifecycle, and how Spring Boot bootstraps on top of Spring Core.*
