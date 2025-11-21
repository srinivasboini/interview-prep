# Java 9-10 Features Interview Guide

## Overview

Java 9 and 10 marked the shift to a 6-month release cycle and introduced transformative features for modularity, productivity, and performance. The module system (JPMS) was the biggest architectural change since Java was created, while Java 10's `var` keyword revolutionized code readability. These releases are critical for senior engineers interviewing at enterprises still on Java 8+ stacks or undergoing modernization initiatives.

**Why This Matters in Interviews**:
- Demonstrates understanding of Java platform evolution
- Shows familiarity with modularity concepts crucial for microservices
- Reflects knowledge of modern Java idioms (local variable type inference)
- Indicates awareness of performance optimizations

---

## Java 9: Module System (JPMS)

### Overview

The Java Platform Module System (JPMS) is Java's answer to the "JAR Hell" problem - a fundamental architectural change that adds explicit dependencies and encapsulation at the platform level.

```
Pre-Java 9:         Java 9+:
classpath           Module path
(implicit)          (explicit)
[chaos]             [clarity]
```

### Key Concepts

**What is a Module?**
A module is:
- A named, self-describing collection of code and resources
- Explicitly declares what it exports (public API)
- Explicitly declares what it requires (dependencies)
- Encapsulates internal implementation details
- Stronger than package-private (module-private)

**Module Declaration (module-info.java)**

```java
module banking.core {
    // Export public API
    exports com.banking.core.api;
    exports com.banking.core.account;

    // Export conditionally (for tools/tests)
    exports com.banking.core.internal to
        banking.service,
        banking.test;

    // Declare dependencies
    requires transitive java.base;
    requires com.fasterxml.jackson.databind;
    requires spring.framework.core;

    // Qualified exports
    opens com.banking.core.entity to
        org.hibernate.orm.core;

    // Service provision
    provides com.banking.core.PaymentProcessor
        with com.banking.core.impl.ACHPaymentProcessor;

    // Service consumption
    uses com.banking.core.NotificationService;
}
```

### Enterprise Patterns

**Layered Architecture with Modules**

```
┌──────────────────────────────────────┐
│  banking.web                         │  UI Layer
│  requires: banking.service           │
└──────────────────────────────────────┘
              │
┌──────────────────────────────────────┐
│  banking.service                     │  Business Logic
│  exports: banking.service.api        │
│  requires: banking.persistence,      │
│           banking.core               │
└──────────────────────────────────────┘
              │
┌──────────────────────────────────────┐
│  banking.persistence                 │  Data Access
│  requires: banking.domain,           │
│           org.hibernate              │
│  opens: banking.domain to hibernate  │
└──────────────────────────────────────┘
              │
┌──────────────────────────────────────┐
│  banking.domain                      │  Core Domain
│  exports: banking.domain.entities    │
└──────────────────────────────────────┘
```

### Common Use Cases

1. **Building Microservices**: Clear boundaries between services
2. **Enforcing Architecture**: Compiler checks dependencies before runtime
3. **Reducing Attack Surface**: Control what's accessible to untrusted code
4. **Platform Optimization**: JVM can eliminate unused modules

### Critical Implementation Details

**requires vs requires transitive**
```java
// Service module
module banking.service {
    requires transitive banking.domain;  // Clients see domain exports
    requires databind;                   // Internal dependency
}

// Client imports
module banking.web {
    requires banking.service;
    // Automatically gets banking.domain exports
    // Does NOT get databind unless explicitly required
}
```

**Automatic Modules** (for legacy JARs without module-info.java)
```java
module banking.app {
    requires old.legacy.library;  // JAR becomes automatic module
}
```

### Gotchas & Production Considerations

- **Reflection Breaking**: `reflection.getFields()` fails on non-exported packages
- **Framework Compatibility**: Older Spring/Hibernate versions don't play well with modules
- **Java Platform Modules**: Nine internal modules expose only supported APIs
- **Split Packages**: Cannot split same package across modules (causes `IllegalModuleNameException`)
- **Cyclic Dependencies**: Not allowed at module level (enforced at compile time)

---

## Java 9: Other Major Features

