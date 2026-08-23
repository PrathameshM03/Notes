# Spring Framework Full Course — Notes 6: Data Access — JDBC to Spring Data JPA
### Covers Lectures 26–34: Raw JDBC → Spring JDBC (`JdbcTemplate`) → Hibernate internals → JPA relationships → Cascading, Lazy Loading & N+1 → Spring Data JPA → Transactions & ACID → Propagation

---

## Lecture 26 — JDBC Fundamentals and CRUD

### 1. How Java talks to a database
A Java app and a database server are **separate programs**. Communication flow:
1. Establish a connection with the DB server.
2. Send SQL to the server.
3. Wait for the server's response.
4. Convert the response into objects Java can work with.

### 2. JDBC API and JDBC driver
**JDBC (Java Database Connectivity)** is the standard Java API for relational database communication. It's an abstraction — the actual database-specific protocol work is done by a **JDBC driver** (e.g. MySQL Connector/J), which:
1. Receives the SQL command + parameter values from your Java call.
2. Converts them into the database's own network protocol.
3. Sends the request to the DB server.
4. The server parses, validates, plans, and executes it.
5. The server returns rows / metadata / update count / an error.
6. The driver decodes the database response back into JDBC types (like `ResultSet`).
7. Your Java code reads and maps the returned data.

### 3. Core JDBC objects
```java
Connection conn = DriverManager.getConnection(url, user, password);
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM students WHERE id = ?");
stmt.setLong(1, id);
ResultSet rs = stmt.executeQuery();
while (rs.next()) {
    Long studentId = rs.getLong("id");
    String name = rs.getString("name");
    // manually build a Student object from each row
}
```
- **`Connection`** — an active DB session.
- **`Statement`** — executes plain SQL text.
- **`PreparedStatement`** (preferred) — executes **parameterized** SQL: placeholders (`?`) protect *values* from being interpreted as SQL (this is the core defense against SQL injection) and allow query plan reuse.
- **`ResultSet`** — a cursor over returned rows; **starts positioned before the first row** — you must call `rs.next()` to advance to (and check for) each row.
- **`executeQuery()`** — for statements that return rows (SELECT).
- **`executeUpdate()`** — for INSERT/UPDATE/DELETE, returns an affected-row count.
- **`execute()`** — a boolean identifying the *type* of the first result, **not** whether the operation "succeeded."

### 4. Manual row-to-object mapping
JDBC does **not** automatically convert a row into a Java object — you must do it manually: move the cursor, read each column, construct the object, assign fields, add to a collection if reading multiple rows. This repetitive mapping code is one of raw JDBC's biggest pain points — and the exact thing Spring JDBC and JPA/Hibernate later automate.

### 5. Closing resources
`ResultSet`, `Statement`, and `Connection` must all be closed to avoid leaking DB resources — **prefer try-with-resources** over manual `finally` blocks:
```java
try (Connection conn = DriverManager.getConnection(url, user, pass);
     PreparedStatement stmt = conn.prepareStatement(sql)) {
    ...
}
```

### Key takeaways
- JDBC is the low-level standard API; a driver implements the actual database wire protocol.
- Use `PreparedStatement` for any dynamic values — placeholders protect *values*, not dynamic SQL identifiers (table/column names still need careful, non-user-controlled handling).
- Raw JDBC's repetitiveness is intentional context for *why* Spring JDBC (next lecture) and JPA add real value.

---

## Lecture 27 — Spring JDBC: From Raw JDBC to `JdbcTemplate`

### 1. The problem with raw JDBC
Every single query repeats the same boilerplate: obtain a connection, create a statement, bind parameters, execute, process the result, handle `SQLException`, close every resource. **Spring JDBC's `JdbcTemplate`** exists to eliminate this repetition **without removing SQL from the application** — you still write SQL yourself; Spring just manages the surrounding plumbing.

### 2. `DataSource` — abstraction over "how to get a connection"
> A **`DataSource`** is an abstraction that provides database connections — it may or may not use connection pooling under the hood.

**Connection pooling** — instead of opening/closing a raw TCP connection to the DB for every single query (expensive), a pool keeps a set of live connections ready to reuse.

