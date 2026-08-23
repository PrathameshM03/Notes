# Spring Framework Full Course — Notes 7: Security, OAuth & Testing
### Covers Lectures 35–40: Spring Security fundamentals → Database-backed authentication & password hashing → The internal security filter chain → JWT authentication → OAuth 2.0/OIDC (Google login) → Testing with JUnit & Mockito

---

## Lecture 35 — Spring Security Introduction

### 1. What "application security" really covers
Beyond "protect the data," security means protecting **assets**: data (passwords, PII, source code, API keys), **operations** (transferring money, deleting an account, changing another user's password), **infrastructure** (DB, servers, deployment pipeline), **security assets themselves** (session IDs, JWT signing keys, OAuth client secrets), and even **availability** (an attacker doesn't need to steal data to cause harm — flooding an endpoint is also an attack).

**Security is not one feature — it's a collection of controls:** authentication, authorization, input validation, session protection, password protection, CSRF protection, CORS policy, secure communication, rate limiting, audit logging, error handling, secret management.

**What Spring Security helps with:** authentication, authorization, security-context management, session security, protection against several common web attacks, and integration with security protocols. **What it does NOT automatically fix:** SQL injection from unsafe queries, insecure file uploads, weak business rules, secrets committed to Git, vulnerable dependencies, incorrect DB permissions, sensitive data written to logs. Security must exist **wherever an untrusted actor can affect a valuable resource** — not just on the login page.

**Assets, Threats, Vulnerabilities** — distinct concepts:
- **Asset** — anything valuable that should be protected.
- **Threat** — something that *could* harm an asset (e.g. "an attacker wants to access an admin account").
- **Vulnerability** — a weakness that *makes an attack possible* (e.g. "the app allows unlimited password attempts").
- **Attack** — the actual exploitation (e.g. running an automated password-guessing tool). **Impact** — the resulting harm (e.g. admin account compromised).

### 2. Spring Security's identity model — core vocabulary
| Term | Meaning |
|---|---|
| **Identity** | The entity whose actions the system recognizes — broader than "a database row" (could be a human, another service, a scheduled job, an IoT device). |
| **Account** | One way an application *represents and manages* an identity (a `users` table row) — identity is the conceptual subject; account is the storage mechanism. |
| **Credentials** | Proof used to verify an identity (password, key, token). |
| **Principal** | The currently-authenticated identity, as Spring Security represents it. |
| **Authorities / Roles / Permissions** | What the identity is allowed to do. |
| **Authentication** | The (Spring) object representing a verified (or verification-attempt) identity. |
| **SecurityContext** | Holds the current `Authentication` for the current execution. |
| **SecurityContextHolder** | The static access point for the current thread's `SecurityContext`. |

### 3. Authentication state — stateful vs stateless
HTTP requests are independent by default — the server must decide **how a later request knows a previous one authenticated**. Two broad models:

**Stateful authentication (session-based):**
```
Login → verify credentials → store authentication in server-side session
   → client gets a session identifier (cookie)
   → client sends the session ID on every later request
   → server retrieves the stored authentication
```
**Stateless authentication** — no server-side session; every request carries its own self-contained proof (e.g. a signed JWT) — covered fully in Lecture 38.

### 4. Authentication mechanism vs. session policy — a crucial distinction
- **Authentication mechanism** answers: *how are credentials obtained and verified?* (e.g. form login, HTTP Basic, OAuth).
- **Session policy** answers: *after authenticating, should the resulting `SecurityContext` be persisted in an HTTP session?*

These are **independent choices**. Form login can either (A) create a session and send a cookie (classic stateful web app), or (B) simply authenticate and issue a token without creating a session (fits a stateless API design). Similarly, HTTP Basic (`Authorization: Basic <base64>`) is just a **credential transport mechanism** — the client typically resends it on every request, which naturally fits per-request (stateless) authentication.

### 5. Spring Boot's security auto-configuration
Spring Boot doesn't implement authentication itself — Spring Security does. Boot's job is to observe the classpath and provide **sensible defaults**, coordinated primarily through `SecurityAutoConfiguration` (default web-security setup) and `UserDetailsServiceAutoConfiguration` (default username/password user).

**Effective default (conceptually):**
```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(authorize -> authorize.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```
Meaning: every request requires authentication; both form login and HTTP Basic are available.

