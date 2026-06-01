# **PRODUCTION, OBSERVABILITY & PERFORMANCE**

8 questions on running Spring Boot in production.

---

### Q01: What is Spring Boot Actuator and which endpoints matter in production? ⭐

**30-Second Answer:** Actuator exposes **operational endpoints** for monitoring/management. In production, expose `/health`, `/info`, `/metrics`, `/prometheus`, `/loggers`. **NEVER expose `/heapdump`, `/threaddump`, `/env`, `/shutdown` publicly** — security risk.

**Deep Dive:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true     # /health/liveness and /health/readiness for K8s
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENV}
  prometheus:
    metrics:
      export:
        enabled: true
  server:
    port: 8081         # separate port for actuator (security)
```

**K8s probes:**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
```

**Common Follow-Ups:**
- "Difference between liveness and readiness?" → Liveness fail → K8s restarts pod; Readiness fail → K8s removes from LB (no restart)
- "How do you protect actuator endpoints?" → Run on separate port; require admin role; restrict via network policy

**Red Flag If You Say:** "Expose all actuator endpoints with `*`." (`/env` leaks secrets, `/heapdump` lets attacker dump memory.)

---

### Q02: How do you add custom metrics with Micrometer?

**30-Second Answer:** Inject `MeterRegistry`, create counters/gauges/timers. Tag with relevant dimensions (endpoint, tenant, status). They're auto-exported to Prometheus/Datadog/CloudWatch via the appropriate registry.

**Deep Dive:**

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final MeterRegistry registry;
    private final Counter charges;
    private final Timer chargeTimer;

    @PostConstruct
    void init() {
        charges = Counter.builder("payment.charges")
            .description("Total payment charges")
            .tags("service", "payment")
            .register(registry);
    }

    public void charge(PaymentRequest req) {
        Timer.Sample sample = Timer.start(registry);
        try {
            paymentClient.charge(req);
            registry.counter("payment.charges", "status", "success", "tenant", req.getTenant()).increment();
        } catch (Exception ex) {
            registry.counter("payment.charges", "status", "failed", "reason", classify(ex)).increment();
            throw ex;
        } finally {
            sample.stop(registry.timer("payment.duration", "endpoint", "charge"));
        }
    }
}

//  Gauge for live values
@Bean
public Gauge activeUsersGauge(MeterRegistry r, UserSessionManager sessions) {
    return Gauge.builder("users.active", sessions, UserSessionManager::activeCount)
        .description("Number of active users")
        .register(r);
}
```

**Naming conventions (Micrometer):**
- Lowercase, dot-separated: `http.server.requests`, `payment.charges`
- Use **tags** for dimensions, not metric names: `payment.charges{status="success"}` NOT `payment.charges.success`

**Common Follow-Ups:**
- "Why tags?" → Cardinality + flexible querying (avg by status, sum by tenant)
- "What's the cardinality risk?" → Too many tag combinations → memory explosion. Don't tag with `user_id` (unbounded)

---

### Q03: How do you do structured logging in Spring Boot?

**30-Second Answer:** Use **JSON output** with **MDC** (Mapped Diagnostic Context) for request-scoped fields (traceId, userId, tenantId). Then ingest into Splunk/ELK/Datadog. Avoid string concatenation; use parameterized logs.

**Deep Dive:**

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
</dependency>
```

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>tenantId</includeMdcKeyName>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

```java
//  Set MDC in a filter
@Component
public class MdcFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        try {
            MDC.put("userId", extractUserId(req));
            MDC.put("tenantId", extractTenantId(req));
            chain.doFilter(req, res);
        } finally {
            MDC.clear();   //  REQUIRED — else MDC leaks across requests
        }
    }
}

//  Use parameterized logs
log.info("Order created order_id={} amount={} user_id={}", orderId, amount, userId);   // 

//  Don't
log.info("Order " + orderId + " created with amount " + amount);   //  No JSON fields
```

**Output:**
```json
{
  "timestamp": "2026-06-01T12:00:00.123Z",
  "level": "INFO",
  "message": "Order created order_id=42 amount=99.99 user_id=u_1",
  "traceId": "a1b2c3...",
  "userId": "u_1",
  "tenantId": "acme"
}
```

**Common Follow-Ups:**
- "Why MDC.clear()?" → Thread is reused (Tomcat pool); next request inherits stale MDC
- "Async + MDC?" → Use `DelegatingTaskExecutor` from `slf4j-jdk-platform-logging` or copy MDC manually

---

### Q04: How do you tune HikariCP for production?

**30-Second Answer:** Set `maximum-pool-size = (2 × DB cores)` per pod. Always set `connection-timeout`, `idle-timeout`, `max-lifetime`, `leak-detection-threshold`. Monitor `hikaricp.connections.active` metric. **If total pods × pool > DB max_connections, use PgBouncer.**

