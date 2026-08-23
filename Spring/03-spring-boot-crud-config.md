# Spring Framework Full Course — Notes 3: Configuration & Your First CRUD API
### Covers Lectures 10–13: `application.properties`, `@Value`, `@ConfigurationProperties`, Runners → Spring Boot CRUD REST APIs → Soft Delete

---

## Lecture 10 — Externalized Configuration & Runner Interfaces

### 1. Why not hardcode values?
```java
@Component
public class PaymentService {
    private String providerName = "Razorpay"; // hardcoded
    private int retryCount = 3;
}
```
If a hardcoded value needs to change (provider, retry count, timeout, feature flag, DB URL, API key, port…), you'd have to edit code, recompile, rebuild, and redeploy — bad for anything that varies across environments.

### 2. `application.properties` / `application.yml`
Spring Boot automatically loads `src/main/resources/application.properties` (or `.yml`) at startup:
```properties
payment.provider=Razorpay
payment.retry-count=3
payment.enabled=true
payment.timeout=5000
```
Equivalent YAML:
```yaml
payment:
  provider: Razorpay
  retry-count: 3
  enabled: true
  timeout: 5000
```
**Nuance:** this file lives under `src/main/resources`, so it's actually packaged *inside* the final JAR — changing only this file still technically requires a rebuild. True flexibility comes from **externalized configuration**: Spring Boot can also read values from environment variables, command-line arguments, system properties, and external config files placed outside the JAR — letting you override behavior *without* recompiling.

### 3. `@Value` — inject a single property
```java
@Component
public class PaymentService {
    private final String providerName;
    private final int retryCount;

    public PaymentService(
            @Value("${payment.provider}") String providerName,
            @Value("${payment.retry-count}") int retryCount) {
        this.providerName = providerName;
        this.retryCount = retryCount;
    }
}
```
**Default value syntax** (avoids startup failure if the property is missing):
```java
@Value("${payment.provider:DefaultProvider}")
private String providerName;
```
Good for one or two simple values — becomes messy for many related properties.

### 4. `@ConfigurationProperties` — bind a whole group
```java
@Component
@ConfigurationProperties(prefix = "payment")
public class PaymentProperties {
    private String provider;
    private int retryCount;
    private boolean enabled;
    private int timeout;
    // + standard getters/setters
}
```
Spring Boot binds every property starting with `payment.` onto this object:
```
payment.provider     → provider
payment.retry-count  → retryCount   (relaxed binding: kebab-case ↔ camelCase)
payment.enabled      → enabled
payment.timeout      → timeout
```
**Relaxed binding** is why `retry-count` (properties-file style) maps to `retryCount` (Java camelCase) — Spring Boot understands multiple equivalent naming styles (`retry-count`, `retryCount`, `retry_count`, `RETRY_COUNT`).

Then inject the whole properties object as one bean:
```java
@Component
public class PaymentService {
    private final PaymentProperties paymentProperties;
    public PaymentService(PaymentProperties paymentProperties) { this.paymentProperties = paymentProperties; }
}
```

**Rule of thumb:** `@Value` for one or two simple values; `@ConfigurationProperties` for a related group — cleaner and more maintainable in larger apps.

### 5. Runner interfaces — running code at startup without a web request
If there's no controller/endpoint to trigger your code (e.g. a batch job, a CLI demo), Spring Boot gives you two interfaces to run logic right after the application context is ready:

**`CommandLineRunner`** — receives raw String arguments:
```java
@Component
public class AppRunner implements CommandLineRunner {
    private final PaymentService paymentService;
    public AppRunner(PaymentService paymentService) { this.paymentService = paymentService; }

    @Override
    public void run(String... args) {
        paymentService.pay();
    }
}
```
Run with `java -jar app.jar hello world` → `args` = `["hello", "world"]`.

**`ApplicationRunner`** — receives structured `ApplicationArguments` (understands `--option=value` style arguments):
```java
@Component
public class AppStartupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        System.out.println(args.getOptionValues("provider"));
    }
}
```
Run with `java -jar app.jar --provider=Razorpay --retry=3` → `args.getOptionValues("provider")` reads it as a named option.

