# **SPRING SECURITY, JWT & OAUTH2**

8 questions on authentication and authorization in Spring Boot.

---

### Q01: Walk me through Spring Security's filter chain. ⭐

**30-Second Answer:** Spring Security is implemented as a **chain of servlet filters**. Each incoming request passes through them in order. Key filters: `SecurityContextPersistenceFilter` → `UsernamePasswordAuthenticationFilter` (login) → `BearerTokenAuthenticationFilter` (JWT) → `AuthorizationFilter` (RBAC) → your controllers.

**Deep Dive:**

```
Request →
  ChannelProcessingFilter           (HTTPS enforcement)
  SecurityContextPersistenceFilter  (load SecurityContext from session)
  HeaderWriterFilter                (security headers: HSTS, X-Frame-Options)
  CsrfFilter                        (CSRF token check)
  LogoutFilter
  UsernamePasswordAuthenticationFilter   (form login)
  BearerTokenAuthenticationFilter   (JWT resource server)
  RequestCacheAwareFilter
  SessionManagementFilter
  ExceptionTranslationFilter        (catches AuthenticationException, AccessDeniedException)
  AuthorizationFilter               (check @PreAuthorize, antMatchers)
                                  → Your Controller
```

**Customize the chain (Spring Security 6+):**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                              // stateless API
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/health").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

**Common Follow-Ups:**
- "How is `SecurityContext` propagated?" → `ThreadLocal` via `SecurityContextHolder` — accessible anywhere in same thread
- "Why is this a problem for async?" → `ThreadLocal` doesn't propagate to other threads — need `DelegatingSecurityContextExecutor`

**Red Flag If You Say:** "Spring Security just works." (Then you can't debug 401s/403s.)

---

### Q02: JWT vs Session-based auth — which and when? ⭐

**30-Second Answer:** **JWT** for stateless APIs (microservices, mobile, SPA) — server doesn't need to remember anything. **Sessions** for traditional server-rendered apps — server can invalidate immediately. Key trade-off: JWTs can't be revoked (until expiry); sessions can.

**Deep Dive:**

| Aspect | Session | JWT |
|---|---|---|
| **Where stored** | Server (DB/Redis) + cookie ID | Client (localStorage/cookie) |
| **State** | Stateful | Stateless |
| **Revocation** | Delete from store | Hard (must wait for expiry or blocklist) |
| **Size** | Small (just session ID) | Big (header + claims + signature) |
| **CSRF** | Vulnerable (need CSRF tokens) | Less (if not in cookies) |
| **XSS** | Cookie HttpOnly helps | localStorage = stealable |
| **Scaling** | Need shared store (Redis) | Stateless = trivially scalable |
| **Mobile** | Awkward | Native fit |

**Best of both — refresh token pattern:**
- **Access token (JWT, 15 min)** — short-lived, stateless
- **Refresh token (opaque, 30 days)** — stored server-side, revocable
- On expiry, use refresh token to get new access token

```java
@PostMapping("/auth/login")
public TokenResponse login(@RequestBody LoginRequest req) {
    User u = authService.authenticate(req.username(), req.password());
    String accessToken = jwtService.generateAccess(u);       // 15-min JWT
    String refreshToken = refreshTokenService.create(u);     // opaque, in DB
    return new TokenResponse(accessToken, refreshToken);
}

@PostMapping("/auth/refresh")
public TokenResponse refresh(@RequestBody RefreshRequest req) {
    RefreshToken rt = refreshTokenService.validate(req.token());
    return new TokenResponse(jwtService.generateAccess(rt.getUser()), rt.getToken());
}

@PostMapping("/auth/logout")
public void logout(@RequestBody RefreshRequest req) {
    refreshTokenService.revoke(req.token());   //  invalidates refresh
    // Access token continues to work until expiry (15 min worst case)
}
```

**Common Follow-Ups:**
- "How do you revoke a JWT immediately?" → Maintain a **blocklist** in Redis keyed by `jti` claim; check on every request
- "Where to store JWT on the client?" → `httpOnly + Secure + SameSite=Strict` cookie is safest; localStorage is XSS-vulnerable

---

### Q03: How do you implement RBAC vs ABAC in Spring Security?

**30-Second Answer:** **RBAC** = role-based — check fixed roles (`hasRole('ADMIN')`). **ABAC** = attribute-based — check arbitrary expressions on user + resource (`@PreAuthorize("#order.owner == authentication.name")`). RBAC is simpler; ABAC is more flexible for fine-grained access.

**Deep Dive:**

```java
//  RBAC — declarative roles
@RestController
public class OrderController {
    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/orders/{id}")
    public void delete(@PathVariable Long id) { ... }

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    @GetMapping("/orders/audit")
    public List<Order> audit() { ... }
}

//  ABAC — owner check
@PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")
@GetMapping("/orders/{orderId}")
public OrderDto get(@PathVariable Long orderId) { ... }

@Component
public class OrderSecurity {
    public boolean isOwner(Long orderId, Authentication auth) {
        return orderRepo.findById(orderId)
            .map(o -> o.getUserId().equals(auth.getName()))
            .orElse(false);
    }
}

//  ABAC — combine conditions
@PreAuthorize("hasRole('MANAGER') and #order.amount < 10000")
public void approve(@P("order") Order order) { ... }

//  Method-level enabled in config
@Configuration
@EnableMethodSecurity   // Spring Security 6+
public class SecurityConfig { ... }
```

**Common Follow-Ups:**
- "`@PreAuthorize` vs `@PostAuthorize`?" → Pre runs before method; Post can use return value (`#returnObject`)
- "Difference between `hasRole('ADMIN')` and `hasAuthority('ROLE_ADMIN')`?" → Same; `hasRole` adds `ROLE_` prefix automatically

---

### Q04: How does OAuth2 authorization code flow work?

**30-Second Answer:** **5 steps**: (1) App redirects user to authorization server, (2) user authenticates and consents, (3) auth server redirects back with a **code**, (4) app exchanges code for **access token** (server-to-server), (5) app calls resource server with token.

**Deep Dive:**

```
1. User clicks "Login with Google"
2. App redirects → https://accounts.google.com/oauth?
                    response_type=code&
                    client_id=YOUR_ID&
                    redirect_uri=https://app.com/callback&
                    scope=openid email profile&
                    state=RANDOM_NONCE   ← CSRF protection

3. User authenticates → consents → Google redirects:
   https://app.com/callback?code=ABC123&state=RANDOM_NONCE

4. App (server-side) exchanges code for token:
   POST https://oauth2.googleapis.com/token
   {
     grant_type: authorization_code,
     code: ABC123,
     client_id: ..., client_secret: ...,
     redirect_uri: https://app.com/callback
   }
   ← { access_token, refresh_token, id_token (OIDC) }

5. Use access_token to call resource servers:
   GET https://api.google.com/userinfo
   Authorization: Bearer <access_token>
```

**Spring Boot client setup:**
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, email, profile
        provider:
          google:
            issuer-uri: https://accounts.google.com
```

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2Login(Customizer.withDefaults())   //  enables OAuth2 client flow
            .build();
    }
}
```

**Common Follow-Ups:**
- "Why not Implicit Flow?" → Deprecated (RFC 8252); tokens in URL = security risk
- "What's PKCE?" → Adds code_verifier/code_challenge — mandatory for SPAs and mobile (no client secret)
- "Authorization Code Flow vs Client Credentials?" → Authorization Code = on behalf of user; Client Credentials = service-to-service

---

### Q05: How do you implement password hashing properly?

**30-Second Answer:** Use **BCrypt** (or Argon2). Spring Security provides `BCryptPasswordEncoder` (default strength 10, ~100ms per hash). NEVER use MD5, SHA-256, plain SHA-512. NEVER store plaintext.

**Deep Dive:**

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);   //  strength 12 = ~250ms per hash
    // Higher = slower = better against brute force, but slower login
}

