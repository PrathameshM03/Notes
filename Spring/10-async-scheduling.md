# Spring Framework — 10: Async & Scheduling
### New topic. Full depth, story format.

---

## Story 1: The signup that took 3 seconds because of an email

**The situation:** A signup endpoint saves the user, then sends a welcome email:
```java
@PostMapping("/signup")
public ResponseEntity<User> signup(@RequestBody SignupDto dto) {
    User user = userService.createUser(dto);
    emailService.sendWelcomeEmail(user); // synchronous SMTP call — can take 1-3+ seconds
    return ResponseEntity.ok(user);
}
```
**Where it breaks:** the user's browser sits there waiting for a response for however long the email provider takes to respond — which has nothing to do with whether their account was actually created successfully. Worse: if the email provider is having a bad day (slow, timing out, rate-limited), **every signup on the whole site slows down or fails**, even though account creation itself is working perfectly. The email is a genuinely **non-critical, fire-and-forget** side effect being treated as if it were blocking, critical work.

**The Spring answer: `@Async`**
```java
@Configuration
@EnableAsync
public class AsyncConfig { }
```
```java
@Service
public class EmailService {
    @Async
    public void sendWelcomeEmail(User user) {
        // runs on a separate thread — the caller does NOT wait for this to finish
        smtpClient.send(user.getEmail(), "Welcome!", buildBody(user));
    }
}
```
```java
@PostMapping("/signup")
public ResponseEntity<User> signup(@RequestBody SignupDto dto) {
    User user = userService.createUser(dto);
    emailService.sendWelcomeEmail(user); // returns almost immediately; email sends in the background
    return ResponseEntity.ok(user);      // responds right away
}
```
Exactly the same proxy mechanism as `@Transactional`/`@Cacheable` — a method annotated `@Async`, called from *outside* its class, is intercepted by a proxy that **dispatches the actual execution to a separate thread pool**, returning control to the caller immediately. (And yes — **the self-invocation trap applies here too**, exactly as covered in the AOP/Transactional scenario docs: calling an `@Async` method from within the same class runs it synchronously, silently defeating the whole point.)

---

## Story 2: The thread pool that quietly ran out — why `@Async`'s defaults aren't "set and forget"

**The situation:** `@Async` works great in testing. In production, under real signup volume, emails start arriving **minutes** late, and eventually stop being sent at all — no errors in the logs, just... nothing happening.

**What's actually happening:** `@Async` methods run on a thread pool — by default, Spring's `SimpleAsyncTaskExecutor`, which (despite the name suggesting otherwise) **does not actually pool threads at all** — it spawns a **new thread for every single call**, with no limit. Under load, this can exhaust system resources (thread creation isn't free), and separately, Spring Boot's *auto-configured* default executor (when using `@EnableAsync` without further config) has modest default pool sizing that can become a bottleneck — tasks pile up in a queue faster than the pool can drain them, and (depending on the queue's configuration) either grow unbounded (memory risk) or start silently rejecting new tasks past a limit.

**The fix — configure a real, sized thread pool deliberately:**
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);       // threads always kept alive
        executor.setMaxPoolSize(20);       // ceiling under load
        executor.setQueueCapacity(100);    // tasks waiting when all threads are busy
        executor.setThreadNamePrefix("async-email-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) ->
            log.error("Async method {} failed: {}", method.getName(), throwable.getMessage());
    }
}
```
**Why each setting is a real, deliberate decision, not a magic number:**
- **`corePoolSize`/`maxPoolSize`** — too small, and tasks queue up under load (Story's symptom); too large, and you risk overwhelming downstream systems (e.g. hammering the email provider's rate limit) or exhausting server memory/CPU with excessive concurrent threads.
- **`queueCapacity`** — a safety valve; without a bound, a sustained traffic spike could queue unboundedly and eventually run the JVM out of memory.
- **`RejectedExecutionHandler`** — decides what happens when the queue is *also* full (pool maxed **and** queue maxed). `CallerRunsPolicy` makes the **calling thread** execute the task itself instead of throwing it away — a deliberate backpressure mechanism: it slows down the caller (the HTTP request thread, in this case) rather than silently dropping the email entirely.
- **`getAsyncUncaughtExceptionHandler`** — **critical and easy to miss**: exceptions thrown inside a `void`-returning `@Async` method have **nowhere to propagate to** (the original caller already moved on) — without this handler, failures are silently swallowed with **zero log trace**, which is exactly Story 2's original symptom ("emails stop being sent, no errors in the logs").

**Named executors for different concerns:**
```java
@Async("emailExecutor")
public void sendWelcomeEmail(User user) { ... }