**Deep Dive:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # rule of thumb: 2 × DB CPU cores
      minimum-idle: 5
      connection-timeout: 2000       # ms to wait for a connection
      idle-timeout: 600000           # 10 min
      max-lifetime: 1800000          # 30 min — rotate before LB closes
      leak-detection-threshold: 60000 # log if connection held > 60s
      pool-name: app-hikari
      auto-commit: false             # if using @Transactional
      transaction-isolation: TRANSACTION_READ_COMMITTED
```

**Sizing formula:**
- DB has 8 cores, 200 max_connections
- App runs 10 pods
- Per-pod pool = (2 × 8) = ~16-20
- Total across pods = 10 × 20 = 200 → AT the DB limit!
- Solution: PgBouncer in transaction-pool mode (multiplexes 1000 app connections → 50 actual DB connections)

**Monitoring (Micrometer auto-exposes):**
- `hikaricp.connections.active` — currently in use
- `hikaricp.connections.pending` — waiting for a connection (RED FLAG if > 0)
- `hikaricp.connections.timeout.total` — couldn't get a connection
- `hikaricp.connections.usage.seconds` — how long held (long = leak suspect)

**Common Follow-Ups:**
- "Why 2× cores?" → Brian Goetz formula: `pool_size = cores × 2 + effective_spindle_count`
- "Why does `max-lifetime` matter?" → Cloud LBs (AWS, GCP) close idle connections at 60-300s; HikariCP must rotate first

---

### Q05: How do you diagnose a production OOM?

**30-Second Answer:** Enable **heap dump on OOM**, ship to S3, analyze with **Eclipse MAT** or **VisualVM**. Look for **leak suspect report**. Check **off-heap memory** (Netty buffers, native, JNI). Verify GC logs for memory pressure patterns.

**Deep Dive:**

```bash
# JVM flags — enable heap dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/oom-$(date +%s).hprof
-XX:OnOutOfMemoryError="aws s3 cp /dumps/ s3://my-bucket/oom/ --recursive"

# GC logs (Java 9+)
-Xlog:gc*:file=/logs/gc.log:time,level,tags:filecount=10,filesize=100M

# Java Flight Recorder (continuous, low overhead)
-XX:StartFlightRecording=name=app,filename=/recordings/jfr.jfr,maxsize=500m,maxage=24h
```

**Diagnosis checklist:**
1. **Heap or non-heap OOM?** Check error message
   - `Java heap space` → heap leak; analyze .hprof
   - `Metaspace` → too many classes loaded (class loader leak)
   - `Direct buffer memory` → Netty / NIO buffer leak (set `-XX:MaxDirectMemorySize`)
   - `unable to create new native thread` → thread leak
2. **Heap dump in MAT** → Leak Suspect report → find dominator
3. **Check usual suspects**:
   - Unbounded caches (Caffeine without `maximumSize`)
   - Result sets without pagination (loading entire table)
   - `ThreadLocal` not cleaned up (web request thread reused)
   - Static collections growing forever
   - Connection/file handle leaks (off-heap)

**Common Follow-Ups:**
- "What's a class loader leak?" → Hot-reload (Tomcat) keeps old class loaders alive when references leak — Metaspace grows
- "Why does midnight crash often happen?" → Batch job loads massive result set without streaming; GC can't keep up

> See `Java/04-Scaling-Spring-Boot-APIs.md` Part 1 for the full diagnosis ladder.

---

### Q06: How do you do zero-downtime deployments with Spring Boot?

**30-Second Answer:** **Graceful shutdown** + **K8s rolling deployment** + **readiness probe** + **connection drain time**. Spring Boot 2.3+ has built-in graceful shutdown: `server.shutdown=graceful`.

**Deep Dive:**

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # max wait for in-flight requests
```

```yaml
# K8s deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0          # zero downtime
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 60   #  > graceful timeout
      containers:
      - lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]   #  drain time before SIGTERM
        readinessProbe:
          httpGet: { path: /actuator/health/readiness, port: 8081 }
```

**Shutdown sequence:**
1. K8s sends SIGTERM
2. preStop hook runs → sleep 10s → K8s LB removes pod from routing
3. SIGTERM hits Spring Boot → graceful shutdown starts
4. New requests rejected; in-flight requests complete (up to 30s)
5. Spring closes DB connections, executors, etc.
6. JVM exits → K8s starts new pod (already rolling)

**For DB schema changes — expand/contract:**
```
1. ADD column (nullable, backward compat)        ← old code OK
2. Deploy new code that writes both old + new
3. Backfill data
4. Deploy new code that reads new column
5. Drop old column (after old code gone)
```