Why not just call the bean manually from `main()`? Because that reverts to the manual-container style this whole course is moving away from — letting Spring detect and invoke a `Runner` bean automatically is the idiomatic Boot approach.

### Complete Boot startup flow (config-aware version)
```
main() → SpringApplication.run()
  → Spring Boot prepares the Environment (properties, YAML, env vars, CLI args)
  → ApplicationContext created
  → @SpringBootApplication processed (@ComponentScan + @EnableAutoConfiguration)
  → Beans discovered & created; dependencies injected
  → @Value / @ConfigurationProperties values bound
  → CommandLineRunner / ApplicationRunner beans execute
  → App keeps running (web app) or exits (batch-style app)
```

---

## Lectures 11–13 — Building a Spring Boot CRUD REST API

These three lectures build up one running example — a `Student` CRUD REST API — layer by layer, culminating in **soft delete**.

### Layered architecture
```
Controller  (handles HTTP, @RequestMapping / @GetMapping / etc.)
    ↓
Service     (business logic)
    ↓
Repository  (data access — Spring Data JPA)
    ↓
Entity      (maps to a DB table via JPA annotations)
```

### The `Student` entity
```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private int age;
    private String email;
    private int rollNo;
    private String subject;
    private Boolean deleted;   // added later, for soft delete
    // getters/setters
}
```
- `@Entity` — marks this class as a JPA-managed database table.
- `@Id` — marks the primary key field.
- `@GeneratedValue(strategy = GenerationType.IDENTITY)` — the database auto-generates the ID (e.g. MySQL `AUTO_INCREMENT`).

