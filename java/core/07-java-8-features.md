# Java 8 LTS Features - Interview Guide

## Overview

Java 8 (released March 2014) was a transformational release that fundamentally changed how Java developers write code. It introduced functional programming capabilities, modernized the Date/Time API, and added powerful new language features that became the foundation for all subsequent Java versions.

**Why This Matters in Interviews**:
- Java 8 is the baseline LTS version most enterprises still use (March 2022 support end, extended support until December 2030)
- Understanding these features demonstrates knowledge of modern Java practices
- Many interview questions assume Java 8+ syntax and patterns
- Foundation for understanding Java 9-21 features and evolution

**Real-World Banking Context**:
Java 8 enabled the transition from verbose, procedural banking code to elegant functional patterns. Transaction processing pipelines, compliance filtering, and data aggregation all became simpler and more maintainable through streams and lambdas. The modern Date/Time API eliminated countless timezone bugs in international settlement systems.

---

## 1. The Five Key Innovation Areas

```
┌─────────────────────────────────────────────────────────────┐
│           Java 8 Innovation Areas                            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Functional Programming                                   │
│     └─ Lambda expressions, Functional interfaces             │
│                                                               │
│  2. Collections & Data Processing                            │
│     └─ Stream API, Method references                         │
│                                                               │
│  3. Enhanced Interfaces                                      │
│     └─ Default methods, Static methods                       │
│                                                               │
│  4. Modern Date/Time                                         │
│     └─ java.time package (JSR-310)                           │
│                                                               │
│  5. Async & Reactive Programming                             │
│     └─ CompletableFuture, parallel streams                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Lambda Expressions & Functional Interfaces

### Brief Recap (Detailed Coverage: See 04-streams-and-functional-programming.md)

Lambda expressions provide concise syntax for anonymous functions:
```java
// Traditional anonymous class
Comparator<String> comp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }
};

// Lambda expression (Java 8+)
Comparator<String> lambda = (a, b) -> a.length() - b.length();
```

**Core Functional Interfaces** (`java.util.function` package):

| Interface | Signature | Use Case |
|-----------|-----------|----------|
| `Function<T, R>` | `T -> R` | Transform data |
| `Predicate<T>` | `T -> boolean` | Filter/test conditions |
| `Consumer<T>` | `T -> void` | Side effects |
| `Supplier<T>` | `() -> T` | Generate/provide values |
| `BiFunction<T, U, R>` | `(T, U) -> R` | Binary transformations |

**Key Point**: Variables in lambda bodies must be **effectively final** (not reassigned after definition). This enables thread-safe functional operations without shared mutable state.

---

## 3. Stream API

### Brief Recap (Detailed Coverage: See 04-streams-and-functional-programming.md)

Streams provide a functional pipeline for data processing without loops:

```java
List<Transaction> transactions = getTransactions();

// Traditional imperative approach
List<BigDecimal> largeAmounts = new ArrayList<>();
for (Transaction tx : transactions) {
    if (tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0) {
        largeAmounts.add(tx.getAmount());
    }
}

// Stream approach (declarative)
List<BigDecimal> largeAmounts = transactions.stream()
    .filter(tx -> tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0)
    .map(Transaction::getAmount)
    .collect(Collectors.toList());

// Complex aggregations
Map<String, BigDecimal> totalByAccount = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::getAccountId,
        Collectors.reducing(BigDecimal.ZERO, Transaction::getAmount, BigDecimal::add)
    ));
```

**Stream Pipeline Structure**:
```
Source (collection/array)
    ↓
Intermediate Operations (filter, map, flatMap, distinct, sorted)
    ↓
Terminal Operation (collect, forEach, reduce, findFirst)
```

**Key Characteristics**:
- **Lazy evaluation**: Intermediate operations don't execute until a terminal operation
- **Immutable**: Doesn't modify source collection
- **Single-use**: Stream can be consumed only once
- **Parallel support**: `.parallelStream()` for multi-threaded processing

---

## 4. Optional - Handling Null Values

### Brief Recap (Detailed Coverage: See 04-streams-and-functional-programming.md)

`Optional<T>` represents a value that may or may not be present, eliminating the need for null checks:

```java
// Problems with null
User user = database.findById(123);
if (user != null && user.getProfile() != null) {
    String email = user.getProfile().getEmail();
    if (email != null) {
        sendNotification(email);
    }
}

// Optional approach
Optional<User> user = database.findById(123);
user.flatMap(User::getProfile)
    .map(Profile::getEmail)
    .ifPresent(this::sendNotification);
```

**Core Methods**:

| Method | Purpose |
|--------|---------|
| `of(value)` | Create with value (fails if null) |
| `ofNullable(value)` | Create, accepting null |
| `isPresent()` / `isEmpty()` | Check presence (Java 11+) |
| `ifPresent(consumer)` | Execute if present |
| `ifPresentOrElse(consumer, runnable)` | If-else logic |
| `get()` | Get value (fails if empty) |
| `getOrElse(default)` | Get or default value |
| `map(function)` | Transform value if present |
| `flatMap(function)` | Chain Optionals |
| `filter(predicate)` | Keep if condition true |

**Anti-Pattern**: Avoid using `Optional` with `get()` without checking:
```java
// BAD - defeats the purpose
user.get().getEmail();  // Can throw NoSuchElementException

