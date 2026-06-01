# **MICROSERVICES & SPRING CLOUD**

8 questions on distributed systems patterns with Spring Boot.

---

### Q01: Service-to-service communication: WebClient vs RestTemplate vs Feign? ⭐

**30-Second Answer:**
- **`RestTemplate`** — synchronous, in maintenance mode (no new features). Use only for legacy
- **`WebClient`** — modern, supports both sync and async, replaces `RestTemplate`
- **`OpenFeign`** — declarative HTTP client (interface-based). Best DX for microservice-to-microservice

**Deep Dive:**

```java
//  WebClient (modern, both sync and reactive)
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
        .responseTimeout(Duration.ofSeconds(5));
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .baseUrl("https://api.payments.com")
        .build();
}

// Sync usage
User user = webClient.get().uri("/users/{id}", id).retrieve().bodyToMono(User.class).block();

// Async usage
Mono<User> userMono = webClient.get().uri("/users/{id}", id).retrieve().bodyToMono(User.class);

//  OpenFeign (declarative, cleanest for microservices)
@FeignClient(name = "payment-service", url = "${payment.service.url}")
public interface PaymentClient {
    @PostMapping("/charges")
    PaymentResponse charge(@RequestBody PaymentRequest req);

    @GetMapping("/charges/{id}")
    PaymentResponse get(@PathVariable String id);
}

// Just inject and call — Feign generates the implementation
@Service
@RequiredArgsConstructor
class OrderService {
    private final PaymentClient paymentClient;
    public void process(Order o) { paymentClient.charge(new PaymentRequest(...)); }
}
```

**Common Follow-Ups:**
- "Why is `RestTemplate` deprecated?" → Not officially deprecated, but in maintenance mode. Spring team recommends `WebClient`
- "What's `RestClient` (Spring 6.1+)?" → Synchronous HTTP client with fluent API similar to WebClient — designed to replace RestTemplate for blocking use cases

---

### Q02: How do you implement a Circuit Breaker pattern in Spring Boot? ⭐

**30-Second Answer:** Use **Resilience4j** (Hystrix is dead). Add `@CircuitBreaker` with a fallback method. When downstream failure rate exceeds threshold (e.g., 50%), the breaker opens and fails fast; after a cooldown, it tests recovery.

**Deep Dive:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        failureRateThreshold: 50
        slowCallRateThreshold: 80
        slowCallDurationThreshold: 2s
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
```

```java
@Service
public class PaymentService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
    public PaymentResponse charge(PaymentRequest req) {
        return paymentClient.charge(req);
    }

    public PaymentResponse fallback(PaymentRequest req, CallNotPermittedException ex) {
        log.warn("Circuit open for payment service");
        return PaymentResponse.unavailable();
    }
}
```

**State machine:**
```
CLOSED ─(failure rate exceeds threshold)→ OPEN
   ↑                                        │
   │ (success in HALF_OPEN)                 │ (after waitDuration)
   │                                        ↓
   └──────────── HALF_OPEN ←────────────────┘
                (test N calls; if any fail → OPEN)
```

> **See `Java/02-Spring-Boot-Resilience-Patterns.md` for the full deep dive.**

**Common Follow-Ups:**
- "What metrics do you alert on?" → CB state transitions (CLOSED→OPEN), failure rate, slow call rate
- "Why combine CB with timeout?" → Timeout catches the slow case; CB catches the high failure rate case

---

### Q03: Saga pattern — orchestration vs choreography? ⭐

**30-Second Answer:** Saga = sequence of local transactions across services with compensating actions on failure.
- **Orchestration** — central coordinator calls each service step-by-step (easier to reason, single point of complexity)
- **Choreography** — services react to events (no central coordinator, looser coupling, harder to trace)

**Deep Dive:**

**Orchestration (state machine in Order Service):**

```java
@Service
public class OrderSaga {
    public void process(OrderId id) {
        try {
            paymentService.charge(id);
            try {
                inventoryService.reserve(id);
                try {
                    shippingService.ship(id);
                } catch (Exception e) {
                    inventoryService.releaseReservation(id);   // compensate
                    paymentService.refund(id);
                    throw e;
                }
            } catch (Exception e) {
                paymentService.refund(id);                      // compensate
                throw e;
            }
        } catch (Exception e) {
            orderService.markFailed(id);
        }
    }
}
```

**Choreography (event-driven):**

```
Order Service: emits OrderCreated → 
Payment Service: listens, charges → emits PaymentSucceeded / PaymentFailed
Inventory Service: listens to PaymentSucceeded, reserves → emits StockReserved
Shipping Service: listens to StockReserved, ships → emits OrderShipped
```

If any step fails, services emit failure events; previous services listen and compensate.

**Pros/Cons:**

| Aspect | Orchestration | Choreography |
|---|---|---|
| Visibility | Easy — all in one place | Hard — distributed events |
| Coupling | Coordinator coupled to all | Loose |
| Adding step | Modify orchestrator | Add new listener |
| Debugging | Easier | Need distributed tracing |
| Best for | Linear workflows | Branching workflows, evolving services |

**Common Follow-Ups:**
- "Tools for orchestration?" → Camunda, Temporal, Conductor, AWS Step Functions, Spring State Machine
- "Why must steps be idempotent?" → Retries during failure = same operation might run twice

---

### Q04: What is the Transactional Outbox pattern? ⭐⭐

**30-Second Answer:** When you save to DB AND publish to Kafka, the two can desync (DB commits but Kafka fails, or vice versa). **Outbox pattern**: write the event to an `outbox` table in the **same DB transaction** as the business data. A separate **CDC** (Change Data Capture) tool like Debezium reads the outbox and publishes to Kafka. Guaranteed at-least-once delivery.

**Deep Dive:**

```java
//  Naive dual-write — DESYNC RISK
@Transactional
public void createOrder(Order o) {
    orderRepo.save(o);
    kafkaTemplate.send("orders", o);   //  What if Kafka send fails AFTER commit?
}                                      //  Or DB rollback AFTER Kafka send?

