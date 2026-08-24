# Scenarios & Deeper Reasoning — 4: The Web Layer's Trick Questions
### Companion to `04-web-layer-mvc-dto-exceptions-config.md`

---

## Story 1: CORS vs CSRF — the classic interview confusion

**The situation:** A frontend (`https://app.example.com`) calls a backend API (`https://api.example.com`). The browser blocks the request with a console error mentioning "CORS policy." A week later, after fixing CORS, a *different* error shows up on a POST request: `403 Forbidden`, this time about a missing CSRF token. Two different errors, both about "cross-origin" something — easy to conflate, and interviewers know it.

**What CORS actually is:** **Cross-Origin Resource Sharing** is a **browser-enforced** rule that says: *by default, JavaScript running on origin A cannot read responses from origin B, unless origin B explicitly says it's allowed.* It exists to protect the **response** — stopping a malicious site from silently reading data back from an API you're logged into.

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://app.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowCredentials(true);
    }
}
```
**Key fact that trips people up:** CORS is enforced by the **browser**, not the server. A server always *executes* the request — CORS only controls whether the **browser lets JavaScript read the response**. Tools like Postman or `curl` never enforce CORS, which is why "it works in Postman but not in the browser" is a classic CORS symptom.

**What CSRF actually is:** **Cross-Site Request Forgery** protects against a different attack entirely: you're logged into `bank.com` (with a session cookie). You visit a malicious site, which secretly submits a form to `bank.com/transfer` — and **your browser automatically attaches your session cookie** to that request, because cookies are sent to whatever domain they belong to, regardless of which page triggered the request. `bank.com`'s server sees a perfectly valid, authenticated-looking request... that you never actually intended to send.

**The core distinction, stated precisely:**
| | CORS | CSRF |
|---|---|---|
| Protects | The **response** (stopping JS from reading data it shouldn't) | The **action** (stopping an unintended state-changing request from being executed) |
| Enforced by | The browser | The server (via a token it checks) |
| Relevant when | Frontend and backend are on different origins, using JS `fetch`/`XHR` | Any browser-based session-cookie-authenticated app, regardless of origin |
| Fixing it means | Configuring `allowedOrigins` etc. on the server | Requiring a CSRF token on state-changing requests |

**Why disabling CSRF for a JWT-based stateless API is actually correct, not a shortcut:** CSRF exploits **cookies**, because browsers attach them automatically without the attacker needing to know the value. A JWT sent via `Authorization: Bearer <token>` header is **not** automatically attached by the browser — the malicious site's forged request wouldn't have the token, so it would simply fail authentication. This is *why* Lecture 38's JWT security config disables CSRF (`csrf.disable()`) — it's not a corner being cut, it's recognizing the attack CSRF protection defends against **doesn't apply** to header-based, stateless auth.

**Interview angle:** *"What's the difference between CORS and CSRF?"* — an extremely common question specifically *because* the names sound similar and both involve "cross-origin" behavior. The strong answer leads with "they protect different things" (response vs. action) and "they're enforced by different parties" (browser vs. server), then explains why a stateless JWT API can safely disable CSRF while still needing careful CORS configuration.

---

## Story 2: The custom validator — when built-in annotations aren't enough

**The situation:** A signup form needs `password` and `confirmPassword` to match. `@NotBlank` and friends validate *one field at a time* — there's no built-in annotation for "these two fields must be equal to each other."

**The naive attempt:** check it manually in the service layer:
```java
public void register(RegistrationDto dto) {
    if (!dto.getPassword().equals(dto.getConfirmPassword())) {
        throw new IllegalArgumentException("Passwords don't match");
    }
    // ... rest of registration
}
```
**Where this is worse than it looks:** this check now lives *outside* the declarative validation system Lecture 16 set up — it won't show up in the same structured `ValidationErrorResponseDto` the global exception handler builds for `@Valid` failures, so the frontend gets an inconsistent error shape for this one specific rule versus every other validation rule.

**The Spring-idiomatic fix — a custom class-level constraint:**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchesValidator.class)
public @interface PasswordMatches {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PasswordMatchesValidator implements ConstraintValidator<PasswordMatches, RegistrationDto> {
    @Override
    public boolean isValid(RegistrationDto dto, ConstraintValidatorContext context) {
        return dto.getPassword() != null && dto.getPassword().equals(dto.getConfirmPassword());
    }
}

@PasswordMatches
public class RegistrationDto {
    @NotBlank private String password;
    @NotBlank private String confirmPassword;
}
```
Now this rule flows through the **exact same** `@Valid` → `MethodArgumentNotValidException` → global exception handler pipeline as every built-in annotation — consistent error shape, no special-casing in the controller or service.

**Interview angle:** *"How would you validate that two fields match, or implement a business rule Spring's built-in annotations don't cover?"* — naming `ConstraintValidator` + a custom annotation (rather than "just check it in the service") shows you understand *why* keeping validation declarative and centralized matters, not just that it's possible to write an `if` statement somewhere.

