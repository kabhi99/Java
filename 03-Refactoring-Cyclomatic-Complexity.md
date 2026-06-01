# **REFACTORING CYCLOMATIC COMPLEXITY — CLEAN CODE PATTERNS**

How to refactor "if-else hell" / SonarQube-flagged methods into clean, testable, extensible code.

### TABLE OF CONTENTS

1. What is Cyclomatic Complexity & Why It Matters
2. The 3 Universal Techniques (Guard Clauses, Polymorphism, Extract Method)
3. **The 4 Core Patterns** (Chain of Responsibility, Strategy, Specification, Rule Engine)
4. **FEATURED PROBLEM: `processPayment()` refactor** (LinkedIn Q — Stripe-like payment API)
5. Similar Problem Patterns (auth, pricing, eligibility, workflow)
6. When NOT to over-engineer
7. Golden Rules & Interview Cheat Codes

---

## PART 1: WHAT IS CYCLOMATIC COMPLEXITY?

 **DEFINITION**: Count of **independent paths** through your code = number of `if`, `else if`, `&&`, `||`, `case`, `catch`, `?:` + 1.

```java
public PaymentResult process(Request r) {        // base: 1
    if (r == null) return error("INVALID");      // +1 = 2
    if (r.amount <= 0) return error("AMOUNT");   // +1 = 3
    if (r.currency.equals("USD")) {              // +1 = 4
        if (r.isInternational) {                 // +1 = 5
            if (!r.kycVerified) return error("KYC");  // +1 = 6
        } else if (r.amount > 10_000) {          // +1 = 7
            return error("LIMIT");
        }
    } else if (r.currency.equals("EUR")) {       // +1 = 8
        if (r.isHighRisk) return error("BLOCKED"); // +1 = 9
    } else return error("UNSUPPORTED");          // +1 = 10
    if (r.fraudSuspected) return error("FRAUD"); // +1 = 11
    return gateway.charge(r);
}
// CC = 11  → SonarQube fails the build (default threshold: 10)
```

### Why high CC is bad:
| Pain | Impact |
|---|---|
| Hard to test | Need 11 test cases just for branch coverage |
| Hard to read | Reviewer holds 11 paths in head |
| Hard to change | Adding currency = touching this method again |
| Bug magnet | Easy to break in one branch while fixing another |
| Violates **OCP** (Open/Closed) | Changing existing code instead of extending |

### Industry thresholds:
- **SonarQube default**: 10 per method
- **Google Java Style**: ~10
- **Aim for**: ≤ 5 per method

---

## PART 2: THE 3 UNIVERSAL TECHNIQUES (apply BEFORE patterns)

### **TECHNIQUE 1: Guard Clauses (Early Return)**

Replace **nested ifs** with **flat early returns**.

```java
//  BEFORE — arrow code (nested)
public PaymentResult process(Request r) {
    if (r != null) {
        if (r.amount > 0) {
            if (isValid(r)) {
                return charge(r);
            } else { return error("INVALID"); }
        } else { return error("AMOUNT"); }
    } else { return error("NULL"); }
}

//  AFTER — guard clauses (flat)
public PaymentResult process(Request r) {
    if (r == null)        return error("NULL");
    if (r.amount <= 0)    return error("AMOUNT");
    if (!isValid(r))      return error("INVALID");
    return charge(r);
}
```

> **MEMORY AID**: "Fail fast, succeed last."

### **TECHNIQUE 2: Replace Conditional with Polymorphism**

Switch / if-else on a type → use polymorphism.

```java
//  BEFORE
double getDiscount(Customer c) {
    if (c.type.equals("PREMIUM")) return 0.20;
    if (c.type.equals("GOLD"))    return 0.15;
    if (c.type.equals("SILVER"))  return 0.10;
    return 0.0;
}

//  AFTER — each type knows its own discount
interface Customer { double discount(); }
class PremiumCustomer implements Customer { public double discount() { return 0.20; } }
class GoldCustomer    implements Customer { public double discount() { return 0.15; } }
class SilverCustomer  implements Customer { public double discount() { return 0.10; } }

double getDiscount(Customer c) { return c.discount(); }   // 1 line, CC=1
```

### **TECHNIQUE 3: Extract Method (Name the Intent)**

Hide complexity behind a well-named method.