//  Transactional Outbox — atomic
@Entity @Table(name = "outbox")
public class OutboxEvent {
    @Id Long id;
    String aggregateType;   // "Order"
    String aggregateId;     // "order-123"
    String eventType;       // "OrderCreated"
    String payload;         // JSON
    Instant createdAt;
    Boolean published = false;
}

@Service
public class OrderService {
    @Transactional
    public void createOrder(Order o) {
        orderRepo.save(o);
        outboxRepo.save(new OutboxEvent(
            "Order", o.getId().toString(), "OrderCreated", toJson(o)
        ));
        //  Both saves in same txn — atomic commit
    }
}
```

**Two ways to publish from outbox:**

1. **Polling publisher** (simple):
```java
@Scheduled(fixedDelay = 1000)
public void publishOutbox() {
    List<OutboxEvent> events = outboxRepo.findUnpublishedTop100();
    for (OutboxEvent e : events) {
        kafkaTemplate.send(topic(e), e.getPayload())
            .whenComplete((res, ex) -> {
                if (ex == null) e.setPublished(true);
            });
    }
    outboxRepo.saveAll(events);
}
```

2. **Debezium CDC** (production):
- Debezium reads Postgres WAL → publishes to Kafka automatically
- No polling, no app code change, exactly-once delivery
- Cleanup outbox rows after publish

**Common Follow-Ups:**
- "Why is dual-write bad in distributed systems?" → No 2-phase commit between DB and Kafka — one will eventually fail
- "What's the inverse — Inbox pattern?" → Consumer-side: dedupe events by ID stored in an `inbox` table within same txn as business logic

---

### Q05: How do you handle distributed tracing in Spring Boot?

**30-Second Answer:** Use **Micrometer Tracing** (replaces Sleuth) with an exporter to **Zipkin/Jaeger/Tempo**. Each request gets a `traceId`; each service hop gets a `spanId`. IDs propagate automatically via HTTP headers (`traceparent`).

**Deep Dive:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0    # 100% in dev; 0.01-0.1 in prod
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

```java
// MDC automatically populated with traceId, spanId
@RestController
public class OrderController {
    public ResponseEntity<Order> create(@RequestBody CreateOrderRequest req) {
        log.info("Creating order");   //  Logs include [traceId, spanId]
        Order o = orderService.create(req);
        return ResponseEntity.ok(o);
    }
}
```

Logback pattern:
```xml
<pattern>%d %-5level [%X{traceId:-},%X{spanId:-}] %logger - %msg%n</pattern>
```

Logs:
```
2026-06-01 12:00:00 INFO [a1b2c3,d4e5f6] OrderController - Creating order
```

**Common Follow-Ups:**
- "What's `traceparent` header?" → W3C standard for trace context propagation
- "How do you debug a slow flow?" → Look up traceId in Jaeger UI → see waterfall of spans across all services

---

### Q06: How do you handle configuration in a microservices setup?

**30-Second Answer:** Three options:
1. **Spring Cloud Config** — centralized config server backed by Git
2. **Kubernetes ConfigMaps + Secrets** — native cloud-native
3. **HashiCorp Vault** — for secrets, integrates with Spring Cloud Vault

**Deep Dive:**

```yaml
# bootstrap.yml — fetched BEFORE application.yml
spring:
  application:
    name: order-service
  cloud:
    config:
      uri: http://config-server:8888
      profile: ${ENVIRONMENT:dev}
      fail-fast: true
