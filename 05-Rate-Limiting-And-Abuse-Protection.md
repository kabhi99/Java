# **RATE LIMITING & ABUSE PROTECTION — PRODUCTION PATTERNS**

How to protect a Spring Boot API from traffic surges, abusive clients, retry storms, and noisy neighbors — without scaling infinitely or hurting healthy customers.

### TABLE OF CONTENTS

1. The 5 Defense Layers (Defense in Depth)
2. Rate Limiting Algorithms (Token Bucket, Sliding Window, Fixed Window, Leaky Bucket)
3. **PATTERN 1: Per-Tenant Rate Limiting (Redis Token Bucket)**
4. **PATTERN 2: 429 + Retry-After (Backpressure Done Right)**
5. **PATTERN 3: Quotas (Long-Window Limits)**
6. **PATTERN 4: Per-Tenant Bulkheads (Thread Isolation)**
7. **PATTERN 5: Priority Shedding (Save Healthy Customers First)**
8. **PATTERN 6: Idempotency Keys (Make Retries Safe)**
9. **PATTERN 7: API Gateway Enforcement (Stop Abuse at the Edge)**
10. **PATTERN 8: Auto-Scaling & Circuit Breaker (Downstream Protection)**
11. **FEATURED PROBLEM: "Your API Went Viral Overnight" (LinkedIn Q)**
12. Golden Rules & Interview Cheat Codes

---

## PART 1: THE 5 DEFENSE LAYERS (Defense in Depth)

> No single layer is enough. Stack them so abuse stops at the cheapest layer.

```
+---------------------------------------------------------------+
|  Layer 1: EDGE (CDN / WAF)                                    |
|   ↓ blocks DDoS, geo-blocks, bot detection                   |
|                                                               |
|  Layer 2: API GATEWAY                                         |
|   ↓ per-tenant rate limits, quotas, auth, 429 + Retry-After  |
|                                                               |
|  Layer 3: APPLICATION (Spring Boot pod)                       |
|   ↓ per-tenant bulkhead, local rate limit, priority shed     |
|                                                               |
|  Layer 4: DOWNSTREAM PROTECTION                               |
|   ↓ circuit breaker, timeout, bulkhead on outbound calls     |
|                                                               |
|  Layer 5: AUTO-SCALING + OBSERVABILITY                        |
|     HPA, alerts on per-tenant spikes, kill-switch            |
+---------------------------------------------------------------+
```

 **GOLDEN RULE**: "Stop abuse at the cheapest layer." Blocking at the API Gateway costs ~$0; blocking at the DB costs everything.

---

## PART 2: RATE LIMITING ALGORITHMS

| Algorithm | How it Works | Pros | Cons | Use When |
|---|---|---|---|---|
| **Token Bucket** | Bucket holds N tokens; refills at rate R; each request consumes 1 | Allows bursts up to bucket size; simple | Slight unfairness on burst | **Default choice** — most APIs |
| **Leaky Bucket** | Requests enter a queue, leak out at fixed rate | Smooths traffic | Doesn't allow bursts; needs queue | Strict outbound rate (e.g., to a metered downstream) |
| **Fixed Window** | Count requests in current minute/hour | Trivial to implement | **Bug**: 2× burst at window boundary | Coarse-grained, low-stakes |
| **Sliding Window Log** | Store timestamp of every request | Exact | Memory-heavy | Low-traffic, exact accuracy needed |
| **Sliding Window Counter** | Weighted blend of two fixed windows | Memory-efficient + smooth | Slightly approximate | **Production go-to** alternative to token bucket |

> **In practice**: 90% of APIs should use **Token Bucket** (allows bursts) or **Sliding Window Counter** (smoother).

### Visualizing Token Bucket
```
Bucket capacity: 100 tokens, refill rate: 10/sec
  t=0s:  bucket=100, request arrives → 99 ✓
  t=0s:  100 more requests arrive    → 99→0, then 99 rejected with 429
  t=10s: 10 tokens refilled          → bucket=10, accepts next 10
```

---

## PART 3: PER-TENANT RATE LIMITING (Redis Token Bucket)

