# Spring Framework — 13: API Documentation, Database Migrations & File Uploads
### New topic. Full depth, story format.

---

## Story 1: The API documentation that lied — Swagger/OpenAPI

**The situation:** A frontend developer asks the backend team "what fields does `POST /api/students/create` expect?" The backend team points to a **Google Doc** written three months ago, describing the API's *original* shape — before someone added a required `subject` field and removed `rollNo` entirely. The frontend developer builds against the stale doc, ships broken code, and the bug isn't caught until QA.

**Why manually-maintained API docs almost always drift out of sync:** nothing **forces** a developer to update a separate document the moment they change a DTO — it's an easy, invisible-until-it-breaks step to skip under deadline pressure, and there's no compiler or test that catches "the docs and the code disagree."

**The Spring answer: springdoc-openapi** — generates API documentation **directly from your actual code** (controllers, DTOs, validation annotations), so the docs literally **cannot** drift out of sync with reality, because they're derived from the same source of truth as the running API.
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```
Adding this dependency alone auto-generates an interactive documentation UI at `/swagger-ui.html`, reading your existing `@RestController`/`@RequestMapping`/`@Valid` DTOs — **no code changes required** to get a basic version working.

**Enriching it with real, precise documentation:**
```java
@RestController
@RequestMapping("/api/students")
@Tag(name = "Student Management", description = "CRUD operations for student records")
public class StudentController {

    @Operation(summary = "Create a new student", description = "Creates a student record and returns the saved entity")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Student created successfully"),
        @ApiResponse(responseCode = "400", description = "Validation failed"),
        @ApiResponse(responseCode = "409", description = "Duplicate email")
    })
    @PostMapping("/create")
    public ResponseEntity<StudentResponseDto> createStudent(@Valid @RequestBody StudentRequestDto dto) {
        ...
    }
}
```
```java
public class StudentRequestDto {
    @Schema(description = "Student's full name", example = "Aditya Sharma")
    @NotBlank
    private String name;

    @Schema(description = "Student's email address", example = "aditya@example.com")
    @Email
    private String email;
}
```
**Why this specific approach (code-derived docs) over "just write really disciplined docs manually":** discipline doesn't scale across a team, deadlines, and turnover — a **structural** guarantee (docs generated from the same code that defines the actual API contract) is far more reliable than a **process** guarantee (everyone remembers to update the doc). This is the same reasoning as preferring compile-time checks over runtime discipline elsewhere in Spring (e.g. constructor injection over field injection, from Doc 1).

**Interview angle:** *"How do you keep API documentation accurate as the API evolves?"* — naming code-generated documentation (springdoc-openapi/Swagger) and explaining *why* it structurally can't drift the way manually-maintained docs do is the expected depth, not just "we write good docs."

---

## Story 2: The schema change that broke production — why `ddl-auto=update` is a trap

**The situation:** A team develops locally with:
```properties
spring.jpa.hibernate.ddl-auto=update
```
This is genuinely convenient in development — add a field to an `@Entity`, restart the app, and Hibernate **automatically alters the database table** to match. No manual `ALTER TABLE` needed. It feels like magic.

**Where it becomes dangerous:** the same `ddl-auto=update` setting is left on in **production**. A developer renames an entity field (`rollNo` → `enrollmentNumber`). Hibernate doesn't understand "this is a rename" — it sees a field that no longer exists (`rollNo`) and a new one that does (`enrollmentNumber`), so it **adds a new column** and **leaves the old one behind**, orphaned, with all its data now invisible to the application (since nothing reads the old column name anymore) but still silently sitting in the production database. In a worse version of this story, a **type change** (e.g. `int` → `Long`) can cause Hibernate to attempt an automatic column type conversion that **fails or silently truncates data**, depending on the database and the specific change.

**Why `ddl-auto=update` is fundamentally the wrong tool for production, not just "risky":** it gives **Hibernate**, not a human, the final say over exactly what SQL runs against your production database on every deploy — with no review step, no way to test the migration in isolation beforehand, and no rollback plan if it goes wrong. This is the schema-management equivalent of `git push --force` directly to production with no one reviewing the diff.

**The fix — versioned, explicit migrations via Flyway (or Liquibase):**
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```
```properties
spring.jpa.hibernate.ddl-auto=validate   # Hibernate CHECKS the schema matches entities, never changes it
spring.flyway.enabled=true
```
```sql
-- src/main/resources/db/migration/V1__create_students_table.sql
CREATE TABLE students (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    roll_no INT
);
```
```sql
-- V2__rename_roll_no_to_enrollment_number.sql
ALTER TABLE students CHANGE COLUMN roll_no enrollment_number INT;
```
```sql
-- V3__add_subject_column.sql
ALTER TABLE students ADD COLUMN subject VARCHAR(255);
```
**How this fundamentally changes the safety model:** every schema change is now an explicit, **numbered, version-controlled SQL file**, reviewed in a pull request like any other code change, applied by Flyway **in order**, and **tracked in a `flyway_schema_history` table** so Flyway always knows exactly which migrations have already run on any given database — no ambiguity, no Hibernate guesswork about what changed and why.