**Auto-configuration is conditional (the recurring Spring Boot pattern from Lecture 9):** if you define your **own** `SecurityFilterChain` bean, Boot backs off and stops supplying its default access rules. Important nuance: replacing the `SecurityFilterChain` does **not** automatically remove the auto-generated default user — you separately replace the user/authentication setup by providing your own `UserDetailsService`, `AuthenticationProvider`, or `AuthenticationManager`. These two concerns are genuinely independent: `SecurityFilterChain` decides **how requests are secured**; `UserDetailsService`/`AuthenticationProvider` decide **how users are authenticated**.

### 6. The default in-memory user
Without any user store, the login page would exist but no credentials could ever succeed — so Spring Boot generates one default user in memory (via `InMemoryUserDetailsManager`):
```
username = user
password = randomly generated, printed to the console at startup
authorities = [ROLE_USER]
```
This user is **not** in your database, not created by your code, and is explicitly documented as **development-only**. You can override it for demos via properties:
```properties
spring.security.user.name=aditya
spring.security.user.password=secret123
spring.security.user.roles=USER,ADMIN
```
Under the default rule `anyRequest().authenticated()`, the *specific* role doesn't matter — Spring Security is only checking "is there an authenticated identity at all," not "does it have `ROLE_ADMIN`."

### 7. Generated password
At startup you'll see a log line like:
```
Using generated security password: 78fa095d-3f4c-48b1-ad50-e24c31d5cf35
```
This is logged at **WARN level** and explicitly documented as development-use-only — never rely on this in production.

### 8. The CSRF 403 "surprise"
A logged-in user sending `POST /api/courses` might get **403 Forbidden** even with valid authentication and a correct role — because Spring Security enables **CSRF protection by default** for servlet web applications, and state-changing requests (POST/PUT/PATCH/DELETE) are rejected without the expected CSRF token.