---

## Story 3: Validation groups — the same DTO, different rules for create vs. update

**The situation:** `StudentRequestDto` is reused for both `POST /students` (create) and `PUT /students/{id}` (update). On create, `email` is required. On update, the client might only send the fields they're changing — requiring `email` on *every* update request forces clients to always resend it even when unchanged.

**The naive attempt:** create two nearly-identical DTOs, `StudentCreateDto` and `StudentUpdateDto`, duplicating every field and annotation. Works, but doubles maintenance for every field change.

**The Spring-idiomatic fix — validation groups:**
```java
public interface OnCreate {}
public interface OnUpdate {}

public class StudentRequestDto {
    @NotBlank(groups = OnCreate.class)
    private String email;   // required only when creating

    @Size(min = 2, groups = {OnCreate.class, OnUpdate.class})
    private String name;    // required in both, when present
}
```
```java
@PostMapping("/create")
public ResponseEntity<?> create(@Validated(OnCreate.class) @RequestBody StudentRequestDto dto) { ... }

@PutMapping("/update")
public ResponseEntity<?> update(@Validated(OnUpdate.class) @RequestBody StudentRequestDto dto) { ... }
```
**Why `@Validated` and not `@Valid` here:** `@Valid` (from `jakarta.validation`) doesn't support groups — you need Spring's own `@Validated` (from `org.springframework.validation.annotation`) to pass a specific group class. This is a small but real gotcha: mixing them up (using `@Valid` and expecting group-scoping to work) silently validates against the **default** group instead, applying every annotation regardless of the group you thought you specified.

**Interview angle:** *"How do you apply different validation rules to the same DTO for create vs. update?"* — validation groups is the textbook answer; a sharp follow-up is knowing `@Valid` doesn't support groups and `@Validated` does, which is exactly the kind of detail that separates "read about it" from "used it."

---

## Story 4: API versioning — the question that comes up once you ship a breaking change

**The situation:** `GET /api/students/{id}` has been live for a year, used by a mobile app that can't be force-updated instantly. The team needs to change the response shape (e.g. splitting `name` into `firstName`/`lastName`) — a breaking change for existing mobile clients still expecting the old shape.

**Why "just change the endpoint" breaks things:** old app versions in the wild will suddenly get a response shape they don't understand, and — because you can't force every user to update their app instantly — you need **both shapes to coexist** for some transition period.

**The main strategies, with real trade-offs (not just "here are four options"):**

**1. URI versioning** — `/api/v1/students/{id}` vs `/api/v2/students/{id}`
```java
@RestController
@RequestMapping("/api/v1/students")
public class StudentControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/students")
public class StudentControllerV2 { ... }
```
*Pro:* dead simple, visible in logs/URLs, easy to route/cache differently per version. *Con:* "the URL is supposed to identify a resource, not a format" (a REST-purist objection) — and you often end up duplicating a lot of controller/service code between versions.

**2. Header versioning** — `Accept: application/vnd.company.v2+json`, same URL for every version.
*Pro:* keeps URLs clean/stable. *Con:* much harder to test manually (you can't just paste a URL in a browser), harder to see which version a request is hitting from logs alone.

**3. Query parameter versioning** — `/api/students/{id}?version=2`
*Pro:* easy to add on top of an existing API. *Con:* mixes "which resource" with "which format" in a way that's easy to get wrong (caching proxies may not treat different query params as genuinely different resources, causing a stale-cache bug across versions).

**Why most real teams pick URI versioning despite the "purist" objection:** operational simplicity wins in practice — it's trivially visible in logs, load-balancer routing rules, API gateways, and monitoring dashboards which version is being hit, which matters enormously when you're trying to figure out "is it safe to finally delete v1?"

**Interview angle:** *"How would you version a REST API, and why?"* — naming all three strategies with their real trade-offs (not just listing them) shows you've thought about the *operational* side (logging, routing, caching), which is what actually differentiates versioning approaches in practice, more than any REST-purity argument.

---

## Quick-fire: gaps between the course's DTO/exception pattern and production reality

| Gap | The course's version | What production usually adds |
|---|---|---|
| **Error response `traceId`** | `ErrorResponseDto` has status/message/path/timestamp | Production error responses often add a unique `traceId`/`correlationId` so a user can report "error XZ192" and you can grep logs for that exact request across distributed services |
| **Mapping boilerplate** | Manual field-by-field DTO↔Entity mapping | Real projects usually use **MapStruct** (compile-time-generated mappers) to eliminate this boilerplate and its associated bugs (forgetting to map a field) |
| **`@ExceptionHandler(Exception.class)` fallback** | Returns a generic message | Production versions also **log the full stack trace** server-side (even though the client only sees a generic message) — otherwise you have a 500 error with zero debugging trail |

---
*Next companion doc: `scenarios-05-filters-interceptors-aop.md` — tying the self-invocation trap (Story 1 of doc 2) into real AOP/Transactional/Async bugs, and a filter-ordering incident.*
