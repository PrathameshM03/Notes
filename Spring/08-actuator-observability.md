# Spring Framework — 08: Observability with Spring Boot Actuator
### New topic, not covered by the original course. Full depth, story format.

---

## Story 1: The 3 AM page — "the app is down," but nobody knows why

**The situation:** A Spring Boot service is running in production behind a load balancer. At 3 AM, the on-call engineer gets paged: "API is returning errors." They SSH into the server. Is the app even running? Is it the database connection? Is it out of memory? Is it just one instance out of five that's unhealthy? With nothing but application logs to grep through, this investigation takes 20 minutes of guesswork before the actual fix even starts.

**The naive approach:** add a `/ping` endpoint yourself:
```java
@GetMapping("/ping")
public String ping() { return "OK"; }
```
**Where it breaks down:** this tells you the app can respond to *an* HTTP request — it says **nothing** about whether the database connection is alive, whether disk space is critically low, whether a downstream service the app depends on is reachable, or what the current memory/thread state looks like. A load balancer polling `/ping` will happily keep routing traffic to an instance that can technically respond "OK" while its database connection pool is completely exhausted.

**The Spring answer: Spring Boot Actuator.** Add one dependency:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
This exposes a family of production-ready **management endpoints** — `/actuator/health`, `/actuator/metrics`, `/actuator/info`, `/actuator/env`, and more — without you writing any of them by hand.

```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```
**Why not expose everything by default?** Some actuator endpoints (`/actuator/env`, `/actuator/heapdump`, `/actuator/beans`) can leak sensitive configuration (database passwords in environment variables!) or expose internal application structure — so Spring Boot deliberately exposes only `health` and `info` by default, and you opt into the rest explicitly.

---

## Story 2: `/actuator/health` — going beyond "the process is running"

```java
GET /actuator/health
{
  "status": "UP",
  "components": {
    "db": { "status": "UP", "details": { "database": "MySQL", "validationQuery": "isValid()" } },
    "diskSpace": { "status": "UP", "details": { "total": 500000000000, "free": 320000000000 } },
    "ping": { "status": "UP" }
  }
}
```
Spring Boot auto-configures **health indicators** for components it detects on the classpath — a `DataSource` gets a DB connectivity check automatically, disk space gets checked automatically, and so on. This directly answers Story 1's real question ("is the DB actually reachable, not just is the JVM alive") without you writing that check yourself.

### Custom health indicators
```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    private final RestTemplate restTemplate;

    @Override
    public Health health() {
        try {
            restTemplate.getForObject("https://payment-provider.com/health", String.class);
            return Health.up().withDetail("provider", "reachable").build();
        } catch (Exception e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```
Now `/actuator/health` reports `DOWN` if your **external payment provider** is unreachable — turning "the app looks fine but checkout is silently broken" into an immediately visible, monitorable signal. This is the real payoff: health checks aren't just "is my app alive," they're "are the things my app depends on alive too."

### Liveness vs Readiness — the Kubernetes-relevant distinction
```properties
management.endpoint.health.probes.enabled=true
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```
- **`/actuator/health/liveness`** — "is the application in a state where it should be **restarted** if this fails?" (e.g. deadlocked, JVM-level failure). Kubernetes uses this to decide whether to **kill and restart** a pod.
- **`/actuator/health/readiness`** — "is the application ready to **receive traffic** right now?" (e.g. still warming up a cache, or a downstream dependency is temporarily down). Kubernetes uses this to decide whether to **route traffic** to a pod, *without* killing it.

**Why this distinction matters — a scenario:** an app depends on a downstream service that's temporarily down for maintenance. If you only had one combined "health" check, the app would report `DOWN`, and Kubernetes would **restart it repeatedly** — pointlessly, since restarting doesn't fix the downstream outage, and the app might genuinely be fine internally. Splitting liveness (restart-worthy) from readiness (traffic-worthy) lets the orchestrator correctly decide: **don't restart, just stop sending traffic until the dependency recovers.**

---

## Story 3: `/actuator/metrics` and Micrometer — answering "how slow, exactly?"

**The situation:** Health checks answer "is it up or down." They don't answer "the checkout endpoint has gotten 40% slower over the last hour" — a degradation, not an outage, that still needs to be caught before it becomes a full outage.