@Async("reportExecutor")
public void generateMonthlyReport(Long userId) { ... }
```
```java
@Bean("emailExecutor")
public Executor emailExecutor() { /* small, fast pool — emails are quick */ }

@Bean("reportExecutor")
public Executor reportExecutor() { /* fewer threads, larger queue — reports are slow, CPU-heavy */ }
```
**Why separate pools matter:** if emails and heavy report generation share one thread pool, a burst of slow report requests can **starve** the email pool of threads — the exact "one feature accidentally breaks an unrelated feature" cascade seen with connection-pool exhaustion in the JDBC/Hibernate scenario doc (N+1 story), just at the thread-pool layer instead of the DB-connection layer.

**Interview angle:** *"What can go wrong with `@Async` in production, and how do you configure it correctly?"* — naming the default `SimpleAsyncTaskExecutor`'s unbounded-thread-creation problem, explaining `corePoolSize`/`maxPoolSize`/`queueCapacity` as a deliberate capacity-planning decision (not defaults you leave alone), and specifically flagging the silent-exception-swallowing risk (`AsyncUncaughtExceptionHandler`) shows real production experience.

---

## Story 3: The nightly job that ran twice — `@Scheduled` on multiple instances

**The situation:** A team adds a nightly cleanup job:
```java
@Component
public class CleanupJob {
    @Scheduled(cron = "0 0 2 * * *") // every day at 2 AM
    public void purgeExpiredSessions() {
        sessionRepository.deleteExpiredBefore(Instant.now().minus(30, ChronoUnit.DAYS));
    }
}
```
Works fine while running as a single instance. The team later scales to **3 instances** for redundancy — and now the job runs **3 times** every night, because `@Scheduled` has no built-in awareness of "other instances of this same app are also running this same job."

**Why this is dangerous, not just wasteful:** for an idempotent cleanup (delete-if-expired), running 3 times is harmless, just redundant work. But for a job that's **not naturally idempotent** — e.g. "charge every customer their monthly subscription fee" — running it on 3 instances simultaneously means **charging every customer three times.** This is a direct callback to the idempotency story in the CRUD scenario doc, showing up again at the scheduling layer.

**The fix — distributed locking, so only one instance actually runs the job:**
```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
</dependency>
```
```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulerConfig {
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(dataSource);
    }
}
```
```java
@Component
public class CleanupJob {
    @Scheduled(cron = "0 0 2 * * *")
    @SchedulerLock(name = "purgeExpiredSessions", lockAtLeastFor = "1m", lockAtMostFor = "10m")
    public void purgeExpiredSessions() {
        sessionRepository.deleteExpiredBefore(Instant.now().minus(30, ChronoUnit.DAYS));
    }
}
```
**How this actually works:** `ShedLock` uses a shared table (or Redis, MongoDB, etc.) as a **distributed lock** — when 2 AM arrives on all 3 instances simultaneously, each tries to acquire the same named lock; **only one succeeds**, that one runs the job, and the other two see the lock is already held and skip execution entirely for this run.

**`lockAtMostFor` vs `lockAtLeastFor` — both matter for different failure modes:**
- **`lockAtMostFor`** — a safety ceiling: if the instance holding the lock **crashes** mid-job without releasing it, this ensures the lock is eventually released anyway (after this duration), so the job isn't permanently stuck un-runnable forever.
- **`lockAtLeastFor`** — prevents a *fast-running* job from finishing so quickly that clock-skew between instances lets a **second** instance also grab the lock and run the job again within the same logical time window.

**Interview angle:** *"You have a scheduled job in a multi-instance deployment — what's the risk, and how do you handle it?"* — naming the duplicate-execution problem, giving a concrete example of *why* it's dangerous (not just "wasteful" — actual double-charging), and naming a distributed locking approach (ShedLock, or an equivalent using a DB/Redis lock table) demonstrates you've actually run into this in a real scaled deployment, not just read `@Scheduled`'s Javadoc.

---

## Story 4: `@Async` returning a value — `CompletableFuture`, and why `void` isn't always enough

**The situation:** Beyond fire-and-forget (welcome emails), sometimes you need to run **multiple independent slow operations in parallel** and combine their results — e.g. a dashboard that needs data from three separate slow external APIs, and doesn't want to call them one after another sequentially (3x the total wait time for no reason, since they don't depend on each other).

```java
@Service
public class DashboardService {