// GOOD - use one of the safe methods
user.map(User::getEmail)
    .ifPresent(System.out::println);
```

---

## 5. Default & Static Methods in Interfaces

### Why This Matters

Pre-Java 8, interfaces were purely contract definitions - all methods were abstract. This made adding methods to existing interfaces a breaking change. Java 8 solved this while enabling functional interface patterns.

### Default Methods

Allow concrete implementations in interfaces without breaking existing implementations:

```java
public interface PaymentProcessor {
    // Abstract method
    void processPayment(BigDecimal amount);

    // Default method - subinterfaces can override
    default void validateAmount(BigDecimal amount) {
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }

    // Real-world example: backward-compatible enhancement
    default void recordAudit(String details) {
        System.out.println("AUDIT: " + details);
    }
}

// Existing implementation works without changes
public class BankTransferProcessor implements PaymentProcessor {
    @Override
    public void processPayment(BigDecimal amount) {
        // Process payment
    }
    // validateAmount() and recordAudit() inherited
}

// New implementation can override default
public class EnhancedPaymentProcessor implements PaymentProcessor {
    @Override
    public void processPayment(BigDecimal amount) {
        validateAmount(amount);
        recordAudit("Processing: " + amount);
        // Process payment
    }

    @Override
    default void recordAudit(String details) {
        // Custom audit implementation
        auditService.log(details);
    }
}
```

**Key Characteristics**:
- Marked with `default` keyword
- Have method body
- Can call other interface methods
- Can be overridden by implementing classes
- Resolved at compile-time through method resolution order

### Static Methods in Interfaces

Static methods provide utility functions in interface context:

```java
public interface TransactionUtils {
    // Static method - cannot be overridden
    static BigDecimal calculateFeeForAmount(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.valueOf(1000)) > 0) {
            return amount.multiply(BigDecimal.valueOf(0.001));  // 0.1%
        }
        return BigDecimal.valueOf(10);  // Fixed fee
    }

    // Another example - factory method pattern
    static Transaction createDummyTransaction(String id, BigDecimal amount) {
        return new Transaction(id, amount, LocalDateTime.now());
    }
}

// Usage - called on interface directly
BigDecimal fee = TransactionUtils.calculateFeeForAmount(BigDecimal.valueOf(5000));
Transaction dummy = TransactionUtils.createDummyTransaction("T001", amount);
```

**Use Cases**:
- Utility/helper functions related to interface concept
- Factory methods for creating interface implementations
- Companion methods that logically belong with interface

**Constraint**: Static methods cannot be overridden - they're bound to the interface.

### Method Resolution Order (Diamond Problem)

When a class implements multiple interfaces with the same default method:

```java
interface Auditable {
    default void audit() { System.out.println("Auditing..."); }
}

interface Reportable {
    default void audit() { System.out.println("Reporting..."); }
}

// COMPILATION ERROR: Must resolve which audit() to use
public class Account implements Auditable, Reportable {
    // Must explicitly override and choose
    @Override
    public void audit() {
        Auditable.super.audit();  // Call Auditable's version
        Reportable.super.audit(); // Then Reportable's version
    }
}
```

---

## 6. Method References

### Brief Recap (Detailed Coverage: See 04-streams-and-functional-programming.md)

Method references provide shorthand for lambdas that delegate to existing methods:

```java
// Lambda version
list.forEach(tx -> logger.log(tx));

// Method reference version (more concise)
list.forEach(logger::log);

// Different types of method references
list.stream()
    .map(Transaction::getAmount)           // Reference to instance method
    .map(String::valueOf)                  // Reference to static method
    .map(BigDecimal::new)                  // Reference to constructor
    .forEach(System.out::println);         // Reference to instance method on object
```

**Forms**:

| Form | Syntax | Equivalent Lambda |
|------|--------|------------------|
| Static method | `ClassName::staticMethod` | `(x) -> ClassName.staticMethod(x)` |
| Instance method (class) | `ClassName::instanceMethod` | `(x) -> x.instanceMethod()` |
| Instance method (object) | `object::instanceMethod` | `(x) -> object.instanceMethod(x)` |
| Constructor | `ClassName::new` | `(x) -> new ClassName(x)` |

---

## 7. Date/Time API (java.time)

### Brief Recap (Detailed Coverage: See 18-date-time-api.md)

The legacy `java.util.Date` and `java.util.Calendar` are notoriously problematic: mutable, thread-unsafe, confusing API, poor timezone support. Java 8 introduced a complete redesign (JSR-310) inspired by Joda-Time.

### Key Design Principles

1. **Immutability**: All classes are immutable and thread-safe
2. **Human vs Machine Time**: Clear distinction in the API
3. **Timezone-Aware**: First-class support for time zones and offsets
4. **ISO-8601 Standard**: Based on international calendar system

### Core Classes

#### LocalDate - Date Only

```java
LocalDate today = LocalDate.now();
LocalDate specific = LocalDate.of(2024, 11, 20);
LocalDate fromString = LocalDate.parse("2024-11-20");

