# Spring Framework — 12: Microservices & Spring Cloud
### New topic. Full depth, story format.

---

## Story 1: "Just call the other service's URL" — until the URL isn't stable anymore

**The situation:** A company splits its monolith into `order-service` and `inventory-service`. The simplest possible integration:
```java
@Service
public class OrderService {
    private final RestTemplate restTemplate = new RestTemplate();

    public void reserveStock(Order order) {
        restTemplate.postForObject("http://192.168.1.50:8081/api/inventory/reserve", order, Void.class);
    }
}
```
**Where it breaks:** the moment `inventory-service` is deployed to a new server (a new container, a new pod, an autoscaling event), its IP changes — and this hardcoded URL is now pointing at nothing. In a cloud/container environment, instances are **constantly** being created and destroyed (deploys, autoscaling, crash-restarts) — hardcoding an address is fundamentally incompatible with that reality, not just a minor inconvenience.

**Why "just use a load balancer with a fixed DNS name" only partly helps:** it solves "which specific instance" but still requires **someone to register/deregister instances** with that load balancer as they come and go — and if that's a manual step, it lags behind reality (a new instance isn't receiving traffic yet; a dead instance is still receiving traffic it can't answer).

---

## Story 2: Service Discovery — Eureka, solving "where is everyone right now?"

**The situation:** What if every service, on startup, **announced itself** to a central registry — "I'm `inventory-service`, and I'm at `10.0.4.22:8081`" — and, on shutdown (or a health-check failure), got automatically removed?

```xml
<!-- Eureka Server (the registry itself, its own small Spring Boot app) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication { ... }
```
```xml
<!-- Every OTHER service (a "Eureka Client") -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
```properties
# in inventory-service's application.properties
spring.application.name=inventory-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```
Now `OrderService` doesn't need to know `inventory-service`'s address at all — it just needs to know its **name**:
```java
@Service
public class OrderService {
    private final RestTemplate restTemplate;

    @LoadBalanced // makes RestTemplate Eureka-aware — resolves service names, not raw IPs
    @Bean
    public RestTemplate restTemplate() { return new RestTemplate(); }

    public void reserveStock(Order order) {
        restTemplate.postForObject("http://inventory-service/api/inventory/reserve", order, Void.class);
        // "inventory-service" is resolved via Eureka to a live instance's actual address, automatically
    }
}
```
**What this actually buys you, concretely, tying back to Story 1:** when `inventory-service` scales from 1 instance to 5, or gets redeployed with new IPs entirely, **zero code changes anywhere else** — every caller keeps saying `http://inventory-service/...`, and Eureka + the load-balanced client keep resolving that to whichever instances are currently alive and registered.

**`@LoadBalanced`, specifically — what it adds:** without it, `RestTemplate`/`WebClient` just makes a normal HTTP call to whatever literal hostname you give it — `@LoadBalanced` intercepts the call, looks up `inventory-service` in the service registry, picks one of the (possibly many) healthy instances (typically round-robin by default), and routes the call there — this is **client-side load balancing**, distinct from a traditional external load balancer sitting in front of the services.

---

## Story 3: Centralized Configuration — the "which DB URL does staging use again?" problem

**The situation:** With 10 microservices, each has its own `application.properties` with environment-specific values (DB URLs, feature flags, third-party API keys). A config change (say, rotating an API key) now means **editing and redeploying 10 separate services** — slow, error-prone, and easy to miss one.

**The fix — Spring Cloud Config Server:** a dedicated service that serves configuration to every other service, backed by a Git repository (so config changes are version-controlled, reviewable, and auditable, just like code):
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { ... }
```
```properties
# Config Server's own application.properties
spring.cloud.config.server.git.uri=https://github.com/company/config-repo
```
```properties
# inventory-service's bootstrap.properties — points to the config server instead of its own local file
spring.application.name=inventory-service
spring.config.import=configserver:http://localhost:8888
```
```
config-repo/
├── inventory-service.yml       # config specific to inventory-service
├── inventory-service-prod.yml  # prod overrides — same profile mechanism from Lecture 18
└── application.yml             # shared defaults for ALL services
```
**Why this is genuinely different from "just put config in a shared file everyone reads":** it comes with the **profile mechanism already familiar from Lecture 18** (`-prod`, `-dev` overrides), version control/audit trail via Git, and — with the optional `spring-cloud-bus` add-on — the ability to **push a config change to every running instance without redeploying anything at all**, via a lightweight broadcast message.

---

## Story 4: The API Gateway — one door, not ten

**The situation:** A frontend team needs to call `order-service`, `inventory-service`, `user-service`, and 7 others. Should the frontend know all 10 individual URLs, handle auth separately for each, and implement CORS config 10 times over?

**The fix — an API Gateway** (Spring Cloud Gateway): a single entry point that routes incoming requests to the correct backend service, and centralizes cross-cutting concerns:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service           # "lb://" = resolve via load-balancer/service-discovery
          predicates:
            - Path=/api/orders/**
        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
      default-filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 100
            redis-rate-limiter.burstCapacity: 200
```
**What consolidating at the gateway actually buys you:**
- **Frontend simplicity** — one base URL (`api.company.com`) instead of tracking 10 service addresses.
- **Centralized JWT validation** — validate the token **once**, at the gateway, instead of duplicating Lecture 38's JWT filter chain config in every single downstream service.
- **Centralized rate limiting** — protects the whole system's front door from abuse, rather than each service defending itself independently (and inconsistently).
- **Centralized CORS config** — one place, not ten.

