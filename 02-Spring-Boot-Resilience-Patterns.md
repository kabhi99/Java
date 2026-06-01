# **SPRING BOOT RESILIENCE PATTERNS — INDUSTRY STANDARD**

Production-grade patterns for calling external services (payment gateways, third-party APIs) in Spring Boot.

### TABLE OF CONTENTS

1. The Five Resilience Pillars
2. Spring Retry (`@Retryable`)
3. Resilience4j (Modern Industry Standard)
4. Timeout Configuration (RestTemplate, WebClient, Feign)
5. Circuit Breaker
6. Bulkhead
7. Rate Limiter
8. **FEATURED PROBLEM: Selective Retry on Network Errors (LinkedIn Q)**
9. Idempotency (Why Retry Needs It)
10. Golden Rules & Interview Cheat Codes

---

## PART 1: THE FIVE RESILIENCE PILLARS

When calling an external service, ALWAYS combine these:

```
+----------------------------------------------------------------------+
| #  | Pattern         | Protects Against                              |
+----+-----------------+-----------------------------------------------+
| 1  | TIMEOUT         | Hanging threads (slow downstream)             |
| 2  | RETRY           | Transient failures (network blip, 5xx)        |
| 3  | CIRCUIT BREAKER | Cascading failure when downstream is dead     |
| 4  | BULKHEAD        | One slow API consuming all threads            |
| 5  | RATE LIMITER    | Overwhelming downstream / cost control        |
+----+-----------------+-----------------------------------------------+
```

 **MEMORY AID**: "**T**ime to **R**etry, but **C**ut the **B**ulkhead **R**ope" (T-R-C-B-R)

> **NEVER retry without a timeout.** That's the #1 reason for the LinkedIn-style production outage.

---

## PART 2: SPRING RETRY (`@Retryable`)

### **Setup**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableRetry   // REQUIRED — easy to forget!
public class App { ... }
```

### **Basic Usage**

```java
@Service
public class PaymentService {

    @Retryable(
        retryFor = { TransientException.class, SocketTimeoutException.class },
        noRetryFor = { ValidationException.class, SSLHandshakeException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000, random = true)
        // 1s, 2s, 4s with ±50% jitter, capped at 10s
    )
    public PaymentResult charge(PaymentRequest req) {
        return paymentGateway.charge(req);
    }

    @Recover   // Called when ALL retries exhausted
    public PaymentResult fallback(TransientException ex, PaymentRequest req) {
        log.error("Payment failed after retries for order={}", req.getOrderId(), ex);
        return PaymentResult.deferred(req.getOrderId());   // queue for async retry
    }
}
```

### **Critical Gotchas**

| Gotcha | Fix |
|---|---|
| `@Retryable` on a method called from **same class** | Won't work (Spring AOP proxy bypass). Move method to another bean. |
| Forgot `@EnableRetry` | Annotation silently ignored. |
| Retrying non-idempotent operation | Risk: charging customer 2x. Use idempotency key (Part 9). |
| No backoff (`@Backoff` missing) | Retry storm — hammers downstream while it's already struggling. |
| `retryFor = Exception.class` | Will retry on `ValidationException` (4xx) — wasteful. Be specific. |

---

## PART 3: RESILIENCE4J (MODERN INDUSTRY STANDARD)

 **Why Resilience4j over Spring Retry?**
- Bundles ALL patterns (retry + circuit breaker + bulkhead + rate limiter + timeout)
- Functional, lightweight (no AOP), works with reactive (WebFlux)
- Built-in Micrometer metrics
- Replaces Netflix Hystrix (which is in maintenance mode)

### **Setup**

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```yaml
# application.yml
resilience4j:
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 1s
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
        retryExceptions:
          - java.net.SocketTimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - javax.net.ssl.SSLHandshakeException
          - com.example.ValidationException

  circuitbreaker:
    instances:
      paymentService:
        failureRateThreshold: 50           # 50% failures → OPEN
        slowCallRateThreshold: 80          # 80% slow → OPEN
        slowCallDurationThreshold: 2s
        permittedNumberOfCallsInHalfOpenState: 5
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        waitDurationInOpenState: 30s
        automaticTransitionFromOpenToHalfOpenEnabled: true

  timelimiter:
    instances:
      paymentService:
        timeoutDuration: 2s
        cancelRunningFuture: true
