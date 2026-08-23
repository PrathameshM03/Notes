# Spring Framework Full Course — Notes 4: The Web Layer
### Covers Lectures 14–18: Servlets & Tomcat internals → Spring MVC architecture → DTOs & Validation → Global Exception Handling → YAML & Spring Profiles

---

## Lecture 14 — Servlets: Complete Masterclass

### 1. The gap Servlets fill
A browser only speaks HTTP text — it has no idea a `HelloController.java` exists. Something must **listen on a port**, receive raw HTTP text, parse it, and route it to Java logic. Java *can* do this manually (`ServerSocket`, `java.net`), but building a full web server this way means manually: reading raw HTTP text, parsing the URL/headers/query params/body, identifying the HTTP method, building a correctly formatted response, and handling many concurrent users via threads. That's exactly the repetitive, error-prone plumbing Servlets exist to remove.

### 2. Normal program vs web application
A normal Java program (`main()`) runs once and exits. A **web application must run continuously**, listening for and responding to many concurrent requests — it needs a long-running server environment, not just a class with a `main()` method.

### 3. Static vs dynamic websites
- **Static** — the server returns an already-existing file as-is (HTML/CSS/JS/images) — no computation needed.
- **Dynamic** — the response depends on input (e.g. `GET /user?id=101` needs to look up user 101 specifically). This requires actual logic: read input → query a data source → build a response. Java (and by extension Servlets) exists to power this dynamic layer.

### 4. What a Servlet is
> A **Servlet** is a Java class that handles HTTP requests and responses, running inside a **Servlet Container**, without the developer manually managing sockets, threads, or raw HTTP parsing.

Key clarifications:
- A Servlet **does not directly listen on port 8080** — the container does that.
- The **browser is the client**; the **server is the receiver** — same client-server model from Lecture 1.
- **Control is inverted**: instead of your code calling the server, the *container* calls *your* servlet method when a matching request arrives (the core IoC idea, applied to the web layer).

### 5. Tomcat = Servlet Container
> **Tomcat is a Servlet Container** — an environment that creates, manages, and controls Servlet objects, and handles the surrounding infrastructure: request/response object creation, thread management, URL mapping, and sending the response back to the browser.

The word **container** here is the same concept as the Spring IoC container managing beans — Tomcat manages servlets the way Spring manages beans.

**What happens when Tomcat starts:** it initializes and prepares to receive requests (it does **not** necessarily instantiate every servlet immediately — servlet instantiation timing depends on configuration). **What Tomcat does per request:** matches the URL to the correct servlet, creates request/response objects, invokes the servlet method on a thread, and sends the response back.

### 6. External Tomcat vs Embedded Tomcat
**Traditional (external Tomcat) flow:**
1. Install Tomcat on your machine/server.
2. Package your Java web app as a **WAR file** (Web Application Archive — the web-app equivalent of a JAR).
3. Drop the WAR into Tomcat.
4. Start Tomcat — it deploys and runs your app.
One Tomcat instance can host multiple WAR-deployed web applications simultaneously.

