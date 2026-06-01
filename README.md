# **JAVA — INTERVIEW READY NOTES**

Industry-standard Java patterns for backend interviews (Spring Boot, concurrency, resilience).

## Index — Production Scenarios (deep dives with full code)

| # | Topic | What's Inside |
|---|-------|---------------|
| 01 | [Concurrency & Race Conditions](./01-Concurrency-And-Race-Conditions.md) | `synchronized`, `Lock`, atomics, `volatile`, `ConcurrentHashMap`, deadlocks, common race-condition bugs and fixes |
| 02 | [Spring Boot Resilience Patterns](./02-Spring-Boot-Resilience-Patterns.md) | `@Retryable`, Resilience4j, circuit breaker, timeout, bulkhead, idempotency — with the **Selective Retry on Network Errors** LinkedIn question fully solved |
| 03 | [Refactoring Cyclomatic Complexity](./03-Refactoring-Cyclomatic-Complexity.md) | Chain of Responsibility, Strategy + Factory, Specification, Rule Engine — with the **`processPayment()` SonarQube refactor** LinkedIn question fully solved |
| 04 | [Scaling Spring Boot APIs](./04-Scaling-Spring-Boot-APIs.md) | Multi-level caching (Caffeine + Redis), Bloom filter, read replicas, HikariCP tuning, async / virtual threads — with the **User Search @ 50M users** LinkedIn question fully solved |
| 05 | [Rate Limiting & Abuse Protection](./05-Rate-Limiting-And-Abuse-Protection.md) | Per-tenant rate limit (Bucket4j + Redis), 429 + Retry-After, quotas, per-tenant bulkheads, priority shedding, idempotency keys, kill-switch — with the **"API Went Viral Overnight"** LinkedIn question fully solved |
| 06 | [Read Replica Lag & Consistency](./06-Read-Replica-Lag-And-Consistency.md) | Read-your-own-writes, write-aware routing, LSN-aware reads, synchronous_commit, RYW correctness probes — with the **"Why Does My API Return Stale Data"** LinkedIn question fully solved |

## Index — [Interview Questions (rapid-fire Q&A)](./Interview-Questions/README.md)

86 senior-level questions asked at high-paying companies (Stripe, Razorpay, Uber, FAANG, top fintechs):

| # | Topic | Questions |
|---|-------|-----------|
| 01 | [Spring Core & IoC](./Interview-Questions/01-Spring-Core-And-IoC.md) | 10 |
| 02 | [Spring Boot Internals & Auto-Configuration](./Interview-Questions/02-Spring-Boot-Internals.md) | 8 |
| 03 | [REST APIs, Validation & Exception Handling](./Interview-Questions/03-REST-APIs-Validation.md) | 8 |
| 04 | [JPA, Transactions & Persistence](./Interview-Questions/04-JPA-Transactions-Persistence.md) | 12 |
| 05 | [Spring Security, JWT & OAuth2](./Interview-Questions/05-Spring-Security-JWT-OAuth2.md) | 8 |
| 06 | [Microservices & Spring Cloud](./Interview-Questions/06-Microservices-Spring-Cloud.md) | 8 |
| 07 | [Async, Concurrency & Reactive](./Interview-Questions/07-Async-Concurrency-Reactive.md) | 8 |
| 08 | [Testing Strategies](./Interview-Questions/08-Testing-Strategies.md) | 6 |
| 09 | [Production, Observability & Performance](./Interview-Questions/09-Production-Observability.md) | 8 |
| 10 | [Common Gotchas & Bugs](./Interview-Questions/10-Common-Gotchas-Bugs.md) | 10 |

## How to Use

- Each file is **self-contained** — code + pitfalls + golden rules
- Code is **production-ready** (industry-standard, not toy examples)
- Look for **"INTERVIEW CHEAT CODES"** sections — direct phrases to use in interviews
- Look for **"COMMON MISTAKES"** sections — to avoid traps

## Related Notes

- System Design conceptual coverage → `System Design/md/Notes-Detailed/Microservices-Architecture/03-Resilience-And-Deployment.md`
- Production issues at scale → `System Design/md/Notes-Detailed/Fundamentals/24-Production-Issues-And-Solutions.md`
- Idempotency deep dive → `System Design/md/Notes-Detailed/Fundamentals/22-Idempotent-API-Design.md`
