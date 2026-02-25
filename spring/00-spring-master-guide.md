# Spring Framework & Spring Boot: Master Interview Guide

## Overview

This master guide provides a comprehensive reference for all Spring Framework and Spring Boot topics covered in the individual guides. Use this as your navigation map — the individual files contain the deep technical content; this guide provides the architecture overview, topic relationships, and high-level summaries.

---

## File Index

| File | Topic | Priority |
|---|---|---|
| `01-spring-core-ioc-di.md` | IoC Container, Dependency Injection, Bean Registration | 🔴 Essential |
| `02-bean-lifecycle-and-scopes.md` | Bean Lifecycle, Scopes, Custom Scopes | 🔴 Essential |
| `03-spring-aop.md` | AOP, Proxies, AspectJ, Cross-cutting Concerns | 🔴 Essential |
| `04-spring-boot-fundamentals.md` | Auto-configuration, Starters, Spring Boot Internals | 🔴 Essential |
| `05-spring-boot-configuration.md` | Configuration Management, Profiles, Properties | 🔴 Essential |
| `06-spring-boot-actuator.md` | Actuator Endpoints, Health Checks, Custom Indicators | 🟡 Important |
| `07-spring-mvc-deep-dive.md` | DispatcherServlet, Controllers, Exception Handling | 🔴 Essential |
| `08-spring-webflux-reactive.md` | Reactive Programming, WebFlux, Reactor | 🟡 Important |
| `09-spring-data-jpa.md` | JPA Repository, N+1, Projections, Migrations | 🔴 Essential |
| `10-transaction-management.md` | @Transactional, Propagation, Isolation, Rollback | 🔴 Essential |
| `11-spring-security-authentication.md` | Security Filter Chain, JWT, OAuth2 | 🔴 Essential |
| `12-spring-security-authorization.md` | Method Security, RBAC, @PreAuthorize | 🟡 Important |
| `13-spring-boot-testing.md` | @WebMvcTest, @DataJpaTest, TestContainers | 🔴 Essential |
| `14-spring-cloud-microservices-patterns.md` | Gateway, Circuit Breaker, Config Server | 🔴 Essential |
| `15-spring-rest-api-design.md` | REST API Best Practices, Versioning, ProblemDetail | 🟡 Important |
| `16-spring-kafka-messaging.md` | Kafka Producer/Consumer, Error Handling, Outbox | 🔴 Essential |
| `17-18-spring-caching-scheduling-async.md` | Caching, Scheduling, @Async, Virtual Threads | 🟡 Important |
| `19-spring-observability-monitoring.md` | Micrometer, Tracing, Structured Logging | 🟡 Important |
| `20-spring-data-redis.md` | Redis Integration, Caching, Pub/Sub | 🟠 Good to Know |
| `21-spring-batch.md` | Batch Jobs, Steps, Chunk Processing | 🟠 Good to Know |
| `22-spring-boot-version-features-migration.md` | 2.x→3.x Migration, jakarta namespace | 🔴 Essential |
| `23-spring-boot-performance-tuning.md` | HikariCP, JVM Tuning, Batch Inserts | 🔴 Essential |
| `24-spring-design-patterns.md` | Factory, Strategy, Template, Builder in Spring | 🟡 Important |

---

## Core Concept Map

```mermaid
graph TB
    subgraph "Spring Container"
        IOC[IoC Container\nApplicationContext]
        BEANS[Bean Registry\n@Component, @Bean]
        LIFE[Bean Lifecycle\nPostConstruct, Destroy]
        SCOPE[Bean Scopes\nSingleton, Prototype, Request]
        
        IOC --> BEANS
        BEANS --> LIFE
        BEANS --> SCOPE
    end
    
    subgraph "AOP"
        PROXY[Proxy Creation\nCGLIB / JDK Dynamic]
        ASPECT[Aspects\n@Before, @After, @Around]
        CROSS[Cross-cutting\n@Transactional, @Cacheable\n@Async, @Scheduled]
        
        PROXY --> ASPECT
        ASPECT --> CROSS
    end
    
    subgraph "Web Layer"
        DS[DispatcherServlet]
        CTRL[@RestController]
        EH[Exception Handling\n@ControllerAdvice]
        
        DS --> CTRL
        CTRL --> EH
    end
    
    subgraph "Data Layer"
        JPA[Spring Data JPA\nRepository]
        TX[@Transactional\nPropagation + Isolation]
        CACHE[@Cacheable\nRedis / Caffeine]
        
        JPA --> TX
        JPA --> CACHE
    end
    
    subgraph "Security"
        FILTER[Security Filter Chain]
        AUTH[Authentication\nJWT / OAuth2]
        AUTHZ[Authorization\n@PreAuthorize / RBAC]
        
        FILTER --> AUTH
        AUTH --> AUTHZ
    end
    
    IOC --> PROXY
    IOC --> DS
    IOC --> JPA
    IOC --> FILTER
```

---

## The Big Picture: How Everything Connects

### A Payment Request Flow Through Spring

