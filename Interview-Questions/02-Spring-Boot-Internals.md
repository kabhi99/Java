# **SPRING BOOT INTERNALS & AUTO-CONFIGURATION**

8 questions on how Spring Boot actually works under the hood — auto-configuration, starters, externalized config. **Top fintechs (Stripe, Razorpay, Goldman) love these.**

---

### Q01: How does Spring Boot auto-configuration actually work? ⭐⭐

**30-Second Answer:** Spring Boot scans for `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3+) or `spring.factories` (Boot 2.x) in every JAR on the classpath. Each listed `@AutoConfiguration` class is conditionally evaluated using `@ConditionalOn...` annotations. If conditions pass, beans are registered.

**Deep Dive:**

```
1. @SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
                                                          ↓
2. @EnableAutoConfiguration → @Import(AutoConfigurationImportSelector.class)
                                                          ↓
3. AutoConfigurationImportSelector reads:
   META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
                                                          ↓
4. For each listed class (e.g., DataSourceAutoConfiguration):
   - Check @ConditionalOnClass(DataSource.class)        ← is JDBC on classpath?
   - Check @ConditionalOnMissingBean(DataSource.class)  ← already user-defined?
   - Check @ConditionalOnProperty                       ← feature toggle?
   - If ALL pass → register the @Bean methods
```

**Example: `DataSourceAutoConfiguration`**
```java
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration { ... }
```

**Common Follow-Ups:**
- "Where can I see what auto-configuration ran?" → Run with `--debug` flag; shows "Positive matches" and "Negative matches"
- "How do I write a custom auto-configuration?" → Create `@AutoConfiguration` class, list it in `AutoConfiguration.imports` file in your starter JAR

**Red Flag If You Say:** "Spring Boot magically configures everything." (Magic = you don't know how it works.)

---

### Q02: How do you create a custom Spring Boot starter?

**30-Second Answer:** Two modules: a **library JAR** (`x-spring-boot-starter`) with your `@AutoConfiguration` class + `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` listing it, and an **API/SPI JAR** (`x-spring-boot-autoconfigure`) for the actual beans. Consumers just add the starter dependency — beans appear automatically.

**Deep Dive:**

```
my-payment-spring-boot-starter/
├── src/main/java/com/example/PaymentAutoConfiguration.java
├── src/main/java/com/example/PaymentProperties.java
└── src/main/resources/META-INF/spring/
    └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
        ← lists "com.example.PaymentAutoConfiguration"
```

```java
@AutoConfiguration
@ConditionalOnClass(PaymentClient.class)
@EnableConfigurationProperties(PaymentProperties.class)
public class PaymentAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public PaymentClient paymentClient(PaymentProperties props) {
        return new PaymentClient(props.getApiKey(), props.getBaseUrl());
    }
}

@ConfigurationProperties(prefix = "payment")
public record PaymentProperties(String apiKey, String baseUrl) {}
```

Consumer just adds:
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-payment-spring-boot-starter</artifactId>
</dependency>
```
And configures via `application.yml`:
```yaml
payment:
  api-key: ${PAYMENT_KEY}
  base-url: https://api.example.com
```

**Common Follow-Ups:**
- "Why two modules?" → Convention: `-starter` is for dependency aggregation; `-autoconfigure` for the beans. Allows consumers to use beans without forcing auto-config

---

### Q03: Explain `@ConfigurationProperties` vs `@Value`. Which is better and why?

**30-Second Answer:** Use `@ConfigurationProperties` for structured config (POJOs, validated, IDE auto-complete). Use `@Value` for one-off single values. **`@ConfigurationProperties` is preferred** because it supports type-safety, JSR-303 validation, IDE hints, and relaxed binding.

**Deep Dive:**

```java
//  @ConfigurationProperties — type-safe, validated, supports nested objects
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
    @NotBlank String apiKey,
    @Min(1000) int timeoutMs,
    Retry retry
) {
    public record Retry(int maxAttempts, long backoffMs) {}
}

//  @Value — quick but no validation, no IDE hints
@Value("${app.payment.api-key}") private String apiKey;
@Value("${app.payment.timeout-ms:5000}") private int timeoutMs;   // default 5000
```

YAML:
```yaml
app:
  payment:
    api-key: sk_test_xxx
    timeout-ms: 3000
    retry:
      max-attempts: 3
      backoff-ms: 500
```

**Why `@ConfigurationProperties` wins:**
- **Relaxed binding**: `api-key`, `apiKey`, `API_KEY` all map to `apiKey`
- **Validation**: `@Validated` + JSR-303 annotations fail at startup
- **Refreshable** (with Spring Cloud Config `@RefreshScope`)
- **IDE auto-complete** via `spring-boot-configuration-processor`

**Common Follow-Ups:**
- "How do you make a property mandatory?" → `@Validated` + `@NotBlank` / `@NotNull` on the field
- "How does relaxed binding work?" → `api-key` (kebab), `apiKey` (camel), `api_key` (snake), `API_KEY` (upper-env-style) all bind to same field

**Red Flag If You Say:** "Always use `@Value` because it's simpler." (Misses validation, refresh, IDE support.)

---

### Q04: What's the precedence order for Spring Boot configuration sources? ⭐

**30-Second Answer:** Highest priority first:
1. **Command-line args** (`--server.port=8080`)
2. **`SPRING_APPLICATION_JSON`** env var
3. **OS env variables**
4. **System properties** (`-Dserver.port=8080`)
5. **`application-{profile}.yml`** (profile-specific)
6. **`application.yml`** (default)
7. **`@PropertySource`** on `@Configuration` class
8. **Default properties**