**HikariCP** is the connection pool Spring Boot uses **by default** when it's on the classpath (it is, automatically, via `spring-boot-starter-jdbc` / `spring-boot-starter-data-jpa`) — chosen for its performance and low overhead.

**How Spring Boot creates the `DataSource` automatically:**
1. Detects Spring JDBC on the classpath.
2. Detects a JDBC driver (e.g. MySQL Connector) on the classpath.
3. Reads `spring.datasource.*` properties (`url`, `username`, `password`, etc.).
4. Detects an available connection-pool implementation.
5. Prefers HikariCP when available.
6. Creates and configures the `DataSource`.
7. Registers it as a Spring bean — ready for injection anywhere.

### 3. `JdbcTemplate` — what it automates
For an **update** (INSERT/UPDATE/DELETE): obtain a connection, create a `PreparedStatement`, bind parameters, execute, read the affected-row count, close the statement, release the connection back to the pool, translate `SQLException` when necessary — **all handled by `JdbcTemplate` internally**; you just call one method.

For a **query**: obtain a connection, create/bind a `PreparedStatement`, execute, obtain the `ResultSet`, iterate rows, call a `RowMapper` for each row, collect mapped objects, close everything, translate exceptions — again, all internal.

```java
@Repository
public class StudentRepository {
    private final JdbcTemplate jdbcTemplate;
    public StudentRepository(JdbcTemplate jdbcTemplate) { this.jdbcTemplate = jdbcTemplate; }

    public int create(Student s) {
        return jdbcTemplate.update(
            "INSERT INTO students (name, age, email) VALUES (?, ?, ?)",
            s.getName(), s.getAge(), s.getEmail());
    }

    public Student findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM students WHERE id = ?",
            new BeanPropertyRowMapper<>(Student.class), id);
    }

    public List<Student> findAll() {
        return jdbcTemplate.query("SELECT * FROM students", new BeanPropertyRowMapper<>(Student.class));
    }

    public int update(Student s) {
        return jdbcTemplate.update(
            "UPDATE students SET name=?, age=?, email=? WHERE id=?",
            s.getName(), s.getAge(), s.getEmail(), s.getId());
    }

    public int delete(Long id) {
        return jdbcTemplate.update("DELETE FROM students WHERE id=?", id);
    }
}
```

### 4. `RowMapper` vs `BeanPropertyRowMapper`
- **`RowMapper<T>`** — you implement `mapRow(ResultSet rs, int rowNum)` yourself, converting one row into one object — full explicit control.
- **`BeanPropertyRowMapper<T>`** — automatically maps columns to bean properties by name/convention, reducing boilerplate but giving less explicit control over the mapping.

### 5. Exception translation
Spring JDBC **translates** raw `SQLException` (a checked exception, database-vendor-specific, with cryptic error codes) into Spring's own **`DataAccessException`** hierarchy — an **unchecked**, consistent, vendor-agnostic set of exception types. This is important context for later: it's the foundation Spring Data JPA repositories also build on (Lecture 32), and it means catching a "duplicate key" style failure doesn't require parsing vendor-specific SQL error codes yourself.

> A database-specific failure (constraint violation, connectivity issue) should not be confused with a legitimate "record not found" — the two are conceptually and exceptionally different outcomes, and Spring's exception hierarchy keeps them distinguishable.

### Key takeaways
- Spring JDBC still uses JDBC internally — SQL doesn't go away, only the boilerplate does.
- `DataSource` abstracts *how* you get a connection; HikariCP is Spring Boot's default pool implementation.
- `JdbcTemplate` handles the repetitive obtain/bind/execute/map/close/translate cycle.
- `RowMapper` = explicit row mapping; `BeanPropertyRowMapper` = convention-based, less code.
- Spring translates raw `SQLException` into the `DataAccessException` hierarchy — this reduces boilerplate while preserving direct control over the actual SQL you write.

---

## Lecture 28 & 29 — Introduction to Hibernate & Hibernate Internals

