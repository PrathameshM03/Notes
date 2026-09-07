# Spring Framework — 09: Caching
### New topic. Full depth, story format.

---

## Story 1: The database that fell over from the same query, asked 10,000 times a minute

**The situation:** An e-commerce homepage shows a "Top 10 Products" list, computed by a moderately expensive query joining orders, products, and ratings. It's identical for every visitor — nothing user-specific about it. On a normal day, fine. On a flash-sale day with a traffic spike, the database starts timing out — monitoring shows the exact same query running thousands of times a minute, each one doing real, repeated work to compute an answer that hasn't actually changed since the last time it ran two seconds ago.

**The naive fix — a manual `HashMap`:**
```java
@Service
public class ProductService {
    private final Map<String, List<Product>> cache = new HashMap<>();

    public List<Product> getTopProducts() {
        if (cache.containsKey("top10")) return cache.get("top10");
        List<Product> result = expensiveQuery();
        cache.put("top10", result);
        return result;
    }
}
```
**Where this breaks immediately:**
1. **No expiration** — the cached list is stale forever unless you manually clear it, which nobody remembers to do consistently.
2. **Not thread-safe** — concurrent requests can race on the `HashMap`, at best duplicating work, at worst corrupting it.
3. **Doesn't scale across multiple app instances** — if you run 5 instances behind a load balancer, each has its *own* separate `HashMap` — no shared cache, 5x the redundant work, and each instance can have a *different* cached answer.
4. **Mixing caching logic with business logic** — the same "cross-cutting concern" problem from the AOP story (Doc/file 5's Story 3 in the AOP course notes) — every method that wants caching now needs this same boilerplate rewritten.

**The Spring answer: the Cache abstraction + `@Cacheable`**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```
```java
@Configuration
@EnableCaching
public class CacheConfig { }
```
```java
@Service
public class ProductService {
    @Cacheable("topProducts")
    public List<Product> getTopProducts() {
        return expensiveQuery(); // only actually runs on a cache MISS
    }
}
```
**This is the exact same mechanism as `@Transactional`/AOP** — `@Cacheable` is implemented via a **proxy** wrapping `ProductService`. On each call, the proxy checks the cache **before** letting the real method run at all; on a hit, the real method's body never executes. (This also means: **the self-invocation trap applies here too** — calling `getTopProducts()` from another method in the same class bypasses the cache entirely, for exactly the reason explained in the AOP scenario docs.)

---

## Story 2: The stale-price bug — cache invalidation, the "hard problem"

**The situation:** `@Cacheable` on `getProduct(id)` works beautifully — until a customer complains they were charged the *old* price after an admin updated it an hour ago. The cache is still serving the pre-update value.

**Why this happens:** `@Cacheable` only handles the **read** side — it has no idea the underlying data changed via a separate `updateProduct()` call. Caching without an invalidation strategy is not "free performance," it's a **correctness bug waiting to happen** the moment the underlying data changes.

**The fix — `@CacheEvict` and `@CachePut`, tied to the actual write operations:**
```java
@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "products", key = "#product.id") // update the cache with the NEW value
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", key = "#id") // remove the (now stale) entry
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    @CacheEvict(value = "topProducts", allEntries = true) // invalidate a computed/aggregate cache entirely
    public void recalculateTopProducts() { ... }
}
```
- **`@CachePut`** — *always* runs the real method (unlike `@Cacheable`), and puts its result into the cache — used for writes that should immediately refresh the cached value.
- **`@CacheEvict`** — removes an entry (or, with `allEntries = true`, wipes an entire cache region) — used when the "correct new value" isn't easily computable inline, so it's simpler to just invalidate and let the next read recompute it.

**Why this is genuinely called "one of the two hard problems in computer science" (the joke: cache invalidation, naming things, and off-by-one errors):** every write path that touches cached data has to remember to evict/update the *right* cache keys — miss even one write path (a bulk import script, a database trigger, a different microservice writing to the same table), and you have a silent, intermittent staleness bug that's brutal to reproduce because it depends on cache timing, not application logic.

**Interview angle:** *"What's the hardest part of adding caching to an existing system?"* — naming invalidation specifically (not "picking a cache library") and giving a concrete example (a write path that bypasses the cache-aware service method — e.g. a batch job writing directly to the DB) shows real understanding.

---

## Story 3: One app instance vs five — why "in-memory cache" isn't always enough

**The situation:** The `@Cacheable` examples above work great with Spring's default, simple in-memory `ConcurrentMapCacheManager`. Then the app scales to 5 instances behind a load balancer for a traffic spike.

**Where it breaks:** each of the 5 instances has its **own separate in-memory cache**. Two problems:
1. **5x redundant work** — the same expensive query gets computed once per instance instead of once total.
2. **Inconsistent invalidation** — an admin updates a product on the instance handling their request; that instance's cache is correctly evicted — but the **other 4 instances** still have the old, now-stale value cached, with no way to know they need to evict it too.

**The fix — an external, shared cache: Redis**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
```
No code changes to `@Cacheable`/`@CacheEvict` — Spring's cache abstraction is exactly designed so **swapping the underlying cache provider is a configuration change, not a rewrite** (the same "program against an abstraction" theme, again). Now all 5 instances share **one** Redis cache: one instance computes the expensive query once, all 5 benefit from the cached result, and an eviction on any one instance is instantly visible to all of them, because they're all reading from the same external store.

