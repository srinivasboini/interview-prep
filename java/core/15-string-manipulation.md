# Java String Manipulation

## Overview

Strings are fundamental to Java applications, yet their behavior—especially immutability, the string pool, and performance implications—often separates junior from senior developers in interviews. Understanding string internals is critical for writing efficient code, debugging memory issues, and designing APIs that don't inadvertently create performance bottlenecks in high-throughput systems like banking platforms.

**Why This Matters in Interviews:**
- Demonstrates knowledge of JVM memory management
- Shows performance consciousness (StringBuilder vs concatenation)
- Reveals understanding of immutability as a design principle
- Required for system design when discussing text processing at scale

---

## String Internals

### Immutability Design

Strings in Java are immutable—once created, their content cannot change. This is a deliberate design choice:

```java
String s1 = "Hello";
String s2 = s1.toUpperCase();
// s1 is still "Hello", s2 is "HELLO"
// s1 was NOT modified; a new String was created
```

**Benefits:**
- **Thread Safety**: Multiple threads can safely share String references without synchronization
- **Security**: Sensitive data (passwords, API keys) can't be accidentally modified
- **Caching**: Strings can be safely cached/reused
- **Hash Consistency**: String hash codes never change, making them ideal for HashMap keys

**Cost:**
- Every string operation creates new String objects
- Naive concatenation in loops causes performance degradation

### String Pool

The **String Pool** is a memory region in the heap where String literals are stored to enable reuse:

```java
String s1 = "Hello";        // Created in String Pool
String s2 = "Hello";        // References same object in String Pool
String s3 = new String("Hello");  // Creates new object in heap

System.out.println(s1 == s2);     // true (same reference)
System.out.println(s1 == s3);     // false (different objects)
System.out.println(s1.equals(s3)); // true (same content)
```

**Key Points:**
- Only string **literals** automatically enter the pool at compile time
- `new String()` always creates a separate heap object
- `.intern()` method explicitly moves a string to the pool
- String pool is garbage collectable (removed in Java 8+)

**Banking Context:**
In a high-throughput transaction system, understanding the difference between == and `.equals()` prevents subtle bugs. Always use `.equals()` for content comparison unless you specifically need reference equality.

### Compact Strings (Java 9+)

Before Java 9, all Strings used UTF-16 encoding (2 bytes per character). Most English text uses only ASCII (1 byte per character), wasting 50% of memory.

**Java 9 Solution: Compact Strings**

```java
// Java 9+: "Hello" stored as 1 byte per character (Latin-1)
String latin = "Hello";      // 5 bytes
String unicode = "你好";     // 4 bytes (2 per char in UTF-16)

// The JVM automatically chooses optimal encoding
// Transparent to developers—no code changes needed
```

**Implementation Detail:**
- Strings use a `coder` field: `0` for Latin-1 (single-byte), `1` for UTF-16
- Reduces memory consumption by ~40% for typical English applications
- Automatic selection based on content—no developer intervention needed

**Impact on Interview:**
Understanding this shows knowledge of JVM evolution and memory optimization. In banking systems processing millions of transactions with field names and IDs, this optimization matters at scale.

---

## String Operations

### Comparison Methods

```java
String s1 = "Hello";
String s2 = "HELLO";

// Content comparison
s1.equals(s2);              // false (case-sensitive)
s1.equalsIgnoreCase(s2);    // true

// Ordering comparison
s1.compareTo(s2);           // -32 (negative: s1 < s2 lexicographically)
s1.compareToIgnoreCase(s2); // 0

// Containment
s1.contains("ell");         // true
s1.startsWith("He");        // true
s1.endsWith("lo");          // true
s1.indexOf('l');            // 2 (first occurrence)
s1.lastIndexOf('l');        // 3 (last occurrence)
```

**Common Pitfall:**
```java
// WRONG - uses == (reference equality)
if (userInput == "admin") { }

// CORRECT - uses equals (content equality)
if (userInput.equals("admin")) { }
```

### Substring and Extraction