### 1. JPA (specification) vs Hibernate (provider) vs Spring Boot (integrator)
| Layer | Role |
|---|---|
| **JPA** (Jakarta Persistence) | A **specification** — defines interfaces/annotations/contracts (`EntityManagerFactory`, `EntityManager`, `@Entity`, `@Id`, `@GeneratedValue`) but **no actual ORM engine**. |
| **Hibernate** | A **JPA provider** — implements the JPA contract: maps entities to tables, generates SQL, talks to JDBC, tracks entity state, performs dirty checking, manages the persistence context. |
| **Spring Boot** | Configures the persistence infrastructure (auto-configures a `DataSource`, `EntityManagerFactory`, transaction management) so JPA + Hibernate "just work." |

Hibernate also exposes a **native API** parallel to JPA's — `SessionFactory extends EntityManagerFactory`, `Session extends EntityManager` — so it can support the standard JPA API *and* Hibernate-specific extras simultaneously. Programming against the **JPA types** (not Hibernate-native ones) keeps your persistence code more provider-independent.

### 2. Core components and their lifetimes
| Component | Responsibility | Typical lifetime |
|---|---|---|
| `EntityManagerFactory` | Creates `EntityManager` instances, holds shared ORM metadata | Entire application |
| `EntityManager` | Operations for interacting with entities + the persistence context | One unit of work / transaction |
| Persistence Context | Stores & tracks managed entity instances | Usually one transaction |
| Database Transaction | Ensures a group of DB operations succeeds/fails as one unit | One business operation |

`EntityManagerFactory` is expensive to create and shared for the app's lifetime; an `EntityManager` is short-lived and **must not be shared across threads concurrently**.

### 3. The Persistence Context — Hibernate's in-memory workspace
> The **persistence context** is Hibernate's in-memory area that tracks managed entity objects by identity, supporting automatic change detection, first-level caching, transactional write-behind, and synchronization with the database.

```java
Student student = entityManager.find(Student.class, 1L);
```
If not already in the current persistence context, Hibernate executes a `SELECT`, builds the `Student` object, places it in the persistence context, and begins tracking its state.

**`EntityManager`'s real purpose** goes beyond "runs queries" — it's the main interface for interacting with the persistence context: `persist()`, `find()`, `remove()`, `flush()`, `detach()`, `clear()`, `refresh()` — each affects an entity, the persistence context, or their synchronization.

### 4. `@PersistenceContext` in Spring — why not one shared `EntityManager`?
```java
@Repository
public class StudentRepository {
    @PersistenceContext
    private EntityManager entityManager;
}
```
Spring repositories are **singleton beans**, but a real `EntityManager` (like a Hibernate `Session`) is **not thread-safe** — sharing one physical instance across concurrent requests would be unsafe. So Spring injects a **shared proxy** that internally routes each call to the `EntityManager` associated with the **current transaction** — meaning two concurrent requests can use the *same* injected repository field while each actually works against a *different* underlying persistence context.

### 5. Transactions, from first principles
Classic example: transferring money between two accounts needs two `UPDATE`s to succeed *together* or *not at all* — otherwise money can vanish if the second update fails after the first succeeds.
```
BEGIN
  UPDATE account A
  UPDATE account B
COMMIT      -- or ROLLBACK if any step fails
```
The region between begin and commit/rollback is the **transaction boundary**. A **unit of work** is the complete set of persistence actions needed for one business operation (e.g. retrieve → validate → modify → save).

### 6. `@Transactional`
Spring's `@Transactional` annotation demarcates a method (typically a **service** method) as a transaction boundary — Spring wraps the method call in a proxy that begins a transaction before the method runs and commits/rolls back after, based on whether the method completes normally or throws.

### 7. Entity Lifecycle States
| State | Meaning |
|---|---|
| **Transient** | A plain Java object — `new Student()` — not yet associated with any persistence context or database row. |
| **Managed** | Tracked by the current persistence context — Hibernate watches it for changes. |
| **Detached** | Was managed once, but the persistence context that tracked it is gone (closed, cleared, or explicitly detached) — changes no longer sync automatically. |
| **Removed** | Marked for deletion — will be deleted from the DB at the next flush/commit. |

The **managed** state is central: dirty checking, automatic updates, first-level caching, relationship synchronization, and flushing all depend on an entity being in this state.

### 8. Dirty Checking
> An entity is **dirty** when its current managed state differs from the state Hibernate previously recorded for it (e.g. `name` was `"Rahul"`, is now `"Rohan"`).

