# Reflection, Dynamic Proxies & AOP — Interview Preparation Guide

> **Why this matters**: "How does Spring `@Transactional` work?" and "How does Hibernate lazy loading work?" are common senior interview questions. The answer is: **reflection + dynamic proxies**.

---

## Table of Contents

- [1. Java Reflection API](#1-java-reflection-api)
- [2. Dynamic Proxies (java.lang.reflect.Proxy)](#2-dynamic-proxies)
- [3. CGLIB Proxies (Class-based)](#3-cglib-proxies)
- [4. Aspect-Oriented Programming (AOP)](#4-aop--how-spring-uses-proxies)
- [5. Annotations & Reflection](#5-annotations--reflection)
- [6. Interview Questions & Answers](#6-interview-questions--answers)

---

## 1. Java Reflection API

Reflection lets you **inspect and manipulate classes, methods, and fields at runtime** — without knowing them at compile time. Used by frameworks (Spring, Hibernate, Jackson) extensively.

```java
/**
 * Core Reflection APIs
 */
public class ReflectionDemo {

    // --- Class Inspection ---
    public static void inspectClass(Object obj) {
        Class<?> clazz = obj.getClass();

        System.out.println("Class: " + clazz.getName());
        System.out.println("Simple: " + clazz.getSimpleName());
        System.out.println("Package: " + clazz.getPackageName());
        System.out.println("Superclass: " + clazz.getSuperclass().getName());
        System.out.println("Interfaces: " + Arrays.toString(clazz.getInterfaces()));
    }

    // --- Field Access (even private!) ---
    public static void accessPrivateField(Object obj, String fieldName) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);  // Bypass private access control
        Object value = field.get(obj);
        System.out.println(fieldName + " = " + value);
    }

    // --- Method Invocation ---
    public static Object invokeMethod(Object target, String methodName, Object... args)
            throws Exception {
        Class<?>[] paramTypes = Arrays.stream(args)
            .map(Object::getClass).toArray(Class[]::new);
        Method method = target.getClass().getDeclaredMethod(methodName, paramTypes);
        method.setAccessible(true);
        return method.invoke(target, args);  // Invoke with runtime dispatch
    }

    // --- Creating Instances ---
    public static Object createInstance(String className) throws Exception {
        Class<?> clazz = Class.forName(className);  // Load class by name
        Constructor<?> constructor = clazz.getDeclaredConstructor();
        constructor.setAccessible(true);
        return constructor.newInstance();
    }
}

// Real-world example: Jackson uses reflection to serialize/deserialize
public class BankAccount {
    private String accountId;
    private BigDecimal balance;

    // Jackson calls getDeclaredFields() and uses getters/setters via reflection
    // It never needs to know BankAccount at compile time!
}
```

### Reflection Performance Considerations

```java
// ❌ Slow: reflect on every call
public Object slowRead(Object obj, String field) throws Exception {
    return obj.getClass().getDeclaredField(field)  // lookup every time
              .get(obj);
}

// ✅ Fast: cache the Field/Method object, only invoke at runtime
private static final Map<String, Field> FIELD_CACHE = new ConcurrentHashMap<>();

public Object fastRead(Object obj, String fieldName) throws Exception {
    Field field = FIELD_CACHE.computeIfAbsent(
        obj.getClass().getName() + "." + fieldName,
        k -> {
            try {
                Field f = obj.getClass().getDeclaredField(fieldName);
                f.setAccessible(true);
                return f;
            } catch (NoSuchFieldException e) { throw new RuntimeException(e); }
        }
    );
    return field.get(obj);
}
```

---

## 2. Dynamic Proxies

**JDK Dynamic Proxies** create a proxy object at runtime that implements one or more interfaces. The proxy intercepts method calls and delegates to an `InvocationHandler`.

**Requirement**: The target must implement at least one interface.

```java
/**
 * Dynamic Proxy — wraps a service to add cross-cutting behavior.
 * This is exactly what Spring does for @Transactional, @Cacheable, @Retryable.
 */
public interface AccountService {
    Account findById(String id);
    void updateBalance(String id, BigDecimal amount);
}

public class AccountServiceImpl implements AccountService {
    @Override
    public Account findById(String id) { return repository.findById(id); }
    @Override
    public void updateBalance(String id, BigDecimal amount) {
        repository.updateBalance(id, amount);
    }
}

// The InvocationHandler — intercepts all method calls on the proxy
public class LoggingInvocationHandler implements InvocationHandler {
    private final Object target;   // The real object being proxied

    public LoggingInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        System.out.println(">>> " + methodName + " called");
        long start = System.currentTimeMillis();

        try {
            Object result = method.invoke(target, args);  // Call real method
            System.out.println("<<< " + methodName + " succeeded in "
                + (System.currentTimeMillis() - start) + "ms");
            return result;
        } catch (InvocationTargetException e) {
            System.out.println("<<< " + methodName + " FAILED: " + e.getCause().getMessage());
            throw e.getCause();  // Unwrap and rethrow real exception
        }
    }
}

// Creating the proxy
AccountService realService = new AccountServiceImpl(repository);
AccountService proxy = (AccountService) Proxy.newProxyInstance(
    AccountService.class.getClassLoader(),
    new Class<?>[] { AccountService.class },   // Interfaces to implement
    new LoggingInvocationHandler(realService)  // Intercept handler
);

// All calls to proxy go through LoggingInvocationHandler first
proxy.findById("ACC-001");       // Logged automatically
proxy.updateBalance("ACC-001", new BigDecimal("1000"));  // Also logged
```

### How Spring's `@Transactional` Works Internally

```java
// When Spring sees @Transactional, it creates a proxy like this:
public class TransactionInvocationHandler implements InvocationHandler {
    private final Object target;
    private final PlatformTransactionManager txManager;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. Check if method has @Transactional
        if (!method.isAnnotationPresent(Transactional.class)) {
            return method.invoke(target, args);  // No transaction needed
        }

        // 2. Begin transaction
        TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());

        try {
            // 3. Call the real method
            Object result = method.invoke(target, args);

            // 4. Commit on success
            txManager.commit(status);
            return result;
        } catch (InvocationTargetException e) {
            // 5. Rollback on exception
            txManager.rollback(status);
            throw e.getCause();
        }
    }
}

// THIS IS WHY self-invocation doesn't work:
@Service
public class PaymentService {
    @Transactional
    public void outerMethod() {
        innerMethod();  // ❌ Calls this.innerMethod() directly — bypasses proxy!
        // The proxy is NOT involved. No transaction for innerMethod.
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void innerMethod() {
        // @Transactional here has no effect when called from outerMethod!
    }
}
```

---

## 3. CGLIB Proxies

When the target class has **no interface**, Spring uses **CGLIB** (Code Generation Library) to generate a **subclass** of the target at runtime.

```java
// No interface — CGLIB creates a subclass
@Service
public class PaymentProcessor {  // Concrete class, no interface

    @Transactional
    public void processPayment(Payment payment) {
        // Spring can't use JDK proxy (no interface)
        // CGLIB generates: class PaymentProcessor$$EnhancerBySpringCGLIB extends PaymentProcessor
        // and overrides processPayment() to add transaction logic
    }

    // ❌ CGLIB cannot proxy final methods!
    @Transactional
    public final void finalMethod() {
        // This @Transactional is silently IGNORED — CGLIB can't override final
    }
}
```

**JDK Proxy vs CGLIB Comparison**:

| Aspect | JDK Dynamic Proxy | CGLIB Proxy |
|--------|-------------------|-------------|
| Required | Target must implement interface | Works on any class |
| Mechanism | Implements same interface | Generates subclass |
| `final` methods | Proxied (interface contract) | Missed — `final` can't be overridden |
| Performance | Slightly faster creation | Slightly slower creation, similar invoke |
| Spring default | Yes, if interface exists | Yes, if no interface (or forced with `proxyTargetClass = true`) |

---

## 4. AOP — How Spring Uses Proxies

**Aspect-Oriented Programming** lets you add cross-cutting concerns (logging, security, transactions, caching) without cluttering business code.

```java
// Spring AOP — declarative proxy configuration
@Aspect
@Component
public class PerformanceMonitoringAspect {

    // Pointcut: matches any method in any class in the payment package
    @Around("execution(* com.bank.payment..*(..))")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().toShortString();
        long start = System.currentTimeMillis();

        try {
            Object result = joinPoint.proceed();  // Call the real method
            long elapsed = System.currentTimeMillis() - start;

            if (elapsed > 500) {
                log.warn("SLOW: {} took {}ms", method, elapsed);
            }
            return result;
        } catch (Exception e) {
            log.error("FAILED: {} with {}", method, e.getMessage());
            throw e;
        }
    }
}

// Under the hood, Spring creates a proxy for every bean matching the pointcut.
// The proxy calls the aspect's monitorPerformance() which wraps the real method.
// TO THE CALLER, it looks identical to the real bean.
```

---

## 5. Annotations & Reflection

Annotations are metadata on classes/methods/fields. Frameworks read them via reflection.

```java
// Defining a custom annotation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)  // Must be RUNTIME for reflection to see it
public @interface Audited {
    String action() default "OPERATION";
    boolean logParams() default false;
}

// Using the annotation
public class TransactionService {
    @Audited(action = "PAYMENT", logParams = true)
    public void processPayment(Payment payment) { ... }
}

// Reading annotations via reflection
public class AuditReflectionReader {
    public void processAuditedMethods(Object bean) throws Exception {
        Class<?> clazz = bean.getClass();

        for (Method method : clazz.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Audited.class)) {
                Audited annotation = method.getAnnotation(Audited.class);
                System.out.println("Method: " + method.getName()
                    + ", Action: " + annotation.action()
                    + ", LogParams: " + annotation.logParams());
            }
        }
    }
}
```

---

## 6. Interview Questions & Answers

**Q1: How does Spring's `@Transactional` work under the hood?**

> "Spring creates a proxy (JDK dynamic proxy if the bean implements an interface, CGLIB subclass otherwise) around your @Service bean. When a transactional method is called, the proxy intercepts the call, starts a transaction using `PlatformTransactionManager`, delegates to the real method, then commits or rolls back depending on outcome.
>
> The critical implication: self-invocation within the same class bypasses the proxy. `this.method()` goes directly to the object, not through the proxy. So `@Transactional` on an internally-called method has no effect."

**Q2: Why can't you proxy `final` methods in Spring with CGLIB?**

> "CGLIB works by generating a runtime subclass. Final methods can't be overridden in Java. So CGLIB's generated subclass cannot intercept them. The `@Transactional` annotation on a final method is silently ignored — no transaction is started. This is a common trap. Solution: remove final, or use interface-based (JDK) proxying."

**Q3: What is the performance cost of reflection?**

> "Reflection has several costs:
> 1. Class/Method lookup (`getDeclaredMethod`) is expensive — it scans through all methods
> 2. `setAccessible(true)` involves a security check
> 3. Method invocation via `method.invoke()` is slower than direct calls due to boxing, varargs overhead
>
> In practice: cache `Field`/`Method` objects (as frameworks like Jackson and Spring do). The lookup is slow; the invoke is acceptable. Never do `getDeclaredMethod` in a hot loop."

**Q4: How does Hibernate lazy loading work?**

> "Hibernate creates a CGLIB proxy subclass for your entity. The proxy has the entity's ID but not its data. When you access a relationship (`order.getItems()`), CGLIB intercepts the call, queries the database for the collection, and populates it. That's the 'lazy load'.
>
> Implication: if the Hibernate Session is closed (common in Spring controllers), accessing a lazy collection throws `LazyInitializationException`. Fix: use @Transactional to keep the session open, use `JOIN FETCH` in JPQL, or use `@EntityGraph`."

---

## Key Takeaways

1. **Reflection** lets you inspect and call code at runtime — powerful but cache the lookups
2. **JDK Dynamic Proxy** needs an interface; generates proxy that delegates to `InvocationHandler`
3. **CGLIB** generates a subclass; doesn't require interface; can't proxy `final` methods
4. **Spring defaults**: JDK proxy if interface exists, CGLIB otherwise (or always CGLIB if `proxyTargetClass=true`)
5. **Self-invocation bypasses proxy** — `@Transactional` / `@Cacheable` on internal calls is ignored
6. **Annotations + Reflection = framework magic** — read metadata at runtime to drive behavior
7. **AOP** = systematic proxying — reduce boilerplate for cross-cutting concerns