```java
String text = "interview.preparation.guide";

text.substring(0, 9);       // "interview"
text.substring(10);         // "preparation.guide"
text.charAt(0);             // 'i'
text.toCharArray();         // char[] {'i','n','t',...}
text.split("\\.");          // ["interview", "preparation", "guide"]
text.replace(".", "_");     // "interview_preparation_guide"
text.replaceAll("[a-z]", "x");  // "xxxxxxxxx.xxxxxxxxxxxxx.xxxxx"
```

### Formatting and Case Conversion

```java
String template = "User %s logged in at %d";
String formatted = String.format(template, "john", 1234567890);
// "User john logged in at 1234567890"

// Case conversion
"Hello".toLowerCase();      // "hello"
"Hello".toUpperCase();      // "HELLO"

// Trimming and padding (Java 11+)
"  hello  ".strip();        // "hello" (removes leading/trailing whitespace)
"  hello  ".trim();         // "hello" (older method, less robust)
"hello".indent(4);          // "    hello\n" (Java 12+)

// Repeating (Java 11+)
"ab".repeat(3);             // "ababab"
```

### Case: Banking Transaction Log Formatting

```java
// Formatting transaction records for audit logs
String auditLog = String.format(
    "User: %s | Amount: $%.2f | Time: %s | Status: %s",
    username, amount, timestamp, status
);
```

---

## StringBuilder vs StringBuffer

### Performance Comparison

**Problem with String Concatenation:**

```java
// INEFFICIENT - Creates 4 intermediate String objects
String result = "User: " + name + " | Amount: " + amount + " | Status: " + status;
// Equivalent to:
// new StringBuilder("User: ")
//     .append(name)
//     .append(" | Amount: ")
//     .append(amount)
//     .append(" | Status: ")
//     .append(status)
//     .toString();
```

**In a Loop - DISASTER:**

```java
// VERY INEFFICIENT - Creates N String objects
String result = "";
for (String item : items) {
    result += item + ", ";  // Creates new String each iteration
}
// For 10,000 items: 10,000 String objects created, massive GC pressure
```

**Correct Approach:**

```java
// EFFICIENT - Single StringBuilder, reused
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item).append(", ");
}
String result = sb.toString();
// Single GC event, linear time complexity
```

### StringBuilder vs StringBuffer

```java
// StringBuilder (Java 5+)
// - NOT thread-safe
// - Faster (no synchronization overhead)
// - Use in single-threaded contexts (99% of cases)
StringBuilder sb = new StringBuilder();
sb.append("Hello").append(" ").append("World");

// StringBuffer (Java 1.0)
// - Thread-safe (synchronized methods)
// - Slower (synchronization overhead)
// - Legacy code or shared across threads
StringBuffer buf = new StringBuffer();
buf.append("Hello").append(" ").append("World");
```

**Interview Answer:**
"Use StringBuilder as default—it's faster and thread-safe in most scenarios because Strings are immutable, so the reference is thread-safe. StringBuffer is only needed if the same StringBuilder instance is shared across threads, which is rare."

### Performance Metrics

```java
// Concatenation in a loop
// 1,000 iterations
// StringBuilder: ~1ms
// String concatenation: ~50-100ms
// StringBuffer: ~5-10ms

// Growth strategy: StringBuilder starts with capacity 16
// Doubles when exceeded: 16→32→64→128...
// To optimize: new StringBuilder(estimatedSize)
StringBuilder sb = new StringBuilder(1000);  // Pre-allocate for 1000 chars
```

---

## String Methods by Version

### Java 11+ Enhancements

```java
// isBlank() - checks for empty or whitespace-only
"  ".isBlank();              // true
"hello".isBlank();           // false

// lines() - splits by line breaks (stream-based)
"line1\nline2\nline3".lines()
    .filter(l -> !l.isEmpty())
    .collect(Collectors.toList());

// strip() - removes leading/trailing whitespace (Unicode-aware)
"  hello  ".strip();         // "hello"
"  hello  ".stripLeading();  // "hello  "
"  hello  ".stripTrailing(); // "  hello"

// repeat() - repeats string N times
"ha".repeat(3);              // "hahaha"

// transform() - pipeline for string transformations (useful!)
String result = "hello"
    .transform(s -> s.toUpperCase())
    .transform(s -> s + "!");
// "HELLO!"
```

### Java 12+ Additions

