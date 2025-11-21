# Java Streams and Functional Programming

## Overview

Java 8 introduced functional programming to Java through lambda expressions, functional interfaces, and the Stream API. These features enable declarative, functional style code for data processing while maintaining backward compatibility. Understanding streams and functional programming is critical for modern Java development and a frequent interview topic.

**Why This Matters**:
- Core to Java 8+ development
- Enables elegant data transformation pipelines
- Foundation for reactive programming and complex queries
- Fundamental for system design discussions involving data processing

**Real-World Context**:
In banking systems, streams are essential for:
- Processing transaction records from databases
- Filtering compliance violations from logs
- Aggregating financial metrics across accounts
- Building data warehouses for analytics

---

## 1. Lambda Expressions

### What Are Lambda Expressions?

A lambda expression is a concise way to represent a function as an anonymous method. It enables you to pass code as a parameter to methods.

**Syntax**:
```java
(parameters) -> { body }
```

### Lambda Expression Rules

| Rule | Example |
|------|---------|
| No parameters | `() -> System.out.println("Hello")` |
| One parameter (optional parentheses) | `x -> x * 2` or `(x) -> x * 2` |
| Multiple parameters | `(a, b) -> a + b` |
| Single expression (implicit return) | `(x, y) -> x + y` |
| Multiple statements | `(x) -> { System.out.println(x); return x * 2; }` |

### Lambda Expression Scope

```java
int multiplier = 5;  // effectively final

List<Integer> numbers = Arrays.asList(1, 2, 3);

// This works - multiplier is effectively final
numbers.forEach(n -> System.out.println(n * multiplier));

// This FAILS - reassignment breaks "effectively final"
// multiplier = 10;
```

**Key Point**: Variables used in lambda expressions must be **effectively final** (not modified after assignment). This enables thread-safe functional operations.

---

## 2. Functional Interfaces

A **functional interface** is an interface with exactly ONE abstract method. It represents a contract for a function.

### Core Functional Interfaces

```
┌─────────────────────────────────────────────────────────────┐
│              Functional Interface Hierarchy                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Function<T, R>: T -> R                                      │
│  ├─ UnaryOperator<T>: T -> T                                │
│  └─ BinaryOperator<T>: (T, T) -> T                          │
│                                                               │
│  Predicate<T>: T -> boolean                                 │
│                                                               │
│  Consumer<T>: T -> void                                     │
│  └─ BiConsumer<T, U>: (T, U) -> void                        │
│                                                               │
│  Supplier<T>: () -> T                                       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### The Four Core Interfaces

#### 1. Function<T, R> - Transform

```java
// Transforms input of type T to output of type R
Function<String, Integer> stringLength = str -> str.length();
Function<Integer, Integer> double_it = n -> n * 2;

System.out.println(stringLength.apply("Hello"));  // 5

// Composition
Function<String, Integer> lengthDoubled =
    stringLength.andThen(double_it);
System.out.println(lengthDoubled.apply("Hello")); // 10

// Real-world: Transform transaction to amount
Function<Transaction, BigDecimal> getAmount = tx -> tx.getAmount();
```

#### 2. Predicate<T> - Filter

```java
// Tests input and returns boolean
Predicate<Integer> isPositive = n -> n > 0;
Predicate<String> isLongerThan5 = str -> str.length() > 5;

System.out.println(isPositive.test(5));    // true
System.out.println(isPositive.test(-5));   // false

// Logical operations
Predicate<String> isValidEmail =
    str -> str.contains("@") && str.contains(".");
Predicate<String> notEmpty = str -> !str.isEmpty();

Predicate<String> validNonEmptyEmail =
    isValidEmail.and(notEmpty);

// Real-world: Filter large transactions
Predicate<Transaction> isLargeTransaction =
    tx -> tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0;
```

#### 3. Consumer<T> - Side Effects

```java
// Accepts input, returns nothing (side effects)
Consumer<String> printUpperCase = str -> System.out.println(str.toUpperCase());
printUpperCase.accept("hello");  // Prints: HELLO

// Chaining consumers
Consumer<Transaction> logTransaction = tx ->
    System.out.println("Transaction: " + tx.getId());
Consumer<Transaction> auditTransaction = tx ->
    auditLog.write(tx);

Consumer<Transaction> processTransaction =
    logTransaction.andThen(auditTransaction);

processTransaction.accept(transaction);

// Real-world: Database operations
Consumer<User> saveUser = user -> userRepository.save(user);
Consumer<User> sendNotification = user -> emailService.send(user.getEmail());
```

#### 4. Supplier<T> - Generate

```java
// No parameters, returns value (lazy evaluation)
Supplier<String> getCurrentTime = () -> LocalDateTime.now().toString();
System.out.println(getCurrentTime.get());