### JShell - Interactive REPL

```java
// Read-Eval-Print Loop for Java
jshell> int fibonacci(int n) {
   ...> return n <= 1 ? n : fibonacci(n-1) + fibonacci(n-2);
   ...> }
|  created method fibonacci(int)

jshell> fibonacci(6)
$1 ==> 8

jshell> /list        // Show all snippets
jshell> /history     // Show command history
```

**Interview Value**: Demonstrates understanding of developer productivity tools and language evolution.

### Private Methods in Interfaces

```java
public interface PaymentProcessor {
    // Public contract
    void processPayment(Payment payment);

    // Default implementation
    default void logTransaction(Payment p) {
        validatePayment(p);
        persistLog(p);
    }

    // Private method (new in Java 9)
    private void validatePayment(Payment p) {
        if (p.getAmount() <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
    }

    // Private static method
    private static void persistLog(Payment p) {
        // Shared logging logic
    }
}
```

**Why This Matters**: Reduces code duplication across default methods, improves maintainability of complex interfaces.

### Collection Factory Methods

```java
// Java 8
List<String> list = Collections.unmodifiableList(
    Arrays.asList("ACH", "Wire", "Card")
);

// Java 9 - Much cleaner
List<String> list = List.of("ACH", "Wire", "Card");
Set<String> set = Set.of("PENDING", "ACTIVE", "CLOSED");
Map<String, BigDecimal> rates = Map.of(
    "USD", new BigDecimal("1.0"),
    "EUR", new BigDecimal("1.15")
);

// Returns immutable collections
list.add("check"); // UnsupportedOperationException
```

**Performance Note**: These are implemented with specialized internal classes, more optimized than Collections.unmodifiableList().

### Stream API Improvements: takeWhile & dropWhile

```java
// Original list (already sorted)
Stream<Transaction> transactions = getTransactionsSorted();

// takeWhile - process while condition true, stop on first false
transactions
    .takeWhile(t -> t.getAmount() > 100)  // Stop at first small amount
    .forEach(this::processHighValue);

// dropWhile - skip while condition true, process rest
transactions
    .dropWhile(t -> t.getStatus().equals("PENDING"))  // Skip pending
    .forEach(this::processSettled);

// Real example: Process transactions up to a certain date
List<Transaction> currentYear = allTransactions.stream()
    .sorted(Comparator.comparing(Transaction::getDate))
    .takeWhile(t -> t.getDate().getYear() == 2024)
    .collect(Collectors.toList());
```

**Key Difference from filter()**: These assume stream is sorted; they stop processing on first mismatch (more efficient than filter).

### Optional Improvements

```java
// Java 9: ifPresentOrElse (more readable than ifPresent + ifEmpty)
Optional<Account> account = getAccount(customerId);

account.ifPresentOrElse(
    acc -> processAccount(acc),
    () -> createNewAccount(customerId)
);

// vs Java 8 way:
if (account.isPresent()) {
    processAccount(account.get());
} else {
    createNewAccount(customerId);
}

// Java 9: or() - chain multiple Optional sources
Optional<Account> account = getAccountById(id)
    .or(() -> getAccountByEmail(email))
    .or(() -> getAccountByPhone(phone));

// Java 9: stream() - convert Optional to Stream
Optional<Account> account = getAccount(customerId);
account.stream()
    .flatMap(acc -> acc.getTransactions().stream())
    .filter(t -> t.getAmount() > 1000)
    .collect(Collectors.toList());
```

### Process API Improvements

```java
// Create and manage OS processes with better control
ProcessBuilder pb = new ProcessBuilder("kubectl", "get", "pods");
pb.directory(new File("/home/user/k8s"));

Process process = pb.start();
process.toHandle()  // New: ProcessHandle for better control
    .getPid();      // Process ID
    .onExit()       // New: returns CompletableFuture
    .thenAccept(p -> System.out.println("Process completed"));

// Shutdown process gracefully
ProcessHandle.current()  // Current JVM process
    .children()          // All child processes
    .forEach(ProcessHandle::destroy);
```

---

## Java 10: Local Variable Type Inference (var keyword)