```

```java
@Service
public class PaymentService {

    @Retry(name = "paymentService", fallbackMethod = "fallback")
    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
    @TimeLimiter(name = "paymentService")
    @Bulkhead(name = "paymentService")
    public CompletableFuture<PaymentResult> chargeAsync(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() -> paymentGateway.charge(req));
    }

    private CompletableFuture<PaymentResult> fallback(PaymentRequest req, Throwable t) {
        log.error("Fallback for order={}", req.getOrderId(), t);
        return CompletableFuture.completedFuture(PaymentResult.deferred(req.getOrderId()));
    }
}
```

 **ORDER OF DECORATORS MATTERS** (outer wraps inner):
```
Bulkhead → TimeLimiter → CircuitBreaker → Retry → Actual call
```
This way: Retry counts each attempt against the breaker; bulkhead limits concurrent retries.

---

## PART 4: TIMEOUTS (THE #1 MOST FORGOTTEN THING)

> Default `RestTemplate` timeout = **INFINITY**. A hanging downstream will silently consume your thread pool until the app dies.

### **RestTemplate (legacy but common)**

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(2000);   // 2s to establish TCP connection
        factory.setReadTimeout(5000);      // 5s to read response after connect
        return new RestTemplate(factory);
    }
}

// Or with Apache HttpClient (recommended — has connection pool):
@Bean
public RestTemplate restTemplate() {
    PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(200);
    cm.setDefaultMaxPerRoute(20);

    RequestConfig rc = RequestConfig.custom()
        .setConnectTimeout(2000)
        .setSocketTimeout(5000)
        .setConnectionRequestTimeout(1000)   // time to get conn from pool
        .build();

    CloseableHttpClient client = HttpClients.custom()
        .setConnectionManager(cm)
        .setDefaultRequestConfig(rc)
        .build();

    return new RestTemplate(new HttpComponentsClientHttpRequestFactory(client));
}
```

### **WebClient (modern, non-blocking)**

```java
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(5))
            .addHandlerLast(new WriteTimeoutHandler(5)));

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

### **OpenFeign**

```yaml
feign:
  client:
    config:
      paymentGateway:
        connectTimeout: 2000
        readTimeout: 5000
```

### **The Two Timeouts You MUST Set**

| Timeout | Purpose | Typical |
|---|---|---|
| **Connect timeout** | Time to establish TCP connection | 1-3s |
| **Read/socket timeout** | Time to wait for response after connecting | 2-10s |

 **GOLDEN RULE**: `caller_timeout > downstream_timeout + retry_overhead`. Otherwise the caller times out *while* the downstream still has the request → wasted work + status uncertainty.

---

## PART 5: CIRCUIT BREAKER

**WHY:** When downstream is dead, fail FAST instead of timing out for every request (which exhausts your threads).

### **State Machine**

```
       CLOSED ────(failure rate > threshold)──→ OPEN
          ▲                                       │
          │                                       │ (after waitDuration)
          │                                       ▼
   (success in        ┌─────────────────── HALF_OPEN
    HALF_OPEN)       │                      (allow N test calls)
          └──────────┘
                  (if any fails → back to OPEN)
```

### **Resilience4j Circuit Breaker — Code**

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "circuitOpenFallback")
public PaymentResult charge(PaymentRequest req) {
    return paymentGateway.charge(req);
}

public PaymentResult circuitOpenFallback(PaymentRequest req, CallNotPermittedException ex) {
    // Specific handler for circuit-open case (vs. other failures)
    return PaymentResult.serviceUnavailable("Payment provider degraded");
}
```

### **Tuning**

| Setting | Rule of thumb |
|---|---|
| `failureRateThreshold` | 50% — too low = flappy, too high = damage done |
| `slidingWindowSize` | 20-100 calls |
| `minimumNumberOfCalls` | At least 10 (avoid opening on 1 fail) |
| `waitDurationInOpenState` | 30s-60s (give downstream time to recover) |
| `slowCallDurationThreshold` | Treat slow calls (close to timeout) as failures |

---

## PART 6: BULKHEAD

**WHY:** Isolate failures so one slow API can't consume ALL threads in the pool. Think of ship bulkheads — one flooded compartment doesn't sink the ship.