Supplier<Integer> randomNumber = () -> new Random().nextInt(100);

// Real-world: Lazy initialization
Supplier<DatabaseConnection> connectionPool = () ->
    connectionFactory.createConnection();

// Called only when needed
DatabaseConnection conn = connectionPool.get();
```

### Functional Interface Comparison

| Interface | Signature | Use Case | Example |
|-----------|-----------|----------|---------|
| `Function<T,R>` | `T -> R` | Transform/Map | `String -> Integer` (length) |
| `Predicate<T>` | `T -> boolean` | Test/Filter | Check if `String.length > 5` |
| `Consumer<T>` | `T -> void` | Side effects | Print or save |
| `Supplier<T>` | `() -> T` | Generate/Create | Lazy initialization |
| `UnaryOperator<T>` | `T -> T` | Transform (same type) | `Integer -> Integer` (double) |
| `BinaryOperator<T>` | `(T,T) -> T` | Reduce two values | `(Integer, Integer) -> Integer` |

---

## 3. Method References

Method references provide a concise syntax to reference existing methods. They replace lambda expressions when the lambda just calls an existing method.

### Four Types of Method References

```
┌─────────────────────────────────────────────────────┐
│          Method Reference Types                      │
├─────────────────────────────────────────────────────┤
│                                                       │
│  1. Static method      ClassName::staticMethod      │
│  2. Instance method    object::instanceMethod       │
│  3. Constructor        ClassName::new               │
│  4. Arbitrary object   ClassName::instanceMethod    │
│                                                       │
└─────────────────────────────────────────────────────┘
```

#### Type 1: Static Method Reference

```java
// Lambda
Function<String, Integer> parse = str -> Integer.parseInt(str);

// Method reference
Function<String, Integer> parse = Integer::parseInt;

// Real-world usage
List<String> numbers = Arrays.asList("1", "2", "3");
List<Integer> ints = numbers.stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());
```

#### Type 2: Instance Method Reference

```java
// Lambda
Consumer<String> print = str -> System.out.println(str);

// Method reference
Consumer<String> print = System.out::println;

// Real-world usage
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
names.forEach(System.out::println);

// Instance variable method
User user = new User();
Supplier<String> getName = user::getName;
```

#### Type 3: Constructor Reference

```java
// Lambda
Supplier<StringBuilder> supplier = () -> new StringBuilder();

// Constructor reference
Supplier<StringBuilder> supplier = StringBuilder::new;

// With parameters
Function<String, User> createUser = User::new;  // Calls User(String name)

// Real-world usage
List<String> data = Arrays.asList("Alice", "Bob", "Charlie");
List<User> users = data.stream()
    .map(User::new)
    .collect(Collectors.toList());
```

#### Type 4: Arbitrary Object Method Reference

```java
// Lambda
Comparator<String> comparator = (s1, s2) -> s1.compareTo(s2);

// Method reference (String::compareTo called on first arg)
Comparator<String> comparator = String::compareTo;

// Sorts using this comparator
List<String> words = Arrays.asList("zebra", "apple", "banana");
words.sort(String::compareTo);

// Real-world usage
List<Transaction> transactions = fetchTransactions();
// Filter transactions where compareTo returns > 0
transactions.stream()
    .filter(tx -> tx.getTimestamp().compareTo(LocalDateTime.now()) < 0)
    .collect(Collectors.toList());
```

### When to Use Method References

| Situation | Use Method Reference | Example |
|-----------|----------------------|---------|
| Just calling a static method | Yes | `Integer::parseInt` |
| Just calling an instance method | Yes | `System.out::println` |
| Just calling a constructor | Yes | `ArrayList::new` |
| Complex logic | No | Lambda with body |
| Need clarity for team | Depends | Document if unclear |

---

## 4. Stream API - Intermediate & Terminal Operations

A **Stream** is a sequence of elements supporting sequential and parallel operations. Streams are **lazy** - intermediate operations don't execute until a terminal operation is called.

```
Data Source → Intermediate Op → Intermediate Op → Terminal Op → Result
  (Lazy)          (Lazy)          (Lazy)          (Executes)
```

### Intermediate Operations (Lazy)

These return a new Stream, allowing chaining. They don't execute until a terminal operation is called.

#### filter(Predicate)
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6);
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
// Result: [2, 4, 6]
```

#### map(Function)
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
List<Integer> lengths = names.stream()
    .map(String::length)
    .collect(Collectors.toList());
// Result: [5, 3, 7]
```

#### flatMap(Function<T, Stream<R>>)
```java
List<List<Integer>> nestedLists = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4),
    Arrays.asList(5, 6)
);

List<Integer> flattened = nestedLists.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());
// Result: [1, 2, 3, 4, 5, 6]

