# Serialization & Object I/O — Interview Preparation Guide

> **Key Topics**: `Serializable`, `serialVersionUID`, `transient`, custom serialization, Jackson/JSON, versioning strategies

---

## Table of Contents

- [1. Java Object Serialization](#1-java-object-serialization)
- [2. serialVersionUID — Why It Matters](#2-serialversionuid--why-it-matters)
- [3. transient Keyword](#3-transient-keyword)
- [4. Custom Serialization](#4-custom-serialization)
- [5. Externalizable Interface](#5-externalizable-interface)
- [6. JSON Serialization with Jackson](#6-json-serialization-with-jackson)
- [7. Serialization Security Risks](#7-serialization-security-risks)
- [8. Interview Questions & Answers](#8-interview-questions--answers)

---

## 1. Java Object Serialization

**Serialization**: Converting an object to a byte stream for storage or transmission.
**Deserialization**: Converting a byte stream back to an object.

```java
/**
 * Making a class Serializable — mark interface only, no methods to implement.
 * The JVM handles field serialization automatically.
 */
public class Transaction implements Serializable {
    // Always declare serialVersionUID explicitly!
    private static final long serialVersionUID = 1L;

    private String transactionId;
    private BigDecimal amount;
    private String currency;
    private LocalDateTime timestamp;
    private String status;

    // Constructors, getters, setters...
}

// Serializing to file
public class SerializationExample {
    public static void serialize(Transaction txn, String filePath) throws IOException {
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(filePath))) {
            oos.writeObject(txn);
        }
    }

    public static Transaction deserialize(String filePath) throws IOException, ClassNotFoundException {
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filePath))) {
            return (Transaction) ois.readObject();
        }
    }
}
```

### What Gets Serialized?

```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;

    // ✅ Serialized — instance fields
    private String accountId;
    private BigDecimal balance;

    // ❌ NOT serialized — static fields (belong to class, not instance)
    private static int totalAccounts = 0;

    // ❌ NOT serialized — transient fields (explicitly excluded)
    private transient Connection dbConnection;

    // ✅ Serialized — even if null
    private String description;
}
```

---

## 2. `serialVersionUID` — Why It Matters

The JVM uses `serialVersionUID` to verify that a serialized class is compatible with the current class definition.

```java
// Version 1 of the class — stored to disk
public class PaymentRecord implements Serializable {
    private static final long serialVersionUID = 1L;
    private String paymentId;
    private BigDecimal amount;
}

// Version 2: field added — serialVersionUID unchanged (compatible)
public class PaymentRecord implements Serializable {
    private static final long serialVersionUID = 1L;  // Same!
    private String paymentId;
    private BigDecimal amount;
    private String currency;         // New field (deserialized to null from old data)
    private String merchantId;       // New field (deserialized to null from old data)
}
```

**What happens WITHOUT explicit serialVersionUID**:

```java
public class PaymentRecord implements Serializable {
    // ❌ No explicit serialVersionUID
    // JVM auto-generates based on class structure
    // Adding ANY field/method changes the auto-generated ID
    // → InvalidClassException when deserializing old data!
    private String paymentId;
}

// Version 2 — just added a field
public class PaymentRecord implements Serializable {
    // Auto-generated ID is DIFFERENT now → InvalidClassException
    private String paymentId;
    private BigDecimal amount;  // Adding this changes the computed UID
}
```

**Best Practice for Versioning**:

```java
public class CustomerProfile implements Serializable {
    private static final long serialVersionUID = 1L;  // Start at 1

    private String customerId;
    private String name;
    private String email;       // v1
    private String phone;       // v2 — added later, null for old records OK
    private Address address;    // v3 — added later, null for old records OK

    // If you make INCOMPATIBLE changes: rename field, change type, remove field
    // Then increment serialVersionUID to 2L → old data properly rejected
}
```

---

## 3. `transient` Keyword

Mark fields `transient` to **exclude them from serialization**. Use for:
- Sensitive data (passwords, security tokens)
- Non-serializable fields (connections, threads, sockets)
- Derived/computed fields (recalculate on deserialization)

```java
public class BankSession implements Serializable {
    private static final long serialVersionUID = 1L;

    private String sessionId;
    private String userId;
    private LocalDateTime loginTime;

    // ❌ Should NOT be serialized — sensitive!
    private transient String passwordHash;

    // ❌ Cannot be serialized — Connection isn't Serializable
    private transient Connection databaseConnection;

    // ❌ Derived field — recalculate after deserialization
    private transient int sessionDurationMinutes;

    // After deserialization, transient fields have default values:
    // passwordHash = null, databaseConnection = null, sessionDurationMinutes = 0
    // Use readObject() to reinitialize them if needed
}
```

---

## 4. Custom Serialization

Override `writeObject` and `readObject` to control exactly how an object is serialized.

```java
public class SecureAccount implements Serializable {
    private static final long serialVersionUID = 1L;

    private String accountId;
    private transient BigDecimal balance;  // transient — custom handling
    private transient String encryptedBalance;

    // Called during serialization
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();  // Write non-transient fields first
        // Custom handling: encrypt balance before writing
        String encrypted = encrypt(balance.toString());
        out.writeObject(encrypted);
    }

    // Called during deserialization
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();  // Read non-transient fields first
        // Custom handling: decrypt the balance
        String encrypted = (String) in.readObject();
        this.balance = new BigDecimal(decrypt(encrypted));
    }

    // Called AFTER deserialization — replace deserialized object with singleton/cached version
    private Object readResolve() throws ObjectStreamException {
        // For singletons: return the singleton instance instead of newly deserialized one
        // This prevents multiple instances of singleton via deserialization
        return AccountCache.getInstance().getOrCache(this);
    }
}
```

**Singleton + Serialization**:

```java
// Without readResolve(), deserialization breaks the Singleton pattern!
public enum ConfigManager {
    INSTANCE;  // Enum Singleton is automatically serialization-safe — no readResolve needed!
    // Java guarantees enum values are singletons even after deserialization
}

// For class-based singletons, protect with readResolve:
public class ConnectionPool implements Serializable {
    private static final ConnectionPool INSTANCE = new ConnectionPool();

    private Object readResolve() {
        return INSTANCE;  // Always return the ONE instance, discard newly deserialized object
    }
}
```

---

## 5. Externalizable Interface

More control than `Serializable`, but requires you to write ALL serialization logic yourself.

```java
/**
 * Externalizable: Complete control over serialization.
 * Must implement writeExternal() and readExternal().
 * Must have a public no-arg constructor (for deserialization).
 */
public class TransactionRecord implements Externalizable {

    private String txnId;
    private BigDecimal amount;
    private String currency;

    // Required for Externalizable
    public TransactionRecord() {}

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(txnId);
        out.writeUTF(amount.toPlainString());
        out.writeUTF(currency);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        this.txnId = in.readUTF();
        this.amount = new BigDecimal(in.readUTF());
        this.currency = in.readUTF();
    }
}
```

**Serializable vs Externalizable**:

| Aspect | Serializable | Externalizable |
|--------|--------------|----------------|
| Control | JVM handles fields automatically | You control everything |
| Performance | Slower (reflective) | Faster (direct write) |
| Backward compatibility | Easier to maintain | Must version manually |
| No-arg constructor | Not required | **Required** |
| Use when | Simple objects, quick setup | Performance-critical, precise control |

---

## 6. JSON Serialization with Jackson

In modern applications, **JSON serialization** (Jackson) is far more common than Java object serialization. This is what REST APIs, Kafka messages, and databases use.

```java
// Jackson annotations for controlling serialization
public class AccountDTO {

    @JsonProperty("account_id")          // Change field name in JSON
    private String accountId;

    @JsonIgnore                           // Exclude from JSON entirely
    private String internalRef;

    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;

    @JsonInclude(JsonInclude.Include.NON_NULL)  // Exclude null fields from JSON
    private String description;

    // Jackson uses default constructor + setters, OR all-args constructor + @JsonCreator
}

// @JsonCreator for immutable objects (no setters)
public class Payment {
    private final String id;
    private final BigDecimal amount;

    @JsonCreator  // Tell Jackson which constructor to use for deserialization
    public Payment(
        @JsonProperty("id") String id,
        @JsonProperty("amount") BigDecimal amount
    ) {
        this.id = id;
        this.amount = amount;
    }
}

// ObjectMapper usage
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());  // For Java 8 Date/Time types
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);  // Ignore extra fields

// Serialize
String json = mapper.writeValueAsString(account);

// Deserialize
AccountDTO account = mapper.readValue(json, AccountDTO.class);

// For generic types (List<T>, Map<K,V>)
List<Transaction> transactions = mapper.readValue(json,
    new TypeReference<List<Transaction>>() {});
```

### Handling Polymorphism in Jackson

```java
// When you have a hierarchy and need to deserialize to the right subtype:
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = CreditCardPayment.class, name = "CREDIT_CARD"),
    @JsonSubTypes.Type(value = BankTransferPayment.class, name = "BANK_TRANSFER")
})
public abstract class Payment {
    private BigDecimal amount;
}

public class CreditCardPayment extends Payment {
    private String cardLastFour;
}

public class BankTransferPayment extends Payment {
    private String iban;
}

// JSON: { "type": "CREDIT_CARD", "amount": 100.00, "cardLastFour": "4242" }
// Jackson uses "type" field to pick CreditCardPayment automatically
```

---

## 7. Serialization Security Risks

**Java serialization is a known security vulnerability**. Deserializing untrusted data can lead to **Remote Code Execution (RCE)**.

```java
// ❌ VULNERABLE — deserializing untrusted input
public Object deserializeFromRequest(byte[] data) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));
    return ois.readObject();  // An attacker can craft data that executes arbitrary code!
}

// ✅ SAFER — use a filtering ObjectInputStream (Java 9+)
public Object safeDeserialize(byte[] data) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data)) {
        @Override
        protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
            // Whitelist only your own types
            String className = desc.getName();
            if (!className.startsWith("com.bank.")) {
                throw new InvalidClassException("Unauthorized class", className);
            }
            return super.resolveClass(desc);
        }
    };
    return ois.readObject();
}

// ✅ BEST — use JSON/JSON Schema validation instead of Java serialization for external data
// REST APIs: always use Jackson/JSON, never Java object serialization
// Kafka messages: use Avro, Protobuf, or JSON with schema registry
```

**Rules for Banking Applications**:
1. **Never deserialize from untrusted sources** using Java object serialization
2. Prefer **JSON (Jackson)** or **Protobuf** for network communication
3. If you must use Java serialization, use **serialization filters** (`ObjectInputFilter`)
4. Use `transient` for all sensitive fields (passwords, keys, tokens)

---

## 8. Interview Questions & Answers

**Q1: What is `serialVersionUID` and what happens if you don't declare it?**

> "It's a class-level version identifier used during deserialization to verify the sender and receiver have compatible class definitions. If you don't declare it, the JVM auto-calculates it from the class structure. Any change to the class — adding a field, changing a method — changes the auto-calculated ID. Then if you try to deserialize data serialized from the old version, you get `InvalidClassException`. Best practice: always declare it explicitly as `1L` and only increment when you make incompatible changes."

**Q2: How do you serialize a field that's not Serializable (like `Connection`)?**

> "Mark it `transient`. Transient fields are excluded from serialization and initialized to their default values on deserialization (null, 0, false). If you need to reinitialize transient fields after deserialization, override `readObject()` — this is called automatically after deserialization and you can re-establish the connection or recalculate the derived field there."

**Q3: Why is Java object serialization considered a security risk?**

> "Java deserialization runs arbitrary code paths during `readObject()` and class initialization. An attacker can craft a serialized byte stream that, when deserialized, triggers a gadget chain — a sequence of legitimate classes whose methods, when called in the right order, execute malicious code. There have been multiple critical CVEs in popular frameworks (Apache Commons Collections, Spring, etc.) exploited via Java serialization. In banking, we never deserialize untrusted input via Java serialization. For APIs, we use Jackson JSON with schema validation. For messaging (Kafka), we use Avro with schema registry."

**Q4: What's the difference between Serializable and Externalizable?**

> "With `Serializable`, the JVM handles everything automatically via reflection — reads and writes all non-transient fields. It's easy but slower. With `Externalizable`, you implement `writeExternal()` and `readExternal()` yourself — you have total control over what's written and in what format. It's faster but requires a public no-arg constructor and more code to maintain. I'd use `Serializable` for most cases and only consider `Externalizable` for performance-critical paths where serialization is a measured bottleneck."

**Q5: How do you handle polymorphism in Jackson?**

> "Use `@JsonTypeInfo` and `@JsonSubTypes` on the base class to include a type discriminator field in the JSON. Jackson reads the discriminator value and instantiates the correct subtype. This is essential for banking APIs where you might have a `Payment` supertype with `CreditCardPayment`, `BankTransferPayment`, `CryptoPayment` subtypes — the deserializer needs to know which concrete class to create. Alternative: use custom deserializers for complex scenarios."

---

## Key Takeaways

1. **Always declare serialVersionUID** — `1L` to start, increment only on incompatible changes
2. **`transient`** excludes fields from serialization — use for sensitive data, non-serializable resources
3. **Override `readObject`/`writeObject`** for custom logic; **`readResolve`** for singleton safety
4. **Java object serialization is a security risk** — never deserialize untrusted input without filtering
5. **Modern apps use JSON (Jackson)** — not Java serialization — for APIs and messaging
6. **Jackson key annotations**: `@JsonProperty`, `@JsonIgnore`, `@JsonFormat`, `@JsonCreator`, `@JsonTypeInfo`
7. **Externalizable** = full control + public no-arg constructor required; faster but more code