### Overview

The `var` keyword allows the compiler to infer the type of local variables, reducing boilerplate while maintaining type safety.

```java
// Java 9 and earlier
List<Transaction> transactions = bankService.getTransactions();
Map<String, Account> accountMap = new HashMap<>();

// Java 10
var transactions = bankService.getTransactions();  // Type: List<Transaction>
var accountMap = new HashMap<String, Account>();  // Type: Map<String, Account>
```

### When to Use var

**Good Use Cases**

```java
// 1. Complex generic types
var accountMap = new HashMap<String, List<Transaction>>();  // Clear from right side

// 2. Obvious from assignment
var accountId = account.getId();  // Clearly Long
var isActive = account.isActive();  // Clearly boolean

// 3. Clear from method return type
var transactions = service.getTransactions();  // Return type is evident

// 4. Intermediate variables in complex streams
var activeAccounts = accounts.stream()
    .filter(Account::isActive)
    .collect(Collectors.toList());

// 5. Try-with-resources
var connection = dataSource.getConnection();
try (var stmt = connection.createStatement()) {
    var result = stmt.executeQuery("SELECT * FROM accounts");
    // ...
}
```

**Poor Use Cases (Avoid)**

```java
// Don't: Unclear what x is
var x = getValue();

// Don't: var with numeric literals (ambiguous)
var rate = 1.5;  // Is this double or Float?
var amount = 100;  // Is this int or long?

// Don't: var with null (no type information)
var config = null;  // Compile error - cannot infer type

// Don't: var breaks API contracts
public var getAmount() {  // Syntax error - only for local variables
    return new BigDecimal("100");
}

// Don't: var in public APIs
public void process(var transaction) {  // Error - only local variables
    // ...
}
```

### var in Practice

```java
// Banking example: Processing transaction batches
public List<Account> updateAccountBalances(List<Transaction> batch) {
    var balanceUpdates = new HashMap<String, BigDecimal>();  // Clear from type

    for (var transaction : batch) {  // Clear: transaction is Transaction
        var accountId = transaction.getAccountId();  // Clear: String
        var newBalance = balanceUpdates
            .getOrDefault(accountId, BigDecimal.ZERO)
            .add(transaction.getAmount());

        balanceUpdates.put(accountId, newBalance);
    }

    return balanceUpdates.entrySet().stream()
        .map(entry -> {
            var account = getAccount(entry.getKey());
            account.setBalance(entry.getValue());
            return account;
        })
        .collect(Collectors.toList());
}
```

### Type Inference Mechanics

```
var x = expression;
        └─ Compiler analyzes expression
           └─ Determines type at compile time
           └─ Type is FINAL and immutable
```

The inferred type is **not** a "generic Object" - it's the specific concrete type, determined at compile time.

---

## Java 10: Other Features

### AppCDS (Application Class-Data Sharing)

Caches application classes in shared archive files, reducing startup time and memory footprint.

```bash
# 1. Create archive with application classes
java -Xshare:dump \
    -XX:+UseAppCDS \
    -XX:SharedArchiveFile=app.jsa \
    -cp my-app.jar \
    com.banking.ApplicationMain

# 2. Run with archive
java -Xshare:on \
    -XX:SharedArchiveFile=app.jsa \
    -cp my-app.jar \
    com.banking.ApplicationMain
```

**Enterprise Impact**: Critical for containerized deployments and serverless architectures (reduced cold start time).

### Parallel Full GC for G1

```java
// Java 10: G1 garbage collector can parallelize full GC
// Configuration:
// java -XX:+UseG1GC -XX:+ParallelGCThreads=8 MyApp

// Before: Full GC was single-threaded (pause times in seconds)
// After: Full GC parallelized across multiple threads
```

**Performance Implication**: Reduces GC pause times in high-memory applications (critical for banking systems with large transaction histories).

### Optional.orElseThrow()

