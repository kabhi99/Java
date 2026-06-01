# **SPRING CORE & IoC — INTERVIEW QUESTIONS**

10 questions on dependency injection, bean lifecycle, scopes, and the IoC container — the foundation EVERY Spring interview tests.

---

### Q01: What's the actual difference between `@Component`, `@Service`, `@Repository`, `@Controller`? ⭐

**30-Second Answer:** All four register the class as a Spring bean — technically interchangeable. The differences are semantic + framework behavior:
- `@Repository` adds **exception translation** (database exceptions → Spring's `DataAccessException`)
- `@Controller` enables **MVC handler mapping**
- `@Service` is purely a marker for business logic
- `@Component` is the generic root annotation

**Deep Dive:**
```java
@Repository  // Translates SQLException → DataAccessException via PersistenceExceptionTranslationPostProcessor
public class UserRepository { ... }

@Controller  // Registers as request handler; needs @RequestMapping methods
public class UserController { ... }

@Service     // No behavior change — semantic marker for business logic
public class UserService { ... }

@Component   // Generic; use when nothing else fits
public class UserMapper { ... }
```

**Common Follow-Ups:**
- "If I replace `@Repository` with `@Component`, will JPA still work?" → Yes, but you lose exception translation
- "Where is exception translation applied?" → `PersistenceExceptionTranslationPostProcessor`

**Red Flag If You Say:** "There's no difference, they're aliases." (Misses the exception translation behavior.)

---

### Q02: Constructor injection vs field injection vs setter injection — which and why? ⭐

**30-Second Answer:** **Always use constructor injection.** Reasons:
1. **Immutability** — fields can be `final`
2. **Required by contract** — class won't compile without dependencies
3. **Easier to test** — no reflection, no Spring needed in unit tests
4. **Catches circular dependencies at startup** instead of at runtime
5. **Lombok `@RequiredArgsConstructor`** makes it as concise as field injection

**Deep Dive:**
```java
//  Constructor injection (preferred)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentClient paymentClient;
    private final OrderRepository orderRepo;
}

//  Field injection
@Service
public class OrderService {
    @Autowired private PaymentClient paymentClient;     // can't be final
    @Autowired private OrderRepository orderRepo;        // hidden dependencies
}
// Problems:
// - Can't unit test without Spring or reflection
// - Circular deps fail at runtime, not startup
// - NullPointerException if used before context is fully initialized
```

**Common Follow-Ups:**
- "What's the difference if there's only one constructor?" → `@Autowired` is optional since Spring 4.3
- "When would you use setter injection?" → Optional dependencies that can be reassigned at runtime

**Red Flag If You Say:** "Field injection is cleaner because no boilerplate." (Shows you haven't suffered through circular deps or testing pain.)

---

### Q03: Explain Spring bean scopes. Which are thread-safe? ⭐

**30-Second Answer:** 6 scopes: `singleton` (default), `prototype`, `request`, `session`, `application`, `websocket`. **Only `prototype` is "thread-isolated"** — but the others ARE shared across threads, so you must make singletons stateless or synchronize state. `singleton` is NOT inherently thread-safe.

**Deep Dive:**
| Scope | Lifetime | Thread Safety |
|---|---|---|
| `singleton` (default) | One per Spring container | Container creates one instance shared by all threads — you handle thread safety |
| `prototype` | New instance per `getBean()` call | Each caller has its own copy — but Spring won't manage destruction |
| `request` | One per HTTP request | Per-thread (request thread) — safe within request |
| `session` | One per HTTP session | Shared if user has parallel requests — needs sync |
| `application` | One per `ServletContext` | Same as singleton, but across all Spring contexts in the app |

**The Singleton Trap:**
```java
@Service
public class CounterService {
    private int count = 0;       //  Mutable state in singleton — race condition!
    public void increment() { count++; }
}
// Fix: AtomicInteger, or scope = "prototype", or make stateless
```

**Common Follow-Ups:**
- "What happens when you inject a prototype bean into a singleton?" → Singleton holds reference to ONE prototype instance forever (defeats the purpose). Use `ObjectFactory<T>` or `@Lookup` method
- "Difference between application scope and singleton scope?" → Same in standalone Spring, different in multi-context web apps

**Red Flag If You Say:** "Singletons are thread-safe by default." (False — Spring guarantees one instance, not thread safety.)

---

### Q04: How does Spring resolve circular dependencies?

**30-Second Answer:** Spring resolves circular deps **only for setter/field injection in singleton beans** via early bean references (3-level cache). **Constructor injection circular deps fail at startup** — and that's a good thing. The fix is to break the cycle (refactor) or use `@Lazy` on one dependency.

**Deep Dive:**

```java
//  Circular dep with field injection — Spring SILENTLY allows this (bad)
@Service class A { @Autowired B b; }
@Service class B { @Autowired A a; }

//  Same with constructor — FAILS at startup with BeanCurrentlyInCreationException
@Service @RequiredArgsConstructor class A { private final B b; }
@Service @RequiredArgsConstructor class B { private final A a; }
```

**Spring's 3-level cache (for singletons):**
1. `singletonObjects` — fully initialized
2. `earlySingletonObjects` — instantiated but not populated (for refs)
3. `singletonFactories` — factory for creating early ref

**Real fix**: Break the cycle. Introduce a 3rd bean, use events, or extract shared interface.

**Quick fix**: `@Lazy` on one side:
```java
@Service @RequiredArgsConstructor class A { private final B b; }
@Service @RequiredArgsConstructor class B {
    private final A a;
    public B(@Lazy A a) { this.a = a; }  // proxy injected, real bean fetched on first use
}
```

**Common Follow-Ups:**
- "Why is Spring Boot 2.6+ stricter about circular deps?" → They disabled allowed-circular-references by default
- "What does `spring.main.allow-circular-references=true` do?" → Re-enables the bad behavior

**Red Flag If You Say:** "Just add `@Lazy` everywhere." (Should refactor, not paper over.)

---

### Q05: Walk me through the Spring bean lifecycle. ⭐

**30-Second Answer:** Instantiation → populate properties (DI) → `BeanNameAware`/`BeanFactoryAware`/`ApplicationContextAware` callbacks → `BeanPostProcessor.postProcessBeforeInitialization` → `@PostConstruct` / `InitializingBean.afterPropertiesSet` / custom init → `BeanPostProcessor.postProcessAfterInitialization` → bean ready → on shutdown: `@PreDestroy` / `DisposableBean.destroy` / custom destroy.

**Deep Dive:**

```java
@Component
public class MyBean implements InitializingBean, DisposableBean {

    public MyBean() { /* 1. Instantiate */ }

    @Autowired
    public void setDeps(SomeDep d) { /* 2. Populate properties */ }

    @PostConstruct
    public void init() { /* 5. After all DI + post-processing */ }

    @Override
    public void afterPropertiesSet() { /* 6. Alternative to @PostConstruct */ }

    @PreDestroy
    public void cleanup() { /* On shutdown */ }

    @Override
    public void destroy() { /* Alternative to @PreDestroy */ }
}
```

**Why this matters**: Order of `BeanPostProcessor` is how Spring injects AOP proxies, validates configs, sets up `@Transactional`, etc.

**Common Follow-Ups:**
- "What's the difference between `@PostConstruct` and `InitializingBean.afterPropertiesSet()`?" → Same purpose; `@PostConstruct` is JSR-250 standard, preferred
- "When is `@PostConstruct` NOT called?" → On prototype beans destroyed by Spring (Spring doesn't track them)

**Red Flag If You Say:** "Constructor is where you initialize stuff." (You can't access injected deps in constructor — use `@PostConstruct`.)

---

### Q06: How does Spring's `@Autowired` actually resolve dependencies?

**30-Second Answer:** `@Autowired` resolves by **type first**. If multiple beans match, it uses **`@Qualifier`** or **`@Primary`**. If still ambiguous, it falls back to **bean name == field name**. Fails with `NoUniqueBeanDefinitionException` if unresolved.

**Deep Dive:**

```java
public interface PaymentGateway { ... }

@Component class StripeGateway implements PaymentGateway { ... }
@Component class PayPalGateway implements PaymentGateway { ... }

@Service
public class OrderService {
    @Autowired
    private PaymentGateway gateway;
    //  Fails: NoUniqueBeanDefinitionException
}

// Fix 1: @Primary
@Component @Primary class StripeGateway implements PaymentGateway { ... }

// Fix 2: @Qualifier
@Service
public class OrderService {
    public OrderService(@Qualifier("stripeGateway") PaymentGateway g) { ... }
}

// Fix 3: Inject all and pick
@Service
public class OrderService {
    private final Map<String, PaymentGateway> gateways;  // bean name → bean
    public OrderService(Map<String, PaymentGateway> gateways) { this.gateways = gateways; }
}
```

**Common Follow-Ups:**
- "What's the difference between `@Autowired`, `@Inject`, `@Resource`?" → `@Inject` is JSR-330 (portable); `@Resource` (JSR-250) resolves by NAME first, then type
- "What does `required = false` do?" → Optional dependency; sets `null` if no bean found

**Red Flag If You Say:** "Autowired matches by name." (It's type-first, name only as tiebreaker.)

---

### Q07: How do you create a bean conditionally?

**30-Second Answer:** Use `@ConditionalOn...` annotations. Most-used:
- `@ConditionalOnProperty(name = "feature.x.enabled", havingValue = "true")`
- `@ConditionalOnMissingBean` — register only if no other bean of this type
- `@ConditionalOnClass` — only if class is on classpath (auto-config staple)
- `@Profile("dev")` — environment-based

**Deep Dive:**

```java
@Configuration
public class CacheConfig {

    // Only create RedisCache if Redis is on classpath AND not already overridden
    @Bean
    @ConditionalOnClass(name = "io.lettuce.core.RedisClient")
    @ConditionalOnMissingBean(CacheManager.class)
    @ConditionalOnProperty(name = "cache.type", havingValue = "redis")
    public CacheManager redisCacheManager() { ... }

    // Fallback: in-memory cache
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() { ... }
}
```

**Common Follow-Ups:**
- "How does Spring Boot's auto-config use these?" → Every auto-config class is `@Configuration` + `@ConditionalOn...`; e.g., `DataSourceAutoConfiguration` only fires if `DataSource` class is on classpath
- "Difference between `@Profile` and `@ConditionalOnProperty`?" → `@Profile` is for environment (dev/prod); `@ConditionalOnProperty` is for feature flags

---

### Q08: What's the difference between `@Configuration` and `@Component`?

**30-Second Answer:** Both register the class as a bean. **`@Configuration` enables `@Bean` method interception** via CGLIB proxy — calling `@Bean` methods from within the config class returns the **same singleton**. With `@Component`, calling a `@Bean` method gives you a **new instance each time** (Spring doesn't know about it).

**Deep Dive:**

```java
@Configuration  //  Spring wraps this in CGLIB proxy
public class AppConfig {
    @Bean public ServiceA serviceA() { return new ServiceA(serviceB()); }   // returns singleton B
    @Bean public ServiceB serviceB() { return new ServiceB(); }
}

@Component  //  No proxy
public class AppConfig {
    @Bean public ServiceA serviceA() { return new ServiceA(serviceB()); }   // returns NEW B every time!
    @Bean public ServiceB serviceB() { return new ServiceB(); }
}
```

**Use `proxyBeanMethods = false` (Spring Boot 2.2+)** for slightly faster startup if your config doesn't call its own `@Bean` methods:

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig { ... }
```

**Common Follow-Ups:**
- "Why does Spring Boot use `proxyBeanMethods = false` in many auto-configs?" → Performance — avoids CGLIB proxy creation overhead

**Red Flag If You Say:** "`@Configuration` is just a marker." (Misses the CGLIB interception, which is the whole point.)

---

### Q09: What is `BeanFactory` vs `ApplicationContext`?

**30-Second Answer:** `BeanFactory` is the basic IoC container — lazy initialization, no extras. `ApplicationContext` extends it with: **eager singleton initialization, event publishing, internationalization, AOP, auto-wiring of beans, environment abstraction**. Always use `ApplicationContext` in real apps.

**Deep Dive:**

| Feature | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Bean instantiation | Lazy by default | Eager (singletons created at startup) |
| Event publishing | No | Yes (`ApplicationEventPublisher`) |
| i18n | No | Yes (`MessageSource`) |
| AOP | Manual | Auto-configured |
| BeanPostProcessor auto-registration | No | Yes |

**Common Follow-Ups:**
- "When would you use `BeanFactory` directly?" → Almost never. Memory-constrained environments only (mobile, embedded)

---

### Q10: What's a `BeanPostProcessor`? Give a real-world use.

**30-Second Answer:** A hook that runs **before and after** every bean's initialization. Spring uses it to inject AOP proxies, validate configs, set up `@Autowired`, `@PostConstruct`, etc. Custom uses: auditing, dynamic decoration, security checks.

**Deep Dive:**

```java
@Component
public class TimingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) {
        // Called BEFORE @PostConstruct
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        // Called AFTER @PostConstruct
        // Return a proxy here to wrap the bean
        if (bean.getClass().isAnnotationPresent(Timed.class)) {
            return Proxy.newProxyInstance(...);   // wrap with timing logic
        }
        return bean;
    }
}
```

**Real-world examples:**
- `AutowiredAnnotationBeanPostProcessor` → resolves `@Autowired`
- `CommonAnnotationBeanPostProcessor` → resolves `@PostConstruct`, `@PreDestroy`
- `AnnotationAwareAspectJAutoProxyCreator` → wraps beans with AOP proxies (`@Transactional`, `@Async`)

**Common Follow-Ups:**
- "Why is `BeanPostProcessor` the secret sauce of Spring?" → Almost every Spring annotation is implemented via one

**Red Flag If You Say:** "Never heard of it." (Then you don't really understand how Spring works under the hood.)

---

## Top 3 from This File (must-know)

1. **Q02** — Constructor injection (asked in ~80% of interviews)
2. **Q03** — Bean scopes + thread safety (singleton trap)
3. **Q05** — Bean lifecycle (foundational)

## Quick Reference Card

```
+-------------------------+-------------------------------------+
| Concept                 | Most Important Thing                |
+-------------------------+-------------------------------------+
| @Service vs @Repository | Repo adds exception translation     |
| Constructor injection   | Default — final fields, no Spring   |
|                         | needed for unit tests               |
| Singleton scope         | NOT thread-safe — you handle state  |
| Prototype scope         | New instance per getBean()          |
| Circular dep            | Constructor → startup fail (good)   |
|                         | Setter/field → silently allowed     |
| @Configuration          | CGLIB proxy → singleton @Bean calls |
| @Component              | No proxy → fresh instance per call  |
| @ConditionalOn...       | Powers Spring Boot auto-config      |
| BeanPostProcessor       | The hook behind every annotation    |
+-------------------------+-------------------------------------+
```