**`ddl-auto=validate` — the crucial complementary setting:** instead of `update` (let Hibernate change the schema) or `none` (Hibernate doesn't check anything), `validate` tells Hibernate to **compare your `@Entity` classes against the actual database schema at startup and fail loudly if they don't match** — catching the exact class of bug where a developer changes an entity but forgets to write the corresponding Flyway migration, **before** the app even starts serving traffic, rather than discovering the mismatch via a confusing runtime SQL error later.

**Interview angle:** *"How do you manage database schema changes in a Spring Boot application, especially across a team?"* — naming Flyway/Liquibase specifically, contrasting it with `ddl-auto=update`'s production dangers (the rename/orphaned-column failure mode is a great concrete example to have ready), and knowing `ddl-auto=validate` as the safe complementary setting shows real production judgment.

---

## Story 3: The file upload that filled the disk — and the security hole hiding inside it

**The situation:** A profile-picture upload feature, built quickly:
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) throws IOException {
    String filePath = "/app/uploads/" + file.getOriginalFilename();
    file.transferTo(new File(filePath));
    return ResponseEntity.ok("Uploaded: " + filePath);
}
```
**Where it breaks — three separate, real problems, not just one:**

**Problem 1 — unbounded file size.** Nothing here limits how large `file` can be. A malicious (or just careless) user uploads a 10 GB file, and the server's disk fills up — taking down the *entire application* (and possibly other apps sharing that disk), not just the upload feature.

**Problem 2 — path traversal via the original filename.** `file.getOriginalFilename()` is **client-supplied, untrusted input**. A malicious filename like `../../etc/cron.d/malicious` could, depending on how naively the path is constructed, let an attacker write a file **outside** the intended uploads directory entirely — a classic, serious security vulnerability (this exact bug pattern is why "never trust client input for file paths" is such an emphasized rule).

**Problem 3 — no content-type validation.** Nothing checks that the uploaded file is actually an image. A user could upload an executable `.jsp`/`.php` file, and — depending on how the server later serves files from that directory — potentially get it **executed** by the server, a severe remote-code-execution risk.

**The fix, addressing all three:**
```properties
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=5MB
```
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) throws IOException {

    // Problem 3 fix — validate actual content type, don't trust the extension or client-provided type blindly
    String contentType = file.getContentType();
    if (!List.of("image/png", "image/jpeg").contains(contentType)) {
        return ResponseEntity.badRequest().body("Only PNG/JPEG images are allowed");
    }

    // Problem 2 fix — NEVER use the client-supplied filename directly; generate your own safe name
    String extension = contentType.equals("image/png") ? ".png" : ".jpg";
    String safeFilename = UUID.randomUUID() + extension;

    Path uploadPath = Paths.get("/app/uploads").resolve(safeFilename).normalize();
    if (!uploadPath.startsWith(Paths.get("/app/uploads"))) {
        throw new SecurityException("Invalid upload path"); // defense in depth, even with a generated name
    }

    file.transferTo(uploadPath.toFile());
    return ResponseEntity.ok(safeFilename);
}
```
**Why generating your own filename (`UUID.randomUUID()`) rather than trying to "sanitize" the client's filename:** sanitization (stripping `../`, special characters) is a **blocklist** approach — it's easy to miss an edge case an attacker can exploit. Generating a completely new, safe filename server-side and **never using client input as a filesystem path at all** is an **allowlist** approach — structurally immune to path traversal, rather than defending against every known variant of it.

