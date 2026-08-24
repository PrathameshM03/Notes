# Scenarios & Deeper Reasoning — 3: Config & CRUD, The Way Production Actually Bites
### Companion to `03-spring-boot-crud-config.md`

---

## Story 1: The double-charge bug — why idempotency matters

**The situation:** A payment API has `POST /api/payments` to charge a customer's card. A mobile user is on a flaky connection: they tap "Pay," the request goes through and charges them, but the *response* never reaches their phone (network drops). The app shows a spinner, times out, and — because the developer built a "retry on timeout" feature — automatically **resends the exact same POST request**.

**Where it breaks:** `POST` is, by HTTP convention, **not idempotent** — each POST is treated as "create a new thing." The server has no way to know "this is a retry of the same intent" vs. "the user genuinely wants to pay twice" — so it processes it again. **The customer is charged twice.**

**Why this isn't really a "REST semantics" problem, it's a design problem:** knowing that POST is "not idempotent" (a course-notes-level fact) doesn't automatically make your API safe — you have to *design for retries*, because in any real distributed system (mobile networks, load balancers, service-to-service calls), retries are inevitable, not exceptional.

**The Spring/backend fix — idempotency keys:**
```java
@PostMapping("/api/payments")
public ResponseEntity<PaymentResponse> pay(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    Optional<PaymentResponse> existing = idempotencyStore.get(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get()); // return the SAME result, don't re-charge
    }

    PaymentResponse response = paymentService.charge(request);
    idempotencyStore.save(idempotencyKey, response);
    return ResponseEntity.ok(response);
}
```
The client generates a unique key **once** per logical payment attempt (often a UUID) and sends the *same* key on every retry. The server checks "have I already processed this exact key?" before doing any real work — turning an inherently non-idempotent operation into a safely-retryable one.

**Why this specific approach (over alternatives):**
- "Just don't let the client retry" — unrealistic; mobile networks and distributed systems retry constantly, often outside your control (load balancers, service mesh retries).
- "Make payments idempotent by nature" — impossible for a true "create" operation; two genuinely separate `$10` charges to the same card in the same second are legitimate and must both go through.
- The idempotency-key pattern is the industry-standard answer precisely because it lets the *client* express intent ("this is one attempt, however many times I resend it") without the server needing to guess.

**Interview angle:** *"Is POST idempotent? Why does it matter?"* — a strong answer doesn't stop at "no, POST is not idempotent" — it connects that fact to a concrete failure mode (double-charge, duplicate order) and names the idempotency-key pattern as the real-world mitigation. This is a very common systems-design-adjacent question for backend roles.

---

## Story 2: PUT vs PATCH — the confusion that ships bugs

**The situation:** A frontend team builds an "edit profile" form. A backend developer implements it as:
```java
@PutMapping("/api/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody UserUpdateDto dto) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(dto.getName());
    user.setEmail(dto.getEmail());
    user.setPhone(dto.getPhone());
    return userRepository.save(user);
}
```
The frontend only sends the field the user actually changed (say, just `email`) to save bandwidth: `{ "email": "new@mail.com" }`.

**Where it breaks:** Because `PUT` semantically means **"replace the entire resource"**, and this implementation blindly overwrites `name` and `phone` with whatever the DTO contains — which, since the frontend didn't send them, deserializes as `null`. **The user's name and phone number silently get wiped out** on every "just update my email" request.

**Why this happened — a semantics mismatch, not a typo:** the *HTTP verb* the team chose (`PUT`) implied full-resource replacement, but the *actual behavior they wanted* was a partial update. This is an extremely common real bug, not a hypothetical.

**The fix — pick the verb that matches the actual operation:**
```java
@PatchMapping("/api/users/{id}")
public User partiallyUpdateUser(@PathVariable Long id, @RequestBody UserPatchDto dto) {
    User user = userRepository.findById(id).orElseThrow();
    if (dto.getName() != null) user.setName(dto.getName());
    if (dto.getEmail() != null) user.setEmail(dto.getEmail());
    if (dto.getPhone() != null) user.setPhone(dto.getPhone());
    return userRepository.save(user);
}
```
`PATCH` semantically means **"apply a partial modification"** — and the implementation matches that by only touching fields that were actually present in the request.

**When you genuinely want PUT:** replacing an entire resource where the client is expected to send the *complete* representation every time (e.g. "save this whole settings object back exactly as edited") — PUT's "wipe and replace" behavior is then the *correct*, intended behavior, not a bug.

**Interview angle:** *"When would you use PUT vs PATCH?"* — the strong answer isn't a dictionary definition, it's this exact bug story: PUT replaces the whole resource (so partial payloads silently null out missing fields), PATCH applies only the fields present — and picking the wrong one is a real, easy-to-ship data-loss bug, not just a style preference.

---

## Story 3: The soft-delete production incident — when "safety" backfires

**The situation:** Following the course's Lecture 13 pattern, a team adds a `deleted` boolean to `Student` and updates every read query to filter `WHERE deleted = false`. Feels safe — nothing is ever "really" gone.