**Embedded Tomcat (Spring Boot's default):** Spring Boot bundles Tomcat *inside* your application via `spring-boot-starter-web` — there's no separate install/configure/deploy step. Running `java -jar app.jar` starts Tomcat *as part of* your app process. This is why Spring Boot apps are easy to run standalone and are a natural fit for microservices.

The Servlet dependency is typically declared with `<scope>provided</scope>` in a WAR-based project — meaning the Servlet API classes are needed to *compile* against but are supplied by the container at *runtime* (so they shouldn't be bundled into the WAR itself). Skipping `provided` can bloat the WAR or cause classloading conflicts with the container's own Servlet API classes.

### 7. From Servlets to Spring MVC
Handling raw request bodies, manually parsing JSON, and managing the Servlet lifecycle methods (`init()`, `service()`, `destroy()`) for every endpoint doesn't scale. **Spring MVC's idea:** instead of many hand-written servlets (one per URL), use **one central Servlet** — the `DispatcherServlet` — that receives *every* request and dispatches it to the right controller method. This is the topic of Lecture 15.

```
Traditional Servlet style:                 Spring MVC style:
/users    → UserServlet                    All requests → DispatcherServlet
/products → ProductServlet                    → routed internally to the matching
/orders   → OrderServlet                        @Controller method
```

---

## Lecture 15 — Spring MVC Architecture

### 1. What is Spring MVC?
**Spring Web MVC** (module `spring-webmvc`) is Spring's web framework for building web apps, REST APIs, and HTTP backends. It's built **on top of the Servlet API** — you don't write `doGet()`/`doPost()` yourself, but under the hood the request still flows through servlet-based handling. **Spring Boot doesn't replace Spring MVC — it auto-configures it** (Tomcat setup, context creation, `DispatcherServlet` registration/mapping, component scanning, JSON conversion), so you can just write:
```java
@GetMapping("/{id}")
public Student getStudent(@PathVariable Integer id) {
    return studentService.getStudent(id);
}
```

### 2. `DispatcherServlet` — the front controller
> `DispatcherServlet` is the **single front controller** of Spring MVC. It receives every incoming HTTP request and coordinates the complete request-processing flow — it is itself a Servlet (so Tomcat knows how to invoke it), but instead of *you* writing per-URL servlets, Spring routes everything through this one central dispatcher.

```
Client → Tomcat → DispatcherServlet → Controller method → Response
```

### 3. High-level Spring MVC flow
1. **`HandlerMapping`** — Spring creates controller beans, reads their `@RequestMapping`/`@GetMapping`/etc. annotations, and builds an internal table mapping URL patterns → controller methods.
2. **`HandlerAdapter`** — knows *how* to actually invoke the matched controller method (handles the mechanics of calling it with the right arguments).
3. **Argument resolution** — request data is extracted and bound to method parameters:
   - **Path variable** (`@PathVariable`) — from the URL path (`/students/{id}`).
   - **Query parameter** (`@RequestParam`) — from `?key=value`.
   - **Request body** (`@RequestBody`) — the JSON payload, converted into a Java object by Jackson (Spring's default JSON converter, wired in automatically by the web starter).
4. The controller method runs and returns a value.
5. Spring converts that return value into the HTTP response (JSON for `@RestController`, or forwards to a view for traditional MVC).

### 4. Manual Spring MVC vs Spring Boot
Building Spring MVC *without* Boot requires you to manually: create the `ApplicationContext`, register `DispatcherServlet`, map it to a URL pattern, wire up component scanning, and configure a JSON message converter. Spring Boot's auto-configuration (from Lecture 9) does every one of these steps for you as soon as `spring-boot-starter-web` is on the classpath — that's the entire reason a Boot web app "just works" with almost no configuration.

### 5. `ViewResolver` — for traditional (non-REST) MVC
When a controller returns a *view name* (a String like `"home"`) instead of raw JSON/data (i.e. it's **not** annotated `@RestController`, just `@Controller`), Spring needs to know **which actual template/JSP file** that name corresponds to. A **`ViewResolver`** maps a logical view name to a physical resource — e.g. `"home"` → `/WEB-INF/views/home.jsp`. This lecture demonstrates a small JSP-based MVC project (`JSPDemo`) alongside the REST-style project to contrast the two.

### 6. REST API vs traditional MVC — the key difference
- `@RestController` = `@Controller` + `@ResponseBody` on every method — the return value **is** the HTTP response body (typically JSON).
- `@Controller` (without `@ResponseBody`) — the return value is treated as a **view name**, resolved via `ViewResolver` to render an HTML page (JSP, Thymeleaf, etc.).

### Final MVC meaning
**M**odel (data), **V**iew (presentation, for traditional web apps), **C**ontroller (request handling) — Spring MVC's real value is providing this single, well-organized, automatically-wired request pipeline (`DispatcherServlet` → `HandlerMapping` → `HandlerAdapter` → your method → response) instead of hand-rolled per-URL servlets.

---

## Lecture 16 — DTOs and Validation

### 1. The real problem: exposing the Entity directly
Early CRUD examples (Lectures 11–13) accept and return the JPA `@Entity` object directly in the controller. This causes several problems:
1. **Client can send fields they shouldn't** — e.g. a client could set `id` or `deleted` directly in a create request.
2. **Client can modify sensitive/internal data** — fields meant to be server-controlled leak into the writable surface.
3. **Response may leak unwanted data** — internal-only fields end up serialized back to the client.
4. **Request shape ≠ Response shape**, usually — what you accept when creating something often differs from what you return.
5. **Database schema changes ripple straight into the API contract** — if the Entity *is* the API shape, changing a DB column changes your public API by accident.

### 2. Two worlds: persistence vs API contract
> An **Entity** represents the database table shape. A **DTO (Data Transfer Object)** represents the shape of data flowing across the API boundary — they're allowed, even expected, to differ.

### 3. Types of DTOs
- **Request DTO** — the shape the client is allowed to *send* (e.g. `StudentRequestDto` with only `name`, `age`, `email`, `subject`, `rollNo` — no `id`, no `deleted`).
- **Response DTO** — the shape the server *returns* (e.g. `StudentResponseDto` — can omit internal fields, or add computed/derived fields).

### 4. Mapping between DTOs and Entities
Two conversion points are needed:
- **Request DTO → Entity** (before saving) — happens typically in the service layer.
- **Entity → Response DTO** (before returning) — also typically in the service layer, keeping the controller and repository unaware of DTO-mapping details.

```java
// Request DTO -> Entity
Student student = new Student();
student.setName(dto.getName());
student.setAge(dto.getAge());
// ... etc.

// Entity -> Response DTO
StudentResponseDto response = new StudentResponseDto();
response.setId(student.getId());
response.setName(student.getName());
// ... etc.
```
(In real projects this mapping is often delegated to a mapping library like MapStruct or ModelMapper to reduce boilerplate — the course keeps it manual here for clarity.)

### 5. Validation
Without validation, invalid data (blank name, negative age, malformed email) can reach the service/DB layer. **Bean Validation** (JSR-380 / `spring-boot-starter-validation`) lets you declare rules directly on the DTO:

```java
public class StudentRequestDto {
    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 1, message = "Age must be positive")
    private int age;
}
```
Common validation annotations: `@NotNull`, `@NotBlank`, `@NotEmpty`, `@Size`, `@Min`/`@Max`, `@Email`, `@Pattern`.

**Triggering validation in the controller:**
```java
@PostMapping("/create")
public ResponseEntity<StudentResponseDto> createStudent(@Valid @RequestBody StudentRequestDto dto) {
    ...
}
```
**What happens internally when `@Valid` is present:**
1. JSON is deserialized into `StudentRequestDto`.
2. Because of `@Valid`, Spring runs the declared validation annotations against the object.
3. If any field fails validation, the **service method is never called**.
4. Spring throws `MethodArgumentNotValidException`, and (by default) an error response goes back to the client automatically.

**Default status code for validation failure:** `400 Bad Request`. The default Spring Boot error body is generic and not very informative — which is exactly the motivation for **global exception handling** (Lecture 17), where you can customize the validation error response shape and add field-specific messages.

### Summary: Entity vs DTO
| | Entity | DTO |
|---|---|---|
| Represents | Database table structure | API request/response contract |
| Annotated with | `@Entity`, `@Id`, JPA mapping annotations | Validation annotations (`@NotBlank`, etc.) |
| Exposed to client? | Ideally never directly | Yes, by design |
| Changes when | DB schema changes | API contract changes |

---

## Lecture 17 — Exception Handling in Spring Boot

### 1. What a professional API response looks like
A response has three parts: **status code**, **headers**, **body**. Beginner CRUD APIs often return raw strings/booleans on error ("not found", `false`) with inconsistent or missing status codes — professional APIs return **consistent, structured error bodies** with the right status code every time.

### 2. Spring Boot's default error handling
By default, an uncaught exception produces Spring Boot's built-in generic error page/JSON (timestamp, status, error, message, path) — functional, but not tailored to your API's needs, and it's the *same shape* regardless of what actually went wrong (a validation failure and a "resource not found" look too similar).

### 3. Key HTTP status codes for CRUD APIs
| Code | Meaning | Typical trigger |
|---|---|---|
| 200 OK | Success | GET/PUT succeeded |
| 201 Created | Resource created | POST succeeded |
| 400 Bad Request | Malformed/invalid input | Validation failure, bad JSON, wrong path-variable type |
| 404 Not Found | Resource doesn't exist | GET/UPDATE/DELETE on missing ID |
| 409 Conflict | State conflict | Duplicate resource (e.g. email already registered) |
| 500 Internal Server Error | Unexpected failure | Uncaught/unexpected exception |

### 4. `ResponseEntity` — full control over the response
```java
return ResponseEntity.status(HttpStatus.CREATED).body(createdStudent);
return ResponseEntity.ok(student);
return ResponseEntity.notFound().build();
```
`ResponseEntity<T>` lets you set the status code, headers, *and* body explicitly instead of relying on defaults — the mental model is "I am building the entire HTTP response myself."

### 5. The problem with handling errors inside the controller
Doing it inline (null checks, deciding status codes, building an error body by hand, in every single controller method) causes: repetition, less readable controllers, a weak service layer (services return `null`/`false` instead of communicating *what* went wrong), inconsistent error shapes across endpoints, and poor separation of concerns.

### 6. Global Exception Handling — the fix
> Convert "some exception occurred somewhere in the backend" into "a proper, consistent HTTP error response" — **in one central place**, not scattered across every controller.

**`@ControllerAdvice`** — a Spring annotation that lets one class provide common cross-cutting behavior for multiple controllers (exception handling, among other things).

**`@RestControllerAdvice`** — shortcut for `@ControllerAdvice` + `@ResponseBody` (so returned objects are serialized as JSON, matching REST API conventions).

**`@ExceptionHandler`** — placed on a method inside the advice class, declares which exception type that method handles:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponseDto> handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponseDto error = new ErrorResponseDto(
            HttpStatus.NOT_FOUND.value(), ex.getMessage(), request.getRequestURI(), LocalDateTime.now());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(Exception.class) // catch-all fallback
    public ResponseEntity<ErrorResponseDto> handleGeneric(Exception ex, HttpServletRequest request) {
        ErrorResponseDto error = new ErrorResponseDto(
            HttpStatus.INTERNAL_SERVER_ERROR.value(), "Something went wrong", request.getRequestURI(), LocalDateTime.now());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```
`HttpServletRequest` is injected into the handler method to read the request path (`getRequestURI()`) for inclusion in the error body.

### 7. Custom exceptions
Instead of throwing generic `RuntimeException`, define meaningful custom exceptions and throw them from the **service layer**:
```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) { super(message); }
}
```
```java
// in the service:
Student student = studentRepository.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("Student not found with id: " + id));
```
This **cleans up the controller** entirely — it just calls the service and lets exceptions propagate to the global handler; no manual null-checking or status-code decisions live in the controller anymore.

### 8. Handling validation errors globally
`MethodArgumentNotValidException` (thrown automatically by `@Valid`) is handled centrally too, extracting **field-level** errors into a structured response:
```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ValidationErrorResponseDto> handleValidation(MethodArgumentNotValidException ex) {
    Map<String, String> fieldErrors = new HashMap<>();
    ex.getBindingResult().getFieldErrors()
        .forEach(err -> fieldErrors.put(err.getField(), err.getDefaultMessage()));
    ValidationErrorResponseDto response = new ValidationErrorResponseDto(HttpStatus.BAD_REQUEST.value(), fieldErrors);
    return ResponseEntity.badRequest().body(response);
}
```

### 9. `400` vs `409` — different meanings
- **400 Bad Request** — the *request itself* is malformed or fails validation (missing field, wrong type, bad JSON).
- **409 Conflict** — the request is well-formed, but it **conflicts with existing server state** (e.g. creating a student with an email that's already registered) — handled via a dedicated `DuplicateResourceException`.

### Categories of exceptions worth distinguishing
1. **Business exceptions** — domain rule violations (duplicate resource, invalid state transition).
2. **Request/input exceptions** — validation failures, malformed JSON, wrong path-variable type.
3. **Unexpected exceptions** — anything uncaught, unforeseen — caught by a generic `Exception.class` fallback handler so the client never sees a raw stack trace.

### Best practices recap
- Never let the entity leak into error responses.
- Throw meaningful custom exceptions from the service layer.
- Handle **all** exception translation in one `@RestControllerAdvice` class.
- Always return a consistent error DTO shape (status, message, path, timestamp) — and for validation, per-field messages.

---

## Lecture 18 — Spring Boot Configuration: YAML & Profiles

### 1. Why not just `application.properties` forever?
Flat `payment.provider=Razorpay` style keys get repetitive and hard to read once configuration grows (nested groups, lists, multiple environments). **YAML** (`application.yml`) offers a more structured, hierarchical way to express the same configuration.

### 2. YAML syntax basics
```yaml
payment:
  provider: Razorpay
  retry-count: 3
  enabled: true
  timeout: 5000
```
**Indentation defines nesting** — YAML is whitespace-sensitive (use spaces, not tabs). Reading via `@ConfigurationProperties`/`@Value` works identically whether the source is `.properties` or `.yml` — Spring Boot's `Environment` abstraction treats them the same way once loaded.

**Lists in YAML:**
```yaml
cities:
  - Delhi
  - Mumbai
  - Bangalore
```
**List of objects:**
```yaml
warehouses:
  - name: North
    city: Delhi
  - name: South
    city: Bangalore
```
Reading via dot notation / index in `@Value` (e.g. `${cities[0]}`) or, more commonly, binding a whole structured list into a `@ConfigurationProperties` class field of type `List<String>` / `List<SomeType>`.

### 3. The multi-environment problem
Real applications run in multiple environments — dev, test, prod — each needing **different config values** (different DB URLs, ports, log levels, feature flags) but the **same code**. Without a mechanism for this, you'd end up manually editing config before every deploy — error-prone and dangerous (e.g. accidentally deploying dev DB credentials to prod).

### 4. Spring Profiles — the solution
Create **profile-specific configuration files** alongside the common one:
```
application.yml           # common/shared config
application-dev.yml       # dev overrides
application-test.yml      # test overrides
application-prod.yml      # prod overrides
```
When a profile is active, Spring Boot loads `application.yml` **plus** the matching profile file, and **profile-specific values override common values** where keys overlap.

**Activating a profile — several ways:**
```properties
# in application.properties / .yml
spring.profiles.active=dev
```
```bash
# command line
java -jar app.jar --spring.profiles.active=prod
```
```bash
# via Maven
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```
```bash
# environment variable (common in production/containers — avoids baking secrets into files)
export SPRING_PROFILES_ACTIVE=prod
```
Environment variables are especially useful in production/CI because sensitive or environment-specific values don't need to live in a checked-in config file at all.

### 5. `@Profile` — conditionally register a whole bean
Beyond just swapping config *values*, you can make an entire **bean** exist only for a specific profile:
```java
@Component
@Profile("dev")
public class MockPaymentGateway implements PaymentGateway { ... }

@Component
@Profile("prod")
public class RealPaymentGateway implements PaymentGateway { ... }
```
This is the annotation-driven equivalent of swapping implementations per environment — Spring only creates the bean matching the active profile(s).

### 6. Profile-specific files vs `@Profile` — when to use which
| Use... | When... |
|---|---|
| **Profile-specific properties/YAML files** | You need different **values** per environment (ports, URLs, flags) |
| **`@Profile` on a bean/class** | You need entirely different **implementations/beans** per environment |

### Common mistakes to avoid
- Forgetting to set `spring.profiles.active` and being surprised the "common" config alone doesn't include environment-specific overrides.
- Hardcoding production secrets directly into a committed `application-prod.yml` instead of using environment variables.
- Mixing tabs and spaces in YAML (causes parse errors).

### Final mental model
```
application.yml (common)
    + application-<active-profile>.yml (overrides)
    = effective configuration for this run
```
Same compiled code, different behavior per environment — purely through configuration and (optionally) profile-scoped beans.

---
*Next file: `05-filters-interceptors-aop.md` — Cross-cutting concerns: Servlet Filters, Spring MVC Interceptors, and a deep dive into Spring AOP (advice types, pointcuts, custom annotations, proxies).*