// Arithmetic
LocalDate nextMonth = today.plusMonths(1);
LocalDate lastYear = today.minusYears(1);

// Queries
int year = today.getYear();
Month month = today.getMonth();
DayOfWeek dayOfWeek = today.getDayOfWeek();
boolean isLeapYear = today.isLeapYear();

// Business logic
LocalDate contractMaturity = today.plusYears(3);
boolean isWeekend = today.getDayOfWeek().getValue() >= 6;
```

**Use Cases**: Birth dates, contract dates, settlement dates, business date calculations

#### LocalTime - Time Only

```java
LocalTime now = LocalTime.now();
LocalTime business = LocalTime.of(14, 30, 0);

// Arithmetic
LocalTime twoHours = now.plusHours(2);

// Check business hours
boolean isBusinessHours =
    now.isAfter(LocalTime.of(9, 0)) &&
    now.isBefore(LocalTime.of(17, 0));
```

**Use Cases**: Business hours validation, batch job scheduling, time-of-day logic

#### LocalDateTime - Date and Time (No Timezone)

```java
LocalDateTime now = LocalDateTime.now();
LocalDateTime specific = LocalDateTime.of(2024, 11, 20, 14, 30, 0);

// Conversion from components
LocalDateTime combined = LocalDateTime.of(
    LocalDate.of(2024, 11, 20),
    LocalTime.of(14, 30)
);

// Extract components
LocalDate date = now.toLocalDate();
LocalTime time = now.toLocalTime();
```

**Use Cases**: Database storage without timezone concerns, audit trails with local time

#### ZonedDateTime - Date, Time, and Timezone

```java
// Current time in specific timezone
ZonedDateTime nyTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
ZonedDateTime londonTime = ZonedDateTime.now(ZoneId.of("Europe/London"));

// Conversion between zones
ZonedDateTime utcTime = nyTime.withZoneSameInstant(ZoneId.of("UTC"));

// Create with specific zone
ZonedDateTime specific = ZonedDateTime.of(
    LocalDateTime.of(2024, 11, 20, 14, 30),
    ZoneId.of("America/New_York")
);

// Banking use-case: Trade settlement in multiple zones
Map<String, ZonedDateTime> settlementTimes = Map.of(
    "NY_OPEN", ZonedDateTime.now(ZoneId.of("America/New_York")),
    "LONDON_OPEN", ZonedDateTime.now(ZoneId.of("Europe/London")),
    "TOKYO_OPEN", ZonedDateTime.now(ZoneId.of("Asia/Tokyo"))
);
```

**Use Cases**: Globalized applications, cross-timezone coordination, API timestamps

#### Instant - Machine Time (Epoch)

```java
// Current instant on UTC timeline
Instant now = Instant.now();

// Specific epoch seconds
Instant specificTime = Instant.ofEpochSecond(1732132200L);

// Conversion from ZonedDateTime
ZonedDateTime zoned = ZonedDateTime.now();
Instant instant = zoned.toInstant();

// Database storage (represents absolute point in time)
Long epochMillis = now.toEpochMilli();  // Store as long
Instant restored = Instant.ofEpochMilli(epochMillis);
```

**Use Cases**: Timestamps for logging, events, database storage, calculating durations

#### Duration & Period

```java
// Duration - time-based (hours, minutes, seconds)
LocalTime start = LocalTime.of(9, 0);
LocalTime end = LocalTime.of(17, 0);
Duration workDay = Duration.between(start, end);  // PT8H

long hours = workDay.toHours();  // 8
long minutes = workDay.toMinutes();  // 480

// Period - date-based (years, months, days)
LocalDate birth = LocalDate.of(1990, 5, 15);
LocalDate today = LocalDate.now();
Period age = Period.between(birth, today);

int years = age.getYears();
int months = age.getMonths();
int days = age.getDays();

// Practical use-case: SLA calculations
Duration sla = Duration.ofHours(24);
Instant createdTime = transaction.getCreatedTime();
Instant now = Instant.now();
Duration elapsed = Duration.between(createdTime, now);

if (elapsed.compareTo(sla) > 0) {
    // SLA violated
}
```

### Critical Timezone Considerations

```
TimeZone Gotchas (Banking Critical):

1. LocalDateTime has NO timezone
   LocalDateTime local = LocalDateTime.now();
   // This is "2024-11-20 14:30:00" with no timezone info
   // Which timezone is this? The server's? Client's? UTC?

   WRONG: LocalDateTime.now() in trading app
   RIGHT: ZonedDateTime.now(ZoneId.of("UTC"))
         or ZonedDateTime.now(ZoneId.of("America/New_York"))

