# **REST APIs, VALIDATION & EXCEPTION HANDLING**

8 questions on building production-grade REST APIs with Spring Boot.

---

### Q01: What's the difference between `@RestController` and `@Controller`? When would you use each?

**30-Second Answer:** `@RestController` = `@Controller` + `@ResponseBody` on every method. **Use `@RestController` for REST APIs** (returns JSON/XML serialized response body). **Use `@Controller` for server-side rendering** (returns view names mapped to templates like Thymeleaf).

**Deep Dive:**

```java
//  REST API — returns serialized JSON
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping("/{id}")
    public UserDto get(@PathVariable Long id) { return service.get(id); }   // serialized to JSON
}

//  MVC — returns view name
@Controller
@RequestMapping("/users")
public class UserViewController {
    @GetMapping("/{id}")
    public String show(@PathVariable Long id, Model model) {
        model.addAttribute("user", service.get(id));
        return "user-detail";   // resolves to user-detail.html
    }
}
```

**Common Follow-Ups:**
- "How is the response body serialized?" → `HttpMessageConverter` (Jackson for JSON by default)
- "What if I want both REST and views in one class?" → Use `@Controller` + add `@ResponseBody` per method

---

### Q02: How do you handle exceptions globally in Spring Boot? ⭐

**30-Second Answer:** Use `@RestControllerAdvice` (or `@ControllerAdvice`) with `@ExceptionHandler` methods. Industry pattern: return a consistent `ErrorResponse` DTO with `timestamp`, `status`, `code`, `message`, `path`, `traceId`. Map domain exceptions to HTTP status codes.

**Deep Dive:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest req) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(404)
                .code("RESOURCE_NOT_FOUND")
                .message(ex.getMessage())
                .path(req.getRequestURI())
                .traceId(MDC.get("traceId"))
                .build());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest().body(
            ErrorResponse.validation(errors));
    }

    @ExceptionHandler(Exception.class)   // catch-all (last)
    public ResponseEntity<ErrorResponse> handleAny(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception", ex);                       //  log full stack
        return ResponseEntity.internalServerError().body(
            ErrorResponse.generic("INTERNAL_ERROR", req));           //  don't leak details to client
    }
}

public record ErrorResponse(
    Instant timestamp, int status, String code, String message,
    String path, String traceId, List<FieldError> errors
) { ... }
```

**Common Follow-Ups:**
- "How do you preserve trace IDs in error responses?" → Pull from MDC (set by Sleuth/OpenTelemetry filter)
- "Should you return stack traces?" → NEVER to clients. Log them server-side with the trace ID

**Red Flag If You Say:** "Return the exception message directly." (Security: leaks internals; might contain SQL, file paths.)

---

### Q03: How does Spring's bean validation work? Explain `@Valid` vs `@Validated`. ⭐

**30-Second Answer:** Both trigger JSR-303 (Jakarta Bean Validation) on the bean. **`@Valid`** (jakarta) — standard, supports nested validation. **`@Validated`** (Spring) — adds **validation groups** support. Use `@Valid` for request bodies; `@Validated` on class for **method-level validation** of `@PathVariable`/`@RequestParam`.

**Deep Dive:**

```java
//  Request body validation
public record CreateUserRequest(
    @NotBlank @Size(min = 3, max = 50) String username,
    @Email String email,
    @Min(18) @Max(120) int age,
    @Valid @NotNull Address address   // cascade validation to nested object
) {}

@PostMapping("/users")
public UserDto create(@Valid @RequestBody CreateUserRequest req) { ... }
// Validation failure → MethodArgumentNotValidException

//  Method-level validation (path params, query params)
@Validated   //  REQUIRED on the class
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public UserDto get(@PathVariable @Min(1) Long id) { ... }
    // Validation failure → ConstraintViolationException (different exception!)
}

//  Validation groups
public interface Create {}
public interface Update {}

public record UserDto(
    @Null(groups = Create.class) @NotNull(groups = Update.class) Long id,
    @NotBlank String name
) {}

@PostMapping public void create(@Validated(Create.class) @RequestBody UserDto u) {}
@PutMapping  public void update(@Validated(Update.class) @RequestBody UserDto u) {}
```

**Common Follow-Ups:**
- "Why two different exceptions?" → `@Valid` on `@RequestBody` → `MethodArgumentNotValidException`; `@Validated` on method args → `ConstraintViolationException`. Handle both in `@ControllerAdvice`
- "How do you write custom validators?" → Implement `ConstraintValidator<MyAnnotation, MyType>`

---

### Q04: What's the right way to design REST API versioning?

**30-Second Answer:** Three strategies, in order of preference:
1. **URL path** — `/api/v1/users`, `/api/v2/users` (simplest, most explicit)
2. **Header** — `Accept: application/vnd.myapi.v2+json` (cleaner URLs but harder to debug)
3. **Query param** — `/api/users?v=2` (worst — caching issues, ugly)

**Deep Dive:**

```java
//  URL versioning (recommended)
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }

