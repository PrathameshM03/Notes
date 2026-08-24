# Scenarios & Deeper Reasoning — 7: Security & Testing, Where "It Works" Isn't Enough
### Companion to `07-security-oauth-testing.md`

---

## Story 1: The endpoint that was "protected" but wasn't — method-level security

**The situation:** A team secures their API purely at the URL level:
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated());
```
This looks thorough. Then someone adds a new **service method** that any authenticated controller can call:
```java
@Service
public class UserService {
    public void deleteAllUsers() { userRepository.deleteAll(); } // no URL-level protection possible here!
}
```
A *different* controller — say, a bulk-import utility endpoint under `/api/tools/**` (not `/api/admin/**`) — ends up calling `deleteAllUsers()` for an unrelated reason, and because `/api/tools/**` only required `.authenticated()`, **any logged-in user**, not just admins, can trigger a full user wipe.

**Where URL-level security breaks down:** `SecurityFilterChain`'s `authorizeHttpRequests` protects **URLs** — it has no idea what a controller method's business logic actually *does*, or which service methods it calls into. If sensitive logic can be reached via more than one path (a new endpoint, a scheduled job, an internal admin tool, a future refactor), URL-level rules alone don't follow the logic there.

**The fix — method-level security, enforced at the point of actual risk:**
```java
@Configuration
@EnableMethodSecurity // enables @PreAuthorize, @PostAuthorize, @Secured
public class SecurityConfig { ... }
```
```java
@Service
public class UserService {
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAllUsers() { userRepository.deleteAll(); }
}
```
Now the protection travels **with the method itself**, regardless of which controller, scheduled job, or future code path calls it — this is the same "move the guard to where the risk actually is" principle that shows up throughout security-minded engineering.

**A more powerful example — object-level authorization:**
```java
@PreAuthorize("#studentId == authentication.principal.id or hasRole('ADMIN')")
public Student getStudentProfile(Long studentId) {
    return studentRepository.findById(studentId).orElseThrow();
}
```
This expresses "a user can view their **own** profile, or an admin can view **anyone's**" — a rule that URL-based `authorizeHttpRequests` genuinely **cannot** express at all, because the URL pattern `/api/students/{id}` has no way to compare `{id}` against *who's asking*.

**`@PreAuthorize` vs `@PostAuthorize`:** `@PreAuthorize` checks **before** the method runs (can block execution entirely, given the arguments); `@PostAuthorize` checks **after** the method returns, with access to the **return value** too — e.g. `@PostAuthorize("returnObject.ownerId == authentication.principal.id")`, useful when the authorization decision depends on data you only have *after* fetching the entity.

**Interview angle:** *"How would you secure a service method that could be called from multiple different places?"* — naming `@PreAuthorize`/`@EnableMethodSecurity` (not just URL-level `authorizeHttpRequests`) and explaining *why* URL-level rules don't follow business logic across multiple call sites is a strong, specific answer that goes beyond "add security config."

---

## Story 2: The token that couldn't be revoked — refresh tokens and the stateless trade-off

**The situation:** Recall Lecture 38's JWT flow — the whole appeal of JWT is **statelessness**: the server doesn't store sessions, so it can verify a token purely by checking its signature, no database lookup needed. Now imagine: an employee is fired. Their JWT, issued that morning, is valid for the next 24 hours (a common expiry). **How do you immediately revoke their access?**

**Where the "pure stateless" design breaks down:** you *can't* — not without contradicting the whole point of statelessness. If the server has to check "is this token on a revocation list?" on every request, you've reintroduced a **server-side lookup per request**, which is exactly the state-per-request cost JWT was meant to eliminate.

**Why short expiry alone doesn't fully solve it either:** making the access token expire in, say, 5 minutes limits the *damage window* of a compromised or "should be revoked" token — but it also means the user has to log in again every 5 minutes, which is a terrible user experience for a normal, legitimate session.

**The standard real-world fix — short-lived access tokens + long-lived refresh tokens:**
```
Login → server issues:
   access_token  (JWT, short-lived, e.g. 15 minutes, used on every API call)
   refresh_token (long-lived, e.g. 7 days, used ONLY to get a new access_token)
```
```java
@PostMapping("/auth/refresh")
public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
    RefreshToken stored = refreshTokenRepository.findByToken(request.getRefreshToken())
        .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));

    if (stored.isRevoked() || stored.getExpiresAt().isBefore(Instant.now())) {
        throw new InvalidTokenException("Refresh token expired or revoked");
    }

    String newAccessToken = jwtService.generateAccessToken(stored.getUser());
    return ResponseEntity.ok(new AuthResponse(newAccessToken, stored.getToken()));
}
```
```java
// Revoking access instantly (e.g. on firing an employee, or "log out everywhere"):
@Transactional
public void revokeAllTokensForUser(Long userId) {
    refreshTokenRepository.revokeAllByUserId(userId); // one DB write, immediate effect
}
```
**Why this specific split is the accepted trade-off, not a compromise nobody's happy with:**
- The **access token** stays fully stateless — every normal API request (the vast majority of traffic) still needs zero DB lookup for auth, preserving JWT's performance benefit where it matters most.
- The **refresh token** *is* checked against a database — but refresh calls happen far less often (every 15 minutes, not every request), so the "reintroduced state" cost is paid on a much smaller slice of total traffic.
- Revocation now works: revoke the refresh token, and even though any *already-issued* access token remains valid until its short expiry naturally passes, the blast radius is capped at that short window (15 minutes), not 24 hours.

**Interview angle:** *"JWTs are stateless — so how do you log a user out, or revoke access, before the token naturally expires?"* — this is a very common, very good follow-up to any JWT discussion, precisely because "just make it stateless" sounds complete until this exact question exposes the gap. The strong answer names the access/refresh token split and explains *why* it's the accepted trade-off (stateless for high-frequency traffic, stateful only for the low-frequency refresh/revocation path) rather than treating JWT as a silver bullet.

---

## Story 3: The test suite that passed but the feature was still broken — the test pyramid, in practice

**The situation:** A team writes extensive **unit tests** for `StudentService`, mocking `StudentRepository` entirely (Lecture 40's `@Mock`/`@InjectMocks` pattern). All green. They ship a new derived query method:
```java
List<Student> findByAgeGreaterThanAndSubjectIgnoreCase(int age, String subject);
```
In production, it throws an exception at startup — the method name has a typo (`Grater` vs `Greater`... or a field name that doesn't match the entity), and Spring Data JPA can't parse it into a valid query.

**Why the unit tests didn't catch this:** the unit test **mocked the repository entirely** —
```java
when(studentRepository.findByAgeGreaterThanAndSubjectIgnoreCase(18, "Math")).thenReturn(List.of(mockStudent));
```
This only proves "if this method is called with these arguments, it returns this value" — it says **nothing** about whether Spring Data JPA can actually *generate* a valid query from that method name. A typo'd or malformed derived-query method name is a **runtime startup failure** that only a test *not* mocking the repository can ever catch (this is exactly what Lecture 40's Story about `@DataJpaTest` warns about — "mocking a repository does not prove the real repository query works").

**The fix — the test pyramid applied deliberately, not accidentally:**
```
        ▲
       / \      Few, slow, expensive:
      / E2E\     Full @SpringBootTest, real HTTP calls, real DB — "does the whole system work together"
     /-------\
    /  Integ. \  Some, moderate cost:
   /  (@DataJpaTest,\ "does this repository's actual query work against a real (test) database"
  / @WebMvcTest)  \
 /------------------\
/    Many, fast, cheap:\ Unit tests (@Mock/@InjectMocks) — "does this ONE class's logic work in isolation"
------------------------
```
- **Unit tests** (many, fast) — verify business *logic* in isolation; **cannot** verify that a derived query method name is valid, that a JPQL/native query compiles, or that a real HTTP request routes correctly.
- **`@DataJpaTest`** (some) — verifies the *actual* query against a real (typically in-memory H2) database — this is the layer that would have caught the typo.
- **`@WebMvcTest`** (some) — verifies routing/serialization/status codes at the web layer, again without needing a real database.
- **`@SpringBootTest`** (few) — the full application context, sometimes with a real embedded server and real HTTP calls — expensive (slow to start, slow to run), so reserved for genuinely critical end-to-end paths, not every scenario.

**Why "just write more unit tests" wouldn't have prevented this bug, no matter how many you wrote:** the bug wasn't in *logic* — it was in whether Spring Data JPA could translate a method *name* into a valid query at all, which is fundamentally something only an integration-level test (with a real, even if in-memory, database and Spring context) can verify. This is the core argument for *why* the test pyramid has multiple layers instead of "just write exhaustive unit tests" — different layers catch fundamentally different classes of bugs.

**Interview angle:** *"You have 100% unit test coverage and a bug still shipped — how?"* — a strong answer doesn't get defensive about coverage numbers; it explains that unit tests (with everything mocked) structurally **cannot** catch certain bug classes — wiring/configuration errors, invalid derived query names, serialization mismatches, real SQL correctness — and that's precisely why `@DataJpaTest`/`@WebMvcTest`/`@SpringBootTest` exist as *distinct, deliberately different* layers, not redundant extra work.

---

## Quick-fire: security/testing concepts worth knowing exist

| Concept | What it is | Why it's beyond the course's core path |
|---|---|---|
| **Rate limiting** | Capping requests per user/IP (e.g. via Bucket4j, or an API gateway) | Prevents brute-force login attempts and abuse — a natural follow-up to any auth discussion, not covered in the course |
| **`@WithMockUser`** | A test annotation that simulates an authenticated user in `@WebMvcTest`/`@SpringBootTest` without a real login flow | The course's testing lecture doesn't cover testing secured endpoints specifically — this is the standard tool for it |
| **Testcontainers** | Spins up a real Dockerized database (not H2) for integration tests, matching production more closely | `@DataJpaTest`'s default in-memory H2 database can behave subtly differently from real MySQL/Postgres (different SQL dialect quirks) — Testcontainers closes that gap for teams that need production-parity in tests |
| **Contract testing** (e.g. Pact) | Verifies that a service's API still matches what its consumers expect, without needing the consumer's full codebase in the test | Relevant once you're in a microservices world with many teams — a natural next question after "how do you test a REST API" |

---
*This closes the 7 companion "scenario" docs. Next up: the 6 new full-depth topic docs the original course never covered — starting with `08-actuator-observability.md`.*