**Spring Boot's answer: Micrometer**, an instrumentation facade bundled with Actuator, which exposes metrics like request counts, latencies, JVM memory usage, and thread pool stats:
```
GET /actuator/metrics/http.server.requests
{
  "name": "http.server.requests",
  "measurements": [
    { "statistic": "COUNT", "value": 12500 },
    { "statistic": "TOTAL_TIME", "value": 340.5 },
    { "statistic": "MAX", "value": 2.1 }
  ]
}
```
### Custom application metrics
```java
@Service
public class OrderService {
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCreatedCounter = Counter.builder("orders.created").register(registry);
        this.orderProcessingTimer = Timer.builder("orders.processing.time").register(registry);
    }

    public void createOrder(Order order) {
        orderProcessingTimer.record(() -> {
            orderRepository.save(order);
            orderCreatedCounter.increment();
        });
    }
}
```
Now you have a **business metric** (`orders.created` — how many orders are actually going through, not just how many HTTP requests hit the endpoint) alongside infrastructure metrics — this is the difference between "the server is technically fine" and "we're actually processing 30% fewer orders than usual," which is often the *real* signal something is wrong, well before any health check would ever go red.

### Why Micrometer, and not just "print numbers to logs"
Micrometer is a **vendor-neutral facade** — the same `Counter`/`Timer` code can be exported to **Prometheus, Datadog, New Relic, CloudWatch**, or others, just by swapping a dependency, without touching your instrumentation code:
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
```properties
management.endpoints.web.exposure.include=prometheus
```
This is the same "program to an abstraction, not a concrete implementation" principle from Spring Core's DI story (Doc 1), applied to metrics/monitoring vendors instead of payment providers.

---

## Story 4: Custom `/actuator/info` — closing the loop on "which version is actually deployed?"

**The situation:** A bug report comes in. The team needs to know: is this bug present in the version currently running in production, or was it already fixed and just not yet deployed? Without an easy way to check, someone has to dig through deployment logs or CI history.

```properties
management.info.env.enabled=true
info.app.version=@project.version@
info.app.name=@project.artifactId@
```
```java
@Component
public class GitInfoContributor implements InfoContributor {
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("git", Map.of(
            "commit", System.getenv("GIT_COMMIT_SHA"),
            "branch", System.getenv("GIT_BRANCH")
        ));
    }
}
```
```
GET /actuator/info
{ "app": { "version": "1.4.2", "name": "order-service" }, "git": { "commit": "a1b2c3d", "branch": "main" } }
```
Now anyone can hit `/actuator/info` on any running instance and know **exactly** what code is deployed there — turning "which version is this?" from a support-ticket-worthy investigation into a 5-second `curl`.

---

## Story 5: Securing Actuator — the endpoint that becomes a liability if left open

**The situation:** A team exposes `management.endpoints.web.exposure.include=*` (everything) for convenience during debugging, and forgets to lock it down before shipping to production. Months later, a security scan finds `/actuator/env` publicly accessible — and it's dumping every environment variable, **including the database password**, to anyone who requests the URL.

**Why this is a real, common mistake:** Actuator endpoints are just normal Spring MVC endpoints by default — they follow whatever `SecurityFilterChain` rules you've set up, but it's extremely easy to write a security config that protects `/api/**` carefully while forgetting `/actuator/**` even exists, especially since it's added by a starter dependency rather than code you wrote yourself.

**The fix — treat actuator endpoints as a distinct, deliberately-configured security zone:**
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/actuator/health", "/actuator/info").permitAll() // safe for load balancers
        .requestMatchers("/actuator/**").hasRole("ADMIN")                  // everything else, locked down
        .anyRequest().authenticated());
    return http.build();
}
```
Common real-world pattern: run actuator's **sensitive** endpoints on a **separate management port**, only reachable from an internal network (not the public internet at all):
```properties
management.server.port=9001
```
This means even a misconfigured `SecurityFilterChain` on the main app port can't accidentally expose actuator data, because it's physically listening somewhere else.

**Interview angle:** *"How do you monitor a Spring Boot application in production?"* — naming Actuator + Micrometer specifically (health checks with custom indicators, metrics exported to Prometheus/Grafana, liveness vs readiness for Kubernetes) shows production experience, and mentioning the security lockdown unprompted shows you've actually thought about the failure mode, not just the happy path.

---

## Quick-fire: other actuator endpoints worth knowing exist

| Endpoint | What it shows | Why it's sensitive |
|---|---|---|
| `/actuator/env` | All environment properties/variables | Can leak secrets (DB passwords, API keys) if not locked down |
| `/actuator/beans` | Every Spring bean in the application context | Reveals internal application structure — useful for debugging, risky if public |
| `/actuator/threaddump` | A snapshot of every JVM thread's state | Useful for diagnosing deadlocks/hangs in production |
| `/actuator/heapdump` | A full JVM heap dump | Can contain sensitive in-memory data (session tokens, cached user data) — never expose publicly |
| `/actuator/loggers` | View/change log levels **at runtime**, without redeploying | Extremely useful for debugging a live production issue ("temporarily bump this package to DEBUG") without a restart |

---
*Next: `09-caching.md` — the "same query runs 10,000 times a minute" problem, `@Cacheable`, Redis, and the invalidation trade-offs that make caching "one of the two hard problems in computer science."*