The key practical implication: **you often don't need to call `save()`/`update()` explicitly** for a managed entity — just modify its fields; Hibernate detects the difference and generates the appropriate `UPDATE` automatically at flush/commit time.

### 9. Flushing & Transactional Write-Behind
**Flush** = the point where Hibernate actually synchronizes the persistence context's changes with the database by executing the necessary SQL. Hibernate does **not** send SQL to the DB the instant you change a Java field — instead it batches up changes and flushes them at strategic points (before a query that might need to see them, or at commit) — this is called **transactional write-behind**, as opposed to a naive "write-through" approach where every Java change would trigger immediate SQL.

### 10. First-Level Cache
The persistence context **is** Hibernate's first-level cache — within one persistence context, requesting the same entity (`Student` id `1`) twice returns the **same Java object reference** without hitting the database a second time (as long as it's still tracked). Calling `entityManager.clear()` or `detach(entity)` removes entities from this cache (and from managed tracking); `refresh(entity)` re-reads the entity's state from the database, overwriting any in-memory changes.

### Common misconceptions addressed
- Dirty ≠ invalid/corrupted — it just means "changed since Hibernate last knew about it."
- Hibernate does not necessarily execute SQL the moment you call a setter — write-behind batches it.
- The injected `EntityManager` field looks like one shared object, but it's a proxy routing to per-transaction instances underneath.

---

## Lecture 30 — JPA Relationships: Mapping Entities

### 1. The fundamental problem
Java objects hold **references** to other objects; relational databases hold **foreign keys**. JPA's relationship annotations exist to bridge these two different worlds.

### 2. Relationship cardinality & direction
- **Cardinality**: `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany` — describes how many entities can be associated on each side.
- **Unidirectional** — only one entity navigates to the other in Java (e.g. `Student` has a `Department` field, but `Department` has no `List<Student>`).
- **Bidirectional** — both sides navigate to each other in Java (`Student.department` **and** `Department.students`) — but this **still represents one single database relationship**, not two.

### 3. Owning side vs Inverse side
> When a relationship is mapped bidirectionally, JPA needs one **authoritative** side — the **owning side** is the side JPA actually reads to determine the foreign-key value written to the database. The other side is the **inverse side** (`mappedBy`), which exists purely for convenient Java navigation and does **not** control the database column.

**Critical consequence:** both sides of a bidirectional relationship must be kept in sync **in Java** (usually via a helper method), because simply setting the inverse side's collection does **not** automatically persist the relationship — only the owning side's value is written to the DB.

### 4. Many-to-One / One-to-Many (the same relationship, two views)
```java
// Owning side — Student holds the foreign key
@Entity
public class Student {
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
}
```
```java
// Inverse side — Department, no additional column, purely navigational
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    private List<Student> students = new ArrayList<>();
}
```
`@ManyToOne` is (almost always) the **owning side** because the foreign key naturally lives on the "many" side's table (`students.department_id`). `mappedBy = "department"` tells JPA "don't create a new column for this — this side just mirrors the relationship owned by the `department` field over on `Student`."

### 5. One-to-One
```java
@Entity
public class Student {
    @OneToOne
    @JoinColumn(name = "profile_id")
    private StudentProfile profile;   // owning side, holds the FK
}
@Entity
public class StudentProfile {
    @OneToOne(mappedBy = "profile")
    private Student student;          // inverse side
}
```