//  Header versioning
@GetMapping(value = "/users", produces = "application/vnd.myapi.v2+json")
public List<UserDtoV2> getV2() { ... }

//  Sunset header for deprecated versions
@GetMapping("/api/v1/users")
public ResponseEntity<List<UserDtoV1>> getV1() {
    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Wed, 11 Nov 2026 23:59:59 GMT")
        .header("Link", "</api/v2/users>; rel=\"successor-version\"")
        .body(...);
}
```

**Common Follow-Ups:**
- "When do you bump a version?" → Breaking changes: removing fields, changing types, changing semantics. Additive changes don't need a bump
- "How long do you keep old versions?" → 6-12 months with Sunset headers; track usage via metrics

---

### Q05: How do you implement pagination correctly?

**30-Second Answer:** For small/medium datasets, use `Pageable` (offset-based). For large datasets or high-write tables, use **cursor-based pagination** (`?after=<lastId>`). Always **cap page size**. Return total count only when needed (it's expensive).

**Deep Dive:**

```java
//  Offset-based (Pageable) — simple but slow for deep pages
@GetMapping("/orders")
public Page<OrderDto> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") @Max(100) int size,    //  cap!
    @RequestParam(defaultValue = "createdAt,desc") String sort
) {
    return repo.findAll(PageRequest.of(page, size, parseSort(sort)))
        .map(OrderDto::from);
}

//  Cursor-based — scales to billions of rows
@GetMapping("/orders")
public CursorPage<OrderDto> list(
    @RequestParam(required = false) String after,    // last seen ID
    @RequestParam(defaultValue = "20") @Max(100) int size
) {
    List<Order> orders = repo.findNextPage(after, size + 1);   // fetch 1 extra
    boolean hasNext = orders.size() > size;
    if (hasNext) orders = orders.subList(0, size);
    String nextCursor = hasNext ? orders.get(orders.size() - 1).getId() : null;
    return new CursorPage<>(orders.stream().map(OrderDto::from).toList(), nextCursor);
}
```

**Why offset-based gets slow:**
```sql
-- Page 1000, size 20 → DB scans 20,000 rows just to skip
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 20000;
```

```sql
-- Cursor-based — uses index directly, O(log n) every page
SELECT * FROM orders WHERE created_at < '2026-05-30T10:00:00Z' ORDER BY created_at DESC LIMIT 20;
```

**Common Follow-Ups:**
- "How do you return total count without a separate query?" → `Page<T>` does it automatically (extra `COUNT(*)`)
- "What about counting expensive queries?" → Use `Slice<T>` instead (no count); or estimate via `pg_class.reltuples`

---

### Q06: What HTTP status codes do you use, and when?

**30-Second Answer:** Use the right code for the semantic — don't just return 200 for everything. Critical ones:

| Status | Meaning | Example |
|---|---|---|
| **200 OK** | Success, body returned | GET / PUT |
| **201 Created** | Resource created | POST with `Location` header |
| **204 No Content** | Success, no body | DELETE / PUT (when no body returned) |
| **400 Bad Request** | Client sent invalid data | Validation failure |
| **401 Unauthorized** | Missing/invalid auth | Wrong API key |
| **403 Forbidden** | Authenticated but not allowed | RBAC failure |
| **404 Not Found** | Resource doesn't exist | `/users/999` |
| **409 Conflict** | State conflict | Idempotency key reused with different body |
| **422 Unprocessable Entity** | Semantically invalid | Business rule violation (not just schema) |
| **429 Too Many Requests** | Rate limited | + `Retry-After` header |
| **500 Internal Server Error** | Our bug | + log with trace ID |
| **502/503/504** | Downstream issue | Gateway, service unavailable, gateway timeout |

**Deep Dive:**

```java
//  POST returns 201 with Location header
@PostMapping
public ResponseEntity<UserDto> create(@Valid @RequestBody CreateUserRequest req) {
    UserDto created = service.create(req);
    return ResponseEntity
        .created(URI.create("/api/v1/users/" + created.id()))
        .body(created);
}