> **Key takeaway:** `401 Unauthorized` usually means authentication is missing/failed. `403 Forbidden` means the request was understood but rejected — and with Spring Security, a 403 on a state-changing request can be a **missing CSRF token**, not just insufficient authority. (For stateless JSON/REST APIs, CSRF protection is commonly disabled since there's no browser-managed session cookie to forge in the first place — a decision made explicitly in security config, not left to default behavior.)

---

## Lecture 36 — Database Authentication and Password Security

### 1. Why the default in-memory user isn't enough
Real applications need many users, registered dynamically, stored durably — authentication needs **a real source of truth** (typically a database), not one hardcoded in-memory credential.

### 2. `UserDetails` — the contract Spring Security needs
```java
public interface UserDetails {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```
Three groups of information: **Identity** (`getUsername()`), **Credential** (`getPassword()`), and **Authorities/account status** (roles + enabled/locked/expired flags). Your own `User` entity doesn't need to *be* a `UserDetails` — you typically wrap/adapt it into one.

### 3. Modeling users and roles with JPA
```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;
    private String password;   // stored HASHED, never plain text
    @ManyToMany
    private Set<Role> roles;
}
@Entity
public class Role {
    @Id @GeneratedValue
    private Long id;
    private String name;   // e.g. "ROLE_USER", "ROLE_ADMIN"
}
```

### 4. Why raw passwords must never be stored
If the database is ever leaked (breach, backup exposure, insider access), plaintext passwords immediately compromise every user's account — and because people commonly reuse passwords, the damage extends to *other* services too.

### 5. Why simple encryption isn't the answer either — hashing instead
Encryption is **reversible** (given the key) — meaning anyone with the key can recover the original password, which is not what you want even internally. **Hashing** is (practically) one-way: you store a hash, and verify a login by hashing the *submitted* password and comparing hashes, never storing or recovering the original.

### 6. Why a fast hash (like plain SHA-256) is not enough
Generic fast cryptographic hashes (SHA-256, MD5) are **designed to be fast** — which is exactly the wrong property for password storage, because it lets an attacker with a stolen hash database brute-force/guess millions of candidate passwords per second.

### 7. BCrypt & `PasswordEncoder`
Spring Security provides `BCryptPasswordEncoder`, implementing the `PasswordEncoder` abstraction — BCrypt is **deliberately slow** (computationally expensive, tunable via a "strength"/work-factor parameter), making brute-force attacks far less practical.
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```
```java
String hashed = passwordEncoder.encode(rawPassword);          // registration
boolean matches = passwordEncoder.matches(rawPassword, hashed); // login
```

### 8. Salt & BCrypt
A stolen hash database can be attacked with **precomputed hash dictionaries** ("rainbow tables") — mapping common passwords straight to their known hashes. A **salt** is unique random data mixed into the hashing process **per password**, so even identical passwords produce different hashes — defeating precomputed-table attacks. BCrypt **generates and embeds its own salt automatically** as part of the encoded output — you don't manage salts manually.

### 9. Registration flow, from first principles
```
Client submits { username, rawPassword }
   → RegistrationService
      → passwordEncoder.encode(rawPassword)  →  hashed password
      → save User(username, hashedPassword, roles) via UserRepository
```
```java
public class RegistrationDto {
    @NotBlank private String username;
    @NotBlank private String password;
}
```
```java
@Service
public class RegistrationService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public void register(RegistrationDto dto) {
        User user = new User();
        user.setUsername(dto.getUsername());
        user.setPassword(passwordEncoder.encode(dto.getPassword())); // encode(...) here
        userRepository.save(user);
    }
}
```
**Registration uses `encode(...)`; login uses `matches(...)`** — this pairing is emphasized because it's a common source of confusion for beginners.

---

## Lecture 37 — Spring Security Internals & Complete Login Flow

### 1. Where Spring Security actually sits
Spring Security works through **servlet filters**, running **before** `DispatcherServlet` — same layer discussed for general Filters in Lecture 19, but Spring Security brings its own dedicated filter infrastructure.

### 2. `DelegatingFilterProxy`
> A **real servlet filter** the container (Tomcat) knows about — but it doesn't implement security logic itself. It simply **finds a Spring-managed filter bean and delegates** the request to it. This is the bridge between the servlet container's filter model and Spring's own bean-managed filter chain.

### 3. `FilterChainProxy`
> The **central filter** of Spring Security's servlet architecture, standing in front of one or more `SecurityFilterChain`s:
```
FilterChainProxy
   ├── SecurityFilterChain for /api/**
   ├── SecurityFilterChain for /admin/**
   └── ... (matched by request pattern; the FIRST matching chain wins)
```

### 4. `SecurityFilterChain` & `HttpSecurity`
A `SecurityFilterChain` bean (built via the fluent `HttpSecurity` DSL) declares which requests need authentication, which authentication mechanisms are enabled, CSRF policy, and more — this is what you override to customize security config.

### 5. Authentication filters extract, they don't verify
Authentication filters (e.g. for form login or HTTP Basic) pull credentials out of the raw request and package them into an **unverified** `UsernamePasswordAuthenticationToken` — they **do not perform database verification themselves**; that's delegated further down the chain.

### 6. `AuthenticationManager` → `ProviderManager` → `AuthenticationProvider`
- **`AuthenticationManager`** — the entry point into Spring Security's authentication engine (an interface).
- **`ProviderManager`** — the standard implementation; it delegates to one of possibly several registered **`AuthenticationProvider`**s, picking whichever one supports the token type it's given.
- **`DaoAuthenticationProvider`** — the common provider for username/password auth; it connects a **`UserDetailsService`** (finds the user) with a **`PasswordEncoder`** (verifies the password).

### 7. `UserDetailsService`
```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```
> `UserDetails` represents **one user**; `UserDetailsService` explains **how to find that user** (from your database, typically).
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;

    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(), user.getPassword(),
            user.getRoles().stream().map(r -> new SimpleGrantedAuthority(r.getName())).toList());
    }
}
```

### 8. Complete internal login flow
```
HTTP Request (with credentials)
   → Authentication Filter (extracts raw credentials into an unverified token)
   → AuthenticationManager
   → ProviderManager
   → DaoAuthenticationProvider
        → UserDetailsService.loadUserByUsername(username)   -- find the user
        → PasswordEncoder.matches(rawPassword, storedHash)  -- verify the password
   → on success: a fully authenticated Authentication object is built
   → stored in the SecurityContext (SecurityContextHolder)
   → request proceeds to the controller with an authenticated principal available
```
**Form login and HTTP Basic share the same underlying authentication engine** (`AuthenticationManager` → `ProviderManager` → `DaoAuthenticationProvider`) — they only differ in *how credentials are transported/extracted* at the filter stage, not in how they're ultimately verified.

**Registration and login are deliberately separate concerns:** registration is *your own* application logic (hash + save a new `User`); login flows entirely through Spring Security's authentication machinery above — they don't share a code path, only the same underlying `PasswordEncoder`.

### Final mental model (key takeaways)
1. Spring Security operates through servlet filters, before the controller.
2. `DelegatingFilterProxy` bridges Tomcat and Spring-managed security components.
3. `FilterChainProxy` selects the first matching `SecurityFilterChain`.
4. `HttpSecurity` builds and configures that chain.
5. Authentication filters extract credentials; they don't verify against the database themselves.
6. `UsernamePasswordAuthenticationToken` represents both the unverified request and, later, the verified result.
7. `AuthenticationManager` is the entry point into the authentication engine.
8. `ProviderManager` delegates to a compatible `AuthenticationProvider`.
9. `DaoAuthenticationProvider` connects `UserDetailsService` with `PasswordEncoder`.
10. After successful login, the authenticated object is placed in the `SecurityContext`.

---

## Lecture 38 — Spring Security JWT Authentication

### 1. The problem: re-authenticating on every request is wasteful
Session-based auth requires server-side session storage — fine for a single traditional web app, but awkward for stateless, horizontally-scaled APIs (which server holds the session? what about multiple backend instances?).

### 2. Why JWT?
A **JWT (JSON Web Token)** lets the *client* carry proof of authentication with it on every request, without the server storing any session state — genuinely **stateless** authentication.

### 3. The security problem with client-carried data
If the client holds its own "I am logged in as admin" claim, what stops it from just editing that claim? E.g., decode a Base64 payload, change `"role": "STUDENT"` to `"role": "ADMIN"`, re-encode, and send the modified token.

**Base64URL is encoding, not encryption** — anyone can decode it; it provides **zero** confidentiality or tamper-protection by itself. A plain (unsigned) hash also isn't enough — if the attacker doesn't know the *algorithm/secret* used, they still can't produce a hash that matches after tampering, but a naive unsalted/unkeyed hash can potentially be recomputed by anyone. The real requirement is a **signature** that only the legitimate issuer can produce.

### 4. Structure of a signed JWT
```
HEADER.PAYLOAD.SIGNATURE
xxxxx.yyyyy.zzzzz
```
Header and payload are Base64URL-encoded JSON; the signature is also Base64URL-encoded bytes.

### 5. Symmetric signing — HS256
`HS256` = **HMAC using SHA-256**:
```
signature = HMAC-SHA256(secretKey, Base64URL(header) + "." + Base64URL(payload))
```
The **same secret key** both signs and verifies. If the payload changes even slightly, recomputing the HMAC with the same secret produces a **completely different** signature — so a tampered token fails verification immediately (this is directly demonstrable via a "tampering experiment": modify the payload, re-encode, and verification fails).

### 6. Asymmetric signing — RS256/ES256
Uses a **key pair**: a **private key** creates signatures, a **public key** verifies them.
| | HS256 (symmetric) | RS256/ES256 (asymmetric) |
|---|---|---|
| Key model | One shared secret | Private/public key pair |
| Signing | Secret key | Private key |
| Verification | Same secret key | Public key |
| Can the verifier also create valid tokens? | Yes, if it has the secret | No, if it only has the public key |
| Setup simplicity | Simple, good for one application | More setup, but lets many services verify without being able to *issue* tokens |

Asymmetric signing is valuable in multi-service architectures — an API gateway or downstream microservice can **verify** tokens with the public key without ever being trusted to **issue** new ones.

### 7. JWT authentication flow in a Spring Boot app
```
POST /auth/login  { username, password }
   → normal Spring Security authentication (DaoAuthenticationProvider, as in Lecture 37)
   → on success: JwtService builds and signs a JWT containing username + authorities
   → JWT returned to the client

Later requests:
   Authorization: Bearer <jwt>
   → a JWT-aware filter extracts and validates the token (via JwtDecoder)
   → if valid, Spring builds an Authentication from the token's claims (no DB/session lookup needed)
   → request proceeds as authenticated
```

### 8. Key building blocks
- **`spring-boot-starter-oauth2-resource-server`** — despite the "OAuth2" name, this starter provides the JWT decoding/validation machinery used here.
- **`JwtEncoder` / `JwtDecoder`** — encode = create+sign a token; decode = parse+verify a token.
```java
@Bean
public SecretKey jwtSecretKey() {
    return Keys.hmacShaKeyFor(secret.getBytes()); // from a configured secret
}

@Bean
public JwtEncoder jwtEncoder(SecretKey key) {
    return new NimbusJwtEncoder(new ImmutableSecret<>(key));
}
```
- **`JwtService`** — application-level helper that builds claims (username, **authorities**, expiration) and calls the encoder.

**Why store authorities in the JWT?** So that later requests can be authorized **without a database round-trip** — the roles travel with the token itself, which is exactly what makes the approach stateless.

- **`JwtDecoder`** — configured with the same secret (HS256) or the public key (RS256); validates signature + expiration on every incoming request.
- **`JwtAuthenticationConverter`** — maps the JWT's claims (e.g. an `authorities` claim) into Spring Security `GrantedAuthority` objects, **without adding another prefix** if the claim already stores properly-prefixed role names (a common gotcha — double-prefixing `ROLE_ROLE_ADMIN`).

### 9. Final `SecurityFilterChain` for JWT
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf.disable())  // no session/cookie to forge — CSRF not applicable
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/auth/login").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
    return http.build();
}
```
`oauth2ResourceServer().jwt()` wires in the JWT-based authentication filter, using the configured `JwtDecoder` to validate incoming bearer tokens on every request.

**Two authentication providers, two purposes:** the `DaoAuthenticationProvider` (username/password) is used **only** at `/auth/login` to issue a token; every *subsequent* request is authenticated purely via JWT validation — they don't overlap in normal operation.

### Session vs JWT — summary
| | Session-based | JWT-based |
|---|---|---|
| Server-side state | Yes (session store) | No (stateless) |
| Scales across multiple servers | Needs shared session store | Naturally stateless-friendly |
| Revoking access immediately | Easy (invalidate session) | Harder (token valid until expiry, unless a blocklist is added) |
| Client storage | Cookie (browser-managed) | Client manages the token explicitly (e.g. `Authorization` header) |

### Important JWT takeaways
- Base64URL encoding provides no security by itself — signing does.
- HS256 = one shared secret for both signing and verifying; RS256/ES256 = separate private (sign) / public (verify) keys.
- Store authorities in the token to avoid per-request DB/session lookups — that's the whole point of stateless auth.
- `oauth2ResourceServer().jwt()` is the Spring Security building block for stateless, JWT-validated APIs.

---

## Lecture 39 — OAuth 2.0, OpenID Connect & Google Login

### 1. Moving beyond your own authentication system
Instead of managing your own usernames/passwords, **delegate** authentication to a trusted third party (Google, GitHub, etc.) — the user logs in with an account they already have, and your app never sees their password.

### 2. The delegated-access problem OAuth solves
Before OAuth, "let this app act on my behalf" meant literally **sharing your password** with that app — giving it far more access than it actually needed, with no ability to limit *what* it could do or later revoke *just that app's* access.

### 3. The four OAuth roles
| Role | Example | Responsibility |
|---|---|---|
| **Resource Owner** | The user | Owns/controls access to the protected resource |
| **Client** | Your Spring Boot app | Requests delegated access on the user's behalf |
| **Authorization Server** | Google's auth system | Authenticates the user, obtains consent, issues tokens |
| **Resource Server** | Google's API (e.g. profile info) | Hosts the protected resource, validates tokens |

### 4. Scope & consent
The client requests specific **scopes** (e.g. "read your email," "read your profile") — the authorization server shows the user exactly what's being requested and requires explicit **consent** before issuing any tokens, limiting the client to only what was approved.

### 5. OAuth tokens vs OIDC tokens
- Pure **OAuth 2.0** is about **authorization** — it issues **access tokens** for calling APIs on the user's behalf.
- **OpenID Connect (OIDC)** is a thin **identity/authentication** layer built on top of OAuth 2.0 — it adds an **ID token** (a signed JWT specifically describing *who the user is*), which is what "Login with Google"-style flows actually rely on for authentication (not just API access).

### 6. Authorization Code Flow
Designed so tokens are **never casually exposed through browser redirects**:
1. The browser receives only a short-lived **authorization code** (not a token) via redirect.
2. The **backend** (server-to-server, not the browser) exchanges that code for actual tokens directly with the authorization server.

```
1. User clicks "Continue with Google"
2. App redirects browser to Google's authorization endpoint (with client_id, redirect_uri, scope, state)
3. Google authenticates the user & shows the consent screen
4. User approves the requested scopes
5. Google redirects back to the app with a short-lived authorization code
6. The app's BACKEND exchanges that code for tokens via a direct server-to-server request
7. App validates/processes the OIDC ID token
8. App creates/updates a local user record from the OIDC identity
9. App establishes its own authenticated session for that user
```

### 7. PKCE (Proof Key for Code Exchange)
An additional protection layered onto the Authorization Code Flow (especially important for public/mobile/SPA clients that can't safely hold a client secret) — the client generates a random secret ("code verifier") up front, sends a hashed version ("code challenge") with the initial authorization request, and must present the original verifier when exchanging the code for tokens — preventing an intercepted authorization code from being redeemed by anyone else.

### 8. Spring Boot implementation outline
Dependencies: `spring-boot-starter-oauth2-client`.
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: <from Google Cloud Console>
            client-secret: <from Google Cloud Console>
            scope: openid, profile, email
```
```java
@Service
public class CustomOidcUserService extends OidcUserService {
    @Override
    public OidcUser loadUser(OidcUserRequest userRequest) {
        OidcUser oidcUser = super.loadUser(userRequest);
        // save-or-update a local AppUser record based on oidcUser's email/sub
        return oidcUser;
    }
}
```
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2Login(oauth2 -> oauth2.userInfoEndpoint(userInfo -> userInfo.oidcUserService(customOidcUserService)));
    return http.build();
}
```

### 9. What authenticates later `/profile` requests?
After the OIDC login flow completes, Spring Security establishes its **own** authenticated session (the classic session-cookie mechanism from Lecture 35) for the user — subsequent requests to protected endpoints (`/profile`) are authenticated the normal session-based way; the OAuth/OIDC dance with Google only happened **once**, at login time.

### Complete flow recap
```
/oauth2/authorization/google → redirect to Google → user authenticates & consents
   → Google redirects back with an authorization code
   → Spring exchanges the code for tokens (back channel)
   → Spring validates the OIDC response, creates an OidcUser
   → CustomOidcUserService saves/updates the local AppUser
   → Spring stores the authenticated state in the session
   → user accesses /profile as a normally-authenticated (session-based) user
```

---

## Lecture 40 — Spring Testing with JUnit and Mockito

### 1. Why testing exists
A test automates what a person would otherwise do manually: provide a known input, execute the production code, compare the actual result against the expected result, and report pass/fail. For an API specifically, a test also has to: start the app (or a slice of it), prepare a request, call the endpoint, inspect the response, and decide if the result is correct — which is why Spring's testing tools go beyond plain JUnit to include web-layer and persistence-layer test support.

### 2. JUnit fundamentals
```java
@Test
void additionWorks() {
    int result = calculator.add(2, 3);
    assertEquals(5, result);
}
```
Standard JUnit 5 annotations: `@Test`, `@BeforeEach`/`@AfterEach` (setup/teardown per test), `@BeforeAll`/`@AfterAll` (once per class), assertions via `org.junit.jupiter.api.Assertions` (`assertEquals`, `assertTrue`, `assertThrows`, etc.).

### 3. Unit testing the Service layer with Mockito
**The dependency problem:** a `ProductService` depends on a `ProductRepository` — a true **unit** test shouldn't hit a real database; it should test the service's logic **in isolation**, with the repository's behavior faked.
```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {
    @Mock
    private ProductRepository productRepository;

    @InjectMocks
    private ProductService productService;

    @Test
    void returnsProductWhenFound() {
        Product product = new Product(1L, "Laptop", 50000);
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        Product result = productService.getProduct(1L);

        assertEquals("Laptop", result.getName());
        verify(productRepository).findById(1L);
    }
}
```
- **`@Mock`** — creates a fake `ProductRepository` whose method behavior you fully control (`when(...).thenReturn(...)`).
- **`@InjectMocks`** — creates a real `ProductService` instance and injects the mocks into it.
- **`verify(...)`** — asserts that a mocked method was actually called (useful for confirming interaction, not just return values).

This is a genuine **unit test** — no Spring context, no database, no HTTP — just the service class's logic exercised directly.

### 4. Testing Spring MVC Controllers
A controller test should verify HTTP-facing behavior: correct status code, correct JSON shape, correct routing — **without** needing a full running server.
```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Test
    void getProductReturnsOk() throws Exception {
        when(productService.getProduct(1L)).thenReturn(new Product(1L, "Laptop", 50000));

        mockMvc.perform(get("/api/products/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Laptop"));
    }
}
```
- **`@WebMvcTest`** — loads only the **web layer** (controllers, `DispatcherServlet` machinery) — not the full application context, not real repositories.
- **`MockMvc`** — simulates HTTP requests against the controller without starting a real embedded server.
- **`@MockBean`** — replaces a bean (here, `ProductService`) in the Spring test context with a Mockito mock.

**What a `MockMvc` call conceptually does under the hood** (matching the Lecture 15 MVC flow): matches the request against `@GetMapping("/{id}")`, extracts and converts the path variable, invokes the controller method, serializes the returned object to JSON, and produces an HTTP response for assertions to check.

### 5. Repository & Integration Testing
**Why repository tests need a real (test) database:** mocking a repository only proves Mockito returns whatever you configured it to return — it proves nothing about whether the *actual query* (derived method name, JPQL, native SQL) is even correct:
```java
// This only proves Mockito works, NOT that the real query is correct:
ProductRepository repository = mock(ProductRepository.class);
when(repository.findByNameIgnoreCase("laptop")).thenReturn(Optional.of(product));
```
**`@DataJpaTest`** — loads just the JPA/repository layer against a real (typically in-memory, e.g. H2) test database, letting you verify that your derived query methods and custom `@Query` annotations actually produce correct SQL and correct results — a genuine (if narrow) integration test, distinct from the service-layer unit test above.

### Testing layers, summarized
| Test type | Annotation | Scope | Database? | HTTP? |
|---|---|---|---|---|
| Unit test (service) | `@ExtendWith(MockitoExtension.class)` | One class, dependencies mocked | No | No |
| Web layer test | `@WebMvcTest` | Controller + MVC machinery | No | Simulated via `MockMvc` |
| Repository/integration test | `@DataJpaTest` | JPA/repository layer | Yes (test DB) | No |
| Full integration test | `@SpringBootTest` | Whole application context | Optionally yes | Optionally real HTTP calls |

### Key takeaways
- **Unit tests** isolate one class using mocks (`@Mock`/`@InjectMocks`) — fast, no Spring context needed.
- **`@WebMvcTest` + `MockMvc`** tests the web/controller layer without a real server or database.
- **`@DataJpaTest`** actually exercises your queries against a real (test) database — mocked repositories cannot validate query correctness.
- Choose the **narrowest** test type that proves what you actually need to prove — full `@SpringBootTest` integration tests are valuable but much slower; reserve them for genuinely end-to-end scenarios.

---

## Course-wide recap
Across these 40 lectures the course builds one continuous mental model, layer by layer:
```
HTTP/Client-Server (L1) → Servlets & Tomcat (L14) → Spring MVC's DispatcherServlet (L15)
   → Spring Core's IoC container & beans (L4-9) wired underneath everything
   → Configuration & CRUD (L10-13) → DTOs, validation, exceptions (L16-17) → Profiles (L18)
   → Cross-cutting concerns: Filters → Interceptors → AOP (L19-25)
   → Data access: JDBC → Spring JDBC → Hibernate → JPA relationships → Spring Data JPA → Transactions (L26-34)
   → Security: authentication fundamentals → DB-backed auth → filter internals → JWT → OAuth2/OIDC (L35-39)
   → Testing every layer you just built (L40)
```
Each later concept explicitly builds on internals explained earlier (e.g. AOP proxies use the bean lifecycle from L7; JWT auth reuses the `SecurityFilterChain` internals from L37; Spring Data JPA sits on the Hibernate persistence-context concepts from L29) — which is why working through the notes in this order (rather than jumping straight to, say, Security or JPA) makes the "why" behind each abstraction much clearer.