### 6. Many-to-Many
Requires a **join table** (no single foreign key can express a many-to-many relationship):
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private List<Course> courses;
}
```

### 7. Lombok with JPA entities — a caution
Lombok's `@Data`/`@ToString`/`@EqualsAndHashCode` can be dangerous on entities with bidirectional relationships — generated `toString()`/`equals()`/`hashCode()` can trigger **infinite recursion** (Student → Department → Students → Department → …) or unwanted lazy-loading. Common practice: exclude relationship fields from generated `toString()`/`equals()`/`hashCode()`, or write them manually.

### Quick reference
1. Java stores entity **references**; relational databases store **foreign keys**.
2. Cardinality describes how many entities associate on each side; unidirectional/bidirectional describes **Java navigation only**, not a second database relationship.
3. The **owning side** determines what relationship value JPA actually writes.
4. Both sides of a bidirectional relationship must be manually kept synchronized in Java — a **helper method** on the "parent" entity (e.g. `addStudent(Student s)` that sets both sides) is the safest pattern.
5. Setting an entity reference alone does **not** automatically persist a *new* referenced entity — cascading (next lecture) controls that.

---

## Lecture 31 — JPA Cascading, Lazy Loading & the N+1 Problem

### 1. Three questions this lecture answers
1. What should happen to related entities when one entity is saved/deleted?
2. When should related data actually be loaded from the database?
3. How do we avoid a huge number of hidden SQL queries?

### 2. Cascading
> **Cascading** propagates a persistence operation (persist, remove, etc.) performed on a parent entity to its related entities automatically.

```java
@OneToMany(mappedBy = "department", cascade = CascadeType.PERSIST)
private List<Student> students;
```
- **`CascadeType.PERSIST`** — saving the parent also saves any *new* (transient) related entities.
- **`CascadeType.REMOVE`** — deleting the parent also deletes related entities.
- Other cascade types (`MERGE`, `REFRESH`, `DETACH`, `ALL`) apply the corresponding operation across the relationship similarly.

**Important distinction:** cascading controls **lifecycle propagation** (save/delete behavior) — it does **not** define or change the relationship's cardinality or mapping itself.

### 3. Relationship Loading — `FetchType.LAZY` vs `FetchType.EAGER`
- **`EAGER`** — the related entity/collection is loaded **immediately**, as part of loading the parent.
- **`LAZY`** — the related entity/collection is loaded **only when actually accessed** in code — Hibernate returns a proxy initially and fires the real query the first time you touch it.

**Default fetch types (JPA spec defaults):**
| Relationship | Default |
|---|---|
| `@ManyToOne` | EAGER |
| `@OneToOne` | EAGER |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

Lazy loading is useful because it avoids pulling in large related graphs you don't actually need for a given operation — but it introduces a well-known trap:

### 4. The N+1 Query Problem
> Occurs when **one query loads a collection of parent entities**, and then **an additional query fires for each parent** when a lazy relationship is accessed — 1 initial query + N follow-up queries.

```java
@ManyToOne(fetch = FetchType.LAZY)
private Department department;
```
```java
List<Student> students = studentRepository.findAll();   // 1 query
for (Student s : students) {
    System.out.println(s.getDepartment().getName());     // N additional queries, one per student!
}
```
- **First-level cache mitigates repeated access to the *exact same* entity** (Lecture 29), but N+1 happens across *different* entities (one department lookup per distinct student) — the cache doesn't prevent this pattern.
- Detecting it: enable SQL logging (`spring.jpa.show-sql=true` / Hibernate SQL logging) and watch for a burst of near-identical queries following one initial `findAll()`-style query.
- **Does N+1 mean lazy loading is bad?** No — lazy loading itself is a sound default; N+1 happens when code *iterates* and *accesses* a lazy relationship for every row without telling Hibernate up front that it'll need that related data for the whole batch.
- **Preventing N+1:** use an **entity graph** (`@EntityGraph` or a JPQL `JOIN FETCH`) to tell Hibernate, in the *original* query, to eagerly fetch the needed relationship for the whole result set in one go — turning N+1 queries into 1 (or 2) well-planned ones.

```java
@EntityGraph(attributePaths = "department")
List<Student> findAll();
```
or
```java
@Query("SELECT s FROM Student s JOIN FETCH s.department")
List<Student> findAllWithDepartment();
```

---

## Lecture 32 — Spring Data JPA

### 1. The problem Spring Data JPA solves
Even with JPA + `EntityManager`, writing a full repository class (find, save, delete, custom queries) for every entity is still substantial boilerplate. **Spring Data JPA** removes almost all of it via a repository **interface hierarchy**.

### 2. JPA vs Hibernate vs Spring Data JPA — final clarification
> **JPA is a specification; Hibernate is a (common) JPA provider; Spring Data JPA is a further abstraction on top of JPA** that generates repository implementations for you at runtime, so you rarely write manual `EntityManager` calls at all for standard CRUD.

### 3. Declaring a repository
```java
public interface StudentRepository extends JpaRepository<Student, Long> {
    // Student = entity type, Long = identifier (primary key) type
}
```
`JpaRepository<Student, Long>` captures **both** the entity type and its ID type — this alone gives you `save()`, `findById()`, `findAll()`, `deleteById()`, `existsById()`, `count()`, and more — **no implementation class needed**.

### 4. Repository proxy internals
At startup, **Spring creates a runtime proxy** implementing your repository interface. Standard methods (`save`, `findById`, etc.) are backed by a shared generic implementation (`SimpleJpaRepository`); your **custom derived query methods** are parsed from their method *names* and translated into JPQL/SQL automatically.

### 5. Derived query methods
```java
List<Student> findByDeletedIsFalseOrderByAgeDesc(int limit);
```
Spring Data parses `findBy` + `Deleted` (field) + `IsFalse` (condition) + `OrderByAgeDesc` (sort), builds the query, and executes it — no query string written by hand. This works well while method names remain readable; very complex conditions are better expressed as an explicit query.

### 6. Custom JPQL & native queries
```java
@Query("SELECT s FROM Student s WHERE s.age > :minAge")
List<Student> findOlderThan(@Param("minAge") int minAge);