// Real-world: Get all transactions from all accounts
List<Transaction> allTransactions = accounts.stream()
    .flatMap(account -> account.getTransactions().stream())
    .collect(Collectors.toList());
```

#### distinct()
```java
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 3, 4);
List<Integer> unique = numbers.stream()
    .distinct()
    .collect(Collectors.toList());
// Result: [1, 2, 3, 4]
```

#### sorted()
```java
List<String> words = Arrays.asList("zebra", "apple", "banana");

// Natural order
words.stream()
    .sorted()
    .forEach(System.out::println);

// Custom comparator
words.stream()
    .sorted((a, b) -> b.compareTo(a))  // Reverse order
    .forEach(System.out::println);

// Real-world: Sort transactions by amount
transactions.stream()
    .sorted(Comparator.comparing(Transaction::getAmount))
    .collect(Collectors.toList());
```

#### peek(Consumer)
```java
// Debugging aid - performs action without changing stream
List<Integer> result = Arrays.asList(1, 2, 3, 4, 5).stream()
    .peek(n -> System.out.println("Processing: " + n))
    .filter(n -> n > 2)
    .peek(n -> System.out.println("After filter: " + n))
    .collect(Collectors.toList());
```

#### limit(long) and skip(long)
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Skip first 3, take 4
List<Integer> result = numbers.stream()
    .skip(3)
    .limit(4)
    .collect(Collectors.toList());
// Result: [4, 5, 6, 7]
```

### Terminal Operations (Execute)

These consume the stream and produce a result or side effect. After a terminal operation, the stream is consumed and cannot be reused.

#### collect(Collector)
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// To List
List<String> list = names.stream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());

// To Set
Set<String> set = names.stream()
    .collect(Collectors.toSet());

// To String (joining)
String result = names.stream()
    .collect(Collectors.joining(", "));
// Result: "Alice, Bob, Charlie"
```

#### forEach(Consumer)
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
names.stream()
    .forEach(System.out::println);

// Or simply:
names.forEach(System.out::println);  // Deprecated in favor of streams for parallel safety
```

#### reduce(BinaryOperator)
```java
// Sum all numbers
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
int sum = numbers.stream()
    .reduce(0, (acc, n) -> acc + n);
// Result: 15

// Multiply all numbers
int product = numbers.stream()
    .reduce(1, (acc, n) -> acc * n);
// Result: 120

// String concatenation
List<String> words = Arrays.asList("Hello", "World");
String result = words.stream()
    .reduce("", (acc, w) -> acc + " " + w).trim();
// Result: "Hello World"

// Real-world: Calculate total transaction amount
BigDecimal total = transactions.stream()
    .map(Transaction::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

#### match operations
```java
List<Integer> numbers = Arrays.asList(2, 4, 6, 8, 10);

// All match condition
boolean allEven = numbers.stream()
    .allMatch(n -> n % 2 == 0);  // true

// Any match condition
boolean hasGreaterThan5 = numbers.stream()
    .anyMatch(n -> n > 5);  // true

// None match condition
boolean noneOdd = numbers.stream()
    .noneMatch(n -> n % 2 != 0);  // true
```

#### count(), min(), max(), findFirst(), findAny()
```java
List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6);

long count = numbers.stream().count();  // 8

Optional<Integer> min = numbers.stream().min(Integer::compareTo);
Optional<Integer> max = numbers.stream().max(Integer::compareTo);

Optional<Integer> first = numbers.stream().findFirst();
Optional<Integer> any = numbers.stream().findAny();  // Better for parallel
```

---

## 5. Collectors

Collectors are used with `collect()` to accumulate stream elements into containers or perform aggregations.

### Common Collectors

#### toList(), toSet(), toCollection()
```java
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());

// Specific collection type
TreeSet<String> treeSet = stream.collect(
    Collectors.toCollection(TreeSet::new)
);
```

#### joining()
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
String result = names.stream()
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"

String result = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie]"
```

#### groupingBy()

Groups elements by a classifier function.

```java
// Group strings by length
List<String> words = Arrays.asList("apple", "ant", "bear", "bat", "cat");
Map<Integer, List<String>> groupedByLength = words.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=[ant, bat, cat], 4=[bear], 5=[apple]}

// Real-world: Group transactions by account
Map<String, List<Transaction>> txByAccount = transactions.stream()
    .collect(Collectors.groupingBy(Transaction::getAccountId));

// With downstream collector - count transactions per account
Map<String, Long> txCountByAccount = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::getAccountId,
        Collectors.counting()
    ));

// Sum transaction amounts by account
Map<String, BigDecimal> totalByAccount = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::getAccountId,
        Collectors.mapping(
            Transaction::getAmount,
            Collectors.reducing(BigDecimal.ZERO, BigDecimal::add)
        )
    ));
```