```java
// Java 10 - Preferred way to get value or throw
Optional<Account> account = getAccount(customerId);

// Old way (Java 9 and earlier)
Account acc = account.orElseThrow(
    () -> new AccountNotFoundException("Not found")
);

// Java 10 - Cleaner (no need for exception supplier if throwing NoSuchElementException)
Account acc = account.orElseThrow();  // Throws NoSuchElementException if empty

// Still supports custom exceptions
Account acc = account.orElseThrow(
    () -> new AccountNotFoundException("ID: " + customerId)
);
```

---

## Interview Questions & Answers

### 1. What is the Java Module System and why was it needed?

**Answer Structure:**
The JPMS (Java Platform Module System) introduced explicit dependencies and encapsulation at the platform level, solving the "JAR Hell" problem where classpath ordering determined which version of a class was loaded.

**Key Points:**
- Solves JAR Hell (implicit, chaotic dependencies)
- Enforces explicit contracts (exports/requires)
- Enables platform optimization (JVM knows which modules can be eliminated)
- Stronger encapsulation than package-private (module-private)

**Enterprise Angle:** In banking systems, modules enforce architectural boundaries - preventing UI layer from directly accessing internal domain models, for example.

---

### 2. How do you migrate a legacy application to use modules?

**Answer Structure:**
Start with automatic modules, then gradually introduce explicit module declarations.

**Steps:**
1. Place legacy JARs on module path (become automatic modules automatically)
2. Create module-info.java for your own code
3. Run with `--permit-illegal-access` to handle reflection issues
4. Gradually update dependencies and add explicit module declarations
5. Test thoroughly (modules break reflection!)

**Common Pitfall:** Assuming `requires` automatically includes transitive dependencies. Only `requires transitive` exports transitive deps.

---

### 3. When would you use `requires transitive` vs `requires`?

**Answer Structure:**

```
requires transitive: Used when your module's PUBLIC API depends on another module
                    Clients of your module need to see those dependencies

requires: Used for internal dependencies clients don't care about
```

**Example:**
```java
// banking.service exports APIs that use banking.domain classes
module banking.service {
    requires transitive banking.domain;  // Clients need this
    requires databind;                    // Internal detail
}

// Client automatically gets banking.domain when importing banking.service
module banking.web {
    requires banking.service;  // Also sees banking.domain
}
```

---

### 4. Explain the difference between `takeWhile()` and `filter()` with an example.

**Answer Structure:**

`takeWhile()` stops processing at the first element that doesn't match the condition (assuming sorted data). `filter()` continues processing all elements.

```java
// Stream: [100, 200, 300, 50, 400]

// filter - processes all 5 elements
stream.filter(x -> x > 99)
    .forEach(System.out::println);
// Output: 100, 200, 300, 400

// takeWhile - stops at first failure
stream.takeWhile(x -> x > 99)
    .forEach(System.out::println);
// Output: 100, 200, 300  (stops when it hits 50)
```

**Interview Point:** `takeWhile` is more efficient for sorted streams where you only care about the prefix.

---

### 5. What are the constraints of the `var` keyword?

**Answer Structure:**

`var` can ONLY be used for local variables, not for method parameters, return types, or fields.

**Constraints:**
```java
// Legal:
var count = 10;
var list = new ArrayList<String>();

// Illegal:
var count;  // No initializer - cannot infer type
var config = null;  // null has no type
public var getBalance() { ... }  // Can't use in return type
public void process(var tx) { ... }  // Can't use in parameters
private var balance;  // Can't use in fields
```

**Why This Matters:** Maintains explicit API contracts. Calling code shouldn't need to read implementation to understand method signatures.

---

### 6. How does the module system handle reflection breaking changes?

**Answer Structure:**

Modules restrict reflection by default. Packages must be explicitly opened for reflection via `opens` declaration.

```java
module banking.domain {
    // Allows reflection on banking.domain.entity classes
    opens banking.domain.entity to org.hibernate.orm.core;

    // Allows reflection on all classes to all modules
    opens banking.domain;
}
```

**Production Issue:** Old versions of Hibernate, Spring, and Jackson use reflection. Must add `opens` declarations or use `--add-opens JVM_ARGS` at runtime.

```bash
# Workaround if you can't modify module-info.java
java --add-opens java.base/java.lang=banking.service MyApp
```

---

### 7. Describe a scenario where you'd use AppCDS in production.