```java
//  BEFORE — what does this even check?
if (user.age >= 18 && user.country.equals("US") && user.verified && !user.banned) { ... }

//  AFTER — intent is clear
if (canMakePayment(user)) { ... }

private boolean canMakePayment(User u) {
    return u.age >= 18 && u.country.equals("US") && u.verified && !u.banned;
}
```

---

## PART 3: THE 4 CORE PATTERNS

### **PATTERN 1: CHAIN OF RESPONSIBILITY** (sequential pipeline)

 **USE WHEN**: A sequence of checks/validators, each can short-circuit.
 **EXAMPLES**: Validation pipeline, middleware/filters, approval flow.

```java
@FunctionalInterface
interface PaymentValidator {
    Optional<PaymentResult> validate(PaymentRequest r);   // empty = pass, present = fail
}

@Component
public class PaymentValidationChain {

    private final List<PaymentValidator> validators;

    public PaymentValidationChain(List<PaymentValidator> validators) {
        this.validators = validators;     // Spring auto-injects all beans in order
    }

    public Optional<PaymentResult> validate(PaymentRequest r) {
        return validators.stream()
            .map(v -> v.validate(r))
            .filter(Optional::isPresent)
            .findFirst()                  // fail-fast on first failure
            .orElse(Optional.empty());
    }
}

// Each validator = a small, single-purpose class
@Component @Order(1)
class NullCheckValidator implements PaymentValidator {
    public Optional<PaymentResult> validate(PaymentRequest r) {
        return r == null
            ? Optional.of(PaymentResult.error("INVALID_REQUEST"))
            : Optional.empty();
    }
}

@Component @Order(2)
class AmountValidator implements PaymentValidator { ... }

@Component @Order(3)
class FraudValidator implements PaymentValidator { ... }
```

**Benefits:**
- Each validator CC = 1-2
- Add new check = add a new class (Open/Closed!)
- Each validator trivially unit-testable
- Order is explicit (`@Order`)

### **PATTERN 2: STRATEGY + FACTORY** (different logic per type)

 **USE WHEN**: Different algorithm per "type" / "category" / "discriminator".
 **EXAMPLES**: Per-currency rules, per-country logic, payment method, shipping rate.

```java
interface CurrencyRule {
    String currency();                              // discriminator
    Optional<PaymentResult> validate(PaymentRequest r);
}

@Component
class UsdRule implements CurrencyRule {
    public String currency() { return "USD"; }
    public Optional<PaymentResult> validate(PaymentRequest r) {
        if (r.isInternational() && !r.isKycVerified())
            return Optional.of(PaymentResult.error("KYC_REQUIRED"));
        if (!r.isInternational() && r.getAmount() > 10_000)
            return Optional.of(PaymentResult.error("LIMIT_EXCEEDED"));
        return Optional.empty();
    }
}

@Component
class EurRule implements CurrencyRule {
    public String currency() { return "EUR"; }
    public Optional<PaymentResult> validate(PaymentRequest r) {
        return r.isHighRiskCountry()
            ? Optional.of(PaymentResult.error("BLOCKED_COUNTRY"))
            : Optional.empty();
    }
}

// FACTORY — picks the right strategy
@Component
public class CurrencyRuleFactory {
    private final Map<String, CurrencyRule> rules;

    public CurrencyRuleFactory(List<CurrencyRule> all) {
        this.rules = all.stream()
            .collect(Collectors.toMap(CurrencyRule::currency, Function.identity()));
    }

    public CurrencyRule forCurrency(String c) {
        CurrencyRule r = rules.get(c);
        if (r == null) throw new UnsupportedCurrencyException(c);
        return r;
    }
}
```

**Adding new currency = drop in a new `@Component` class. ZERO changes to existing code.**

### **PATTERN 3: SPECIFICATION** (composable boolean rules)

 **USE WHEN**: Business rules combine with AND/OR/NOT and need to be reusable.
 **EXAMPLES**: Eligibility, filtering, access control.

```java
@FunctionalInterface
interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return t -> this.isSatisfiedBy(t) && other.isSatisfiedBy(t);
    }
    default Specification<T> or(Specification<T> other) {
        return t -> this.isSatisfiedBy(t) || other.isSatisfiedBy(t);
    }
    default Specification<T> not() {
        return t -> !this.isSatisfiedBy(t);
    }
}

// Define rules as small specs
Specification<PaymentRequest> isUSD          = r -> "USD".equals(r.getCurrency());
Specification<PaymentRequest> isInternational = PaymentRequest::isInternational;
Specification<PaymentRequest> isKycVerified  = PaymentRequest::isKycVerified;

// Compose
Specification<PaymentRequest> needsKyc = isUSD.and(isInternational).and(isKycVerified.not());

if (needsKyc.isSatisfiedBy(req)) return error("KYC_REQUIRED");
```