2. Daylight Saving Time transitions
   ZoneId zone = ZoneId.of("America/New_York");

   // March 10, 2024 - clocks spring forward (2:00 AM -> 3:00 AM)
   LocalDateTime ambiguous = LocalDateTime.of(2024, 3, 10, 2, 30);

   // This creates an ambiguous time - doesn't exist!
   // Java handles gracefully (moves to next valid time)

3. Printing and parsing
   // Different zones produce different strings
   ZonedDateTime.now(ZoneId.of("UTC"))
       .format(DateTimeFormatter.ISO_INSTANT);
   // "2024-11-20T14:30:00Z"

   ZonedDateTime.now(ZoneId.of("America/New_York"))
       .format(DateTimeFormatter.ISO_INSTANT);
   // "2024-11-20T19:30:00Z" (same instant, different local time)
```

---

## 8. CompletableFuture - Async & Reactive Programming

### What Problem Does It Solve?

Pre-Java 8, asynchronous programming in Java was clunky. `java.util.concurrent.Future` required blocking calls to get results. CompletableFuture provides:
- Non-blocking async operations
- Composable async pipelines
- Exception handling for async code
- Timeout support

### Basic Usage

```java
// Create a CompletableFuture
CompletableFuture<String> future = new CompletableFuture<>();

// Manually complete it
future.complete("Success");
future.get();  // "Success"

// Create from async task
CompletableFuture<String> async = CompletableFuture.supplyAsync(() -> {
    // Long-running operation on ForkJoinPool
    return fetchUserData("user123");
}, executorService);

// Non-blocking - register callback
async.thenAccept(result ->
    System.out.println("User data: " + result)
);

// Continue and transform
CompletableFuture<UserProfile> profile = async
    .thenApply(this::parseUserData)
    .thenApply(this::enrichWithTransactions)
    .exceptionally(ex -> handleError(ex));
```

### Composition Patterns

```java
// Sequential chaining
CompletableFuture<String> name =
    fetchUser(123)  // CompletableFuture<User>
    .thenApply(User::getName);  // CompletableFuture<String>

// Combining multiple futures
CompletableFuture<User> user = fetchUser(123);
CompletableFuture<Profile> profile = fetchProfile(123);

CompletableFuture<Pair<User, Profile>> combined =
    user.thenCombine(profile, Pair::new);

// Either/Or - complete with first result
CompletableFuture<String> fastest =
    CompletableFuture.anyOf(
        fetchFromDB(),
        fetchFromCache(),
        fetchFromAPI()
    )
    .thenApply(result -> (String) result);

// All of - combine multiple futures
CompletableFuture<Void> allDone = CompletableFuture.allOf(
    saveUser(user),
    sendNotification(user),
    updateCache(user)
);

allDone.thenRun(() -> System.out.println("All operations completed"));

// Exception handling
CompletableFuture<String> withErrorHandling =
    fetchData()
    .exceptionally(ex -> {
        logger.error("Error fetching data", ex);
        return "default value";
    })
    .handle((result, ex) -> {
        if (ex != null) {
            return "error: " + ex.getMessage();
        }
        return "success: " + result;
    });
```

### Real-World Banking Example

```java
public CompletableFuture<TransactionSummary> processTransaction(Transaction tx) {
    return fetchAccountDetails(tx.getAccountId())
        .thenCompose(account -> validateTransaction(tx, account))
        .thenCompose(validated -> executePayment(validated))
        .thenCompose(result -> persistTransaction(result))
        .thenCompose(saved -> sendNotification(saved))
        .exceptionally(this::handlePaymentFailure);
}

// Execute multiple settlement operations in parallel
CompletableFuture<Void> settleAllTransactions =
    transactionList.stream()
    .map(this::processTransaction)
    .collect(Collectors.toList())
    .parallelStream()
    .reduce(CompletableFuture.completedFuture(null),
        (acc, future) -> acc.thenCompose(v -> future),
        (acc1, acc2) -> acc1.thenCompose(v -> acc2));

settleAllTransactions.get();  // Wait for all to complete
```

### Key Methods

| Method | Use Case |
|--------|----------|
| `supplyAsync(supplier)` | Async computation returning value |
| `runAsync(runnable)` | Async computation returning void |
| `thenApply(function)` | Transform result |
| `thenAccept(consumer)` | Consume result (side effect) |
| `thenCompose(function)` | Chain with another CompletableFuture |
| `thenCombine(other, function)` | Combine two futures |
| `anyOf(...)` | Complete with first result |
| `allOf(...)` | Complete when all done |
| `exceptionally(function)` | Handle exception |
| `handle(bifunction)` | Handle result or exception |
| `complete(value)` | Manually complete with value |
| `completeExceptionally(ex)` | Manually complete with exception |
| `get()` / `join()` | Block until complete (avoid in async code) |

---

## 9. Functional Interfaces Summary

### Why Custom Functional Interfaces?

While `java.util.function` provides standard interfaces, you often need custom ones for domain clarity:

```java
// Generic but unclear what validation means
Predicate<Transaction> validation = tx -> tx.isValid();