### **Two Types**

```yaml
resilience4j:
  # THREAD POOL BULKHEAD — dedicated pool per service (Hystrix-style)
  thread-pool-bulkhead:
    instances:
      paymentService:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20

  # SEMAPHORE BULKHEAD — caps concurrent calls (lighter)
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms
```

**When to use which:**
- **Semaphore**: Lightweight, no extra threads — for non-blocking code (WebClient, reactive)
- **Thread pool**: Full isolation — for blocking calls that might hang

---

## PART 7: RATE LIMITER

```yaml
resilience4j:
  ratelimiter:
    instances:
      paymentService:
        limitForPeriod: 100        # 100 calls
        limitRefreshPeriod: 1s     # per second
        timeoutDuration: 500ms     # max wait when limit exceeded
```

```java
@RateLimiter(name = "paymentService", fallbackMethod = "rateLimitedFallback")
public PaymentResult charge(PaymentRequest req) { ... }
```

---

## PART 8: FEATURED — SELECTIVE RETRY ON NETWORK ERRORS

### **THE PROBLEM** (LinkedIn interview question)

> Spring Boot Order Service calls an external Payment Gateway.
>
> **Requirements:**
> - Retry transient network errors (timeouts) and HTTP 5xx
> - **Fail fast** on HTTP 4xx and validation failures
> - Apply **explicit timeouts** to avoid hanging threads
> - **Do NOT retry** non-transient network failures like `SSLHandshakeException`
>
> **Production incident:** During a traffic spike, payments failed due to timeouts, but no retries happened because retry config was wrong.

### **WHAT CAN GO WRONG (the bug behind the incident)**

1. `@Retryable(value = Exception.class)` — too broad; retries 4xx and `SSLHandshakeException`, masking real bugs.
2. `@Retryable` only on `IOException` — `SocketTimeoutException` IS an `IOException`, but so is `SSLHandshakeException`. Need finer control.
3. No timeout on `RestTemplate` → calls hang forever → thread pool exhausted → upstream queues build → cascading failure.
4. Retries with no backoff → retry storm → downstream falls over harder.
5. RestTemplate throws `HttpServerErrorException` for 5xx (retryable) and `HttpClientErrorException` for 4xx (NOT retryable) — many devs lump them together.

### **THE INDUSTRY-STANDARD SOLUTION**

#### **Step 1: Configure RestTemplate with explicit timeouts + connection pool**

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public RestTemplate paymentRestTemplate() {
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(100);
        cm.setDefaultMaxPerRoute(20);

        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectTimeout(2_000)              // 2s TCP connect
            .setSocketTimeout(5_000)               // 5s read response
            .setConnectionRequestTimeout(1_000)    // 1s wait for pool slot
            .build();

        CloseableHttpClient client = HttpClients.custom()
            .setConnectionManager(cm)
            .setDefaultRequestConfig(requestConfig)
            .build();

        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(client));
    }
}
```

#### **Step 2: Define exception hierarchy (be precise about retryable)**

```java
// Retryable (transient)
public class TransientPaymentException extends RuntimeException { }

// Non-retryable (permanent)
public class NonTransientPaymentException extends RuntimeException { }
```

#### **Step 3: Translate raw HTTP errors into our exceptions**

```java
@Service
public class PaymentGatewayClient {
    private final RestTemplate restTemplate;

    public PaymentResult charge(PaymentRequest req) {
        try {
            return restTemplate.postForObject("/charges", req, PaymentResult.class);
        }
        // Retryable: timeouts and transient network blips
        catch (ResourceAccessException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof SocketTimeoutException
             || cause instanceof ConnectTimeoutException) {
                throw new TransientPaymentException("Network timeout", ex);
            }
            // SSLHandshakeException, UnknownHostException → NOT transient
            throw new NonTransientPaymentException("Non-transient network error", ex);
        }
        // Retryable: server-side errors
        catch (HttpServerErrorException ex) {     // 5xx
            throw new TransientPaymentException("Gateway 5xx: " + ex.getStatusCode(), ex);
        }
        // NOT retryable: client errors (bad request, auth failure, etc.)
        catch (HttpClientErrorException ex) {     // 4xx
            throw new NonTransientPaymentException("Gateway 4xx: " + ex.getStatusCode(), ex);
        }
    }
}
```

#### **Step 4: Selective retry with backoff + jitter**

```java
@Service
public class PaymentService {