#### partitioningBy()

Special case of groupingBy that divides into two groups (true/false).

```java
// Partition numbers into even/odd
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6);
Map<Boolean, List<Integer>> evenOdd = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// {false=[1, 3, 5], true=[2, 4, 6]}

// Real-world: Partition transactions into large and small
Map<Boolean, List<Transaction>> partitioned = transactions.stream()
    .collect(Collectors.partitioningBy(
        tx -> tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0
    ));

Map<Boolean, Long> counts = transactions.stream()
    .collect(Collectors.partitioningBy(
        tx -> tx.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0,
        Collectors.counting()
    ));
```

#### summarizing collectors
```java
// IntSummaryStatistics
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
IntSummaryStatistics stats = numbers.stream()
    .collect(Collectors.summarizingInt(Integer::intValue));

System.out.println(stats.getCount());   // 5
System.out.println(stats.getSum());     // 15
System.out.println(stats.getAverage()); // 3.0
System.out.println(stats.getMin());     // 1
System.out.println(stats.getMax());     // 5
```

### Collector Composition Flow

```
Stream → groupingBy(classifier, downstream) → Result
                           ↓
                  Processes each group
                    with downstream
```

---

## 6. Parallel Streams

Parallel streams enable processing using multiple threads for potential performance gains.

### Creating Parallel Streams

```java
// From collection
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
numbers.parallelStream()
    .filter(n -> n > 2)
    .forEach(System.out::println);

// From regular stream
numbers.stream()
    .parallel()
    .filter(n -> n > 2)
    .forEach(System.out::println);

// Convert back to sequential
numbers.stream()
    .parallel()
    .filter(n -> n > 2)
    .sequential()
    .forEach(System.out::println);
```

### Parallel Stream Performance

```java
// Performance is only gained with:
// 1. Large datasets (1000s+ elements)
// 2. Expensive operations (database calls, complex computations)
// 3. Stateless operations (no shared state modifications)

List<Integer> largeList = generateMillionIntegers();

// Good candidate for parallelization
long sum = largeList.parallelStream()
    .filter(n -> isPrime(n))          // Expensive operation
    .map(n -> expensiveCalculation(n))  // Expensive operation
    .reduce(0, Integer::sum);

// NOT suitable - forEach modifies shared state (UNSAFE)
List<String> results = new ArrayList<>();
largeList.parallelStream()
    .forEach(n -> results.add(String.valueOf(n)));  // WRONG!
```

### Parallel vs Sequential Comparison

| Aspect | Sequential | Parallel |
|--------|-----------|----------|
| Thread Count | 1 | Multiple (ForkJoinPool) |
| Overhead | Minimal | Splitting, merging |
| Best for | Small datasets | Large datasets |
| Ordering | Guaranteed | May not be preserved |
| Unsafe operations | All safe | Only stateless ops safe |
| Lock-free | Yes | Not always |

### When to Use Parallel Streams

**Use Parallel When**:
- Dataset > 1,000 elements
- Operation is CPU-intensive or blocking I/O
- No shared mutable state
- Stateless operations (map, filter, reduce)

**Avoid Parallel When**:
- Small datasets (< 100 elements)
- Operations are fast (string operations, simple arithmetic)
- Shared mutable state (thread-unsafe)
- Fine-grained control needed (use ExecutorService instead)

---

## 7. Optional Basics

Optional is a container for a possibly-null value, encouraging explicit null handling and reducing NullPointerExceptions.

### Creating Optionals

```java
// Empty Optional
Optional<String> empty = Optional.empty();

// Optional with value
Optional<String> name = Optional.of("Alice");

// Optional that might be empty
String value = null;
Optional<String> maybe = Optional.ofNullable(value);
```

### Using Optional

```java
Optional<String> name = Optional.of("Alice");

// Check if present
if (name.isPresent()) {
    System.out.println(name.get());
}

// Or use orElse
String result = name.orElse("Unknown");

// Or use orElseGet (lazy evaluation)
String result = name.orElseGet(() -> fetchDefaultName());

// Or throw exception
String result = name.orElseThrow(() ->
    new IllegalArgumentException("Name is missing")
);

// Or consume
name.ifPresent(System.out::println);

// Combined with if-else
name.ifPresentOrElse(
    n -> System.out.println("Name: " + n),
    () -> System.out.println("No name provided")
);
```

### Optional with Streams