### **PATTERN 4: RULE ENGINE / DECISION TABLE** (data-driven rules)

 **USE WHEN**: Rules change often, business wants to edit without redeploying, or you have 50+ rules.
 **EXAMPLES**: Pricing engine, eligibility rules, fraud scoring, insurance underwriting.

```java
// Define rules as DATA, not code
public record PaymentRule(
    Predicate<PaymentRequest> condition,
    String errorCode,
    String description
) {}

@Component
public class PaymentRuleEngine {
    private final List<PaymentRule> rules = List.of(
        new PaymentRule(r -> r == null,           "INVALID_REQUEST", "Null request"),
        new PaymentRule(r -> r.getAmount() <= 0,  "INVALID_AMOUNT",  "Non-positive amount"),
        new PaymentRule(r -> r.isFraudSuspected(),"FRAUD_DETECTED",  "Fraud check failed"),
        new PaymentRule(r -> "USD".equals(r.getCurrency()) && r.isInternational() && !r.isKycVerified(),
                            "KYC_REQUIRED",    "USD intl needs KYC"),
        new PaymentRule(r -> "USD".equals(r.getCurrency()) && !r.isInternational() && r.getAmount() > 10_000,
                            "LIMIT_EXCEEDED",  "USD domestic limit"),
        new PaymentRule(r -> "EUR".equals(r.getCurrency()) && r.isHighRiskCountry(),
                            "BLOCKED_COUNTRY", "EUR high-risk")
    );

    public Optional<PaymentResult> evaluate(PaymentRequest r) {
        return rules.stream()
            .filter(rule -> rule.condition().test(r))
            .findFirst()
            .map(rule -> PaymentResult.error(rule.errorCode()));
    }
}
```

**Take it further**: load rules from DB/YAML so business can edit without code change. Or use a real rule engine (Drools, Easy Rules).

---

## PART 4: FEATURED — `processPayment()` REFACTOR

### **THE PROBLEM** (LinkedIn interview question)

> Stripe-like Payments platform. `processPayment()` started simple, but over 2 years:
> - Compliance rules added
> - Fraud checks bolted on
> - Region-specific logic crept in
> - Limits added by product teams
>
> SonarQube now blocks merges — **cyclomatic complexity = 11+** (threshold 10).

### **ORIGINAL CODE (the mess)**

```java
public PaymentResult processPayment(PaymentRequest request) {
    if (request == null) return PaymentResult.error("INVALID_REQUEST");
    if (request.getAmount() <= 0) return PaymentResult.error("INVALID_AMOUNT");

    if (request.getCurrency().equals("USD")) {
        if (request.isInternational()) {
            if (!request.isKycVerified()) return PaymentResult.error("KYC_REQUIRED");
        } else {
            if (request.getAmount() > 10_000) return PaymentResult.error("LIMIT_EXCEEDED");
        }
    } else if (request.getCurrency().equals("EUR")) {
        if (request.isHighRiskCountry()) return PaymentResult.error("BLOCKED_COUNTRY");
    } else {
        return PaymentResult.error("UNSUPPORTED_CURRENCY");
    }

    if (request.isFraudSuspected()) return PaymentResult.error("FRAUD_DETECTED");

    return paymentGateway.charge(request);
}
```

### **REFACTORED — Chain of Responsibility + Strategy (best of both)**

#### Step 1: Universal validators (apply to ALL payments)

```java
@FunctionalInterface
interface PaymentValidator {
    Optional<PaymentResult> validate(PaymentRequest r);
}

@Component @Order(1)
class NullCheckValidator implements PaymentValidator {
    public Optional<PaymentResult> validate(PaymentRequest r) {
        return r == null
            ? Optional.of(PaymentResult.error("INVALID_REQUEST"))
            : Optional.empty();
    }
}

@Component @Order(2)
class AmountValidator implements PaymentValidator {
    public Optional<PaymentResult> validate(PaymentRequest r) {
        return r.getAmount() <= 0
            ? Optional.of(PaymentResult.error("INVALID_AMOUNT"))
            : Optional.empty();
    }
}

@Component @Order(99)   // run last
class FraudValidator implements PaymentValidator {
    public Optional<PaymentResult> validate(PaymentRequest r) {
        return r.isFraudSuspected()
            ? Optional.of(PaymentResult.error("FRAUD_DETECTED"))
            : Optional.empty();
    }
}
```