> The MOST important pattern for multi-tenant APIs. Rate limit per **API key / tenant / user**, NOT globally.

### Why per-tenant?
**Global rate limit fails the whole API** when one tenant misbehaves. **Per-tenant rate limit** punishes only the abuser.

### **Option A: Bucket4j + Redis (Spring Boot industry standard)**

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-redis</artifactId>
</dependency>
```

```java
@Configuration
public class RateLimitConfig {

    @Bean
    public ProxyManager<String> proxyManager(RedisClient redis) {
        return Bucket4jRedis.casBasedBuilder(redis).build();
    }
}

@Component
public class TenantRateLimiter {

    private final ProxyManager<String> proxyManager;

    public boolean tryConsume(String apiKey, TenantTier tier) {
        // Different buckets per tier (free=100/min, pro=1000/min, enterprise=10K/min)
        BucketConfiguration cfg = BucketConfiguration.builder()
            .addLimit(Bandwidth.simple(tier.limit(), Duration.ofMinutes(1)))
            .build();

        Bucket bucket = proxyManager.builder().build("rl:" + apiKey, cfg);
        return bucket.tryConsume(1);
    }
}
```

### **Spring Filter — apply to every request**

```java
@Component
@Order(1)
public class RateLimitFilter extends OncePerRequestFilter {

    private final TenantRateLimiter rl;
    private final TenantService tenants;

    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        String apiKey = req.getHeader("X-API-Key");
        if (apiKey == null) {
            res.sendError(401, "Missing API key");
            return;
        }

        Tenant tenant = tenants.lookup(apiKey);

        if (!rl.tryConsume(apiKey, tenant.tier())) {
            res.setStatus(429);
            res.setHeader("Retry-After", "1");                   // seconds
            res.setHeader("X-RateLimit-Limit", String.valueOf(tenant.tier().limit()));
            res.setHeader("X-RateLimit-Remaining", "0");
            res.getWriter().write("{\"error\":\"rate_limited\"}");
            return;
        }
        chain.doFilter(req, res);
    }
}
```

### **Option B: Resilience4j RateLimiter (per-config)**

```yaml
resilience4j:
  ratelimiter:
    instances:
      payments-free:
        limitForPeriod: 100
        limitRefreshPeriod: 1m
      payments-pro:
        limitForPeriod: 1000
        limitRefreshPeriod: 1m
```

```java
@RateLimiter(name = "payments-${tenant.tier}", fallbackMethod = "rateLimited")
public PaymentResponse authorize(PaymentRequest req) { ... }
```

> **Limitation**: Resilience4j RateLimiter is **in-process** — each pod has its own counter. For 10 pods × 100/min limit, the real cap is 1000/min. Use Redis-backed (Bucket4j) for accurate distributed limits.

### **Hybrid: Local + Redis (lowest latency, highest accuracy)**

```java
//  Per-pod local rate limit (cheap, fast) — catches most abuse
if (!localBucket.tryConsume(1)) {
    return false;
}
//  Redis-backed limit (accurate across pods) — backstop
return redisBucket.tryConsume(1);
```

---

## PART 4: 429 + Retry-After (Backpressure Done Right)

> Rejecting requests is HALF the job. Telling clients **when to retry** prevents them from hammering you harder.

### **The headers you MUST send**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30                       ← seconds OR HTTP-date
X-RateLimit-Limit: 1000               ← your limit
X-RateLimit-Remaining: 0              ← how many left in this window
X-RateLimit-Reset: 1717245600         ← unix timestamp when bucket refills

{"error":"rate_limited","retry_after":30}
```

### **Why this matters**

| Without `Retry-After` | With `Retry-After` |
|---|---|
| Client retries immediately → more 429s | Client backs off intelligently |
| Tight retry loop → 100x amplification | Smooth recovery |
| Your DB still hammered | Healthy system after refill |

### **Critical: respond FAST on 429**

Make sure the 429 response is **cheap**. If your rate-limit check itself is slow (e.g., synchronous DB call), the rejected requests still consume threads. Keep the check in Redis (~1ms) or local (microseconds).