    @Async
    public CompletableFuture<UserStats> fetchUserStats(Long userId) {
        return CompletableFuture.completedFuture(userStatsClient.fetch(userId));
    }

    @Async
    public CompletableFuture<OrderStats> fetchOrderStats(Long userId) {
        return CompletableFuture.completedFuture(orderStatsClient.fetch(userId));
    }

    public DashboardDto buildDashboard(Long userId) throws Exception {
        CompletableFuture<UserStats> userStatsFuture = fetchUserStats(userId);   // fires immediately
        CompletableFuture<OrderStats> orderStatsFuture = fetchOrderStats(userId); // fires immediately too, in parallel

        CompletableFuture.allOf(userStatsFuture, orderStatsFuture).join(); // wait for BOTH to finish

        return new DashboardDto(userStatsFuture.get(), orderStatsFuture.get());
    }
}
```
**Why this matters over sequential calls:** if each external call takes ~1 second, sequential calls take **~2 seconds total**; running them concurrently via `@Async` + `CompletableFuture` takes **~1 second total** (bounded by the slowest single call, not the sum of all of them) — a real, measurable latency win for genuinely independent operations.

**Note — this crosses two files' worth of context deliberately:** `fetchUserStats`/`fetchOrderStats` must be called from **outside** `DashboardService`'s own proxy for `@Async` to actually apply — if `buildDashboard` were calling these on `this` within the same class, they'd run synchronously and sequentially anyway, silently losing all the parallelism this pattern is meant to provide (the self-invocation trap, once again — worth internalizing that it shows up in *every* proxy-based Spring feature, not just `@Transactional`).

---

## Quick-fire: async/scheduling concepts worth knowing exist

| Concept | What it is | Why it matters |
|---|---|---|
| **`@Async` + `@Transactional` combined on one method** | Both apply proxy-based interception — order and interaction can get subtle | A transaction started in the *calling* thread does **not** automatically carry over to the new thread `@Async` dispatches to — the async method needs its **own** `@Transactional` if it touches the DB |
| **Cron expression precision** | `@Scheduled(cron = "0 0 2 * * *")` — six fields: second, minute, hour, day-of-month, month, day-of-week | A wrong field order/count is a classic "job ran at the wrong time" bug — worth double-checking against Spring's cron format specifically, which differs slightly from traditional Unix cron |
| **`@Scheduled(fixedDelay=...)` vs `fixedRate=...`** | `fixedDelay` waits N ms **after the previous run finishes** before starting the next; `fixedRate` tries to start every N ms **regardless of whether the previous run finished** | Using `fixedRate` for a job whose duration can exceed the interval risks **overlapping executions** stacking up — usually the wrong choice unless you're certain the job is always fast |
| **Virtual threads (Project Loom, Java 21+)** | A lighter-weight threading model that can reduce the need for careful thread-pool tuning for I/O-bound async work | An emerging alternative worth knowing exists, though the `ThreadPoolTaskExecutor` patterns above remain the standard, broadly-compatible approach as of most current production Spring Boot deployments |

---
*Next: `11-messaging-kafka-rabbitmq.md` — the two-service coupling problem, why direct HTTP calls between services eventually break down, and how message brokers fix it.*