#### Step 2: Currency-specific strategies

```java
interface CurrencyRule {
    String currency();
    Optional<PaymentResult> validate(PaymentRequest r);
}

@Component
class UsdRule implements CurrencyRule {
    public String currency() { return "USD"; }
    public Optional<PaymentResult> validate(PaymentRequest r) {
        if (r.isInternational() && !r.isKycVerified())
            return Optional.of(PaymentResult.error("KYC_REQUIRED"));
        if (!r.isInternational() && r.getAmount() > 10_000)
            return Optional.of(PaymentResult.error("LIMIT_EXCEEDED"));
        return Optional.empty();
    }
}

@Component
class EurRule implements CurrencyRule {
    public String currency() { return "EUR"; }
    public Optional<PaymentResult> validate(PaymentRequest r) {
        return r.isHighRiskCountry()
            ? Optional.of(PaymentResult.error("BLOCKED_COUNTRY"))
            : Optional.empty();
    }
}

@Component
public class CurrencyRuleFactory {
    private final Map<String, CurrencyRule> rules;
    public CurrencyRuleFactory(List<CurrencyRule> all) {
        rules = all.stream().collect(Collectors.toMap(CurrencyRule::currency, r -> r));
    }
    public Optional<CurrencyRule> forCurrency(String c) {
        return Optional.ofNullable(rules.get(c));
    }
}
```

#### Step 3: Orchestrator — now trivially simple (CC = 3)

```java
@Service
public class PaymentService {

    private final List<PaymentValidator> universalValidators;
    private final CurrencyRuleFactory currencyFactory;
    private final PaymentGateway gateway;

    public PaymentResult processPayment(PaymentRequest request) {
        // 1. Universal checks (null, amount, fraud)
        Optional<PaymentResult> failure = runChain(universalValidators, request);
        if (failure.isPresent()) return failure.get();

        // 2. Currency-specific checks
        Optional<CurrencyRule> rule = currencyFactory.forCurrency(request.getCurrency());
        if (rule.isEmpty()) return PaymentResult.error("UNSUPPORTED_CURRENCY");

        Optional<PaymentResult> currencyFailure = rule.get().validate(request);
        if (currencyFailure.isPresent()) return currencyFailure.get();

        // 3. All checks passed — charge
        return gateway.charge(request);
    }

    private Optional<PaymentResult> runChain(List<PaymentValidator> chain, PaymentRequest r) {
        return chain.stream()
            .map(v -> v.validate(r))
            .filter(Optional::isPresent)
            .findFirst()
            .orElse(Optional.empty());
    }
}
```

### **WHY THIS IS BETTER**

| Metric | Before | After |
|---|---|---|
| Cyclomatic complexity (main method) | **11** | **3** |
| Add new currency | Edit `processPayment()` | Add 1 new class |
| Add new validator | Edit `processPayment()` | Add 1 new class |
| Unit-testable in isolation | No (must mock everything) | Yes (each class tested alone) |
| Open/Closed Principle | Violated | Satisfied |
| Single Responsibility | Violated | Satisfied |
| Risk when changing one rule | High (whole method changes) | Low (only one class changes) |

### **Interview narrative (use this!)**

> "I'd refactor using two patterns: **Chain of Responsibility** for the universal validation pipeline (null, amount, fraud), and **Strategy + Factory** for currency-specific rules. The orchestrator collapses to ~10 lines with CC=3. Adding a new currency or validator becomes adding one Spring `@Component` — the orchestrator is never touched. This satisfies SOLID's Open/Closed Principle and dramatically improves testability — each rule is now its own unit test."

---

## PART 5: SIMILAR PROBLEMS & WHICH PATTERN TO USE

| Problem | Pattern | Why |
|---|---|---|
| Authorization (RBAC/ABAC) | **Specification** | Rules compose with AND/OR/NOT |
| HTTP middleware/filters | **Chain of Responsibility** | Sequential, short-circuiting |
| Pricing engine | **Strategy + Rule Engine** | Per-product-type + data-driven |
| Shipping rate calculation | **Strategy** | Per-carrier algorithm |
| Tax calculation | **Strategy + Factory** | Per-country logic |
| Eligibility check | **Specification** | Composable booleans |
| Multi-step approval | **Chain of Responsibility** | Sequential approvers |
| Discount stacking | **Strategy** + chain | Apply discounts in order |
| Fraud scoring | **Rule Engine** | 50+ rules, business edits |
| Form validation | **Chain of Responsibility** | Field-by-field |
| Payment routing | **Strategy + Factory** | Route by type, amount, country |
| Notification dispatch (email/SMS/push) | **Strategy + Factory** | Per-channel logic |