//  DELETE returns 204 (no body)
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    service.delete(id);
    return ResponseEntity.noContent().build();
}
```

**Common Follow-Ups:**
- "400 vs 422?" → 400 = malformed request (bad JSON, missing field); 422 = well-formed but semantically wrong (e.g., trying to confirm an already-confirmed order)
- "401 vs 403?" → 401 = "I don't know who you are" (no auth); 403 = "I know you, but you can't do this"

---

### Q07: How do you secure / sanitize file uploads?

**30-Second Answer:** Validate **MIME type**, **file extension**, **size** (use `MaxUploadSize`), and **content** (don't trust headers — check magic bytes). Stream large files (don't load into memory). Store outside the app server (S3, GCS), never in `webroot`. Scan with antivirus (ClamAV) before serving.

**Deep Dive:**

```java
@PostMapping(value = "/upload", consumes = MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<UploadResponse> upload(@RequestParam MultipartFile file) {
    //  Size check
    if (file.getSize() > 5 * 1024 * 1024) throw new PayloadTooLargeException();

    //  Whitelist content types (don't blacklist)
    Set<String> allowed = Set.of("image/jpeg", "image/png", "application/pdf");
    if (!allowed.contains(file.getContentType())) throw new UnsupportedMediaTypeException();

    //  Magic bytes check (Tika)
    Tika tika = new Tika();
    String detectedType = tika.detect(file.getInputStream());
    if (!allowed.contains(detectedType)) throw new UnsupportedMediaTypeException();

    //  Sanitize filename
    String safeName = StringUtils.cleanPath(file.getOriginalFilename());
    if (safeName.contains("..")) throw new SecurityException("Path traversal");

    //  Stream to S3 (not local disk)
    String key = UUID.randomUUID() + "/" + safeName;
    s3.putObject(PutObjectRequest.builder().bucket(bucket).key(key).build(),
                 RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

    return ResponseEntity.ok(new UploadResponse(key));
}
```

application.yml:
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 5MB
      max-request-size: 10MB
```

**Common Follow-Ups:**
- "Why check magic bytes?" → Clients can lie about Content-Type; an .exe renamed to .jpg still has PE header
- "How do you serve images securely?" → Through your API with auth/signed URLs, never direct from S3 public bucket

---

### Q08: How do you generate OpenAPI/Swagger docs in Spring Boot 3?

**30-Second Answer:** Use **springdoc-openapi** (NOT springfox — it's dead). Add `springdoc-openapi-starter-webmvc-ui` dependency and it auto-generates docs at `/swagger-ui.html` and JSON at `/v3/api-docs`. Document with `@Operation`, `@ApiResponse`, etc.

**Deep Dive:**

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

```java
@Tag(name = "Orders", description = "Order management API")
@RestController
public class OrderController {

    @Operation(summary = "Create order", description = "Creates a new order with idempotency support")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Order created"),
        @ApiResponse(responseCode = "400", description = "Validation error"),
        @ApiResponse(responseCode = "409", description = "Idempotency key conflict")
    })
    @PostMapping("/orders")
    public ResponseEntity<OrderDto> create(
        @Parameter(description = "Idempotency key (UUID)") @RequestHeader("Idempotency-Key") String key,
        @Valid @RequestBody CreateOrderRequest req
    ) { ... }
}
```

**Custom info:**
```yaml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger
    operations-sorter: method
```

```java
@Bean
public OpenAPI openApi() {
    return new OpenAPI()
        .info(new Info().title("Orders API").version("2.0").description(...))
        .components(new Components().addSecuritySchemes("bearer",
            new SecurityScheme().type(HTTP).scheme("bearer").bearerFormat("JWT")));
}
```

**Common Follow-Ups:**
- "Why not springfox?" → Last release 2020, doesn't work with Spring Boot 2.6+, abandoned
- "How do you exclude an endpoint from docs?" → `@Hidden` annotation

---

## Top 3 from This File (must-know)

1. **Q02** — Global exception handling with `@RestControllerAdvice`
2. **Q03** — `@Valid` vs `@Validated` (often confused)
3. **Q05** — Pagination (offset vs cursor) — every backend interview asks

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| @RestController                   | @Controller + @ResponseBody      |
| Global exception handling         | @RestControllerAdvice + DTO      |
| @Valid                            | Body / nested objects             |
| @Validated                        | Method params (class-level too)  |
| Versioning                        | URL paths > headers > query      |
| Pagination                        | Cursor-based for large datasets  |
| File upload security              | Magic bytes + whitelist + size   |
| OpenAPI                           | springdoc (NOT springfox)        |
| 200/201/204/400/401/403/404/422/429/500 | Use semantic codes          |
+----------------------------------+----------------------------------+
```