**The trade-off worth naming honestly:** the gateway becomes a **single point of failure and a potential bottleneck** — if it goes down, the *entire system* is unreachable from outside, even if every individual microservice is perfectly healthy. This is why gateways are typically deployed with **multiple redundant instances** behind their own load balancer, and kept as **lightweight and stateless** as possible (routing + a few centralized cross-cutting concerns — not business logic, which belongs in the actual services).

---

## Story 5: The cascading failure — why one slow service took the whole system down, and Resilience4j's answer

**The situation:** `order-service` calls `inventory-service` on every checkout. `inventory-service` starts responding slowly (a DB issue on *its* end) — not down, just slow, taking 30 seconds per call instead of 200ms.

**Where it cascades:** every `order-service` thread handling a checkout request is now stuck **waiting** on the slow `inventory-service` call. Under load, `order-service`'s own thread pool fills up entirely with requests stuck waiting — and `order-service` **itself** becomes unresponsive, even though *its own* code and database are perfectly healthy. The failure has **cascaded** from one slow service to an entirely unrelated, otherwise-healthy one — this is the distributed-systems analog of the connection-pool-exhaustion cascade from the N+1 story in the JPA scenario doc, just one layer up the stack.

**The fix — Circuit Breaker pattern, via Resilience4j:**
```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```
```java
@Service
public class OrderService {

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "reserveStockFallback")
    @TimeLimiter(name = "inventoryService")
    public CompletableFuture<Void> reserveStock(Order order) {
        return CompletableFuture.runAsync(() -> inventoryClient.reserve(order));
    }

    public CompletableFuture<Void> reserveStockFallback(Order order, Throwable t) {
        // don't let the whole checkout fail — queue it for retry, or degrade gracefully
        pendingReservationQueue.add(order);
        return CompletableFuture.completedFuture(null);
    }
}
```
```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        sliding-window-size: 20
        failure-rate-threshold: 50    # if 50%+ of the last 20 calls failed...
        wait-duration-in-open-state: 30s # ...stop calling it entirely for 30 seconds
  timelimiter:
    instances:
      inventoryService:
        timeout-duration: 2s          # give up waiting after 2 seconds, don't hang forever
```
**How a circuit breaker actually behaves, in plain terms** (the electrical-circuit analogy it's named for): normally **closed** (calls flow through normally). If failures exceed the configured threshold, it **opens** — for the next `wait-duration-in-open-state`, calls to `inventory-service` **fail immediately** (triggering the fallback) *without even attempting* the network call — protecting `order-service`'s own threads from getting stuck waiting on a service that's known to be struggling. After the wait period, it goes **half-open**, allowing a few test calls through to check if `inventory-service` has recovered — if they succeed, it closes again (normal operation resumes); if they fail, it reopens.

**Why "fail fast" is *better* than "keep trying, hope it works," here:** without the circuit breaker, every request still tries the full 2-second timeout against a service that's already known (from very recent history) to be failing — burning threads and resources for a call that's very likely to fail anyway. Failing immediately via the fallback keeps `order-service` itself healthy and responsive, and lets its own threads keep serving requests that *don't* depend on the struggling service, instead of blocking everything indiscriminately.

**Interview angle:** *"How do you prevent one failing microservice from taking down the services that depend on it?"* — the strong answer names the **circuit breaker pattern specifically** (not just "add a timeout," which only partially helps — a 2-second timeout under heavy concurrent load can still exhaust a thread pool if *every* request pays that 2 seconds before failing), explains the closed/open/half-open state machine, and connects it back to the cascading-failure mechanism it's specifically designed to prevent.

---

## Quick-fire: microservices concepts worth knowing exist

| Concept | What it is | Why it matters |
|---|---|---|
| **Saga pattern** | Managing a business transaction that spans multiple services (no single DB transaction can cover them all) via a sequence of local transactions + compensating actions on failure | The distributed-systems answer to "how do you roll back an order **and** a payment **and** an inventory reservation across three separate services if one step fails" — ACID transactions (Lecture 33) don't cross service boundaries |
| **Distributed tracing (e.g. Sleuth/Zipkin, or OpenTelemetry)** | Tags a request with a trace ID that follows it across every microservice it touches | Without it, debugging "why was this specific user's checkout slow" across 10 services means correlating separate logs by timestamp and guesswork — with it, one trace ID shows the whole request's journey |
| **Strangler Fig pattern** | Gradually migrating a monolith to microservices by routing an increasing share of traffic to new services while the old monolith still handles the rest | The realistic, incremental alternative to "rewrite the whole monolith as microservices at once," which is a notoriously high-risk, high-failure-rate approach |
| **Database-per-service** | Each microservice owns its own database; no service reads another's DB directly | Enforces the same "depend on an interface, not an implementation detail" principle from Doc 1's DI story, applied at the service level — prevents services becoming secretly, tightly coupled through a shared schema |

---
*Next: `13-api-docs-migrations-uploads.md` — Swagger/OpenAPI, Flyway vs. `ddl-auto`, and handling file uploads correctly.*