// Custom functional interface - domain clear
@FunctionalInterface
public interface TransactionValidator {
    boolean isValid(Transaction transaction);
}

// Even better - more expressive
@FunctionalInterface
public interface ComplianceChecker {
    /**
     * Check if transaction violates sanctions or AML rules
     */
    boolean passesCompliance(Transaction transaction);
}

TransactionValidator validator = tx -> {
    return tx.getAmount().compareTo(BigDecimal.ZERO) > 0 &&
           tx.getAccountId() != null;
};
```

### Creating Custom Functional Interfaces

```java
// Single abstract method - functional interface
@FunctionalInterface
public interface TransactionProcessor {
    TransactionResult process(Transaction tx) throws Exception;

    // Can have default methods
    default void audit(String details) {
        System.out.println("AUDIT: " + details);
    }

    // Can have static methods
    static TransactionProcessor loggingProcessor() {
        return tx -> {
            logger.info("Processing: " + tx.getId());
            return new TransactionResult("processed");
        };
    }
}

// Usage with lambda
TransactionProcessor processor = tx -> {
    // Process transaction
    return new TransactionResult("success");
};

processor.process(myTransaction);
processor.audit("Transaction processed successfully");
```

### Common Functional Interface Patterns

| Pattern | Purpose | Common Interfaces |
|---------|---------|------------------|
| Transformation | Input -> Output | `Function`, `UnaryOperator` |
| Filtering | Input -> boolean | `Predicate` |
| Side Effects | Input -> void | `Consumer`, `Runnable` |
| Generation | () -> Output | `Supplier` |
| Binary Operations | (Input, Input) -> Output | `BinaryOperator` |
| Combining | (Input1, Input2) -> Output | `BiFunction` |

---

## 10. Why Java 8 Was Transformational

### Historical Context

Before Java 8 (1995-2013):
- Java was imperative/object-oriented only
- Had to use verbose anonymous classes for callbacks
- Null handling was dangerous (`NullPointerException`)
- Date/Time APIs were a laughingstock (Date is mutable!)
- No elegant data processing pipelines

### The Transformation

```
Pre-Java 8:
┌──────────────────────────────────────────┐
│ Imperative + Verbose + Error-Prone       │
│                                          │
│ for (Transaction tx : list) {            │
│     if (tx.getAmount() > 10000) {        │
│         List<BigDecimal> amounts = ...   │
│     }                                    │
│ }                                        │
│                                          │
│ Anonymous inner classes everywhere       │
│ Date bugs, null checks, boilerplate      │
└──────────────────────────────────────────┘

Post-Java 8:
┌──────────────────────────────────────────┐
│ Functional + Concise + Declarative       │
│                                          │
│ list.stream()                            │
│     .filter(tx -> tx.getAmount() > 10000)│
│     .map(Transaction::getAmount)         │
│     .collect(toList());                  │
│                                          │
│ Lambdas, Optionals, Modern APIs          │
│ Parallel processing, immutability        │
└──────────────────────────────────────────┘
```

### Impact on the Industry

1. **Language Evolution Enabled**
   - Java 9 modules, 10-11 local variables, 14-16 records, 17-21 pattern matching
   - None of this would be possible without Java 8's functional foundation

2. **Framework Evolution**
   - Spring Data with query DSLs
   - Spring WebFlux (reactive programming)
   - Kafka Streams (functional topology)
   - Modern testing with streams and assertions

3. **Enterprise Development**
   - Shorter, more readable code
   - Easier to reason about immutable data flows
   - Parallel streams for high-throughput processing
   - Better null handling with Optional

4. **Performance Improvements**
   - Parallel streams enable multi-core utilization
   - Lazy evaluation prevents unnecessary processing
   - JVM optimizations for lambda bytecode

### Backward Compatibility

Java 8's genius was achieving all this while maintaining 100% backward compatibility:
- Old code still works unchanged
- Gradual adoption possible
- Default methods allowed interfaces to evolve
- No version-specific class files needed

---

## 11. Interview Questions & Answers

### Question 1: Explain Why Variables in Lambdas Must Be Effectively Final

**Answer**:
Variables captured by lambdas must be effectively final because lambdas create closures over the variable's VALUE, not a reference to the variable itself. If the variable could be reassigned, it would create race conditions in multithreaded scenarios.

```java
int multiplier = 5;

// If multiplier could be changed, threads would see inconsistent values
list.parallelStream()
    .map(n -> n * multiplier)  // What value of multiplier?
    .collect(toList());