---

## PART 6: WHEN NOT TO OVER-ENGINEER

> "Just because you CAN use a pattern doesn't mean you SHOULD."

**STAY WITH SIMPLE `if-else` WHEN:**
- Total CC ≤ 5
- Logic is **truly** static (won't change for 6+ months)
- Only 1-2 branches
- Team is small and code is local

**REACH FOR PATTERNS WHEN:**
- CC > 10 (SonarQube territory)
- ≥ 3 branches per dimension (e.g., 3+ currencies, 3+ user types)
- Rules expected to grow (new currency, new region, new check)
- Multiple teams need to add rules without stepping on each other
- Need to test each rule in isolation

> **Rule of thumb**: Refactor on the **third** occurrence of a similar pattern. (Rule of Three.)

---

## PART 7: GOLDEN RULES

```
+----------------------------------------------------------------------+
|  1. CC > 10 = refactor; aim for ≤ 5 per method                       |
|  2. Guard clauses first — flatten nesting before adding patterns     |
|  3. Each branch in a switch/if-else ladder → ask "polymorphism?"     |
|  4. Sequential checks → Chain of Responsibility                      |
|  5. Per-type algorithm → Strategy + Factory                          |
|  6. Composable boolean rules → Specification                         |
|  7. Data-driven / business-edited rules → Rule Engine                |
|  8. Each new rule should be a NEW CLASS, not new branch in old code  |
|     (Open/Closed Principle)                                          |
|  9. If you can't name what a method does in 5 words, it's too big    |
| 10. Hidden complexity > complexity in your face → extract & name     |
| 11. Don't pattern-spray on simple code — Rule of Three before refactor |
| 12. Spring's @Component + List<Interface> injection = free chain     |
+----------------------------------------------------------------------+
```

---

## PART 8: INTERVIEW CHEAT CODES

> "The orchestrator violates the **Open/Closed Principle** — every new rule modifies the same method. I'd refactor with **Chain of Responsibility** for the validation pipeline and **Strategy + Factory** for currency-specific rules, so adding a new currency becomes adding a new class without touching the orchestrator."

> "I prefer **polymorphism over conditionals**. Each type knows its own behavior, so the `if-else` ladder collapses into a single dispatch — and cyclomatic complexity drops from double digits to single digits."

> "Spring makes Chain of Responsibility almost free — I inject `List<Validator>` and Spring auto-collects every `@Component` implementing the interface. `@Order` controls the sequence."

> "For business rules that change frequently — pricing, fraud scoring, eligibility — I'd use a **Rule Engine** pattern with rules stored as data (DB, YAML, or even Drools). Business can change rules without a deployment."

> "Specifications are great for composable boolean logic — `isAdult.and(isVerified).and(isNotBanned)`. They're reusable, testable in isolation, and read like English."

> "Before adding patterns I always try **guard clauses** first — they often cut complexity in half with zero abstraction cost."

> "I follow the **Rule of Three** — first time, write it inline. Second time, copy-paste with a sigh. Third time, refactor into a pattern. Premature abstraction is as bad as no abstraction."

---

## QUICK PATTERN SELECTION FLOWCHART

```
+-------------------------------------------------------+
| Is the method's CC > 10?                              |
|   NO  → don't refactor (premature abstraction)        |
|   YES ↓                                               |
|                                                       |
| Are you doing a sequence of checks that short-circuit?|
|   YES → CHAIN OF RESPONSIBILITY                       |
|                                                       |
| Different algorithm per "type"/"category"?            |
|   YES → STRATEGY + FACTORY                            |
|                                                       |
| Composable boolean business rules?                    |
|   YES → SPECIFICATION                                 |
|                                                       |
| 50+ rules, business edits without code changes?       |
|   YES → RULE ENGINE (Drools, Easy Rules, or custom)   |
|                                                       |
| Switch/if-else on a TYPE field?                       |
|   YES → POLYMORPHISM (each type implements interface) |
|                                                       |
| Deeply nested ifs?                                    |
|   YES → GUARD CLAUSES first                           |
+-------------------------------------------------------+
```