@Query(value = "SELECT * FROM students WHERE age > :minAge", nativeQuery = true)
List<Student> findOlderThanNative(@Param("minAge") int minAge);
```
- **JPQL** (`@Query` without `nativeQuery = true`) works with **entities and their relationships** (object-oriented query language) — portable across databases.
- **Native SQL** (`nativeQuery = true`) works directly with **tables and columns** — full SQL power, but tied to the specific database dialect.

### 7. Sorting & Pagination
```java
Page<Student> findByDeletedIsFalse(Pageable pageable);
```
- **`Sort`** — explicit ordering; should always be specified when order actually matters (don't rely on incidental DB ordering).
- **`Page<T>`** — includes total element/page counts, which typically requires an extra **count query** behind the scenes.
- **`Slice<T>`** — like `Page` but **omits the total count** (cheaper — no count query) — useful for infinite-scroll-style UIs that only need "is there a next page," not an exact total.

### Complete mental model / key takeaways
1. JPA = specification; Hibernate = a common provider; Spring Data JPA = a further abstraction generating repository implementations for you.
2. `JpaRepository<Student, Long>` captures the entity type + its identifier type together.
3. Spring creates a **repository proxy at runtime**; standard CRUD methods are mostly generic, shared implementation.
4. Managed entities update via **dirty checking** even without an explicit `save()` call, as long as the change happens inside a transaction on a managed instance.
5. Derived query methods are effective while their method names stay readable — reach for `@Query` once they get unwieldy.
6. JPQL works with entities/relationships; native SQL works with raw tables/columns.
7. Sorting should be explicit whenever order genuinely matters.
8. `Page` provides totals (usually needs a count query); `Slice` omits totals for a cheaper query.
9. Spring Data JPA reduces boilerplate, but correct **transactions**, **mappings**, and **fetch strategies** are still entirely your responsibility — the abstraction doesn't remove the need to understand what's underneath it (which is exactly why Lectures 29–31 come first).

---

## Lecture 33 — Introduction to Transactions & ACID

### 1. Database Transaction
A **transaction** groups multiple database operations into one all-or-nothing logical unit — either every operation succeeds (**commit**), or none of them take effect (**rollback**).

### 2. ACID Properties
| Property | Guarantee |
|---|---|
| **Atomicity** | All operations in the transaction succeed together, or none do — no partial application. |
| **Consistency** | A transaction moves the database from one valid state to another valid state — constraints/rules are never violated at the end of a committed transaction. |
| **Isolation** | Concurrently running transactions don't interfere with each other's intermediate (uncommitted) state. |
| **Durability** | Once committed, changes survive — even a crash immediately after commit doesn't lose the data. |

These four together form **one combined guarantee**: "a transaction behaves as a single, reliable, isolated, permanent unit of change" — none of the four properties is very useful in isolation from the others.

### 3. Transaction boundary & begin/commit/rollback
```
BEGIN
  ... operations ...