```java
// Filter with Optional
Optional<User> user = findUserById(123);
Optional<String> email = user
    .filter(u -> u.isActive())
    .map(User::getEmail);

// flatMap for chaining operations
Optional<String> department = findUserById(123)
    .flatMap(User::getDepartment)
    .flatMap(Department::getManager)
    .map(Manager::getName);

// Stream of Optionals
List<Optional<User>> maybeUsers = Arrays.asList(
    Optional.of(user1),
    Optional.empty(),
    Optional.of(user3)
);

List<User> users = maybeUsers.stream()
    .flatMap(Optional::stream)  // Flat-map empty to zero elements
    .collect(Collectors.toList());
```

### Optional Anti-patterns

```java
// WRONG - defeats the purpose of Optional
Optional<String> name = Optional.of("Alice");
if (name.isPresent()) {
    String value = name.get();  // Just check if != null instead
}

// WRONG - can throw NoSuchElementException
String value = Optional.empty().get();  // Throws!

// CORRECT - use orElse or orElseThrow
String value = Optional.empty()
    .orElseThrow(() -> new IllegalArgumentException("Required"));

// CORRECT - use ifPresent for side effects
Optional.of("Alice").ifPresent(System.out::println);
```

---

## 8. Stream Pipeline Example: Banking Use Case

```java
// Real-world scenario: Find top 5 accounts by transaction volume,
// only include active accounts, group by customer, show details

public class TransactionAnalyzer {

    public Map<String, AccountSummary> analyzeTopAccounts(
        List<Account> accounts) {

        return accounts.stream()
            .filter(Account::isActive)                    // Intermediate
            .filter(acc -> !acc.getTransactions().isEmpty()) // Intermediate
            .flatMap(acc -> acc.getTransactions().stream()
                .map(tx -> new AccountTx(acc.getId(), tx)))  // Intermediate
            .collect(Collectors.groupingBy(
                AccountTx::getAccountId,                  // Group by account
                Collectors.collectingAndThen(            // Custom reduction
                    Collectors.toList(),
                    txList -> new AccountSummary(
                        txList.size(),
                        txList.stream()
                            .map(AccountTx::getAmount)
                            .reduce(BigDecimal.ZERO, BigDecimal::add),
                        txList.stream()
                            .map(AccountTx::getTimestamp)
                            .max(LocalDateTime::compareTo)
                            .orElse(null)
                    )
                )
            ))
            .entrySet().stream()
            .sorted((e1, e2) ->
                e2.getValue().getTransactionCount()
                    .compareTo(e1.getValue().getTransactionCount()))
            .limit(5)
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue
            ));
    }
}
```

---

## 9. Common Pitfalls and Solutions

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Stream reuse | `Stream closed` | Create new stream each time |
| Modifying during iteration | Concurrent modification | Collect first, then modify |
| Peek debugging | Not executed in sequential | Add terminal operation |
| Shared mutable state in parallel | Race conditions, data loss | Use thread-safe collectors |
| Optional.get() without check | NoSuchElementException | Use orElse or orElseThrow |
| All operations are parallel | Performance degradation | Check dataset size first |
| Lazy evaluation surprise | Code doesn't execute | Remember to add terminal op |
| `forEach` for all side effects | Order not guaranteed in parallel | Use `forEachOrdered` if needed |

---

## 10. Interview Questions & Answers

### Q1: What's the difference between intermediate and terminal operations?

**Answer**:
Intermediate operations are **lazy** - they return a Stream and don't execute until a terminal operation is called. They enable chaining multiple operations efficiently.

Terminal operations **consume** the stream and produce a result or side effect. Examples: `collect()`, `forEach()`, `reduce()`, `count()`.

```java
// No execution happens until collect() (terminal)
numbers.stream()
    .filter(n -> {
        System.out.println("Filter: " + n);  // Never prints
        return n > 2;
    })
    .map(n -> {
        System.out.println("Map: " + n);     // Never prints
        return n * 2;
    });
    // No terminal operation - nothing executes!

// Now execution happens
    .collect(Collectors.toList());  // NOW everything executes in order
```

**Interview Tip**: Emphasize lazy evaluation as a performance optimization - computations only run when results are needed.

---

### Q2: Explain method references vs lambda expressions. When would you use each?

**Answer**:
Method references are syntactic sugar for lambdas that simply call an existing method. They improve readability when applicable.

**Lambda**: `name -> name.length()` (explicit parameter)
**Method Reference**: `String::length` (concise, references method directly)

Use method references when:
- Just calling a method without transformation
- Code is more readable
- Refactoring to avoid "arrow hell"

Use lambdas when:
- Logic is more complex than a method call
- You need multiple lines or conditions
- Only used once (no method to reference)

```java
// Good method reference - clearly intent
list.forEach(System.out::println);

// Good lambda - complex logic
list.stream()
    .filter(name -> name.length() > 5 && !name.startsWith("_"))
    .collect(Collectors.toList());
```

**Interview Tip**: Show you understand the trade-off between conciseness and clarity.

---

