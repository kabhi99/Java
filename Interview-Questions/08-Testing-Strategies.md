# **TESTING STRATEGIES IN SPRING BOOT**

6 questions on how to test Spring Boot apps efficiently.

---

### Q01: `@SpringBootTest` vs slice tests (`@WebMvcTest`, `@DataJpaTest`)? ⭐

**30-Second Answer:** `@SpringBootTest` loads the **entire application context** — slow (~5-30s) but realistic. **Slice tests** load only what's needed for one layer — fast (~1-3s) and focused. Use slices for unit-of-feature tests; full `@SpringBootTest` only for integration smoke tests.

**Deep Dive:**

```java
//  @SpringBootTest — loads EVERYTHING
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderE2ETest {
    @Autowired TestRestTemplate rest;

    @Test void createOrder_returns201() {
        ResponseEntity<Order> r = rest.postForEntity("/orders", req, Order.class);
        assertEquals(201, r.getStatusCodeValue());
    }
}

//  @WebMvcTest — only MVC layer (no @Service, no DB)
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;   //  mocked dependency

    @Test void createOrder_returnsLocation() throws Exception {
        when(orderService.create(any())).thenReturn(Order.of(1L));
        mockMvc.perform(post("/orders").content(json).contentType(JSON))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", "/orders/1"));
    }
}

//  @DataJpaTest — only JPA (in-memory H2 by default)
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repo;

    @Test void findByStatus_returnsActive() {
        repo.save(new Order(...));
        assertThat(repo.findByStatus(ACTIVE)).hasSize(1);
    }
}
```

**All Spring Boot slices:**
| Slice | Loads |
|---|---|
| `@WebMvcTest` | Controllers, filters, advice |
| `@DataJpaTest` | Repositories, EntityManager |
| `@JsonTest` | Jackson, ObjectMapper |
| `@RestClientTest` | RestTemplate / WebClient |
| `@WebFluxTest` | WebFlux controllers |
| `@DataMongoTest` | Mongo repositories |
| `@DataRedisTest` | Redis repositories |

**Common Follow-Ups:**
- "How do you test only a service?" → Plain JUnit + Mockito; no Spring context needed
- "Why is `@SpringBootTest` slow?" → Loads all beans, all auto-configs, embedded server

---

### Q02: When should you use `@MockBean` vs `@Mock`?

**30-Second Answer:**
- **`@Mock`** (Mockito) — plain mock, no Spring context, used in pure unit tests
- **`@MockBean`** (Spring Boot) — replaces a bean in the Spring context with a mock; use in slice/integration tests

**Deep Dive:**

```java
//  @Mock — pure unit test, no Spring
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @Mock PaymentClient paymentClient;
    @InjectMocks OrderService service;

    @Test void create_savesOrder() {
        when(repo.save(any())).thenReturn(savedOrder);
        Order result = service.create(req);
        verify(paymentClient).charge(any());
    }
}

//  @MockBean — Spring context aware
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;   //  replaces real bean in context

    @Test void ...
}

//  @SpyBean — wrap real bean (test some methods, mock others)
@SpringBootTest
class OrderServiceIntegrationTest {
    @SpyBean OrderRepository repo;   //  real, but you can verify/stub

    @Test void ...
}
```

**Performance trap:**
`@MockBean` **invalidates Spring's context cache** between tests — your test suite slows dramatically if many test classes use different `@MockBean` configurations.

**Fix:** Group tests with same `@MockBean` configs, use abstract base test classes.

**Common Follow-Ups:**
- "How does Spring's test context caching work?" → Spring caches contexts keyed by configuration. Tests with identical configs share a context. `@MockBean` differs → new context
- "When use `@SpyBean`?" → Rarely — usually a smell. Prefer mocking dependencies and unit-testing the real class

---

### Q03: How do you test code that uses external services (HTTP, DB)?

**30-Second Answer:** Use **Testcontainers** for real DB/Kafka/Redis in a Docker container per test (or per class). Use **WireMock** for stubbing external HTTP APIs. Avoid mocking JDBC/HTTP — too far from reality.

**Deep Dive:**

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.github.tomakehurst</groupId>
    <artifactId>wiremock-jre8-standalone</artifactId>
    <scope>test</scope>
</dependency>
```

```java
//  Testcontainers + Postgres
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withReuse(true);   //  Speed up: reuse across test runs

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    @Test void canPersistAndQuery() { ... }
}

//  WireMock — stub external HTTP
@SpringBootTest
class PaymentClientIT {
    static WireMockServer wm = new WireMockServer(8089);

    @BeforeAll static void start() { wm.start(); }
    @AfterAll static void stop() { wm.stop(); }

    @Test void charge_returns200() {
        wm.stubFor(post("/charges")
            .willReturn(aResponse().withStatus(200).withBody("{\"id\":\"ch_1\"}")));

        PaymentResponse r = paymentClient.charge(req);
        assertEquals("ch_1", r.getId());

        wm.verify(postRequestedFor(urlEqualTo("/charges"))
            .withHeader("X-API-Key", equalTo("test")));
    }
}
```

**Common Follow-Ups:**
- "Why Testcontainers over H2?" → H2 has SQL dialect differences (functions, types) — tests pass locally but fail in prod with real Postgres
- "Why WireMock vs Mockito for HTTP?" → WireMock tests the full HTTP layer (URL, headers, body); Mockito skips it

---

### Q04: How do you test `@Transactional` rollback behavior?

**30-Second Answer:** By default, `@Transactional` on a test method **rolls back the transaction at the end** — DB state never persists. To verify rollback behavior of your code (not the test framework), use `TestTransaction` or `@Commit`.

**Deep Dive:**

```java
@SpringBootTest
@Transactional   //  Rolled back automatically after each test
class OrderServiceTest {