```
1. HTTP Request arrives
   └─► [Servlet Container: Tomcat]
       └─► [DelegatingFilterProxy] → SecurityFilterChain
           └─► [BearerTokenFilter] validates JWT
           └─► [SecurityContextHolder] stores Authentication
           └─► [DispatcherServlet]
               └─► [HandlerMapping] finds @RequestMapping method
               └─► [HandlerInterceptor.preHandle] (CorrelationID, rate limiting)
               └─► [HandlerAdapter → RequestMappingHandlerAdapter]
                   └─► [ArgumentResolver] @RequestBody → JSON → Java object
                   └─► Bean Validation (@NotNull, @Valid)
               └─► [@RestController method]
                   └─► [AOP Proxy]
                       └─► [CachingInterceptor] @Cacheable
                       └─► [TransactionInterceptor] @Transactional
                           └─► [Actual Service Logic]
                               └─► [Repository] → JPA → SQL
                               └─► [@Async] notification sender
                               └─► [KafkaTemplate] publish event
                       └─► [MetricsInterceptor] @Timed
               └─► [HandlerInterceptor.postHandle]
               └─► [MappingJackson2HttpMessageConverter] → JSON
2. HTTP Response returns (with Correlation-ID, tracing headers)
```

---

## Critical Interview Topics: Proxy and Self-Invocation

The most common Spring interview trap question:

```java
@Service
public class OrderService {
    
    @Transactional
    public void placeOrder(Order order) {
        saveOrder(order);
        // ❌ this.sendNotification() bypasses proxy!
        this.sendNotification(order);  // @Async is IGNORED!
    }
    
    @Async  // This annotation has NO effect when called from same bean
    public void sendNotification(Order order) {
        notificationService.send(order);
    }
}

// ✅ Fix: Separate bean
@Service
public class OrderService {
    private final OrderNotificationService notificationService;  // Separate bean
    
    @Transactional
    public void placeOrder(Order order) {
        saveOrder(order);
        notificationService.sendAsync(order);  // Through proxy ✅
    }
}
```

**Rule**: `@Transactional`, `@Cacheable`, `@Async`, `@Secured`, and any AOP annotation ONLY works through proxy. Self-invocation (`this.method()`) bypasses the proxy and the annotation is ignored.

---

## Key Interview Questions Checklist

### Spring Core
- [ ] How does Spring resolve circular dependencies?
- [ ] What is the difference between @Component, @Service, @Repository, @Controller?
- [ ] What is the difference between BeanFactory vs ApplicationContext?
- [ ] How does @Autowired resolve ambiguity when multiple beans match?
- [ ] What are the different bean scopes and when to use each?
- [ ] When does Spring's proxy mechanism NOT work? (private methods, self-invocation, final classes with CGLIB)

### Spring AOP
- [ ] When does Spring use JDK dynamic proxy vs CGLIB proxy?
- [ ] What is the order of AOP advice types?
- [ ] How is @Transactional implemented internally?

### Spring Boot
- [ ] How does Spring Boot auto-configuration work?
- [ ] What is the auto-configuration order in Spring Boot 3.x?
- [ ] What is the difference between @EnableAutoConfiguration and @SpringBootApplication?

### Spring MVC
- [ ] DispatcherServlet end-to-end request processing flow
- [ ] Filter vs HandlerInterceptor — differences and use cases
- [ ] How does @RequestBody deserialization work?

### Spring Data JPA
- [ ] What is the N+1 problem and 3 solutions?
- [ ] EAGER vs LAZY fetch — which to use and why?
- [ ] What is the difference between @Transactional(propagation=REQUIRED) and REQUIRES_NEW?
- [ ] How does optimistic locking work (@Version)?

### Transaction Management
- [ ] Why doesn't @Transactional work on private/final methods?
- [ ] What is the difference between checked exceptions and @Transactional rollback rules?
- [ ] What does readOnly = true actually do?

### Spring Security
- [ ] How does the security filter chain work?
- [ ] What changed in Spring Security 6.x? (WebSecurityConfigurerAdapter removed)
- [ ] JWT vs Session-based authentication — pros/cons

### Testing
- [ ] @Mock vs @MockBean differences
- [ ] When to use @WebMvcTest vs @DataJpaTest vs @SpringBootTest?

### Microservices
- [ ] Circuit breaker states: CLOSED, OPEN, HALF_OPEN
- [ ] What is the Saga pattern and when to use choreography vs orchestration?
- [ ] What is the Outbox pattern and why is it needed?

---

## Banking Domain Context

Many interview questions in financial services will reference these patterns:

| Pattern | Technology | Banking Use Case |
|---|---|---|
| Circuit Breaker | Resilience4j | Payment gateway protection |
| CQRS | Spring Data (separate read/write repos) | Account balance reads vs transaction writes |
| Event Sourcing | Spring + Kafka | Complete transaction audit trail |
| Saga | Spring + Kafka/events | Distributed payment processing |
| Outbox | JPA + scheduled relay | Reliable event publishing |
| Optimistic Locking | JPA @Version | Concurrent balance updates |
| Two-Phase Lock | `@Lock(PESSIMISTIC_WRITE)` | High-conflict balance operations |
| Rate Limiting | Spring Cloud Gateway | API protection against abuse |
| Idempotency | Redis + UUID key | Prevent duplicate payment processing |

---

## Recommended Study Order

1. **Foundation** (Days 1-3): Files 01-04 — IoC/DI, Bean Lifecycle, AOP, Boot Fundamentals
2. **Web + Data** (Days 4-5): Files 07, 09, 10 — MVC, JPA, Transactions
3. **Security + Testing** (Day 6): Files 11, 13 — Security, Testing
4. **Microservices** (Days 7-8): Files 14, 16 — Cloud, Kafka
5. **Advanced** (Days 9-10): Files 17-23 — Caching, Observability, Performance, Migration