// multiplier = 10;  // Would break thread safety
```

This design enforces functional programming principles - no shared mutable state across closures.

---

### Question 2: When Would You Use CompletableFuture vs Streams vs Regular Threads?

**Answer**:
- **Streams**: Data transformation and processing in-process. Good for: filtering, mapping, collecting data from collections.

- **CompletableFuture**: Async operations that might block (I/O, network calls). Good for: calling external services, parallel independent operations, non-blocking architectures.

- **Regular Threads**: Long-running background tasks. Good for: daemon threads, batch processing, monitoring tasks.

```java
// Streams - process local data
List<Transaction> largeTransactions = transactions.stream()
    .filter(tx -> tx.getAmount() > 10000)
    .collect(toList());

// CompletableFuture - call external services
CompletableFuture<CustomerData> customer =
    CompletableFuture.supplyAsync(() -> callCRMService(customerId));

// Regular Thread - long background task
new Thread(() -> {
    while (true) {
        reconcileTransactions();
        Thread.sleep(3600000);  // Every hour
    }
}).start();
```

---

### Question 3: What's the Difference Between map() and flatMap() in Streams?

**Answer**:
- **map()**: Transforms each element with a function, wrapping results. Use when function returns a simple value.
- **flatMap()**: Transforms each element into a stream, then flattens all streams. Use when function returns a Stream/Collection.

```java
// map - wraps each result
List<Integer> lengths = words.stream()
    .map(String::length)  // String -> Integer
    .collect(toList());   // List<Integer>

// flatMap - flattens results
List<String> allChars = words.stream()
    .flatMap(word -> word.chars()  // String -> IntStream
        .mapToObj(c -> String.valueOf((char)c)))
    .collect(toList());  // Single flattened List<String>

// Real example: database relationships
List<String> allAccountIds = customers.stream()
    .flatMap(customer -> customer.getAccounts().stream())
    .map(Account::getId)
    .collect(toList());
```

---

### Question 4: Describe the Method Resolution Order When a Class Implements Multiple Interfaces with Same Default Method

**Answer**:
Java uses this order to resolve ambiguity:
1. Class implementation takes precedence over interface default
2. Most specific interface (subinterface) takes precedence
3. If ambiguous between unrelated interfaces: compiler error

```java
interface Auditable {
    default void process() { System.out.println("Audit"); }
}

interface Loggable {
    default void process() { System.out.println("Log"); }
}

// Must explicitly resolve
public class Reporter implements Auditable, Loggable {
    @Override
    public void process() {
        Auditable.super.process();  // Call Auditable's version
        Loggable.super.process();   // Call Loggable's version
    }
}

// Subinterface takes precedence
interface EnhancedAuditable extends Auditable {
    @Override
    default void process() { System.out.println("Enhanced Audit"); }
}

public class BetterReporter implements EnhancedAuditable, Loggable {
    // No conflict - uses EnhancedAuditable's version
}
```

---

### Question 5: What are the Key Differences Between LocalDateTime and ZonedDateTime in Banking Systems?

**Answer**:
- **LocalDateTime**: Date and time without timezone context. Ambiguous and dangerous for cross-timezone systems. Never use for timestamps in banking.
- **ZonedDateTime**: Date, time, AND timezone. Precise, unambiguous. Always use for trading, settlement, and cross-timezone operations.

```java
// WRONG - ambiguous
LocalDateTime tradeTime = LocalDateTime.now();  // Which timezone?
// Saved as "2024-11-20 14:30:00" - at what timezone was trade executed?

// RIGHT - unambiguous
ZonedDateTime tradeTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
// Saved as "2024-11-20 14:30:00-05:00" - clearly EST
// Can convert to any other timezone without ambiguity

// Conversion is WRONG without timezone context
ZonedDateTime.of(LocalDateTime.now(), ZoneId.of("UTC"));  // Assumption!
```

In banking, always use UTC for storage and conversion for display:
```java
ZonedDateTime tradeTime = ZonedDateTime.now(ZoneId.of("UTC"));
ZonedDateTime displayTimeNY = tradeTime.withZoneSameInstant(ZoneId.of("America/New_York"));
```

---

### Question 6: Why Are Streams Single-Use and What Happens If You Try to Reuse One?

**Answer**:
Streams are single-use by design - once consumed by a terminal operation (like `collect()`), they cannot be reused. This is because streams represent a data pipeline flow, not a collection.

```java
Stream<Integer> stream = Arrays.asList(1, 2, 3).stream();

stream.forEach(System.out::println);  // Terminal operation - consumes stream
stream.forEach(System.out::println);  // IllegalStateException - stream already used!

// Solution: Create new stream or use intermediate collection
List<Integer> list = Arrays.asList(1, 2, 3);
list.stream().forEach(System.out::println);  // OK
list.stream().forEach(System.out::println);  // OK - new stream from list
```

This design enforces functional purity - streams are immutable transformations, not mutable collections.

---

### Question 7: Explain lazy Evaluation in Streams and Its Performance Impact

**Answer**:
Intermediate stream operations (`map`, `filter`, `distinct`) are lazy - they don't execute until a terminal operation forces evaluation. This enables optimization:

```java
// Without streams - evaluates all elements
list.stream()
    .filter(n -> {
        System.out.println("Filter: " + n);  // Prints for EVERY element
        return n > 5;
    })
    .limit(3)  // Take first 3
    .forEach(System.out::println);