**Where it breaks, months later:**
1. **Every new query anyone writes must remember the filter.** A junior developer adds a new report query directly against the `students` table (or a native SQL query) and forgets `deleted = false` — the report silently includes "deleted" students, confusing stakeholders.
2. **Unique constraints stop working correctly.** If `email` has a unique DB constraint, and a user is soft-deleted, a *new* user can't register with that same email — the "deleted" row still occupies the unique slot in the database, even though the application layer treats it as gone.
3. **The table grows forever.** Nothing is ever actually removed, so a high-churn table (e.g. cart items, sessions) balloons in size indefinitely, slowing down every query that touches it — even the ones correctly filtering `deleted = false`, since the DB still has to scan past all those "invisible" rows unless properly indexed.

**Why the naive "just add a boolean" design breaks down:** soft delete isn't free — it converts a delete operation into an **ongoing tax on every future query against that table**, forever. The course's version (filtering in derived query method names) is fine for a small demo but doesn't scale to "every developer, every query, forever, correctly remembering the filter."

**Better real-world approaches (in order of increasing sophistication):**
1. **A dedicated, indexed partial index** (Postgres: `CREATE INDEX ... WHERE deleted = false`) — keeps "live" queries fast even as the deleted rows pile up, and a database-level index makes it harder to *forget* the intent (though not impossible to forget in the query itself).
2. **Move soft-deleted rows to an `archived_students` table** instead of a flag on the live table — live queries never need a filter at all (there's nothing "deleted" in the live table to accidentally include), and archived data can be queried separately when actually needed.
3. **Enforce the filter at the JPA/Hibernate level, not per-query** — Hibernate's `@Where(clause = "deleted = false")` (or `@SQLRestriction` in newer Hibernate) applies the filter to *every* query Hibernate generates for that entity automatically, removing the "did the developer remember to filter" risk entirely — this is a genuinely better fix than the course's per-method-name filtering.
```java
@Entity
@SQLRestriction("deleted = false")
public class Student { ... }
```

**Interview angle:** *"What are the trade-offs of soft delete vs hard delete?"* — the strong answer names the specific failure modes above (forgotten filters, unique constraint collisions, unbounded table growth), not just "soft delete keeps data for auditing." Bonus points for knowing `@SQLRestriction`/`@Where` as the "enforce it centrally" fix.

---

## Story 4: Where do secrets actually go? (`application.properties` isn't it)

**The situation:** A developer follows the course pattern and puts the database password directly in `application.properties`:
```properties
spring.datasource.password=SuperSecret123
```
...and commits it to Git, because that's where the course's demo puts everything.

**Where it breaks:** this file is packaged **inside the JAR** (Lecture 10 already flags this) and, worse, is now sitting in **Git history forever** — even if you delete the line in a later commit, the old commit still has it. If the repo is ever made public, or an employee's laptop with repo access is compromised, the production database password is exposed.

**Why "just use `.gitignore`" isn't a complete fix:** it stops the *next* commit from having the secret, but doesn't retroactively scrub Git history, and doesn't solve the deeper problem — *someone* still has to get that real password onto the production server somehow, and "a plaintext file someone manually copies over SSH" isn't much better than committing it.

**The real-world fix — externalize secrets outside the codebase entirely**, using exactly the "externalized configuration" mechanism from Lecture 10, but pointed at a **secret manager** instead of a properties file:
```bash
# environment variables injected by the deployment platform (Docker/Kubernetes/CI secrets), never committed:
export SPRING_DATASOURCE_PASSWORD=SuperSecret123
```
Or, in more mature setups: **HashiCorp Vault**, **AWS Secrets Manager**, or **Kubernetes Secrets** — the application fetches the credential at startup/runtime from a system specifically designed to store secrets securely, audit who accessed them, and rotate them without a code deploy.

**Interview angle:** *"How do you manage secrets in a Spring Boot application?"* — naming `application.properties` as the answer is actually a red flag in an interview; the strong answer is "never commit secrets — use environment variables injected by the deployment platform, or a dedicated secret manager like Vault/AWS Secrets Manager, and `application.properties` should only ever hold non-sensitive defaults or placeholders that get overridden externally."

---

## Quick-fire: things worth knowing even if the course glossed over them

| Concept | Why it matters | The gap in the course's version |
|---|---|---|
| **`@RequestParam` vs `@PathVariable`** for identifying a resource | REST convention: `/students/{id}` (path) not `/students?id=5` (query param) for *identifying* a specific resource | The course's demo actually uses `@RequestParam Long id` for get/update/delete — functional, but not REST-idiomatic; `@PathVariable` is the convention interviewers expect |
| **204 No Content** | The correct status for a successful DELETE with no response body | The course's demo returns `200 OK` with a string message — fine for learning, but `204` is the REST-correct choice in production APIs |
| **Response wrapping / envelope pattern** | Some APIs wrap every response in `{ "data": ..., "meta": ... }` for consistency | Not covered — worth knowing as a design choice (consistency + room for pagination metadata) vs. the course's "return the raw object" style |

---
*Next companion doc: `scenarios-04-web-layer.md` — CORS vs CSRF (a classic trick question), custom validators & validation groups, and API versioning strategies.*