**Answer Structure:**

AppCDS is critical for containerized and serverless applications where startup time matters.

**Scenario - Serverless Banking API:**
```
Cold Start Benchmark (500MB heap):
- Without AppCDS: 3.5 seconds
- With AppCDS: 1.8 seconds  (~50% improvement)
```

**Implementation:**
1. Build Docker image with AppCDS archive included
2. Run container with `-XX:SharedArchiveFile=app.jsa`
3. Reduces cold start latency for AWS Lambda/Azure Functions

---

### 8. What's the practical difference between Java 9 and 10's var keyword and dynamic languages?

**Answer Structure:**

`var` is NOT dynamic typing. Type is determined at **compile time** and fixed.

```java
var count = 10;  // Type: int (determined at compile time, never changes)
// count = "hello";  // COMPILER ERROR - cannot assign String to int

// vs JavaScript (dynamic):
let count = 10;
count = "hello";  // Allowed at runtime
```

**Interview Point:** `var` gives Java the **readability benefit** of dynamic languages while maintaining the **type safety** of static languages.

---

### 9. In what scenarios would you avoid using `var` even though it's syntactically valid?

**Answer Structure:**

Use `var` judiciously. Prefer explicit types when they clarify code intent.

```java
// Avoid - unclear what method returns
var result = service.processTransaction();

// Better - explicit type clarifies intent
TransactionResult result = service.processTransaction();

// Avoid - numeric ambiguity
var rate = 0.15;  // Is this double or Float?

// Better - explicit
double rate = 0.15;

// Avoid - weakens API contracts
public List<Account> getAccounts(var filter) { ... }  // Illegal anyway
```

**Principle:** Code is read more often than written. Type clarity helps future maintainers understand intent immediately.

---

### 10. How does the G1 parallel full GC improvement benefit enterprise applications?

**Answer Structure:**

Full Garbage Collection pauses the entire application (Stop-the-World). Parallelizing this reduces pause duration.

**Banking Example:**
```
Transaction processing during GC pause:
- Before Java 10: 8-second GC pause
  → Customer requests timeout
  → Batch processing misses SLA

- After Java 10: 2-second GC pause
  → Acceptable latency
  → Batch processing completes on time
```

**Configuration:**
```bash
java -XX:+UseG1GC \
     -XX:+ParallelGCThreads=16 \
     -Xmx8g \
     BankingApplication
```

---

### 11. Compare List.of() vs Arrays.asList() with immutability implications.

**Answer Structure:**

`List.of()` creates truly immutable collection. `Arrays.asList()` creates a view of an array that's mutable and resizable (but fixed-size).

```java
// Arrays.asList - fixed size, backed by array
List<String> types = Arrays.asList("ACH", "Wire");
types.add("Card");  // UnsupportedOperationException
types.set(0, "Check");  // Works! (modifies backing array)

// List.of - truly immutable
List<String> types = List.of("ACH", "Wire");
types.add("Card");  // UnsupportedOperationException
types.set(0, "Check");  // UnsupportedOperationException

// Production impact: use List.of for read-only data
final List<String> VALID_STATUSES = List.of(
    "PENDING", "ACTIVE", "CLOSED", "FRAUD"
);
```

**Interview Point:** Immutable collections are thread-safe, safer for concurrent access in banking systems.

---

### 12. How would you design a modular banking application using Java 9+ JPMS?

**Answer Structure:**

Design layered architecture with clear module boundaries. Each layer is a separate module with explicit dependencies.

```
Module Structure:

banking.api (Public contracts)
├── exports: com.banking.api.account
├── exports: com.banking.api.payment
└── requires: (none)

banking.domain (Entities & Value Objects)
├── exports: com.banking.domain.model
├── requires: (none)

banking.service (Business Logic)
├── exports: com.banking.service (only public interfaces)
├── requires transitive: banking.domain
├── requires: banking.persistence (internal)
└── opens: to org.springframework.core (for AOP/proxies)

banking.persistence (Data Access)
├── exports: (internal only)
├── requires: banking.domain
├── requires: org.hibernate
└── opens: com.banking.persistence to org.hibernate

banking.infrastructure (External Integrations)
├── requires: banking.service
├── requires: com.fasterxml.jackson
└── provides: com.banking.api.PaymentGateway

banking.web (REST API)
├── requires: banking.api
├── requires: banking.service
└── requires: spring.web
```