@Service
public class UserService {
    private final PasswordEncoder encoder;

    public void register(String username, String rawPassword) {
        String hash = encoder.encode(rawPassword);   //  bcrypt hash
        userRepo.save(new User(username, hash));
    }

    public boolean checkPassword(User user, String rawPassword) {
        return encoder.matches(rawPassword, user.getPasswordHash());
    }
}
```

**BCrypt format**: `$2a$12$N9qo8uLOickgx2ZMRZoMye...` (algorithm $cost $22-char-salt + 31-char-hash)
- Cost (work factor) is in the hash → can upgrade without forcing re-login
- Salt is built-in → no need to store separately

**Best practice — `DelegatingPasswordEncoder`** (allows upgrading algorithms):
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    //  Encodes with bcrypt; reads any of {bcrypt}, {scrypt}, {argon2}, {sha256}, etc.
    //  Easy migration from old hashes
}
```

**Common Follow-Ups:**
- "Why not SHA-256 with salt?" → Too fast → GPU can brute force billions/sec. BCrypt's iteration count slows it intentionally
- "What if you have a leaked password DB?" → Use Have I Been Pwned API; reject known-compromised passwords

**Red Flag If You Say:** "Use SHA-256 with salt." (Fast hashes are NOT secure for passwords.)

---

### Q06: How do you protect against CSRF? When can you disable it?

**30-Second Answer:** CSRF = malicious site tricks user's browser into making authenticated request to your site. Spring Security adds a CSRF token to forms; server verifies on state-changing requests. **Disable CSRF only for stateless APIs (JWT)** — tokens aren't auto-sent by browser, so CSRF is impossible.

**Deep Dive:**