// Output:
// Filter: 1
// Filter: 2
// ... (all elements filtered before limiting)

// With lazy evaluation - only necessary elements processed
// The limit() causes only 3 elements to be evaluated through filter()

// Real example - stopping early
List<String> names = getAllUserNames();  // Huge list
Optional<String> first = names.stream()
    .filter(name -> nameMatches(name))  // Not evaluated for all names!
    .findFirst();  // Stops after finding first match
```

This is crucial for performance with large datasets or expensive operations.

---

### Question 8: What's the Difference Between parallelStream() and Regular Stream?

**Answer**:
- **stream()**: Single-threaded, processes elements in order, predictable
- **parallelStream()**: Multi-threaded using ForkJoinPool, processes elements in parallel, non-deterministic order

```java
List<Transaction> transactions = getMillionTransactions();

// Single-threaded
BigDecimal total = transactions.stream()
    .map(Transaction::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);  // Sequential

// Parallel (for LARGE datasets)
BigDecimal total = transactions.parallelStream()
    .map(Transaction::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);  // Parallel

// NOT always better!
// Overhead of thread coordination can exceed benefit for small datasets
// Only use parallelStream() when:
// 1. Dataset is LARGE (thousands+ of elements)
// 2. Operation is CPU-intensive (not I/O)
// 3. No side effects or shared state
```

In banking, be careful:
```java
// BAD - updating shared mutable state
List<Account> accounts = ...;
accounts.parallelStream()
    .forEach(account -> account.balance += 100);  // Race condition!

// GOOD - immutable reduce
BigDecimal totalIncrease = accounts.parallelStream()
    .map(a -> BigDecimal.valueOf(100))
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

---

### Question 9: Explain How Method References Are Resolved at Runtime

**Answer**:
Method references are converted to lambda expressions by the compiler, then invoke the appropriate method at runtime:

```java
// Method reference
list.forEach(System.out::println);

// Equivalent lambda (what compiler generates)
list.forEach(item -> System.out.println(item));

// Four types of method references:

1. Static method
   Integer::parseInt  ==  str -> Integer.parseInt(str)

2. Instance method on class
   String::toUpperCase  ==  str -> str.toUpperCase()

3. Instance method on object
   logger::log  ==  msg -> logger.log(msg)

4. Constructor
   Integer::new  ==  i -> new Integer(i)
```

At runtime, the JVM uses invokedynamic bytecode instruction to call the appropriate method. This allows JVM optimizations to inline the call, making method references as efficient as direct method calls.

---

### Question 10: How Would You Handle Exceptions in a CompletableFuture Chain?

**Answer**:
Use `exceptionally()` or `handle()` to catch and recover from exceptions in async chains:

```java
// Simple recovery
CompletableFuture<String> result =
    fetchData()
    .exceptionally(ex -> {
        logger.error("Failed to fetch", ex);
        return "default value";
    });

// More control with handle()
CompletableFuture<String> result =
    fetchData()
    .handle((data, ex) -> {
        if (ex != null) {
            // Handle exception
            logger.error("Error: " + ex.getMessage(), ex);
            return "error: " + ex.getMessage();
        }
        // Handle success
        return "success: " + data;
    });

// Nested error handling
CompletableFuture<TransactionResult> result =
    fetchAccount(accountId)
    .thenCompose(account -> processTransaction(account))
    .thenCompose(this::persistTransaction)
    .exceptionally(ex -> {
        if (ex instanceof AccountNotFoundException) {
            return new TransactionResult("ACCOUNT_NOT_FOUND", null);
        }
        return new TransactionResult("FAILED", null);
    });

// Note: catch exceptions before they occur
// Use validation in thenCompose()
CompletableFuture<Transaction> validated =
    fetchTransaction(id)
    .thenCompose(tx -> {
        if (tx.isValid()) {
            return CompletableFuture.completedFuture(tx);
        }
        return CompletableFuture.failedFuture(
            new InvalidTransactionException("Invalid transaction")
        );
    });
```

---

### Question 11: Compare Declarative (Stream) vs Imperative Approaches in Real Banking Scenario

**Answer**:
Imperative code tells "what to do step-by-step"; declarative code describes "what you want".

**Scenario**: Find total amount of all transactions over $10,000 from the last month for a specific account, sorted by amount.