COMMIT     -- success path
-- or --
ROLLBACK   -- failure path, undoes everything since BEGIN
```
The **transaction boundary** is the region between `BEGIN` and the final `COMMIT`/`ROLLBACK`.

### 4. Transaction layers: JDBC, JPA, Hibernate, Spring
Each layer has its own notion of "a transaction," stacked on top of the one below:
- **JDBC** — the lowest level: `Connection.setAutoCommit(false)`, `commit()`, `rollback()`.
- **JPA/Hibernate** — `EntityTransaction`/Hibernate's transaction API, wrapping the underlying JDBC transaction and coordinating it with the persistence context's flush behavior.
- **Spring's `@Transactional`** — the highest, most convenient level: a **declarative** transaction boundary — you annotate a method, and a Spring proxy handles beginning, committing, or rolling back the transaction around that method call, coordinating the JDBC/JPA layers underneath automatically.

```java
@Service
public class StudentService {
    @Transactional
    public void transferCourse(Long studentId, Long fromCourseId, Long toCourseId) {
        // if any exception is thrown here, the WHOLE method's DB changes roll back
    }
}
```

### Key takeaways
- ACID is the contract a transaction must honor to be trustworthy.
- `@Transactional` is Spring's declarative wrapper around the same JDBC/JPA transaction mechanics you'd otherwise manage by hand.
- The transaction boundary should generally sit at the **service layer** (Lecture 29) — the layer that represents one coherent business operation, not the controller (too broad/HTTP-specific) or the repository (too narrow — often just one query).

---

## Lecture 34 — Transactions: Propagation and Rollback Rules

### 1. Rollback-only marking
When an exception occurs inside a `@Transactional` method, Spring's interceptor may mark the **current transaction as rollback-only** even if the exception is subsequently caught somewhere. Consequence: **the transaction cannot commit anymore**, even if the surrounding code "handles" the exception and completes normally.

**Example failure scenario:**
1. Outer method starts Transaction A.
2. Inner call throws, and its transactional interceptor marks Transaction A as **rollback-only**.
3. The outer method catches the Java exception and completes "normally."
4. When the outer method tries to commit, Spring finds Transaction A is rollback-only — it **cannot commit**.
5. Spring rolls back Transaction A and throws **`UnexpectedRollbackException`** instead of silently succeeding.

This is a common source of confusion: *catching* the exception in Java doesn't undo the *transactional* rollback-only marking that already happened.

### 2. Transaction propagation — what happens when a `@Transactional` method calls another `@Transactional` method
Propagation settings (`@Transactional(propagation = ...)`) control this. The example flow the course walks through (`REQUIRES_NEW`):
1. Outer method is running inside Transaction A.
2. Inner method is annotated to require a **brand-new, independent** transaction.
3. Spring **suspends** Transaction A.
4. Spring starts a new Transaction B.
5. Inner method runs entirely inside Transaction B.
6. Transaction B commits or rolls back **independently** of Transaction A.
7. Spring releases Transaction B's resources.
8. Spring **resumes** Transaction A, which continues (and commits/rolls back) on its own terms.

This means a failure inside the inner (`REQUIRES_NEW`) transaction does **not** automatically roll back the outer transaction — the two are fully decoupled, which is exactly the point of using `REQUIRES_NEW` (e.g. writing an audit log entry that should persist *even if* the main business operation later fails and rolls back).

### Practical implications
- Understand **which propagation behavior you actually want** before reaching for `@Transactional` on nested calls — the default (`REQUIRES` — join the existing transaction if one exists, or start one if not) is right most of the time, but audit-logging / "must persist regardless" scenarios are a classic case for `REQUIRES_NEW`.
- Catching an exception in Java code does **not** undo transactional rollback-only marking — if you need the transaction to actually still commit, the rollback-only decision has to be avoided in the first place (e.g. by not letting the exception propagate out of the transactional boundary that set the flag).

---
*Next file: `07-security-oauth-testing.md` — Spring Security fundamentals → database-backed authentication → the internal filter chain → JWT authentication → OAuth 2.0/OIDC → Testing with JUnit, Mockito & MockMvc.*