```java
//  Stateful (browser-session app) — CSRF enabled (default)
@Bean SecurityFilterChain web(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
        .build();
}

//  Stateless API (JWT) — disable CSRF
@Bean SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())   //  Safe because no cookie-based auth
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}
```

**How CSRF attack works:**
```html
<!-- malicious-site.com -->
<form action="https://yourbank.com/transfer" method="POST">
    <input type="hidden" name="to" value="attacker" />
    <input type="hidden" name="amount" value="1000" />
</form>
<script>document.forms[0].submit();</script>
```
If user is logged into yourbank.com (cookie session), browser sends cookie → transfer happens.

**Defense**: Server requires a CSRF token in the form that attacker can't know.

**Common Follow-Ups:**
- "Why doesn't JWT need CSRF?" → JWT in `Authorization: Bearer` header. Browser doesn't auto-send headers cross-origin (only cookies). Attacker can't read your localStorage from their site (CORS)
- "What if JWT is in a cookie?" → Then CSRF matters again. Use `SameSite=Strict` cookie

---

### Q07: How do you implement custom UserDetailsService?

**30-Second Answer:** Implement `UserDetailsService` interface to load user from your DB. Spring Security calls `loadUserByUsername(String)` during authentication. Return `UserDetails` with username, password hash, granted authorities.

**Deep Dive:**

```java
@Service
@RequiredArgsConstructor
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository userRepo;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepo.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPasswordHash())
            .authorities(user.getRoles().stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r.getName()))
                .toList())
            .accountLocked(user.isLocked())
            .disabled(!user.isEnabled())
            .build();
    }
}
```

**Common Follow-Ups:**
- "How does Spring know to use this?" → Auto-detected as a `UserDetailsService` bean
- "How do you cache user lookups?" → `@Cacheable("users")` on `loadUserByUsername` (but invalidate on user updates)

---

### Q08: What is `SecurityContext` and how does it propagate to async threads?

**30-Second Answer:** `SecurityContext` (holds authenticated user) is stored in a `ThreadLocal` via `SecurityContextHolder`. **`ThreadLocal` doesn't propagate to async threads** — you need `DelegatingSecurityContextExecutor`, `DelegatingSecurityContextRunnable`, or set `SecurityContextHolder` strategy to `MODE_INHERITABLETHREADLOCAL`.

**Deep Dive:**

```java
//  Bug — Security context lost in async
@Service
public class OrderService {
    @Async
    public CompletableFuture<Order> processAsync(Long id) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        //  auth is NULL — different thread, different ThreadLocal
    }
}

//  Fix 1: DelegatingSecurityContextExecutor (wrap your executor)
@Bean
public Executor asyncExecutor() {
    ThreadPoolTaskExecutor base = new ThreadPoolTaskExecutor();
    base.initialize();
    return new DelegatingSecurityContextExecutor(base);   //  copies context
}

//  Fix 2: Inheritable ThreadLocal (global change)
@PostConstruct
public void init() {
    SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL);
    //  Child threads inherit context — but careful with thread pools (stale context)
}

//  Fix 3: Manually copy
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
CompletableFuture.runAsync(() -> {
    SecurityContextHolder.getContext().setAuthentication(auth);
    try { doWork(); }
    finally { SecurityContextHolder.clearContext(); }
});
```

**Common Follow-Ups:**
- "Why is `MODE_INHERITABLETHREADLOCAL` dangerous with thread pools?" → Thread pool reuses threads — child inherits the FIRST request's context, then subsequent requests use a stale context
- "Reactive (WebFlux) version?" → Use `ReactiveSecurityContextHolder` — propagates via reactive context, not ThreadLocal

---

## Top 3 from This File (must-know)

1. **Q02** — JWT vs Session (always asked)
2. **Q03** — RBAC vs ABAC (`@PreAuthorize`)
3. **Q08** — SecurityContext propagation to async (catches juniors)

## Quick Reference Card

```
+----------------------------------+----------------------------------+
| Concept                           | Most Important Thing             |
+----------------------------------+----------------------------------+
| Filter chain                      | Servlet filters, in order        |
| JWT                               | Stateless, can't revoke immediately|
| Refresh token                     | Server-side, revocable           |
| RBAC                              | @PreAuthorize("hasRole('X')")    |
| ABAC                              | @PreAuthorize("#x == auth.name") |
| OAuth2 Auth Code Flow             | Redirect → code → token exchange |
| PKCE                              | Required for SPAs / mobile       |
| BCrypt                            | Slow hash; built-in salt         |
| DelegatingPasswordEncoder         | Migrate algorithms safely        |
| CSRF                              | Disable ONLY for stateless API   |
| UserDetailsService                | Custom user loading              |
| SecurityContext + Async           | ThreadLocal doesn't propagate    |
+----------------------------------+----------------------------------+
```