```java
// indent() - adds/removes indentation
"hello\nworld".indent(2);
// "  hello\n  world\n"

"hello\nworld".indent(-1);   // remove indentation

// describeConstable() - for constant metadata (advanced)
Optional<String> described = "hello".describeConstable();
```

### Java 15+ Text Blocks

**Problem Being Solved:**

```java
// BEFORE: Verbose, escaped newlines, concatenation
String json = "{\n" +
    "  \"name\": \"John\",\n" +
    "  \"age\": 30\n" +
    "}";

String html = "<div>\n" +
    "  <p>Hello World</p>\n" +
    "</div>";
```

**Solution - Text Blocks (Java 15+):**

```java
// Text blocks use triple quotes
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;

String html = """
    <div>
      <p>Hello World</p>
    </div>
    """;

String sql = """
    SELECT u.id, u.name, t.amount
    FROM users u
    JOIN transactions t ON u.id = t.user_id
    WHERE t.amount > ?
    ORDER BY t.date DESC
    """;
```

**Text Block Rules:**

```java
// Leading whitespace is stripped to the leftmost line
String multiline = """
    line1
    line2
    line3""";
    // Result: "line1\nline2\nline3"

// Escape sequences still work
String withNewline = """
    Line 1\
    Line 2""";  // Backslash continues line without newline

// Empty text block
String empty = "";

// String interpolation NOT supported in standard text blocks
// (Feature discussed but not yet standard; use String.format instead)
String name = "John";
String message = """
    Hello %s
    """.formatted(name);  // Java 15: formatted() method
```

**Banking Use Case:**

```java
// Audit log template with indented SQL for readability
String auditTemplate = """
    Transaction Audit Log
    --------------------
    User: %s
    Amount: $%.2f
    Query Executed:
        %s
    Timestamp: %s
    """;

String auditLog = String.format(auditTemplate, user, amount, sqlQuery, timestamp);
```

---

## Architecture & Performance Diagram

```
String Lifecycle and Memory

┌─────────────────────────────────────────────────────────┐
│                   HEAP MEMORY                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  String Pool (Java < 8: Permanent Gen, Java 8+: Heap) │
│  ┌──────────────────────────────────────────┐          │
│  │ Literal: "Hello"                         │          │
│  │ Literal: "World"                         │          │
│  │ Literal: "Hello" → references same obj   │          │
│  └──────────────────────────────────────────┘          │
│                                                         │
│  Regular Heap                                           │
│  ┌──────────────────────────────────────────┐          │
│  │ new String("Hello") - separate object    │          │
│  │ new String("World") - separate object    │          │
│  │ sb.toString() - new String object        │          │
│  └──────────────────────────────────────────┘          │
│                                                         │
│  StringBuilder (Mutable)                                 │
│  ┌──────────────────────────────────────────┐          │
│  │ char[] backing array (resizable)         │          │
│  │ capacity: 16, 32, 64... (doubles)        │          │
│  │ count: current length                    │          │
│  └──────────────────────────────────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘

String Object Structure (Java 9+):
┌─────────────────────────────────────┐
│ String {                            │
│   private final byte[] value;       │  Actual character data
│   private final byte coder;         │  0=Latin-1, 1=UTF-16
│   private int hash;                 │  Cached hash code
│ }                                   │
└─────────────────────────────────────┘
```

---

## String Concatenation Performance Diagram

```
Concatenation Performance: String vs StringBuilder

Scenario: Building "User1,User2,User3...User1000"

STRING CONCATENATION:
result = ""
Loop iteration 1: result = "" + "User1" + ","  → Creates new String
Loop iteration 2: result = "User1," + "User2" + "," → Creates new String
Loop iteration 3: result = "User1,User2," + "User3" + "," → Creates new String
...
Loop iteration 1000: + "User1000," → Creates new String
═════════════════════════════════════════════════════════════════
Result: 1000+ intermediate String objects, O(n²) time complexity
Memory: Garbage collection pressure spikes
GC Pauses: Potentially visible in high-throughput systems

STRINGBUILDER:
sb = new StringBuilder()
Loop 1000x: sb.append(userN).append(",")
sb.toString()
═════════════════════════════════════════════════════════════════
Result: Single StringBuilder, capacity: 16→32→64→...→2048
Memory: Linear, minimal GC pressure
GC Pauses: Negligible
Time: O(n) complexity

Performance (1000 iterations):
├─ String concat: ~100ms
├─ StringBuilder: ~1ms
└─ 100x faster with StringBuilder
```