```

Config server (Spring Cloud Config):
```yaml
# Backed by Git repo with /order-service-dev.yml, /order-service-prod.yml, etc.
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          search-paths: '{application}'
```

**Refreshable config (no restart):**
```java
@RestController
@RefreshScope
public class FeatureController {
    @Value("${feature.new-checkout.enabled:false}")
    private boolean enabled;
}

// POST /actuator/refresh to re-fetch config
```

**Kubernetes alternative:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  application.yml: |
    server:
      port: 8080
    payment:
      url: http://payment-service:8080
```

Mount as volume in pod spec — Spring Boot picks it up.

**Common Follow-Ups:**
- "Spring Cloud Config vs Kubernetes ConfigMap — which?" → ConfigMap if on K8s (one less moving part); Config Server if multi-cluster, multi-cloud, or version control needed
- "How do you handle secrets?" → Vault, AWS Secrets Manager, GCP Secret Manager, K8s Secrets (less secure)

---

### Q07: API Gateway — what does it do and which one in Spring?

**30-Second Answer:** API Gateway is the **single entry point** to microservices. Handles: routing, authentication, rate limiting, request/response transformation, observability. Spring Cloud Gateway (reactive, modern) or Spring Cloud Netflix Zuul (legacy, deprecated). Production alternatives: Kong, AWS API Gateway, Envoy, Istio.

**Deep Dive:**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service           # load-balanced via service discovery
          predicates:
            - Path=/api/orders/**
            - Header=X-API-Key, .+
          filters:
            - StripPrefix=2                  # /api/orders/123 → /123
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
            - name: CircuitBreaker
              args:
                name: orderCircuit
                fallbackUri: forward:/fallback
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - StripPrefix=2
```

**What it gives you:**
- Single TLS termination point
- Centralized auth (JWT validation once, not in every service)
- Cross-cutting filters: logging, tracing, CORS
- Rate limiting at the edge (cheap)

**Common Follow-Ups:**
- "Why prefer API Gateway over service-level rate limiting?" → Edge is cheapest place to drop requests
- "When NOT to use Spring Cloud Gateway?" → Service mesh handles routing (Istio) + you want config in YAML, not code

---

### Q08: How does service discovery work? Eureka vs Consul vs Kubernetes DNS?

**30-Second Answer:** Services register themselves; clients query for current addresses (handles dynamic IPs in cloud/K8s).
- **Eureka** (Netflix) — Spring Cloud native, simple, in-process registry
- **Consul** — more features (KV store, health checks), but more ops overhead
- **Kubernetes DNS** — built-in via Service objects (`http://order-service.namespace.svc.cluster.local`). **If you're on K8s, just use this**.

**Deep Dive:**

**Eureka client setup:**
```yaml
spring:
  application:
    name: order-service
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka
  instance:
    prefer-ip-address: true
```

```java
//  Use the registered name in WebClient/Feign
WebClient.builder().baseUrl("http://order-service").build();   //  Eureka resolves IP

@FeignClient("order-service")   //  Same — Feign resolves via Eureka
public interface OrderClient { ... }
```

**Kubernetes (no extra setup):**
```java
//  Just use the K8s Service DNS name
WebClient.builder().baseUrl("http://order-service.default.svc.cluster.local").build();
//  Or with Spring Cloud Kubernetes:
WebClient.builder().baseUrl("http://order-service").build();   // resolves via K8s
```

**Common Follow-Ups:**
- "What about service mesh?" → Istio/Linkerd handle discovery + retries + load balancing transparently — app code just makes HTTP calls
- "Why is client-side load balancing preferred over server-side?" → Removes the load balancer bottleneck; better latency

---

## Top 3 from This File (must-know)

1. **Q02** — Circuit breaker pattern
2. **Q03** — Saga pattern (orchestration vs choreography)
3. **Q04** — Transactional Outbox (most-asked microservices pattern 2024-2026)

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| Service comm                      | OpenFeign (decl) / WebClient     |
| Circuit breaker                   | Resilience4j (Hystrix is dead)   |
| Saga                              | Compensating actions on failure  |
| Orchestration vs Choreography     | Central state vs event-driven    |
| Transactional Outbox              | DB + outbox in 1 txn; CDC pubs   |
| Distributed tracing               | Micrometer Tracing + Zipkin/Tempo|
| Trace propagation                 | traceparent header (W3C)         |
| Config                            | Cloud Config / K8s ConfigMap     |
| Secrets                           | Vault, NOT in application.yml    |
| API Gateway                       | Spring Cloud Gateway / Kong      |
| Service discovery (K8s)           | DNS via Service objects          |
| Service discovery (non-K8s)       | Eureka / Consul                  |
+----------------------------------+----------------------------------+
```