---

## PART 5: QUOTAS (Long-Window Limits)

> Rate limiting is per-second/minute. **Quotas** are per-day/month — for billing, fair use, and SLA enforcement.

```java
@Component
public class QuotaService {

    private final RedisTemplate<String, Long> redis;

    public boolean consumeDaily(String apiKey, long cost) {
        String key = "quota:" + apiKey + ":" + LocalDate.now();
        Long newCount = redis.opsForValue().increment(key, cost);
        if (newCount == cost) {
            redis.expire(key, Duration.ofDays(2));                // auto-cleanup
        }
        long limit = limitForApiKey(apiKey);
        return newCount <= limit;
    }
}
```

**Always set TTL** on Redis keys to prevent unbounded growth.

---

## PART 6: PER-TENANT BULKHEADS (Thread Isolation)

> Even with rate limits, one tenant's slow requests can monopolize the thread pool. **Bulkheads** cap concurrent calls per tenant.

```yaml
resilience4j:
  bulkhead:
    instances:
      payments-per-tenant:
        maxConcurrentCalls: 20         # any one tenant ≤ 20 concurrent
        maxWaitDuration: 100ms         # reject after 100ms wait
```

```java
@Bulkhead(name = "payments-per-tenant", fallbackMethod = "overloaded")
public PaymentResponse authorize(String apiKey, PaymentRequest req) { ... }

public PaymentResponse overloaded(String apiKey, PaymentRequest req, BulkheadFullException ex) {
    return PaymentResponse.tooBusy("Try again shortly");           // 503
}
```

> Combined with per-tenant rate limit: rate limit caps **rate**, bulkhead caps **concurrency**. You need both.

---

## PART 7: PRIORITY SHEDDING (Save Healthy Customers First)

> When you must drop traffic, drop **abusive / low-priority** first. Don't let one tenant's burst harm your enterprise customers.

### **Priority Queue Pattern**

```java
public class PriorityRequestExecutor {

    private final PriorityBlockingQueue<PrioritizedRequest> queue = new PriorityBlockingQueue<>();

    public CompletableFuture<Response> submit(Request req, Tenant tenant) {
        int priority = computePriority(tenant);     // enterprise=1, pro=5, free=10
        var prioReq = new PrioritizedRequest(priority, req);
        queue.offer(prioReq);
        return prioReq.future();
    }

    // Drop low-priority when queue is full
    public boolean shouldShed(Tenant tenant, int queueDepth) {
        if (queueDepth > 8000 && tenant.tier() == FREE) return true;
        if (queueDepth > 9000 && tenant.tier() == PRO)  return true;
        return false;
    }
}
```

### **Adaptive shedding** (Netflix Concurrency Limits)