### Q3: When should you use parallel streams?

**Answer**:
Only when:
1. **Large dataset**: > 1,000 elements (overhead amortization)
2. **CPU-intensive operations**: Complex calculations, not I/O bound
3. **Stateless operations**: No shared mutable state
4. **Unordered**: Don't need guarantees

**Avoid** for:
- Small datasets
- Simple fast operations
- Operations with shared mutable state
- When you need ordered results from forEach

```java
// GOOD: Large dataset, expensive CPU operation
List<Integer> largeList = generateMillionNumbers();
long result = largeList.parallelStream()
    .map(this::expensiveCalculation)  // CPU-intensive
    .filter(n -> n > threshold)
    .count();

// BAD: Small dataset, minimal overhead not worth it
List<Integer> small = Arrays.asList(1, 2, 3);
small.parallelStream()
    .map(n -> n * 2)  // Too fast for parallel overhead
    .collect(Collectors.toList());

// BAD: Modifying shared state (thread-unsafe)
Set<Integer> results = new HashSet<>();
largeList.parallelStream()
    .forEach(n -> results.add(n));  // RACE CONDITION!
```

**Interview Tip**: Show you understand the **ForkJoinPool** overhead and when it's worth the cost.

---

### Q4: Explain groupingBy vs partitioningBy with an example.

**Answer**:
- **`groupingBy`**: Creates multiple groups based on a classifier function. Uses **key-value pairs** in Map.
- **`partitioningBy`**: Special case of groupingBy that divides into exactly **two groups** (true/false).

```java
// groupingBy - multiple groups
List<String> words = Arrays.asList("apple", "apricot", "banana", "blueberry");
Map<Character, List<String>> byFirstLetter = words.stream()
    .collect(Collectors.groupingBy(w -> w.charAt(0)));
// {a=[apple, apricot], b=[banana, blueberry]}

// partitioningBy - exactly two groups
Map<Boolean, List<String>> longVsShort = words.stream()
    .collect(Collectors.partitioningBy(w -> w.length() > 6));
// {false=[apple, banana], true=[apricot, blueberry]}

// Real-world banking example
List<Transaction> txs = fetchTransactions();

// groupingBy - group by status (multiple groups)
Map<TransactionStatus, List<Transaction>> byStatus = txs.stream()
    .collect(Collectors.groupingBy(Transaction::getStatus));

// partitioningBy - split into settled/unsettled
Map<Boolean, List<Transaction>> settled = txs.stream()
    .collect(Collectors.partitioningBy(
        tx -> tx.getStatus() == TransactionStatus.SETTLED
    ));
```

**Interview Tip**: Mention that `partitioningBy` is more memory-efficient for binary splits since it has only 2 keys.

---

### Q5: What's the difference between map and flatMap?

**Answer**:
- **`map(Function<T, R>)`**: Transforms each element into one result. Stream<T> → Stream<R>
- **`flatMap(Function<T, Stream<R>>)`**: Transforms each element into a Stream, then **flattens** all streams into one. Stream<Stream<R>> → Stream<R>

```java
// map - one-to-one
List<String> words = Arrays.asList("hello", "world");
List<Integer> lengths = words.stream()
    .map(String::length)  // Each word -> one length
    .collect(Collectors.toList());
// [5, 5]

// flatMap - one-to-many then flatten
List<String> words = Arrays.asList("hello", "world");
List<Character> chars = words.stream()
    .flatMap(word -> word.chars()  // Each word -> many chars
        .mapToObj(c -> (char) c))
    .collect(Collectors.toList());
// [h, e, l, l, o, w, o, r, l, d]

// Real-world: Flatten account transactions
List<Account> accounts = fetchAccounts();
List<Transaction> allTx = accounts.stream()
    .flatMap(acc -> acc.getTransactions().stream())
    .collect(Collectors.toList());
```

**Interview Tip**: Draw a diagram showing how flatMap "unwraps" nested streams.

---

### Q6: What happens if you call a terminal operation twice on the same stream?

**Answer**:
You get a **`IllegalStateException`** because streams are **one-time consumable**. Once a terminal operation is called, the stream is closed.

```java
Stream<Integer> stream = Arrays.asList(1, 2, 3).stream();

stream.forEach(System.out::println);  // Terminal operation
stream.forEach(System.out::println);  // THROWS: IllegalStateException
                                      // stream already operated upon or closed

// Solution: Create a new stream
List<Integer> list = Arrays.asList(1, 2, 3);
list.stream().forEach(System.out::println);
list.stream().forEach(System.out::println);  // OK - separate streams
```

**Interview Tip**: Mention this is a feature, not a bug - enforces functional programming purity.

---

### Q7: Explain the difference between Optional.of() and Optional.ofNullable().