    private final PaymentGatewayClient gateway;

    @Retryable(
        retryFor = TransientPaymentException.class,
        noRetryFor = NonTransientPaymentException.class,
        maxAttempts = 3,
        backoff = @Backoff(
            delay = 500,          // start at 500ms
            multiplier = 2.0,     // 500ms → 1s → 2s
            maxDelay = 5_000,
            random = true         // jitter! prevents thundering herd
        )
    )
    public PaymentResult chargeWithRetry(PaymentRequest req) {
        return gateway.charge(req);
    }

    @Recover
    public PaymentResult onTransientFailure(TransientPaymentException ex, PaymentRequest req) {
        log.error("All retries exhausted for order={}", req.getOrderId(), ex);
        return PaymentResult.deferred(req.getOrderId());   // queue for async retry
    }

    @Recover
    public PaymentResult onPermanentFailure(NonTransientPaymentException ex, PaymentRequest req) {
        log.error("Permanent failure for order={}", req.getOrderId(), ex);
        return PaymentResult.failed(req.getOrderId(), ex.getMessage());
    }
}
```

#### **Step 5 (RECOMMENDED): Use Resilience4j instead — cleaner**

```yaml
resilience4j:
  retry:
    instances:
      paymentGateway:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
        retryExceptions:
          - com.example.TransientPaymentException
        ignoreExceptions:
          - com.example.NonTransientPaymentException
          - javax.net.ssl.SSLHandshakeException
  circuitbreaker:
    instances:
      paymentGateway:
        failureRateThreshold: 50
        slowCallRateThreshold: 80
        slowCallDurationThreshold: 4s
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        waitDurationInOpenState: 30s
  timelimiter:
    instances:
      paymentGateway:
        timeoutDuration: 6s         # > socket timeout
        cancelRunningFuture: true
```

```java
@Retry(name = "paymentGateway")
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallback")
@TimeLimiter(name = "paymentGateway")
public CompletableFuture<PaymentResult> charge(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() -> gateway.charge(req));
}
```

### **WHY THIS SOLUTION IS "INDUSTRY STANDARD"**

| Requirement | How it's met |
|---|---|
| Retry transient (timeout, 5xx) | `TransientPaymentException` → in `retryExceptions` |
| Fail fast on 4xx / validation | `HttpClientErrorException` → `NonTransientPaymentException` → in `ignoreExceptions` |
| Explicit timeouts | RestTemplate `connect=2s`, `read=5s`; `TimeLimiter` outer cap |
| No retry on `SSLHandshakeException` | Mapped to non-transient; also explicit ignore |
| Avoid retry storms | Exponential backoff + jitter |
| Avoid cascading failure | Circuit breaker opens when downstream degrades |

---

## PART 9: IDEMPOTENCY (RETRY'S BEST FRIEND)

> Retry is **DANGEROUS** without idempotency. If the original request succeeded but the response was lost, your retry charges the customer twice.

### **Idempotency Key Pattern**

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResult> charge(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody PaymentRequest req
) {
    return paymentService.chargeIdempotent(idempotencyKey, req);
}

@Service
public class PaymentService {

    public PaymentResult chargeIdempotent(String key, PaymentRequest req) {
        // 1. Check if we've seen this key before
        Optional<PaymentResult> cached = idempotencyStore.get(key);
        if (cached.isPresent()) return cached.get();   // return saved response

        // 2. Acquire lock on key (prevent concurrent duplicates)
        if (!idempotencyStore.acquireLock(key, Duration.ofSeconds(30))) {
            throw new ConflictException("Concurrent request with same key");
        }

        try {
            // 3. Process the payment
            PaymentResult result = gateway.charge(req);

            // 4. Store result keyed by idempotency key (TTL: 24h)
            idempotencyStore.save(key, result, Duration.ofHours(24));
            return result;
        } finally {
            idempotencyStore.releaseLock(key);
        }
    }
}
```

**Store options:** Redis (most common), DynamoDB, RDBMS with unique constraint on `idempotency_key`.

---

## PART 10: GOLDEN RULES