### The repository — Spring Data JPA
```java
public interface StudentRepository extends JpaRepository<Student, Long> {
    Optional<Student> findByIdAndDeletedIsFalse(Long id);
    List<Student> findByDeletedIsFalse();
}
```
Extending `JpaRepository<Student, Long>` (entity type + primary key type) gives you `save()`, `findById()`, `findAll()`, `deleteById()`, `existsById()`, etc. **for free** — no implementation needed. Spring Data JPA generates a proxy implementation at runtime. Custom finder methods like `findByDeletedIsFalse()` are **derived query methods**: Spring parses the method name (`findBy` + `Deleted` + `IsFalse`) and builds the query automatically (covered in depth in the Spring Data JPA lecture, #32).

### The service layer
```java
@Service
public class StudentService {
    private StudentRepository studentRepository;
    public StudentService(StudentRepository studentRepository) { this.studentRepository = studentRepository; }

    public Student createStudent(Student studentReq) {
        studentReq.setDeleted(false);
        return studentRepository.save(studentReq);
    }

    public Student getStudent(Long id) {
        return studentRepository.findById(id).orElse(null);
    }

    public List<Student> getAllStudent() {
        return studentRepository.findByDeletedIsFalse();
    }

    public Student updateStudent(Long id, Student studentReq) {
        Optional<Student> existingStudent = studentRepository.findByIdAndDeletedIsFalse(id);
        if (existingStudent.isEmpty()) return null;

        Student studentToSave = existingStudent.get();
        studentToSave.setName(studentReq.getName());
        studentToSave.setRollNo(studentReq.getRollNo());
        studentToSave.setSubject(studentReq.getSubject());
        studentToSave.setEmail(studentReq.getEmail());
        studentToSave.setAge(studentReq.getAge());
        studentToSave.setDeleted(false);
        return studentRepository.save(studentToSave);
    }

    public Boolean deleteStudent(Long id) {          // hard delete
        if (!studentRepository.existsById(id)) return false;
        studentRepository.deleteById(id);
        return true;
    }

    public Boolean deleteStudentSoftly(Long id) {     // soft delete
        Optional<Student> existingStudent = studentRepository.findByIdAndDeletedIsFalse(id);
        if (existingStudent.isEmpty()) return false;

        Student studentToSave = existingStudent.get();
        studentToSave.setDeleted(true);
        studentRepository.save(studentToSave);
        return true;
    }
}
```

### The controller — REST endpoints
```java
@RestController
@RequestMapping("/api/students")
public class StudentController {
    private StudentService studentService;
    public StudentController(StudentService studentService) { this.studentService = studentService; }

    @PostMapping("/create")
    public ResponseEntity<Student> createStudent(@RequestBody Student student) {
        Student created = studentService.createStudent(student);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @GetMapping("/get")
    public ResponseEntity<Student> getStudent(@RequestParam Long id) {
        Student student = studentService.getStudent(id);
        if (student == null) return ResponseEntity.notFound().build();
        return ResponseEntity.ok(student);
    }

    @GetMapping("/getAll")
    public ResponseEntity<List<Student>> getAllStudent() {
        List<Student> list = studentService.getAllStudent();
        if (list.isEmpty()) return ResponseEntity.notFound().build();
        return ResponseEntity.ok(list);
    }

    @PutMapping("/update")
    public ResponseEntity<Student> updateStudent(@RequestParam Long id, @RequestBody Student req) {
        Student updated = studentService.updateStudent(id, req);
        if (updated == null) return ResponseEntity.notFound().build();
        return ResponseEntity.ok(updated);
    }

    @DeleteMapping("/delete")
    public ResponseEntity<String> deleteStudent(@RequestParam Long id) {
        if (!studentService.deleteStudent(id)) return ResponseEntity.notFound().build();
        return ResponseEntity.ok("Record deleted");
    }

    @PatchMapping("/delete-soft")
    public ResponseEntity<String> deleteStudentSoftly(@RequestParam Long id) {
        if (!studentService.deleteStudentSoftly(id)) return ResponseEntity.notFound().build();
        return ResponseEntity.ok("Record deleted");
    }
}
```

### Key mapping annotations used here
| Annotation | HTTP verb | Typical use |
|---|---|---|
| `@PostMapping` | POST | Create a resource |
| `@GetMapping` | GET | Read one / read all |
| `@PutMapping` | PUT | Full update |
| `@PatchMapping` | PATCH | Partial update (here: soft-delete flag) |
| `@DeleteMapping` | DELETE | Hard delete |
| `@RequestBody` | — | Deserialize the JSON request body into a Java object |
| `@RequestParam` | — | Bind a query parameter (`?id=5`) to a method parameter |
| `@PathVariable` | — | Bind a URI path segment (e.g. `/{id}`) — used more in Lecture 15's MVC deep dive |

### Why soft delete?
A **hard delete** (`deleteById`) permanently removes the row. A **soft delete** instead flips a `deleted` boolean flag to `true` and *keeps the row* — the record is excluded from normal reads (`findByDeletedIsFalse`) but still exists for audit trails, recovery, or referential integrity with related data. This lecture's design puts the `deleted` filter directly into the derived-query method names (`findByDeletedIsFalse`, `findByIdAndDeletedIsFalse`) so every "normal" read automatically excludes soft-deleted rows.

### Progression across Lectures 11 → 13
- **Lecture 11:** builds the basic CRUD skeleton — Create, Get All, Get by ID, Update, Delete, first drawing the distinction between the raw approach and letting Spring Data JPA do the heavy lifting.
- **Lecture 12:** wires up the real MySQL connection — adding the MySQL driver dependency, configuring the JDBC URL, entity type/primary-key type generics on `JpaRepository`.
- **Lecture 13:** introduces the `deleted` flag and refactors every read/update/delete path to respect it, adding the `/delete-soft` endpoint alongside the original hard `/delete`.

### Practical takeaways
- Spring Data JPA repositories eliminate almost all boilerplate SQL/JDBC code for standard CRUD.
- `ResponseEntity` gives you full control of status code + body + headers (used consistently here instead of returning raw objects/strings).
- Keep validation of "does this even exist" (`existsById`, `findById...IsFalse`) in the **service layer**, not the controller — the controller here just decides what HTTP response to send based on what the service returns.
- Soft delete is a data-modeling decision that ripples through every query in the repository and service layers — not just the delete endpoint.

---
*Next file: `04-web-layer-mvc-dto-exceptions-config.md` — Servlets & Tomcat internals, Spring MVC architecture, DTOs & validation, global exception handling, and YAML/Profiles.*