**Answer**:
- **`of(T)`**: Throws **NullPointerException** if value is null. Use when you know value is non-null.
- **`ofNullable(T)`**: Returns empty Optional if value is null. Use for potentially null values.

```java
// of - expect non-null
String name = "Alice";
Optional<String> opt = Optional.of(name);  // OK

String maybeNull = null;
Optional<String> opt = Optional.of(maybeNull);  // NPE!

// ofNullable - handle possibly null
String maybeNull = null;
Optional<String> opt = Optional.ofNullable(maybeNull);  // OK, returns empty()

// Real-world
User user = findUserById(123);
Optional<User> userOpt = Optional.ofNullable(user);  // Safe
Optional<String> email = userOpt.map(User::getEmail);  // Safe chaining
```

**Interview Tip**: Explain that `of()` is for **contract guarantees** (design-by-contract) while `ofNullable()` is for **defensive programming**.

---

### Q8: How would you implement a custom collector?

**Answer**:
Custom collectors implement the `Collector` interface with four methods:

```java
public class StringBuilderCollector implements Collector<String, StringBuilder, String> {

    @Override
    public Supplier<StringBuilder> supplier() {
        return StringBuilder::new;  // Create mutable accumulator
    }

    @Override
    public BiConsumer<StringBuilder, String> accumulator() {
        return (sb, str) -> sb.append(str).append(",");  // Accumulate
    }

    @Override
    public BinaryOperator<StringBuilder> combiner() {
        return (sb1, sb2) -> sb1.append(sb2);  // Combine for parallel
    }

    @Override
    public Function<StringBuilder, String> finisher() {
        return sb -> sb.toString();  // Transform to final result
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.singleton(Characteristics.CONCURRENT);
    }
}

// Usage
List<String> words = Arrays.asList("hello", "world");
String result = words.stream()
    .collect(new StringBuilderCollector());
// "hello,world,"
```

**Interview Tip**: Show understanding of **mutable reduction** and why collectors need supplier, accumulator, combiner, and finisher.

---

### Q9: What's "effectively final" and why does it matter for lambdas?

**Answer**:
A variable is **effectively final** if it's not modified after initialization. Lambda expressions can only access effectively final variables from enclosing scope.

```java
// OK - effectively final
int multiplier = 5;
List<Integer> nums = Arrays.asList(1, 2, 3);
nums.forEach(n -> System.out.println(n * multiplier));

// NOT OK - modified after initialization
int multiplier = 5;
nums.forEach(n -> System.out.println(n * multiplier));
multiplier = 10;  // Reassignment breaks "effectively final"
                   // Compilation error!

// Solution: Use final explicitly or different variable
final int multiplier = 5;
nums.forEach(n -> System.out.println(n * multiplier));  // OK

// Or use effectively final
int multiplier = 5;
int finalMultiplier = multiplier;  // Different variable
nums.forEach(n -> System.out.println(n * finalMultiplier));  // OK
```

**Why It Matters**:
Lambda expressions capture variables by **value**, not by reference. The variable must be effectively final to ensure thread-safety and predictable behavior across multiple calls.

**Interview Tip**: Explain the **closure** concept and why this restriction enables safe parallelization.

---

### Q10: Compare reduce vs collect - when would you use each?

**Answer**:
- **`reduce()`**: Combines elements into a single value. Use for **immutable** aggregations.
- **`collect()`**: Combines elements into a container (list, set, map). Use for **mutable** accumulations.

```java
// reduce - immutable aggregation
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
int sum = numbers.stream()
    .reduce(0, Integer::sum);  // Single value
// 15

// collect - mutable container
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());  // Container
// [2, 4]

// reduce requires combining operation (parallel-safe)
// collect handles complexity through Collector (supplier, combiner, finisher)

// Real-world
List<Transaction> txs = fetchTransactions();

// Use reduce for simple aggregations
BigDecimal total = txs.stream()
    .map(Transaction::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// Use collect for complex aggregations
Map<String, List<Transaction>> byAccount = txs.stream()
    .collect(Collectors.groupingBy(Transaction::getAccountId));
```

**Interview Tip**: Mention that `reduce()` is simpler but `collect()` is more powerful for complex aggregations. `collect()` handles parallel merging automatically.

---

### Q11: What's the performance impact of using streams vs loops?

**Answer**:
Streams have overhead (function calls, intermediate object creation) but can be faster in certain scenarios:

**Streams are Slower When**:
- Simple iterations with direct index access
- Very small datasets
- Cache locality matters (arrays better than streams)

**Streams are Faster When**:
- Parallel processing (multi-core advantage)
- Multiple chained operations (intermediate streams reuse)
- Lazy evaluation skips unnecessary work
- Better JIT compilation optimizations (in modern JVMs)