**Key Benefits:**
- Circular dependencies impossible (enforced at compile time)
- Clear API surface (only exports are public)
- Framework compatibility managed explicitly (opens declarations)
- Performance optimized (JVM knows module dependencies)

---

## Gotchas & Best Practices

### Module System Gotchas

1. **Reflection Breaking Changes**
   - JUnit, Mockito, Jackson, Hibernate all use reflection
   - Solution: Add `opens` declarations or use `--add-opens` flag
   - Test with `--limit-modules` to catch issues early

2. **Transitive Dependency Explosion**
   - `requires transitive` exports unnecessary modules
   - Use sparingly; prefer explicit `requires` for internal deps
   - Document why you used `requires transitive`

3. **Split Packages**
   ```java
   // Package "com.banking.core" cannot exist in both:
   module m1 { exports com.banking.core; }
   module m2 { exports com.banking.core; }  // Compile error!
   ```

4. **Automatic Modules are Unreliable**
   - Old JARs placed on module path become automatic modules
   - Their module name is derived from filename (fragile)
   - Eventually migrate to explicit modules or require new versions

### var Keyword Best Practices

1. **Use when right-side type is obvious**
   ```java
   var list = new ArrayList<Transaction>();  // OK - clear
   var list = createTransactions();  // Questionable - is return type clear?
   ```

2. **Never use with null**
   ```java
   var config = null;  // Compile error
   ```

3. **Be explicit in ambiguous cases**
   ```java
   var rate = 1.5;  // Bad - is it double or Float?
   double rate = 1.5;  // Clear
   ```

4. **Don't use to hide complexity**
   ```java
   // Bad - hides complex type
   var result = service.process(request);

   // Better - shows contract
   ProcessingResult result = service.process(request);
   ```

### Collection Factory Methods Best Practices

1. **Prefer for immutable data**
   ```java
   // Good - read-only reference data
   private static final Set<String> VALID_CURRENCIES = Set.of(
       "USD", "EUR", "GBP", "JPY"
   );

   // Avoid - data that needs modification
   List<Transaction> transactions = List.of(...);  // Will break when you need to add
   ```

2. **No null elements allowed**
   ```java
   var list = List.of("USD", null);  // NullPointerException
   ```

3. **Performance characteristics**
   - Faster than Collections.unmodifiableList()
   - No extra wrapper layer
   - Optimized for common sizes (1-2 elements especially cheap)

---

## References

- [Java 9 Release Notes](https://www.oracle.com/java/technologies/javase/9-all-relnotes.html)
- [Java Platform Module System (JPMS) Specification](https://docs.oracle.com/javase/9/docs/api/java.base-summary.html)
- [Java 10 Release Notes](https://www.oracle.com/java/technologies/javase/10-all-relnotes.html)
- [Local Variable Type Inference (JEP 286)](https://openjdk.org/jeps/286)
- [AppCDS Documentation](https://docs.oracle.com/en/java/javase/10/docs/specs/cds.html)
- [Venkat Subramaniam - Java 9+ Features](https://www.agilelearner.com/)

---

## Summary

Java 9-10 represented a maturation of the Java platform with transformative features:

- **Module System (Java 9)**: Enforces architectural boundaries, solves JAR Hell, enables platform optimization
- **Local Variable Type Inference (Java 10)**: Improves readability without sacrificing type safety
- **Stream Enhancements (Java 9)**: `takeWhile`/`dropWhile` for efficient prefix operations
- **Collection Factories (Java 9)**: `List.of()`, `Set.of()`, `Map.of()` for cleaner, immutable collections
- **Performance (Java 10)**: Parallel full GC and AppCDS reduce latency in enterprise deployments

For senior engineers in enterprise banking, understanding modules is essential for designing scalable microservices architectures, while `var` and collection factories represent modern Java idioms you'll encounter in contemporary codebases.