**Deep Dive:**

```bash
# All of these set server.port; last command wins via precedence
java -jar app.jar --server.port=9090                       # 1st priority
SERVER_PORT=8080 java -jar app.jar                          # 3rd priority
java -Dserver.port=7070 -jar app.jar                        # 4th priority
# application-prod.yml: server.port=6060                    # 5th priority
# application.yml: server.port=5050                         # 6th priority
```

**Common Follow-Ups:**
- "Where would you put DB password in production?" → Env var or secrets manager (Vault, AWS Secrets Manager), NEVER in `application.yml`
- "What's `application.properties` vs `application.yml`?" → Same precedence; YAML is just easier to read for nested config

---

### Q05: How do Spring Profiles work? How do you organize config for dev/staging/prod?

**30-Second Answer:** Profiles let you load **different beans and config** per environment. Activate via `spring.profiles.active=prod`. Spring loads `application.yml` + `application-{profile}.yml` (profile overrides default). `@Profile("prod")` on a bean → only loaded in that profile.

**Deep Dive:**

```yaml
# application.yml — shared defaults
spring:
  application:
    name: orders-service
  datasource:
    hikari:
      maximum-pool-size: 10

---
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa

---
# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50    # overrides default of 10
```

Activate:
```bash
java -jar app.jar --spring.profiles.active=prod
# or env var: SPRING_PROFILES_ACTIVE=prod
```

Profile-specific beans:
```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean public DataSource dataSource() { return new H2DataSource(); }
}

@Configuration
@Profile({"staging", "prod"})
public class CloudConfig {
    @Bean public DataSource dataSource() { return new HikariDataSource(); }
}
```

**Common Follow-Ups:**
- "What's `spring.profiles.include`?" → Adds profiles in addition to active ones (composability)
- "How do you set the default profile?" → `spring.profiles.default=dev` in `application.yml`

---

### Q06: What's the difference between `@SpringBootApplication` and writing `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` separately?

**30-Second Answer:** **Functionally identical.** `@SpringBootApplication` is a convenience meta-annotation. The only practical difference is that `@SpringBootApplication` lets you **customize parts via attributes** (e.g., `exclude = DataSourceAutoConfiguration.class`).

**Deep Dive:**

```java
//  Equivalent
@SpringBootApplication
public class App { ... }

//  Equivalent (verbose)
@SpringBootConfiguration   // @Configuration variant
@EnableAutoConfiguration
@ComponentScan
public class App { ... }
```

**Customizing:**
```java
//  Exclude specific auto-configs
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class App { ... }

//  Custom scan base packages
@SpringBootApplication(scanBasePackages = {"com.example.core", "com.example.api"})
public class App { ... }
```

**Common Follow-Ups:**
- "Why does the main class location matter?" → `@ComponentScan` defaults to the package of the main class → scans sub-packages only
- "When would you exclude an auto-configuration?" → When you want to provide your own (e.g., custom DataSource setup)

---

### Q07: How does Spring Boot's embedded Tomcat work? Can you switch to Jetty/Undertow?

**30-Second Answer:** `spring-boot-starter-web` includes embedded Tomcat by default. To switch, exclude Tomcat and add the other starter. The `EmbeddedServletContainerFactory` auto-config detects which is on the classpath.

**Deep Dive:**

```xml
<!-- Switch from Tomcat to Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

**When to switch:**
- **Undertow**: faster, lower memory, but smaller community
- **Jetty**: better long-lived connection support
- **Tomcat**: default, most battle-tested

**Common Follow-Ups:**
- "Why embedded?" → Self-contained JAR, easier deployment, no WAR/EAR
- "How do you tune Tomcat's thread pool?" → `server.tomcat.threads.max=200`, `server.tomcat.accept-count=100`

---

### Q08: How do you exclude specific auto-configuration classes?

**30-Second Answer:** Three ways:
1. `@SpringBootApplication(exclude = { X.class })` — at app level
2. `spring.autoconfigure.exclude=com.example.X` — in `application.yml`
3. Conditional check (e.g., remove the JAR or unset the property)

**Deep Dive:**

```java
//  Method 1: Annotation
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,           // don't auto-configure DB
    SecurityAutoConfiguration.class               // don't auto-configure security
})
public class App { ... }
```

```yaml
#  Method 2: Property
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
      - org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration
```

**Common scenario**: You want to use Spring Boot but configure DataSource manually (e.g., multi-tenant) — exclude `DataSourceAutoConfiguration` and define your own bean.

**Common Follow-Ups:**
- "What happens if you don't exclude but provide your own bean?" → `@ConditionalOnMissingBean` usually skips the auto-config — exclude is only needed in edge cases

---

## Top 2 from This File (must-know)

1. **Q01** — How auto-configuration works (asked in ~70% of senior interviews)
2. **Q04** — Configuration source precedence (everyone gets this wrong)

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| Auto-config mechanism             | AutoConfiguration.imports +      |
|                                   |  @ConditionalOn... gates         |
| @SpringBootApplication            | Convenience for 3 annotations    |
| Custom starter                    | Library + @AutoConfiguration +   |
|                                   |  AutoConfiguration.imports       |
| @ConfigurationProperties          | Type-safe, validated, relaxed    |
| Config source precedence          | CLI > env > sys-props > yml      |
| Profiles                          | application-{profile}.yml +       |
|                                   |  spring.profiles.active          |
| Excluding auto-config             | exclude attribute or property    |
| Embedded server                   | Tomcat default, swap via deps    |
+----------------------------------+----------------------------------+
```