**Common Follow-Ups:**
- "What's `maxUnavailable: 0`?" → Never let total available pods drop below desired count → zero downtime
- "Why preStop sleep?" → K8s removes from service AFTER preStop completes; gives LB time to stop sending traffic

---

### Q07: How do you tune JVM for a Spring Boot service?

**30-Second Answer:** Use **container-aware JVM** (Java 10+ does it automatically). Set heap to `-Xmx = 75% of container memory`. Use **G1GC** (default Java 11+) for most apps; ZGC/Shenandoah for low-latency. Always set **`-XX:MaxDirectMemorySize`** if using Netty.

**Deep Dive:**

```bash
# Standard Java 17 production flags
-XX:MaxRAMPercentage=75            # use 75% of container memory for heap
-XX:InitialRAMPercentage=75        # same as initial
-XX:+UseG1GC                       # default — good for most
-XX:MaxGCPauseMillis=200           # G1 target pause time
-XX:+ParallelRefProcEnabled
-XX:+UnlockExperimentalVMOptions
-XX:+UseStringDeduplication        # G1 dedup duplicate Strings (saves heap)
-XX:MaxDirectMemorySize=512m       # cap off-heap for Netty
-XX:+ExitOnOutOfMemoryError        # fail fast on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/
-XX:+UseContainerSupport           # default in Java 10+
-XX:NativeMemoryTracking=summary   # diagnose native leaks
```

**GC choice:**
| GC | Use case |
|---|---|
| **G1GC** (default) | Most apps; balanced throughput vs latency |
| **ZGC** | Very large heaps (TBs); ultra-low pauses |
| **Shenandoah** | Low-latency, smaller heaps |
| **ParallelGC** | Batch jobs (max throughput, big pauses OK) |
| **Serial** | Tiny heaps / single CPU |

**Common Follow-Ups:**
- "Why not just `-Xmx2g`?" → If container is later resized, JVM won't grow. `MaxRAMPercentage` adapts
- "Why `+ExitOnOutOfMemoryError`?" → A "limping" JVM in OOM is worse than dead pod (K8s restarts it cleanly)

---

### Q08: What metrics + alerts do you set up for a production Spring Boot service?

**30-Second Answer:** Six categories:
1. **Golden signals** — latency (p50/p95/p99), traffic (RPS), errors (5xx rate), saturation (CPU/memory)
2. **JVM** — GC pause time, heap usage, thread count
3. **HTTP** — request rate by endpoint, error rate by status code
4. **DB** — Hikari active/pending, query time, connection wait
5. **Downstream** — call latency, circuit breaker state, retry rate
6. **Business** — orders/sec, payments/sec, signups/sec

**Deep Dive:**

```yaml
# Example alert rules (Prometheus syntax)
# 5xx error rate > 1% for 5 minutes
- alert: HighErrorRate
  expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) /
        rate(http_server_requests_seconds_count[5m]) > 0.01
  for: 5m

# p99 latency > 1s
- alert: HighLatency
  expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m])) > 1
  for: 5m

# Hikari pool pending (waiting for connection)
- alert: DbPoolPending
  expr: hikaricp_connections_pending > 5
  for: 2m

# Heap > 90%
- alert: HighHeapUsage
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
  for: 5m

# Circuit breaker open
- alert: CircuitBreakerOpen
  expr: resilience4j_circuitbreaker_state{state="open"} == 1
  for: 1m

# Restart loop
- alert: PodRestarting
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
```

**Common Follow-Ups:**
- "What's an SLO?" → Service Level Objective — target metric (e.g., "99.9% of requests < 200ms over 30 days")
- "Error budget?" → 100% - SLO = how much you can be down before paging

---

## Top 3 from This File (must-know)

1. **Q01** — Actuator endpoints (security implications!)
2. **Q04** — HikariCP tuning
3. **Q05** — Diagnosing production OOM

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| Actuator                          | Expose health, metrics, prometheus|
| Actuator security                 | Separate port, NEVER expose env  |
| K8s probes                        | liveness=restart, readiness=LB   |
| Micrometer custom metric          | Tags for dimensions              |
| Structured logging                | JSON + MDC + traceId             |
| MDC                               | clear() in finally               |
| Hikari pool size                  | 2 × DB cores per pod             |
| PgBouncer                         | When total conns > DB max        |
| OOM diagnosis                     | HeapDumpOnOOM + MAT/VisualVM     |
| GC logs                           | -Xlog:gc* enabled always         |
| Graceful shutdown                 | server.shutdown=graceful         |
| K8s preStop sleep                 | Drain time before SIGTERM        |
| Schema migration                  | Expand-contract pattern          |
| JVM in container                  | MaxRAMPercentage=75              |
| G1GC                              | Default, good for most apps      |
| Golden signals                    | Latency, traffic, errors, saturation |
+----------------------------------+----------------------------------+
```