    @Test
    void createOrder_persists() {
        orderService.createOrder(req);
        assertThat(repo.count()).isEqualTo(1);
    }   //  rollback — DB is clean for next test

    @Test
    @Commit   //  Forces commit — useful when you want side effects
    void createOrder_writesToOutbox() {
        orderService.createOrder(req);
        // ... external system can now see the data
    }

    @Test
    void rollbackOnError_doesNotPersistOrder() {
        try {
            orderService.createOrderWithError(req);   // throws
        } catch (Exception ignored) {}
        // Even though we caught the error, the @Transactional method rolled back
        assertThat(repo.count()).isZero();
    }
}
```

**`TestTransaction` for manual control:**

```java
@Test
@Transactional
void multipleTxnsInOneTest() {
    repo.save(order1);
    TestTransaction.flagForCommit();
    TestTransaction.end();

    TestTransaction.start();
    repo.save(order2);
    TestTransaction.flagForRollback();
    TestTransaction.end();

    // Now order1 is committed, order2 is rolled back
}
```

**Common Follow-Ups:**
- "Why does `@Transactional` on tests default to rollback?" → Test isolation — each test starts with clean DB
- "What if you want to test propagation behaviors?" → Use real services with Testcontainers (not in-memory DB)

---

### Q05: How do you write fast and reliable integration tests?

**30-Second Answer:** Six rules:
1. **Reuse containers** (`.withReuse(true)`) — save 5-10s per test class
2. **Share Spring context** across tests (avoid varying `@MockBean`)
3. **Parallel tests** with proper isolation
4. **`@DirtiesContext` only when needed** (kills caching)
5. **Use `@DynamicPropertySource`** instead of static config
6. **Use `@Sql` for test data**, not custom setup per test

**Deep Dive:**

```java
//  Base class — shared container, shared context
@SpringBootTest
@Testcontainers
@AutoConfigureMockMvc
public abstract class IntegrationTest {

    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withReuse(true);

    static {
        postgres.start();   //  start ONCE, share across all subclass tests
    }

    @DynamicPropertySource
    static void config(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }
}

class OrderServiceIT extends IntegrationTest { ... }
class PaymentServiceIT extends IntegrationTest { ... }
//  Same context, same container — both very fast

//  @Sql for test data
@Test
@Sql("/test-data/orders.sql")
void findActiveOrders_returns3() { ... }

//  Enable reuse in ~/.testcontainers.properties
// testcontainers.reuse.enable=true
```

```yaml
# junit-platform.properties — parallel execution
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
```

**Common Follow-Ups:**
- "What about Flyway migrations?" → Run them on the container; same migrations as production = realistic schema

---

### Q06: How do you test reactive code (`Mono`/`Flux`)?

**30-Second Answer:** Use **`StepVerifier`** from `reactor-test`. It subscribes to a publisher and asserts the sequence of events (next, complete, error, time-based delays).

**Deep Dive:**

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@Test
void mono_returnsUser() {
    Mono<User> userMono = userService.findById(1L);

    StepVerifier.create(userMono)
        .expectNextMatches(u -> u.getId().equals(1L))
        .verifyComplete();
}

@Test
void flux_emitsExpectedSequence() {
    Flux<Integer> nums = Flux.just(1, 2, 3);

    StepVerifier.create(nums)
        .expectNext(1)
        .expectNext(2, 3)
        .verifyComplete();
}

@Test
void error_propagated() {
    Mono<User> faulty = Mono.error(new RuntimeException("oops"));

    StepVerifier.create(faulty)
        .expectErrorMatches(ex -> "oops".equals(ex.getMessage()))
        .verify();
}

//  Virtual time for testing delays
@Test
void delayedFlux_eventuallyEmits() {
    StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofMinutes(1)).take(3))
        .expectSubscription()
        .thenAwait(Duration.ofMinutes(3))   //  fast-forward!
        .expectNext(0L, 1L, 2L)
        .verifyComplete();
}

//  WebTestClient for WebFlux controllers
@WebFluxTest(UserController.class)
class UserControllerTest {
    @Autowired WebTestClient web;
    @MockBean UserService service;

    @Test void getUser_returns200() {
        when(service.findById(1L)).thenReturn(Mono.just(user));
        web.get().uri("/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class).isEqualTo(user);
    }
}
```

**Common Follow-Ups:**
- "Why virtual time?" → Real delays of 5s × 100 tests = 8 min wait; virtual time makes them instant
- "What's `WebTestClient`?" → Reactive counterpart of `MockMvc`; works with both WebFlux and MVC

---

## Top 3 from This File (must-know)

1. **Q01** — `@SpringBootTest` vs slice tests
2. **Q02** — `@MockBean` vs `@Mock` (and context caching)
3. **Q03** — Testcontainers + WireMock

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| @SpringBootTest                   | Full context, slow, integration  |
| @WebMvcTest                       | MVC layer only                   |
| @DataJpaTest                      | JPA layer (H2 by default)        |
| @Mock                             | Pure unit test (no Spring)       |
| @MockBean                         | Replace bean in Spring context   |
| @MockBean cache trap              | Invalidates Spring context cache |
| @SpyBean                          | Wrap real bean (rarely needed)   |
| Testcontainers                    | Real DB/Kafka/Redis in Docker    |
| WireMock                          | Stub external HTTP services      |
| @Transactional on test            | Auto-rollback after test         |
| @Commit                           | Force commit on test             |
| @DynamicPropertySource            | Inject container URLs            |
| @Sql                              | Load test data                   |
| StepVerifier                      | Test reactive Mono/Flux          |
| WebTestClient                     | Reactive MockMvc                 |
+----------------------------------+----------------------------------+
```