---

## String Methods Comparison Table

| Method | Purpose | Immutable | Time | Notes |
|--------|---------|-----------|------|-------|
| `equals()` | Content comparison (case-sensitive) | ✓ | O(n) | Preferred for security decisions |
| `equalsIgnoreCase()` | Content comparison (case-insensitive) | ✓ | O(n) | Single pass |
| `compareTo()` | Lexicographic ordering | ✓ | O(n) | Returns int: <0, =0, >0 |
| `contains()` | Substring check | ✓ | O(n) | Uses indexOf internally |
| `indexOf()` | Find first occurrence | ✓ | O(n*m) | m = pattern length |
| `substring()` | Extract portion | ✓ | O(1)* | Java 9+: copies; older: shared |
| `split()` | Divide by regex | ✓ | O(n) | Regex compilation cost |
| `replace()` | String replacement | ✓ | O(n) | All occurrences |
| `replaceAll()` | Regex replacement | ✓ | O(n*m) | More powerful, slower |
| `toUpperCase()` | Convert to uppercase | ✓ | O(n) | Locale-aware |
| `toLowerCase()` | Convert to lowercase | ✓ | O(n) | Locale-aware |
| `trim()` | Remove whitespace | ✓ | O(n) | Java 8 and earlier |
| `strip()` | Remove whitespace (Unicode) | ✓ | O(n) | Java 11+, preferred |
| `repeat()` | Repeat string N times | ✓ | O(n*k) | k = repetitions, Java 11+ |

*Java 9+ optimized substring; pre-Java 9 shared the char array (could prevent GC).

---

## Interview Questions & Answers

### Q1: Explain String Immutability and Its Benefits

**Answer:**
"Strings are immutable—once created, their content cannot change. Every modification like `toUpperCase()` creates a new String object. This design provides:

1. **Thread Safety**: Multiple threads can safely share String references without synchronization, since the content can't change.
2. **Security**: Sensitive data like passwords can't be accidentally modified by other code holding the reference.
3. **Caching/Optimization**: The JVM can cache strings and reuse them. The String pool exists specifically because of this.
4. **HashMap Keys**: Hash codes are consistent throughout the String's lifetime, making it safe for use as HashMap keys.

The trade-off is memory overhead—every operation creates new objects. That's why we use StringBuilder for repeated modifications."

### Q2: What's the Difference Between `==` and `.equals()` for Strings?

**Answer:**
"**`==`** checks **reference equality**—whether two variables point to the same object in memory.
**`.equals()`** checks **content equality**—whether two Strings have the same characters.

```java
String s1 = "Hello";              // String pool
String s2 = "Hello";              // Same reference in pool
String s3 = new String("Hello");  // Different object

s1 == s2  // true (same reference in pool)
s1 == s3  // false (different objects)
s1.equals(s3)  // true (same content)
```

**Always use `.equals()` for content comparison** unless you specifically need reference identity. Using `==` for string comparison is a common bug source, especially in security-sensitive code like password validation."

### Q3: When Would You Use StringBuilder vs StringBuffer?

**Answer:**
"Use **StringBuilder** as the default—it's faster because it's not synchronized. Use **StringBuffer** only if the same buffer is intentionally shared across threads.

In practice, StringBuffer is legacy code. Modern Java applications use StringBuilder because:
1. Strings themselves are immutable, so the reference is thread-safe
2. If you need thread-safe concatenation, you'd typically use thread-local StringBuilder or other synchronization patterns
3. StringBuffer's synchronization adds 5-10x overhead for minimal real-world benefit

**Example:** Building audit logs in a transaction processing system:
```java
// Correct approach
StringBuilder auditLog = new StringBuilder();
auditLog.append(userId).append("|").append(amount).append("|");
// Efficient, no thread safety needed
```"

### Q4: Explain String Pool Behavior

**Answer:**
"The String Pool is a memory region where String literals are stored to enable reuse:

```java
String a = "Hello";        // Created in pool
String b = "Hello";        // Reuses same object
String c = new String("Hello");  // Separate heap object

a == b  // true (same pool reference)
a == c  // false (c is separate)
c.intern()  // moves c into pool, returns pool reference
```

**Key Points:**
- Only **string literals** automatically enter the pool at compile time
- `new String()` explicitly creates a separate heap object
- `.intern()` explicitly adds a string to the pool
- Pre-Java 8: Pool was in PermGen (limited, not GC'd). Java 8+: Pool is in the heap (GC-able)
- This prevents memory leaks from calling `.intern()` on user input

**Interview insight:** Understanding this shows you comprehend JVM memory regions and can avoid performance problems in high-throughput systems."

### Q5: What Are Compact Strings (Java 9+)?

**Answer:**
"Before Java 9, all Strings used UTF-16 (2 bytes per character). Most English text is ASCII (1 byte per character), wasting ~50% memory.

Java 9 introduced **Compact Strings**:
```java
// Java 9+: "Hello" stored as 1 byte per character
String latin = "Hello";      // 5 bytes (Latin-1 encoding)
String mixed = "Hello你好";  // optimized automatically

// Transparent—no code changes needed
```

The JVM automatically chooses encoding based on content:
- `coder = 0`: Latin-1 (single byte)
- `coder = 1`: UTF-16 (two bytes)

This is a great example of JVM evolution optimizing for real-world usage patterns. Saved roughly **40% memory** for typical English-heavy applications without any developer effort."

### Q6: What's the Performance Impact of Concatenation in a Loop?

**Answer:**
"String concatenation in a loop is a **classic performance anti-pattern**:

```java
// BAD: Creates ~1000 String objects
String result = "";
for (String item : items) {
    result += item + ", ";  // New String created each iteration
}
// Time: O(n²), Memory: excessive GC pressure

// GOOD: Reuses single StringBuilder
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item).append(", ");
}
String result = sb.toString();
// Time: O(n), Memory: single GC event
```

**Impact:** For 1,000 items:
- String concat: 100-200ms, spikes GC
- StringBuilder: 1-2ms, minimal GC

In banking systems processing millions of transactions or audit logs, this difference is critical. This is why code review standards explicitly flag string concatenation in loops."

### Q7: How Do Text Blocks (Java 15+) Improve Code?

**Answer:**
"Text blocks solve the 'stringly-typed' problem with multiline strings:

```java
// Before: Escaped newlines, concatenation
String json = "{\n  \"name\": \"John\",\n  \"age\": 30\n}";

// After: Readable as-is
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;
```

**Key advantages:**
1. **Readability**: JSON, SQL, XML look natural
2. **No Escaping**: No `\n`, no string concatenation
3. **Reduced Errors**: Fewer escape sequence mistakes
4. **Leading Whitespace Handling**: Automatically strips indentation

**Banking example—SQL readability:**
```java
String auditQuery = """
    SELECT u.id, u.name, t.amount, t.date
    FROM users u
    JOIN transactions t ON u.id = t.user_id
    WHERE t.amount > ?
      AND t.status = 'PENDING'
    ORDER BY t.date DESC
    LIMIT 100
    """;
```

Much cleaner than `"SELECT u.id... " + "FROM users... "` approach."

### Q8: Explain `.substring()` Behavior Change in Java 9

**Answer:**
"Pre-Java 9 and Java 9+ behave differently:

```java
// Java 8 and earlier: Shared char array
String str = "HelloWorld";
String sub = str.substring(0, 5);  // "Hello"
// 'sub' shared the char[] with 'str', held reference to entire array
// Could prevent 'str' from being GC'd even if 'sub' was only using 5 chars
// Memory leak potential if substring was retained

// Java 9+: Copies data
String str = "HelloWorld";
String sub = str.substring(0, 5);  // "Hello"
// 'sub' gets its own char array with just "Hello"
// Original can be GC'd independently
```

**Interview point:** This shows JVM evolution fixing subtle memory bugs. The pre-Java 9 behavior was a hidden source of memory leaks in applications that retained long-lived substrings while discarding originals."

### Q9: When Should You Use `.split()` vs `.replace()`?

**Answer:**
"**`.split(String regex)`**: Divides string by delimiter, returns array
**`.replace(String target, String replacement)`**: Replaces all occurrences

```java
String csv = "john,jane,bob";
String[] names = csv.split(",");  // ["john", "jane", "bob"]

String log = "ERROR: auth failed, ERROR: timeout";
String cleaned = log.replace("ERROR", "WARN");
// "WARN: auth failed, WARN: timeout"
```

**Key differences:**
- `.split()` takes a **regex pattern**—slower, more powerful
- `.replace()` takes **literal strings**—faster for simple cases
- `.replaceAll()` takes a **regex pattern**—most powerful but slowest

```java
// Careful: split() treats argument as regex
"a.b.c".split(".");      // ERROR: returns empty! (. matches any char)
"a.b.c".split("\\.");    // ["a", "b", "c"] (escaped dot)

// replace() doesn't use regex
"a.b.c".replace(".", "_");  // "a_b_c" (literal replacement)
```

**Performance tip:** Use `.replace()` for literal string replacement; reserve `.replaceAll()` for actual regex needs."

### Q10: Design a Logging Framework Method for High-Throughput Systems

**Answer:**
"For a transaction processing system handling thousands of operations per second:

```java
public class AuditLogger {
    private static final int LOG_BUFFER_CAPACITY = 4096;

    /**
     * Builds audit log entry efficiently for high-throughput scenarios.
     * Uses StringBuilder to avoid intermediate String objects.
     */
    public static String formatAuditLog(String userId, BigDecimal amount,
                                       String txnId, String status) {
        // Pre-allocate sufficient capacity to avoid resizing
        StringBuilder sb = new StringBuilder(LOG_BUFFER_CAPACITY);

        sb.append("user_id=").append(userId)
          .append("|txn_id=").append(txnId)
          .append("|amount=").append(String.format("%.2f", amount))
          .append("|status=").append(status)
          .append("|timestamp=").append(System.currentTimeMillis());

        return sb.toString();
    }

    // For repeated formatting with same structure
    private static final String AUDIT_TEMPLATE = """
        User: %s
        Amount: $%.2f
        Status: %s
        Time: %d
        """;

    public static String formatAuditReport(String user, BigDecimal amount,
                                          String status, long timestamp) {
        return AUDIT_TEMPLATE.formatted(user, amount, status, timestamp);
    }
}
```

**Design considerations:**
1. **Pre-allocate capacity**: `new StringBuilder(4096)` for expected log size
2. **Avoid concatenation operators**: Use `.append()` chaining
3. **Use text blocks**: For template strings (Java 15+)
4. **Leverage `.formatted()`**: Type-safe alternative to `String.format()`

This prevents unnecessary GC pressure in high-throughput systems."

### Q11: Explain String Validation Best Practices

**Answer:**
"Proper string validation prevents security issues and runtime errors:

```java
public class StringValidator {
    // Use isBlank() (Java 11+) instead of isEmpty()
    private boolean isValidInput(String input) {
        // Null-safe check
        if (input == null) {
            return false;
        }

        // Whitespace-only strings treated as invalid
        if (input.isBlank()) {
            return false;
        }

        // Length constraints (prevent DoS)
        if (input.length() > 1000) {
            return false;
        }

        return true;
    }

    // Case-insensitive comparison for user input
    public boolean isAdminUser(String username) {
        // Don't use ==, always use equals()
        return username.equalsIgnoreCase("admin");
    }

    // Sanitizing user input to prevent injection
    public String sanitizeInput(String input) {
        if (!isValidInput(input)) {
            throw new IllegalArgumentException("Invalid input");
        }

        return input.strip()  // Remove whitespace (Java 11+)
                    .replaceAll("\\s+", " ");  // Normalize spaces
    }
}
```

**Key takeaways:**
- Use `.isBlank()` (Java 11+) instead of `.isEmpty()` or `.trim().isEmpty()`
- Always use `.equals()`, never `==` for content comparison
- Validate length to prevent DoS attacks
- Use `.strip()` instead of `.trim()` (Unicode-aware)"

### Q12: Performance Tuning: When to Use `.intern()`?

**Answer:**
"`.intern()` moves a String into the pool, but use with extreme caution:

```java
// Good use case: Limited, known set of values
String productCategory = "BANKING_PRODUCT".intern();  // One-time cost
if (productCategory.equals("BANKING_PRODUCT")) {  // Fast comparison possible
    // process
}

// BAD use case: User input or unbounded strings
String userInput = request.getParameter("name").intern();
// WARNING: If users supply 100K unique names, pool grows unbounded
// Can cause OutOfMemoryError or performance degradation
```

**When to use:**
1. Fixed, limited set of strings (< 100 values)
2. Strings created once and referenced repeatedly
3. Performance-critical comparisons (though `equals()` is usually fast enough)

**When NOT to use:**
1. User input (could DOS attack via pool exhaustion)
2. Dynamically generated strings
3. CSV/XML/JSON parsing (millions of unique values)
4. Logging (new log messages likely unique)

**Modern best practice:** Almost never use `.intern()` in new code. JVM string caching is sufficient, and the pool can cause memory issues if misused."

---

## Common Pitfalls & Best Practices

### 1. String Concatenation in Loops
```java
// WRONG
String result = "";
for (String item : items) {
    result += item;
}

// RIGHT
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item);
}
String result = sb.toString();
```

### 2. Using `==` for String Comparison
```java
// WRONG
if (status == "ACTIVE") { }

// RIGHT
if (status.equals("ACTIVE")) { }
if (status.equalsIgnoreCase("active")) { }
```

### 3. Forgetting Null Safety
```java
// WRONG - NullPointerException if str is null
if (str.isEmpty()) { }

// RIGHT
if (str != null && !str.isEmpty()) { }
if (str != null && str.isBlank()) { }  // Java 11+

// Or using Optional
String str = ...;
Optional.ofNullable(str)
    .filter(s -> !s.isBlank())
    .ifPresent(System.out::println);
```

### 4. Misusing `.split()`
```java
// WRONG - dot is regex metacharacter (matches any char)
String[] parts = "a.b.c".split(".");  // returns empty array!

// RIGHT
String[] parts = "a.b.c".split("\\.");
```

### 5. Unnecessary `.intern()` Calls
```java
// WRONG - can cause memory issues with user input
String[] usernames = getUsernamesFromDB();  // 1M records
for (String name : usernames) {
    name.intern();  // DON'T DO THIS
}

// RIGHT - only for known, fixed values
private static final String ADMIN = "ADMIN";
```

### 6. Inefficient Formatting
```java
// LESS EFFICIENT - concatenation
String log = "User: " + user + " | Amount: " + amount;

// MORE EFFICIENT - String.format
String log = String.format("User: %s | Amount: %.2f", user, amount);

// BEST (Java 15+) - formatted() method
String log = "User: %s | Amount: %.2f".formatted(user, amount);
```

---

## Production Checklist

When working with Strings in enterprise applications:

✅ **Use StringBuilder** for repeated concatenations or loop concatenation
✅ **Use `.equals()`** for content comparison, never `==`
✅ **Use `.strip()`** (Java 11+) instead of `.trim()`
✅ **Validate length** to prevent DoS attacks
✅ **Use text blocks** (Java 15+) for multiline strings
✅ **Pre-allocate StringBuilder capacity** when size is known
✅ **Avoid `.intern()`** unless strings are known/fixed and performance-critical
✅ **Use `.formatted()`** (Java 15+) instead of `String.format()` where possible
✅ **Handle null strings** explicitly before calling methods
✅ **Use `.equalsIgnoreCase()`** for case-insensitive comparisons
✅ **Consider locale** when converting case (`.toUpperCase(Locale.ENGLISH)`)
✅ **Log carefully**—avoid logging sensitive data in plain text

---

## References

- **JEP 254**: Compact Strings (Java 9) - https://openjdk.java.net/jeps/254
- **JEP 378**: Text Blocks (Java 15) - https://openjdk.java.net/jeps/378
- **Java String Documentation**: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/String.html
- **String Performance**: "The Coding Interview Guide by Gyle McDowell" - String optimization patterns
- **Java Memory Management**: "Understanding the JVM" by Zhou Zhiming - String pool internals