```java
// Stream - one pass, lazy, potential parallel
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * 2)
    .collect(Collectors.toList());

// Loop - imperative, explicit, predictable
List<Integer> evens = new ArrayList<>();
for (Integer n : numbers) {
    if (n % 2 == 0) {
        evens.add(n * 2);
    }
}

// Benchmark: Streams ~10% slower for simple iteration (small datasets)
// Streams ~2x faster with parallel operations (large datasets, expensive ops)
```

**Interview Tip**: Show you understand trade-offs and can choose appropriately. Mention that readability often matters more than micro-optimizations.

---

### Q12: Design a stream pipeline for a real-world banking scenario.

**Answer**:
Scenario: Find customers with largest balances, who haven't made transactions in 30 days, and group by branch.

```java
public Map<String, List<CustomerSummary>> findInactiveRichCustomers(
    List<Customer> customers,
    LocalDate cutoffDate) {

    return customers.stream()
        // Filter active customers
        .filter(Customer::isActive)

        // Calculate last transaction date
        .map(customer -> new CustomerWithLastTx(
            customer,
            customer.getTransactions().stream()
                .map(Transaction::getDate)
                .max(LocalDate::compareTo)
                .orElse(LocalDate.MIN)
        ))

        // Filter inactive (no transactions in 30 days)
        .filter(c -> c.getLastTx().isBefore(cutoffDate))

        // Filter by high balance
        .filter(c -> c.getCustomer().getBalance()
            .compareTo(BigDecimal.valueOf(100000)) > 0)

        // Group by branch
        .collect(Collectors.groupingBy(
            c -> c.getCustomer().getBranch(),

            // Downstream: Create summary for each customer
            Collectors.mapping(
                c -> new CustomerSummary(
                    c.getCustomer().getId(),
                    c.getCustomer().getBalance(),
                    c.getLastTx()
                ),
                Collectors.toList()
            )
        ))

        // Sort by balance within each branch
        .entrySet().stream()
        .collect(Collectors.toMap(
            Map.Entry::getKey,
            e -> e.getValue().stream()
                .sorted(Comparator.comparing(
                    CustomerSummary::getBalance,
                    (b1, b2) -> b2.compareTo(b1)  // Descending
                ))
                .collect(Collectors.toList())
        ));
}
```

**Interview Tip**: This demonstrates understanding of:
- Chaining multiple operations
- Stateless transformations
- Complex collectors with grouping
- Handling Optional properly
- Real-world use case

---

## 11. Key Takeaways

**Lambda Expressions**:
- Syntactic sugar for functional interfaces
- Must reference effectively final variables
- Enable concise, readable code

**Functional Interfaces**:
- One abstract method contract
- Core: Function, Predicate, Consumer, Supplier
- Build blocks of functional programming

**Method References**:
- Cleaner alternative to lambdas when just calling methods
- Four types: static, instance, constructor, arbitrary object
- Improve readability when used appropriately

**Streams**:
- Lazy, chainable processing pipelines
- Intermediate operations don't execute until terminal op
- Enable parallel processing with minimal code changes

**Collectors**:
- Powerful aggregation beyond simple lists
- `groupingBy` for multi-group aggregation
- `partitioningBy` for binary splits
- Composition enables complex transformations

**Parallel Streams**:
- Overhead only worth cost for large datasets (1000+) with expensive operations
- Require stateless, non-interfering operations
- ForkJoinPool divides and conquers automatically

**Optional**:
- Container for null-safety, not a replacement for null checks
- Use `orElse`, `orElseGet`, `ifPresent` instead of `get()`
- Chains nicely with streams via `flatMap`

**Performance**:
- Streams have overhead; loops faster for simple iterations
- Streams shine with complex pipelines and parallelization
- Choose based on clarity, complexity, and actual performance needs

**Best Practices**:
1. Prefer immutable operations and pure functions
2. Keep lambda bodies concise; extract complex logic to methods
3. Use method references for code clarity
4. Avoid shared mutable state in streams
5. Profile before optimizing; readability often matters more
6. Use `peek()` for debugging, not production
7. Remember streams are one-time consumable

---

## References

- [Oracle Java Streams Tutorial](https://docs.oracle.com/javase/tutorial/collections/streams/)
- [Java Functional Interfaces Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)
- [Optional Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html)
- [Collectors Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html)
- [Modern Java in Action - Raoul-Gabriel Urma](https://www.manning.com/books/modern-java-in-action) (Chapters 2-6)
- [Maurice Naftalin's Collections Tutorial](https://www.oreilly.com/library/view/functional-programming-in/9781491927281/)

---

**Last Updated**: November 2024
**Relevance**: Essential for Java 8+ development and senior-level interviews
**Complexity**: Intermediate to Advanced
