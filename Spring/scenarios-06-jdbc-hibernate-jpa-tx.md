# Scenarios & Deeper Reasoning — 6: Data Access, Where Small Mistakes Get Expensive
### Companion to `06-jdbc-hibernate-jpa-transactions.md`

---

## Story 1: The N+1 bug that took down a dashboard (a realistic timeline)

**The situation:** An internal admin dashboard shows a list of 50 orders per page, each with its associated customer name. Works fine in dev with a handful of test records. Ships. Three months later, with 40,000 real orders in the database, the page takes **11 seconds** to load, and the DB monitoring dashboard shows a spike to thousands of queries per second whenever someone opens that page.

**What's actually happening:**
```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```
```java
List<Order> orders = orderRepository.findAll(); // 1 query — fine
for (Order o : orders) {
    dto.add(new OrderSummaryDto(o.getId(), o.getCustomer().getName())); // triggers lazy load, EACH TIME
}
```
For a 50-row page: **1 query to fetch orders, plus 50 more queries, one per order, to fetch each customer** — 51 queries for what should be a single page load. This is exactly the N+1 problem from Lecture 31, but seeing the *actual production timeline* is more instructive than the abstract definition:

1. **Dev/testing:** 5 test orders → 6 queries total → feels instant, nobody notices.
2. **Early production:** 200 orders, paginated at 50/page → 51 queries per page load → still "fast enough," maybe 200ms, still unnoticed.
3. **Growth:** 40,000 orders, same pagination → still only 51 queries per *page* (pagination limits this, thankfully) — but now the *customer* table itself has grown, indexes are under more pressure, connection pool is shared with other growing features, and 51 queries per request, multiplied across concurrent dashboard users, starts exhausting the connection pool (HikariCP's default pool size is often just 10) — **other, unrelated features start timing out too**, because they can't get a DB connection.

**The real lesson:** N+1 doesn't "break" at a fixed data size — it degrades gradually and then **cascades**: connection pool exhaustion from one badly-fetching feature starves *every other* feature sharing that pool. This is why N+1 bugs are notorious for shipping silently and then causing outages that look, at first, completely unrelated to the actual root cause.

**The fix, applied properly:**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
Page<Order> findAllWithCustomer(Pageable pageable);
```
Now it's **1 query total** for the whole page, regardless of how many orders are on it. `JOIN FETCH` tells Hibernate "when you fetch `Order`, eagerly pull in `customer` in the *same* SQL query" — turning N+1 queries into exactly 1.

**Detecting it before production, not after:** enable `spring.jpa.show-sql=true` (or, better, a proper SQL logging/metrics tool like **p6spy** or **Hibernate statistics**) in a staging environment seeded with realistic data volumes — the N+1 pattern (one query, followed by a burst of near-identical queries) is visually obvious in logs once you know to look for it, but invisible in a 5-row dev database.

**Interview angle:** *"Tell me about a performance bug you'd expect to encounter with JPA, and how you'd both detect and fix it."* — N+1 is the textbook answer, but a strong response includes the *why it's dangerous specifically* (connection pool exhaustion cascading to unrelated features), not just "it's slow."

---

## Story 2: `LazyInitializationException` — the exception every JPA beginner hits eventually

**The situation:** A controller does this:
```java
@GetMapping("/{id}")
public Student getStudent(@PathVariable Long id) {
    return studentRepository.findById(id).orElseThrow(); // returns the raw entity
}
```
`Student` has a lazy `@OneToMany` list of `Enrollment`s. Jackson (the JSON serializer) tries to serialize the entire `Student` object, including calling the getter for `enrollments` to write it into the JSON response — and the app throws:
```
org.hibernate.LazyInitializationException: could not initialize proxy - no Session
```

**Why this happens — connecting back to Lecture 29's persistence context:** the lazy `enrollments` collection is a **proxy** — Hibernate hasn't actually loaded it yet, and loading it requires an active `EntityManager`/session. But by the time Jackson tries to serialize the response (outside the repository method, back up in the web layer), **the transaction (and the persistence context/session with it) has already closed** — the repository method returned, the transaction committed, the session is gone. The lazy proxy has nowhere to fetch its data from anymore.

**Why this is precisely the "two worlds" problem Lecture 16 introduced (entities vs. DTOs), made concrete:** this exception is one of the clearest, most common real-world reasons **never to return a JPA entity directly from a controller** — not just "it's bad practice," but "it will literally crash with an exception the moment a lazy relationship is touched outside a transaction."

**Fixes, in order of how commonly they're actually used:**
1. **Always map to a DTO inside the service layer, while the transaction/session is still open** (the Lecture 16 pattern) — access the lazy collection *inside* the `@Transactional` service method, converting it to a DTO *before* returning, so the controller/Jackson never touches the raw lazy proxy at all. This is the standard, recommended fix.
2. **`JOIN FETCH` or `@EntityGraph`** the relationship up front if you know you'll need it — same fix as the N+1 story, and it also happens to sidestep this exception since the data's already loaded.
3. **`FetchType.EAGER`** — technically "fixes" it by always loading the relationship, but reintroduces N+1-style problems for *every* query on that entity, even ones that never needed the relationship. Generally discouraged as the primary fix.
4. **`@Transactional` on the controller method itself** (keeping the session open longer, sometimes called "Open Session In View") — works, but is broadly considered an anti-pattern: it keeps DB connections/transactions open for the *entire* HTTP request/response cycle (including JSON serialization, which can be slow for large payloads), tying up connection-pool resources for longer than necessary — this is directly connected to Story 1's connection-pool-exhaustion risk.

**Interview angle:** *"What causes `LazyInitializationException`, and how do you prevent it?"* — extremely common. The strong answer explains the session-closed-before-access mechanism (not just "lazy loading failed"), and names DTO-mapping-inside-the-transaction as the standard fix, while being able to explain *why* "Open Session In View" is considered an anti-pattern rather than just naming it as an alternative.

---

## Story 3: Two customers buy the last item at the same time — optimistic vs. pessimistic locking

**The situation:** An e-commerce app has exactly 1 unit of a product left in stock. Two customers click "Buy" within the same millisecond.
```java
@Transactional
public void purchase(Long productId) {
    Product product = productRepository.findById(productId).orElseThrow();
    if (product.getStock() > 0) {
        product.setStock(product.getStock() - 1); // dirty checking will flush this UPDATE
        orderRepository.save(new Order(productId));
    } else {
        throw new OutOfStockException();
    }
}
```
**Where it breaks — a classic race condition:** both transactions can read `stock = 1` **before either commits** (this is exactly what transaction **isolation**, from Lecture 33's ACID properties, is supposed to prevent — but the *default* isolation level in most databases, READ COMMITTED, doesn't fully prevent this specific race by itself). Both see stock available, both proceed, both decrement — **stock goes negative, or two orders are created for one unit of inventory.**

**Fix option 1 — Optimistic Locking (`@Version`):**
```java
@Entity
public class Product {
    @Id private Long id;
    private int stock;
    @Version
    private int version; // Hibernate manages this automatically
}
```
Hibernate adds `WHERE id = ? AND version = ?` to every UPDATE, and increments `version` on each successful update. If Transaction A reads `version = 5` and commits an update, `version` becomes `6`. If Transaction B *also* read `version = 5` (before A committed) and tries to update, its `WHERE version = 5` matches **zero rows** (since it's now `6`) — Hibernate detects this as **zero rows affected** and throws `OptimisticLockException`. **The application must catch this and retry** (or tell the user "someone else just bought this, please retry").

*Why "optimistic":* it **assumes conflicts are rare** and only checks for them at commit time — cheap in the common case (no lock held while the user is "thinking"), but requires retry logic for the conflict case.

**Fix option 2 — Pessimistic Locking:**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdForUpdate(@Param("id") Long id);
```
This issues a `SELECT ... FOR UPDATE` — the database **physically locks the row** the moment it's read. The second transaction trying to read the same row **blocks and waits** until the first transaction commits or rolls back, rather than racing ahead and failing later.

*Why "pessimistic":* it **assumes conflicts are likely** (or the cost of a conflict is too high to risk) and prevents them up front by serializing access — safer in high-contention scenarios, but **hurts throughput** (other transactions literally wait in line) and carries real deadlock risk if lock-acquisition order isn't carefully controlled across different code paths.

**Choosing between them — the actual decision, not just a definition:**
| Scenario | Preferred lock | Why |
|---|---|---|
| Low-contention resource (most rows are rarely fought over) | Optimistic | Avoids the overhead of locking for the (common) non-conflict case |
| High-contention "hot" resource (flash sale, limited inventory drop) | Pessimistic | Retrying over and over under heavy contention is wasteful; better to just wait in line once |
| User-editable long-lived records (e.g. a shared document) | Optimistic | Locking a row while a user "thinks" for minutes would be disastrous for throughput |
| A financial ledger balance update | Pessimistic (often) | Correctness under concurrent writes matters more than throughput; conflicts are common and costly to get wrong |

**Interview angle:** *"How would you prevent two users from overselling the last item in stock?"* — a strong answer names both strategies, explains the underlying mechanism (version column + zero-rows-affected detection vs. `SELECT FOR UPDATE` row locking), and picks one **based on the specific scenario's contention level**, rather than presenting them as interchangeable.

---

## Story 4: `@Transactional` on a public method, called from a private helper inside the same class — the sneaky variant

**The situation:** revisiting Doc 5's self-invocation story with one more twist worth knowing:
```java
@Service
public class InventoryService {

    @Transactional
    public void adjustStock(Long productId, int delta) {
        Product product = fetch(productId); // private helper, fine — no annotation on it
        product.setStock(product.getStock() + delta);
        productRepository.save(product);
    }

    private Product fetch(Long productId) {
        return productRepository.findById(productId).orElseThrow();
    }
}
```
This one is **fine** — `adjustStock` is the entry point called *from outside* the class (through the proxy), so its own `@Transactional` fires correctly; the private `fetch()` helper doesn't need its own transaction, it's just running inside the transaction `adjustStock` already started. **The self-invocation problem only bites when the annotated method itself is the one being called internally** — not when a plain internal helper is called *from within* an already-correctly-proxied method. Knowing this distinction (rather than becoming afraid of *all* internal calls) prevents over-engineering simple, correct code out of misplaced caution.

**Interview angle:** interviewers sometimes probe whether you *overcorrect* — showing you can tell "this internal call is fine" from "this internal call breaks the proxy" demonstrates real understanding rather than a memorized rule applied too broadly.

---

## Quick-fire: JPA/Hibernate concepts worth knowing exist

| Concept | What it is | Why it's not in the course's core path |
|---|---|---|
| **Second-level cache** (Ehcache/Redis + Hibernate) | Caches entities **across sessions/transactions**, unlike the first-level (persistence-context-scoped) cache | Adds real complexity (cache invalidation across a cluster of app instances) — only worth it for read-heavy, rarely-changing data at scale |
| **`@BatchSize`** | Tells Hibernate to fetch lazy collections in batches (e.g. 10 at a time) instead of one row at a time | A middle ground between full N+1 and full `JOIN FETCH` — useful when eager-fetching everything would pull in too much data, but N+1 is still too slow |
| **`EntityManager.merge()` vs `persist()`** | `persist()` makes a transient entity managed; `merge()` copies a detached entity's state onto a managed one | Rarely needed directly once you're using Spring Data JPA's `save()`, which internally picks the right one based on whether the entity has an ID |
| **Native SQL + `@SqlResultSetMapping`** | Mapping raw SQL results to custom (non-entity) DTOs directly | Useful for complex reporting queries where JPQL/entity mapping gets in the way — the course only briefly mentions native queries |

---
*Next companion doc: `scenarios-07-security-testing.md` — method-level security, refresh tokens & the revocation trade-off, and the test pyramid in practice.*