```java
// IMPERATIVE - "how to do it"
LocalDate oneMonthAgo = LocalDate.now().minusMonths(1);
BigDecimal total = BigDecimal.ZERO;
List<BigDecimal> sorted = new ArrayList<>();

for (Transaction tx : transactions) {
    if (tx.getAccountId().equals(accountId)) {  // Step 1: filter by account
        LocalDate txDate = tx.getTimestamp().toLocalDate();
        if (txDate.isAfter(oneMonthAgo)) {  // Step 2: filter by date
            if (tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0) {  // Step 3: filter by amount
                sorted.add(tx.getAmount());
            }
        }
    }
}
sorted.sort((a, b) -> b.compareTo(a));  // Step 4: sort descending
for (BigDecimal amount : sorted) {
    total = total.add(amount);  // Step 5: sum
}

// DECLARATIVE - "what you want"
LocalDate oneMonthAgo = LocalDate.now().minusMonths(1);
BigDecimal total = transactions.stream()
    .filter(tx -> tx.getAccountId().equals(accountId))
    .filter(tx -> tx.getTimestamp().toLocalDate().isAfter(oneMonthAgo))
    .filter(tx -> tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0)
    .map(Transaction::getAmount)
    .sorted(Collections.reverseOrder())
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// Benefits of declarative:
// 1. Clearer intent - reads like business requirements
// 2. Easier to modify - add/remove filters without touching loop logic
// 3. Thread-safe - no mutable intermediate collections
// 4. Enables optimization - JVM can inline, parallelize, eliminate dead code
// 5. Immutable - original list unchanged
```

---

### Question 12: What are Anti-Patterns to Avoid With Java 8 Features?

**Answer**:

```java
// ANTI-PATTERN 1: Optional.get() without checking
Optional<User> user = findUser(id);
User u = user.get();  // Can throw NoSuchElementException!

// CORRECT:
User u = user.orElseThrow(() -> new UserNotFoundException(id));
user.ifPresent(u -> processUser(u));
user.ifPresentOrElse(
    u -> processUser(u),
    () -> createDefaultUser()
);

// ANTI-PATTERN 2: Using parallelStream() everywhere
List<String> names = smallList.parallelStream()  // Overhead > benefit
    .map(String::toUpperCase)
    .collect(toList());

// CORRECT: Use only for large datasets
List<String> names = millionNames.parallelStream()
    .map(String::toUpperCase)
    .collect(toList());

// ANTI-PATTERN 3: Stateful operations in streams
List<Integer> list = new ArrayList<>();
numbers.stream()
    .forEach(n -> list.add(n * 2));  // Mutable shared state!

// CORRECT:
List<Integer> list = numbers.stream()
    .map(n -> n * 2)
    .collect(toList());

// ANTI-PATTERN 4: Complex nested lambdas (hard to read)
transactions.stream()
    .map(tx -> tx.getAccount())
    .filter(acc -> acc.isActive())
    .flatMap(acc -> acc.getTransactions().stream())
    .filter(t -> t.getAmount() > 1000)
    .forEach(t -> System.out.println(t));

// BETTER: Use method references for clarity
transactions.stream()
    .map(Transaction::getAccount)
    .filter(Account::isActive)
    .flatMap(acc -> acc.getTransactions().stream())
    .filter(this::isLargeAmount)
    .forEach(System.out::println);

// ANTI-PATTERN 5: Ignoring timezone in banking
LocalDateTime tradeTime = LocalDateTime.now();  // Which timezone?
database.save(tradeTime);

// CORRECT: Always include timezone or use UTC
ZonedDateTime tradeTime = ZonedDateTime.now(ZoneId.of("UTC"));
database.save(tradeTime);
```

---

## 12. Summary: Java 8's Legacy and Forward Impact

### What Java 8 Gave Us

1. **Functional Programming** - Changed how Java developers think about code
2. **Streams API** - Elegant data processing without loops
3. **Modern Date/Time API** - Finally, a reliable time library
4. **Interface Evolution** - Default/static methods enabled framework innovation
5. **Async Support** - CompletableFuture foundation for reactive systems

### Why It Still Matters Today (2024+)

- Java 17-21 builds on Java 8 foundation (streams, lambdas, immutability)
- Pattern matching (Java 17+) evolves functional concepts
- Records (Java 16+) embrace immutability principle
- Text blocks, sealed classes, modules all follow Java 8's design philosophy
- Enterprise systems still run Java 8 LTS (support until 2030)

### For Interviews

Java 8 is the **baseline expectation** for senior roles. Interviewers expect you to:
- Use streams naturally, not as a novelty
- Understand the reasoning behind Optional
- Know when and why to use CompletableFuture
- Write immutable, functional-style code
- Handle dates/times correctly (no timezone bugs!)

The candidates who understand the "why" behind Java 8 features (functional purity, immutability, lazy evaluation) will clearly stand out from those who just know the syntax.

---

## References

- [Java 8 Date/Time API - JCP](https://jcp.org/en/jsr/detail?id=310)
- [CompletableFuture - Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
- [Functional Interfaces - Java Documentation](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)
- [Stream API - Comprehensive Guide](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)
- "Effective Java" (3rd Edition) - Items 43-48 (Lambdas and Streams)
- Stephen Colebourne's Blog on java.time API design