**Trade-off worth naming:** an in-memory cache is faster (no network hop to Redis) but doesn't scale across instances; Redis adds a small network round-trip per cache access but solves both the redundant-work and the invalidation-consistency problems for multi-instance deployments. **For a single-instance app, in-memory is genuinely fine** — reaching for Redis before you actually have multiple instances is often premature complexity.

---

## Story 4: `time-to-live` — the safety net for invalidation bugs you didn't catch

**The situation:** Despite careful `@CacheEvict` placement, a team still occasionally ships a write path that forgets to invalidate the cache (Story 2's core risk never fully goes away — it's a discipline problem, not a solved problem).

**The pragmatic mitigation — TTL (time-to-live):**
```properties
spring.cache.redis.time-to-live=600000  # 10 minutes, in milliseconds
```
```java
@Cacheable(value = "products", key = "#id")
// this cache region expires after 10 minutes regardless of whether anyone explicitly evicted it
public Product getProduct(Long id) { ... }
```
**Why this matters as a *design* decision, not just a config knob:** TTL puts a **hard upper bound** on how stale any cached value can ever be, even if every explicit invalidation path has a bug. It converts "this cache entry might be stale forever if we missed an eviction" into "this cache entry is stale for **at most** 10 minutes" — a much safer failure mode. Choosing the TTL value is itself a real trade-off: too short, and you lose most of the performance benefit (cache misses become frequent again); too long, and staleness windows get uncomfortably wide for data that changes moderately often.

**Interview angle:** *"How do you decide on a cache TTL?"* — the strong answer frames TTL as a deliberate trade-off between staleness tolerance and cache hit-rate, tied to **how often the underlying data actually changes** and **how costly staleness is for that specific piece of data** (a "top products" list tolerates 10 minutes of staleness fine; an account balance does not).

---

## Quick-fire: caching concepts worth knowing exist

| Concept | What it is | When it matters |
|---|---|---|
| **Cache stampede / thundering herd** | Many concurrent requests all miss the cache at the exact same moment (e.g. right after a TTL expiry) and all hit the database simultaneously | A real risk for very hot cache keys — mitigated with request coalescing/locking so only one request recomputes while others wait for that result |
| **Write-through vs write-behind caching** | Write-through updates cache + DB synchronously on every write; write-behind updates the cache immediately and writes to the DB asynchronously later | Write-behind is faster but risks data loss if the app crashes before the async DB write completes — a genuine availability-vs-durability trade-off |
| **`@Caching`** | Combines multiple `@Cacheable`/`@CachePut`/`@CacheEvict` on one method when a single write needs to touch multiple cache regions | Useful when one update needs to both refresh one cache and evict a related, differently-keyed one |
| **Cache key design (`key = "#id"` vs composite keys)** | SpEL expressions can build composite keys, e.g. `key = "#userId + '-' + #productId"` | Getting key design wrong (e.g. two different logical queries accidentally sharing one cache key) causes subtle correctness bugs, similar to Story 2 but harder to spot |

---
*Next: `10-async-scheduling.md` — the request that blocks for 3 seconds sending an email, `@Async`, `@Scheduled`, and why thread pool configuration is where async silently goes wrong.*