```
+----------------------------------------------------------------------+
|  1. NEVER retry without a timeout                                    |
|  2. NEVER retry without backoff + jitter                             |
|  3. Retry only IDEMPOTENT operations (or use idempotency keys)       |
|  4. Be SPECIFIC about retryable exceptions (not Exception.class)     |
|  5. 4xx = client bug → do NOT retry                                  |
|     5xx = server hiccup → retry                                      |
|     SSLHandshake / UnknownHost = config bug → do NOT retry           |
|     SocketTimeout / ConnectTimeout = transient → retry               |
|  6. Always set BOTH connect timeout AND read timeout                 |
|  7. Caller timeout > downstream timeout (so caller knows the truth)  |
|  8. Combine: Timeout + Retry + Circuit Breaker + Bulkhead            |
|  9. Decorator order (outer → inner):                                 |
|     Bulkhead → TimeLimiter → CircuitBreaker → Retry → call           |
| 10. Use a connection pool (never one connection per call)            |
| 11. Use Resilience4j for new projects (Hystrix is dead)              |
| 12. @Retryable doesn't work on self-invocation (same-class calls)    |
| 13. Don't forget @EnableRetry                                        |
| 14. Always have a @Recover / fallback method                         |
| 15. Emit Micrometer metrics for retry attempts and CB state          |
+----------------------------------------------------------------------+
```

---

## PART 11: INTERVIEW CHEAT CODES

> "I'd combine **five resilience patterns**: explicit timeouts, retry with exponential backoff and jitter, circuit breaker to fail fast when downstream is dead, bulkhead to isolate the thread pool, and a rate limiter to protect downstream."

> "The single most important thing — and where most production incidents start — is **setting an explicit timeout**. RestTemplate defaults to infinity, which silently exhausts your thread pool when downstream hangs."

> "For retry, I classify exceptions into **transient** (timeout, 5xx) and **non-transient** (4xx, SSLHandshake, validation). Only the transient bucket gets retried."

> "I always retry with **exponential backoff + jitter** — without jitter, 1000 clients retry at exactly 1s, 2s, 4s, creating a synchronized retry storm that hammers a recovering service."

> "Retries make sense **only for idempotent operations**. For non-idempotent calls like `POST /payments`, I require an **idempotency key**: the server caches the response keyed by it, so a retried request returns the cached result instead of charging twice."

> "I prefer **Resilience4j** over Spring Retry today — it bundles retry, circuit breaker, timeout, bulkhead, and rate limiter into one library, works with both blocking and reactive code, and has first-class Micrometer metrics."

> "Circuit breaker config matters: `minimumNumberOfCalls` prevents opening on a single failure, and `slowCallDurationThreshold` treats latency degradation as a failure signal — not just outright errors."

> "Decorator order is **Bulkhead → TimeLimiter → CircuitBreaker → Retry → call**. This way the breaker sees each retry as a separate sample, and the bulkhead caps total concurrency including retries."

---

## QUICK REVISION TABLE

```
+-------------------------+---------+----------+--------------------+
| Scenario                | Retry?  | Backoff? | Circuit Breaker?   |
+-------------------------+---------+----------+--------------------+
| SocketTimeout (5xx)     |  Yes    |  Yes     |  Yes               |
| ConnectTimeout          |  Yes    |  Yes     |  Yes               |
| HTTP 503 / 502 / 504    |  Yes    |  Yes     |  Yes               |
| HTTP 500                |  Yes*   |  Yes     |  Yes               |
| HTTP 429 (rate limit)   |  Yes**  |  Yes     |  No (respect 429)  |
| HTTP 400 / 422          |  NO     | N/A      | N/A (client bug)   |
| HTTP 401 / 403          |  NO     | N/A      | N/A                |
| HTTP 404                |  NO     | N/A      | N/A                |
| SSLHandshakeException   |  NO     | N/A      | N/A (config bug)   |
| UnknownHostException    |  NO     | N/A      | N/A (DNS/config)   |
| ValidationException     |  NO     | N/A      | N/A                |
+-------------------------+---------+----------+--------------------+

  * Retry 500 ONLY if idempotent (GET, PUT, DELETE — not raw POST)
 ** Retry 429 after Retry-After header (respect server's hint)
```