Use library like [Netflix `concurrency-limits`](https://github.com/Netflix/concurrency-limits) which **dynamically discovers** the right concurrency limit based on latency.

```java
Limiter<Void> limiter = AimdLimiter.newBuilder()
    .maxLimit(1000)
    .build();
```

---

## PART 8: IDEMPOTENCY KEYS (Make Retries Safe)

> If the partner has a retry bug, idempotency keys make those retries **harmless** instead of catastrophic.

```http
POST /api/payments/authorize
Idempotency-Key: 7f9e8d6c-...
Content-Type: application/json
```

```java
@PostMapping("/authorize")
public PaymentResponse authorize(
    @RequestHeader("Idempotency-Key") String key,
    @RequestBody PaymentRequest req
) {
    // 1. Already processed? Return cached response.
    Optional<PaymentResponse> cached = idempotencyStore.get(key);
    if (cached.isPresent()) return cached.get();

    // 2. Acquire lock to prevent concurrent duplicates
    if (!idempotencyStore.tryLock(key, Duration.ofSeconds(30))) {
        throw new ConflictException("Concurrent request with same idempotency key");
    }

    try {
        PaymentResponse resp = paymentService.authorize(req);
        idempotencyStore.save(key, resp, Duration.ofHours(24));
        return resp;
    } finally {
        idempotencyStore.unlock(key);
    }
}
```

> See `02-Spring-Boot-Resilience-Patterns.md` Part 9 for the full pattern.

---

## PART 9: API GATEWAY ENFORCEMENT

> **Best place to enforce rate limits is at the edge** — Kong, Spring Cloud Gateway, AWS API Gateway, Cloudflare. Backend pods never see rejected traffic.

### **Spring Cloud Gateway example**

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: payments
        uri: lb://payments-service
        predicates:
        - Path=/api/payments/**
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 100      # tokens/sec
            redis-rate-limiter.burstCapacity: 200       # max burst
            key-resolver: "#{@apiKeyResolver}"          # per API key
```

```java
@Bean
public KeyResolver apiKeyResolver() {
    return exchange -> Mono.justOrEmpty(
        exchange.getRequest().getHeaders().getFirst("X-API-Key"));
}
```

### **Why edge enforcement wins**

| Concern | Backend Pod | API Gateway |
|---|---|---|
| Cost of rejecting | Spawns thread, allocates objects | Drops in ~1ms, near-zero CPU |
| TLS termination | Per-pod | Centralized |
| Auth | Duplicated per service | Centralized |
| Per-tenant config | Hard | YAML/admin UI |

---

## PART 10: AUTO-SCALING & CIRCUIT BREAKER

### **Horizontal Pod Autoscaler (Kubernetes)**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payments-service
spec:
  scaleTargetRef: { kind: Deployment, name: payments-service }
  minReplicas: 4
  maxReplicas: 50
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  - type: Pods
    pods:
      metric: { name: http_requests_per_second }
      target: { type: AverageValue, averageValue: "200" }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30        # react fast
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
    scaleDown:
      stabilizationWindowSeconds: 300       # scale down slowly
```

> **WARNING**: Scaling does NOT solve an abuse problem. If 80% traffic is abuse, scaling 10x just costs 10x for nothing. **Rate limit first, then scale.**

### **Circuit Breaker on the Payment Gateway (downstream)**

```java
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "gatewayFallback")
public PaymentResponse callGateway(PaymentRequest req) { ... }
```

> When gateway latency spikes, the breaker opens → fail fast → recover the upstream. See `02-Spring-Boot-Resilience-Patterns.md`.

---

## PART 11: FEATURED — "YOUR API WENT VIRAL OVERNIGHT"

### **THE PROBLEM** (LinkedIn interview question)

> Spring Boot merchant checkout API. `POST /api/payments/authorize`.
> - **Normal**: 2,000 req/min
> - **One day**: partner integration has retry bug → traffic jumps to **250,000 req/min** (125×)
>
> **Symptoms:**
> - CPU spikes to 95%
> - DB connection pool exhausted
> - Redis overloaded
> - Payment gateway latency jumps
> - Healthy customers begin receiving 500s
>
> **Investigation finds: 80% of traffic comes from 3 abusive API keys.**
> Checkout availability dropped to 62%.

### **DIAGNOSIS (read the symptoms!)**

| Signal | Inference |
|---|---|
| **80% from 3 API keys** | No per-tenant rate limit — this is the killer |
| Partner retry bug → 125× | No backpressure (clients amplify failures) |
| CPU 95%, DB exhausted | Abusive traffic is admitted into the system |
| Healthy customers get 500s | No tenant isolation — noisy neighbor effect |
| Gateway latency jumps | Downstream cascading failure (no CB on gateway) |
| Redis overloaded | Rate-limit check itself hitting Redis hot path |

### **THE FIX (in priority order)**

#### **STEP 1: STOP THE BLEEDING (5 minutes)**

Block the 3 abusive API keys at the API Gateway. Even a static IP/key allowlist + denylist in the gateway buys time to deploy the real fix.

```yaml
# Spring Cloud Gateway — emergency block
filters:
- name: RequestHeaderFilter
  args:
    name: X-API-Key
    blockedValues: ["abusekey1", "abusekey2", "abusekey3"]
```

#### **STEP 2: PER-TENANT RATE LIMIT AT THE GATEWAY**

```yaml
# Per-API-key token bucket — applied at the edge
filters:
- name: RequestRateLimiter
  args:
    redis-rate-limiter.replenishRate: 100    # 100/sec base
    redis-rate-limiter.burstCapacity: 200
    key-resolver: "#{@apiKeyResolver}"
```

With **per-key** limiting, the 3 abusers get 429s; healthy customers go through.

#### **STEP 3: 429 + Retry-After + RFC 6585 headers**

So the partner's retry bug doesn't tighten into an infinite loop.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 5
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
```

#### **STEP 4: IDEMPOTENCY KEY on `/authorize`**

Make the partner's retries safe — same key returns cached response instead of double-charging.

```java
@PostMapping("/authorize")
public PaymentResponse authorize(
    @RequestHeader("Idempotency-Key") String key,
    @RequestBody PaymentRequest req
) { ... }
```

#### **STEP 5: BULKHEAD per tenant (in-app)**

Even if rate limits leak, no single tenant can monopolize the thread pool.

```java
@Bulkhead(name = "payments-${tenant.tier}")
public PaymentResponse authorize(...) { ... }
```

#### **STEP 6: CIRCUIT BREAKER on payment gateway (downstream)**

When the gateway gets slow, fail fast instead of piling up.

```java
@CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallback")
public GatewayResponse callGateway(...) { ... }
```

#### **STEP 7: PRIORITY SHEDDING**

When still over capacity, drop free-tier first, never enterprise.

#### **STEP 8: QUOTAS (long-term fix)**

Daily/monthly quotas per tier — prevents future surprise spend & abuse.

#### **STEP 9: AUTO-SCALING (CPU + custom metric)**

Scale up when healthy demand grows — but never as a substitute for rate limiting.

#### **STEP 10: ALERTING (per-API-key spike detection)**

PagerDuty alert when a single API key's RPS jumps 10× from its baseline — catches this incident in minutes instead of hours.

### **WHY THIS COMBINATION WORKS**

| Requirement | How it's met |
|---|---|
| Stop abuse | Per-API-key rate limit at API Gateway |
| Protect healthy customers | Per-tenant bulkhead + priority shedding |
| Tell client to back off | 429 + Retry-After |
| Make retries safe | Idempotency keys |
| Protect downstream | Circuit breaker on payment gateway |
| Survive future spikes | Quotas + per-key alerts + HPA |

### **EXPECTED OUTCOME**

| Metric | During incident | After fix |
|---|---|---|
| Checkout availability | 62% | 99.9%+ for healthy customers |
| Abusive traffic admitted | 100% | 0% (429 at gateway) |
| Pod CPU | 95% | <50% |
| DB pool exhausted | Yes | No |
| Gateway latency | Spiking | Normal |

### **INTERVIEW NARRATIVE (use this!)**

> "The killer signal is **80% of traffic from 3 API keys** — this isn't a scaling problem, it's an **abuse / isolation problem**. Scaling up would just cost 10× to fail in the same way.
>
> I'd stack defenses in layers:
>
> **First** — emergency: block the 3 keys at the API Gateway. Stops the bleeding in minutes.
>
> **Second** — permanent: **per-API-key rate limiting** at the gateway using a Redis-backed token bucket (Bucket4j or Spring Cloud Gateway's `RequestRateLimiter`). The abusers get 429s, healthy customers go through.
>
> **Third** — every 429 carries `Retry-After` so the partner's retry bug backs off instead of tightening the loop.
>
> **Fourth** — **idempotency keys** on `/authorize`. The partner's retries become safe; same key returns cached response instead of re-charging.
>
> **Fifth** — defense in depth at the application: **per-tenant bulkhead** so even a leaking rate limit can't starve healthy customers, and a **circuit breaker on the downstream payment gateway** to fail fast when it degrades.
>
> **Sixth** — long-term: **quotas** per tier, **priority shedding** for free-tier traffic when over capacity, and **per-API-key alerting** so we catch the next anomaly in minutes.
>
> The principle is **stop abuse at the cheapest layer** — blocking at the gateway costs nearly nothing, blocking at the DB costs everything."

---

## PART 12: GOLDEN RULES

```
+----------------------------------------------------------------------+
|  1. ALWAYS rate-limit per-tenant (API key/user), NEVER globally only |
|  2. Enforce at the EDGE (API Gateway), not at every backend pod       |
|  3. Every 429 MUST carry Retry-After + X-RateLimit-* headers          |
|  4. Token Bucket = default; Sliding Window Counter = smoother         |
|  5. Use Redis-backed limits for distributed pods (Bucket4j)           |
|  6. Combine rate limit (req/sec) + bulkhead (concurrency) + quota (day)|
|  7. Idempotency keys make retries SAFE — required for any POST         |
|  8. Stop abuse at the CHEAPEST layer (CDN → Gateway → App → DB)        |
|  9. Don't scale to absorb abuse — block it first, then scale          |
| 10. Priority-shed: drop free-tier before enterprise; abusive first    |
| 11. Alert on per-API-key RPS anomalies (10× baseline), not just total |
| 12. Make the 429 response CHEAP (Redis hit, not DB)                   |
| 13. Use circuit breaker on every outbound call (downstream protection)|
| 14. HPA on CPU + custom request-rate metric                           |
| 15. Have a kill-switch: emergency block list at the gateway           |
+----------------------------------------------------------------------+
```

---

## PART 13: INTERVIEW CHEAT CODES

> "The very first thing I look at in any traffic surge is the **per-tenant distribution**. If 80% of traffic comes from 3 tenants, that's an **abuse / isolation problem**, not a scaling problem — and scaling up would just cost 10× to fail the same way."

> "I enforce rate limits at the **API Gateway with a Redis-backed token bucket** — `Bucket4j` or Spring Cloud Gateway's `RequestRateLimiter`. The bucket is keyed by API key, not IP, so legitimate large customers aren't punished for one rogue caller behind a NAT."

> "Every 429 must carry `Retry-After`. Without it, a buggy client retries immediately, creating a tighter loop and making everything worse — the exact mechanism behind most retry-storm incidents."

> "Rate limit and bulkhead are **complementary**, not redundant. Rate limit caps the **rate** (req/sec); bulkhead caps **concurrency**. A slow tenant doing 10 req/sec can still monopolize threads if each request hangs for 30s — that's what bulkhead prevents."

> "**Idempotency keys** are the unsung hero of retry-storms — if the partner has a retry bug, idempotency means those retries are free instead of catastrophic. They return cached responses instead of double-charging."

> "I never **scale to absorb abuse**. The principle is to stop bad traffic at the cheapest layer — blocking at the API Gateway costs ~$0, blocking at the DB costs everything. HPA is for healthy demand, not abusive demand."

> "I also add a **kill-switch** — an emergency block list at the gateway, deployable in minutes, so we can stop a runaway client even before the engineering team gets paged."

> "For long-term protection: **quotas** per tier (daily/monthly), **priority shedding** to drop free-tier before enterprise, and **per-tenant alerts** on 10× baseline so we catch the next anomaly in minutes."

---

## QUICK REVISION TABLE

```
+-----------------------------------+------------------------------------+
| Problem                            | Fix                               |
+-----------------------------------+------------------------------------+
| One tenant dominates traffic      | Per-tenant rate limit (per-key)    |
| Partner retry bug → 125× spike    | 429 + Retry-After + idempotency    |
| CPU 95% under abuse                | Block at gateway, don't scale up   |
| Healthy customers get 500s         | Per-tenant bulkhead + priority     |
| Redis hot on rate-limit check     | Hybrid: local bucket + Redis       |
| Downstream gateway latency spikes | Circuit breaker on outbound        |
| Need to stop a specific client    | Kill-switch / denylist at gateway  |
| Long-term fair use enforcement    | Daily/monthly quotas               |
| Want to detect next spike fast    | Per-API-key RPS alerts (10× base)  |
| Auto-grow for legit demand        | HPA on CPU + req-rate metric       |
| Stop attacks before app          | WAF / CDN (Cloudflare, etc.)       |
| Differentiated SLA per tier       | Per-tier rate limit + bulkhead     |
+-----------------------------------+------------------------------------+
```