**Where uploaded files should actually live in production — not the app server's local disk at all:** even with the fixes above, storing uploads on the same disk as the running application doesn't scale (Problem 1's disk-exhaustion risk doesn't fully go away — it's now bounded per-file, but not bounded in *total* across many uploads) and doesn't survive a redeploy (a fresh container/instance has an empty disk). Real production systems typically upload to **object storage** (AWS S3, Google Cloud Storage, Azure Blob Storage) instead:
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) throws IOException {
    String safeFilename = UUID.randomUUID() + getValidatedExtension(file);
    s3Client.putObject(PutObjectRequest.builder()
        .bucket("company-uploads")
        .key(safeFilename)
        .build(), RequestBody.fromInputStream(file.getInputStream(), file.getSize()));
    return ResponseEntity.ok(safeFilename);
}
```
This sidesteps disk exhaustion entirely (object storage scales independently of the app server), and survives redeploys/scaling naturally, since the files aren't tied to any specific app instance's local disk at all.

**Interview angle:** *"What do you need to think about when implementing file upload?"* — a strong answer volunteers all three problems (size limits, path traversal via client-supplied filenames, content-type validation) **without being prompted for each one separately**, and knows that production file storage generally means object storage, not the app server's own disk — this is a very common practical/coding-round topic, not just a theory question.

---

## Quick-fire: concepts worth knowing exist

| Concept | What it is | Why it matters |
|---|---|---|
| **Presigned URLs (S3)** | A temporary, scoped URL that lets a client upload directly to S3, bypassing your app server entirely | Removes the app server from the upload's data path completely — better for very large files, since your server's bandwidth/memory never touches the actual file bytes |
| **Content-Disposition header on download** | Controls whether a browser displays a file inline or prompts a download, and under what filename | A commonly-missed detail — serving user-uploaded content without the right header can expose your app to reflected content-type confusion issues |
| **`spring.jpa.open-in-view`** | A related-but-distinct Boot default (`true` by default) that keeps the Hibernate session open through view rendering | Connects back to the `LazyInitializationException`/Open-Session-In-View discussion in the JPA scenario doc — worth explicitly setting `false` in most modern REST API projects, since the DTO-mapping pattern from Lecture 16 makes it unnecessary and it carries the same connection-pool-holding-too-long risk |
| **Liquibase vs Flyway** | Liquibase supports XML/YAML/JSON changesets (not just raw SQL) and has more built-in rollback tooling; Flyway is simpler, SQL-first | Flyway's simplicity is often preferred for straightforward projects; Liquibase's format flexibility and rollback support appeal to larger, more complex, multi-database-vendor projects |

---

## Closing note — how these 13 documents fit together

The original 7 course-notes files teach **what** each Spring feature does and how to use it. The 7 companion scenario docs teach **where it breaks in practice** and why the "obvious" fix is often wrong — with the self-invocation/proxy trap as the connective thread running through beans, AOP, transactions, caching, and async. These final 6 topic docs (08–13) extend the same story-driven approach to everything a real production Spring system needs beyond a single course: observability, caching, async/scheduling, messaging, microservices, and the operational concerns (docs, migrations, uploads) that don't show up until an application actually has to survive contact with real users, real scale, and real failure modes.
