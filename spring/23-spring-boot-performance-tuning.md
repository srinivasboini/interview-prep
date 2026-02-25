# Spring Boot Performance Tuning and Best Practices

## Overview

Performance tuning Spring Boot applications for high-throughput, low-latency banking systems requires understanding each layer: JVM, embedded container, connection pools, JPA/Hibernate, JSON serialization, caching, and thread management. Most performance issues have a root cause in one of these areas, and an experienced engineer can identify the layer from observable symptoms (latency histograms, GC logs, connection pool metrics).

Staff/Principal engineers are expected to have hands-on experience tuning these settings and diagnosing production performance issues — not just knowing the theory.

---

## Startup Performance

### Lazy Initialization

```yaml
# application.yml - defer bean creation until first use
spring:
  main:
    lazy-initialization: true
    # Advantages: faster startup (useful for dev, tests, serverless)
    # Disadvantages: first request is slow; startup errors may not appear until runtime
    # Best: Enable in dev/test, evaluate for production based on K8s startup time requirements
```

### Class Data Sharing (Spring Boot 3.3+)

```bash
# Spring Boot 3.3+ provides easy CDS support via spring.aot.enabled
mvn spring-boot:process-aot  # Generate AOT-optimized code

# Multi-step CDS setup for maximum startup performance
java -Dspring.context.exit=onRefresh -jar app.jar     # Training run
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar    # Dump classes
java -XX:SharedArchiveFile=app.jsa -jar app.jar       # Run with CDS (~30-50% faster startup)
```

### Thin JARs vs Fat JARs

```xml
<!-- Thin JAR: dependencies in separate layer (better Docker caching)  -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

```dockerfile
# Multi-stage Docker build with layer extraction (Spring Boot 2.3+)
FROM eclipse-temurin:21-jre AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# JVM tuning for containers
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Embedded Container Tuning

```yaml
server:
  # Tomcat thread pool (critical for throughput vs latency trade-off)
  tomcat:
    threads:
      max: 200        # Max threads (default: 200, increase for slow/I/O-bound endpoints)
      min-spare: 20   # Minimum threads kept alive
    accept-count: 100   # Queue depth when all threads busy
    max-connections: 8192  # Max concurrent connections
    connection-timeout: 20000  # 20s connection timeout
    
    # Performance: Compress responses for API clients that accept it
    compression:
      enabled: true
      min-response-size: 2048  # Only compress if > 2KB
      mime-types: application/json,application/xml,text/html
    
    # HTTP/2 support (requires SSL)
    remoteip:
      remote-ip-header: X-Forwarded-For
      protocol-header: X-Forwarded-Proto

  # HTTP/2 for newer clients
  http2:
    enabled: true
    
  # Read/write timeouts
  max-http-request-header-size: 16KB
```

### Virtual Threads (Java 21 + Spring Boot 3.2)

```yaml
# With virtual threads enabled, thread pool tuning is largely unnecessary for I/O-bound apps
spring:
  threads:
    virtual:
      enabled: true

# Virtual threads with Tomcat:
# - Each request gets its own virtual thread
# - No max-threads limit needed (JVM manages millions of VTs)
# - Blocking I/O operations don't consume OS threads
# - Dramatically simplified configuration
```

---

## HikariCP Connection Pool

```yaml
spring:
  datasource:
    hikari:
      # Pool size formula: (core_count * 2) + effective_spindle_count
      # For 8-core server with SSD: (8 * 2) + 1 = 17 → ~20
      maximum-pool-size: 20
      minimum-idle: 10
      
      connection-timeout: 30000    # Max 30s to get connection from pool
      idle-timeout: 600000         # Connection idle 10min → removed from pool
      max-lifetime: 1800000        # Recycle connections after 30min (prevents stale connections)
      keepalive-time: 60000        # Send keepalive if idle > 1min
      
      # Connection validation
      connection-test-query: SELECT 1
      validation-timeout: 5000
      
      # Connection leak detection
      leak-detection-threshold: 60000  # Log warning if connection held > 60s
      
      # Pool name for metrics
      pool-name: payment-pool
      
      # PostgreSQL JDBC optimizations
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        useLocalSessionState: true
        rewriteBatchedStatements: true  # Batch INSERT optimization
        cacheResultSetMetadata: true
        zeroDateTimeBehavior: convertToNull
```

---

## JPA / Hibernate Performance

```properties
# application.properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true

# Query plan caching (avoid repeated query parsing)
spring.jpa.properties.hibernate.query.plan_cache_max_size=2048
spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size=128

# Second-level cache (Ehcache or Redis)
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
spring.jpa.properties.hibernate.cache.use_structured_entries=false

# Statistics for performance monitoring (DISABLE in production)
spring.jpa.properties.hibernate.generate_statistics=false
spring.jpa.show-sql=false        # NEVER true in production (log pollution + performance)

# Slow query logging
logging.level.org.hibernate.SQL=DEBUG     # Log all SQL (dev only)
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE  # Log bind params (dev only)
```

```java
// Batch inserts (requires batch_size > 1 AND IDENTITY strategy replaced with SEQUENCE)
@Entity
public class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "payment_seq")
    @SequenceGenerator(name = "payment_seq", sequenceName = "payment_id_seq", allocationSize = 50)
    private Long id;  
    // allocationSize=50: Hibernate allocates 50 IDs per DB roundtrip
    // IDENTITY would disable batching! SEQUENCE supports batching.
}

// Batch insert service
@Service
public class BulkPaymentService {
    
    @Autowired
    private EntityManager entityManager;
    
    @Transactional
    public void createBulkPayments(List<Payment> payments) {
        int batchSize = 50;
        for (int i = 0; i < payments.size(); i++) {
            entityManager.persist(payments.get(i));
            
            if (i % batchSize == 0 && i > 0) {
                entityManager.flush();   // Execute batch to DB
                entityManager.clear();   // Clear persistence context (prevent OOM)
            }
        }
    }
}
```

---

## JSON Serialization Performance

```java
// Jackson configuration for banking APIs
@Configuration
public class JacksonPerformanceConfig {
    
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer customizer() {
        return builder -> builder
            // ─── Disable expensive defaults ───────────────────────────
            .featuresToDisable(
                SerializationFeature.WRITE_DATES_AS_TIMESTAMPS,
                SerializationFeature.FAIL_ON_EMPTY_BEANS,   // Avoids reflection exception
                DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES  // API evolution safety
            )
            // ─── Enable performance features ──────────────────────────
            .featuresToEnable(
                MapperFeature.DEFAULT_VIEW_INCLUSION  // @JsonView support
            )
            // ─── Date handling ─────────────────────────────────────────
            .modules(new JavaTimeModule())
            // ─── Naming strategy ──────────────────────────────────────
            .propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
            // ─── Visibility ────────────────────────────────────────────
            .visibility(PropertyAccessor.FIELD, JsonAutoDetect.Visibility.ANY)
            .visibility(PropertyAccessor.GETTER, JsonAutoDetect.Visibility.NONE);
    }
}

// ─── @JsonView for conditional serialization ──────────────────────────────
// Avoid sending sensitive fields to some clients
public class PaymentViews {
    public interface Summary {}
    public interface Details extends Summary {}
    public interface Audit extends Details {}
}

@Entity
public class Payment {
    @JsonView(PaymentViews.Summary.class)
    private UUID id;
    
    @JsonView(PaymentViews.Summary.class)
    private BigDecimal amount;
    
    @JsonView(PaymentViews.Details.class)
    private String senderIban;          // Not in summary
    
    @JsonView(PaymentViews.Audit.class)
    private String internalReference;   // Admin only
}

@GetMapping("/payments/{id}")
@JsonView(PaymentViews.Summary.class)  // Only Summary fields sent
public Payment getPaymentSummary(@PathVariable UUID id) {
    return paymentService.findById(id);
}
```

---

## Response Time Optimization Patterns

```java
// ─── Parallel processing with CompletableFuture ──────────────────────────
@GetMapping("/dashboard")
public CompletableFuture<DashboardResponse> getDashboard(@RequestParam UUID accountId) {
    
    // Sequential (SLOW): 3 DB calls in sequence = 100ms + 200ms + 150ms = 450ms
    // Account account = accountService.find(accountId);
    // List<Payment> payments = paymentService.findRecent(accountId);
    // BigDecimal balance = balanceService.calculate(accountId);
    
    // Parallel (FAST): All 3 simultaneous = max(100ms, 200ms, 150ms) = 200ms
    CompletableFuture<Account> accountFuture = 
        CompletableFuture.supplyAsync(() -> accountService.find(accountId));
    CompletableFuture<List<Payment>> paymentsFuture = 
        CompletableFuture.supplyAsync(() -> paymentService.findRecent(accountId));
    CompletableFuture<BigDecimal> balanceFuture = 
        CompletableFuture.supplyAsync(() -> balanceService.calculate(accountId));
    
    return CompletableFuture.allOf(accountFuture, paymentsFuture, balanceFuture)
        .thenApply(v -> DashboardResponse.builder()
            .account(accountFuture.join())
            .recentPayments(paymentsFuture.join())
            .balance(balanceFuture.join())
            .build());
}

// ─── Streaming large responses ────────────────────────────────────────────
@GetMapping(value = "/payments/export", produces = MediaType.APPLICATION_NDJSON_VALUE)
public ResponseEntity<StreamingResponseBody> exportPayments(
        @RequestParam UUID accountId) {
    
    StreamingResponseBody stream = outputStream -> {
        try (Writer writer = new OutputStreamWriter(outputStream)) {
            // Stream results directly to response, don't load all into memory
            paymentRepository.streamByAccountId(accountId).forEach(payment -> {
                try {
                    writer.write(objectMapper.writeValueAsString(payment));
                    writer.write("\n");
                    writer.flush();
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
        }
    };
    
    return ResponseEntity.ok()
        .header("Content-Disposition", "attachment; filename=payments.ndjson")
        .body(stream);
}
```

---

## JVM Tuning for Spring Boot

```bash
# Recommended JVM flags for Spring Boot 3 + Java 21 in containers
JAVA_OPTS="
  # ─── Memory (container-aware) ─────────────────────────────────────
  -XX:MaxRAMPercentage=75          # Use 75% of container RAM for heap
  -XX:InitialRAMPercentage=50      # Start with 50%
  # Avoid hard-coding: -Xmx512m  (doesn't adapt to container limits)
  
  # ─── GC (G1GC is default, tune if needed) ─────────────────────────
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200        # Target max 200ms GC pauses
  -XX:G1HeapRegionSize=16m        # For larger heaps
  -XX:+G1UseAdaptiveIHOP          # Adaptive IHOP for mixed GC
  
  # ─── Metaspace ───────────────────────────────────────────────────── 
  -XX:MetaspaceSize=256m          # Initial metaspace
  -XX:MaxMetaspaceSize=512m       # Prevent unlimited growth
  
  # ─── JIT optimization ────────────────────────────────────────────── 
  -XX:+TieredCompilation          # Run both C1 and C2 compilers
  -XX:ReservedCodeCacheSize=256m  # Code cache for JIT (avoid CodeCache flush)
  
  # ─── Diagnostics ─────────────────────────────────────────────────── 
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/dumps/heapdump.hprof
  -XX:+PrintGCDetails             # GC logging (minimal overhead)
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=10m
  
  # ─── Virtual Threads (Java 21) ────────────────────────────────────── 
  # No special flags needed — virtual threads are in the JVM
  
  # ─── Startup optimization ─────────────────────────────────────────── 
  -XX:SharedArchiveFile=app.jsa   # CDS archive (if generated)
"
```

---

## Key Performance Anti-Patterns

| Anti-Pattern | Problem | Solution |
|---|---|---|
| `N+1 Queries` | 1 + N DB queries for N entities | JOIN FETCH, @EntityGraph, @BatchSize |
| `EAGER fetch on collections` | Cartesian product on every load | Always use LAZY for collections |
| `@SpringBootTest for every test` | Slow test suite (10-20min) | Use @WebMvcTest, @DataJpaTest slices |
| `show-sql=true in prod` | Log pollution, slightly slower | Use dedicated query profiling |
| `synchronized blocks with VT` | Pins carrier thread | Use ReentrantLock |
| `Large transactions` | Lock contention, connection pool exhaustion | Minimise transaction scope |
| `RestTemplate without pool` | New connection per request | Configure RestTemplate with HttpClient pool |
| `Disabled circuit breakers` | Cascading failures | Always configure Resilience4j |
| `IDENTITY generation with batch inserts` | Disables JDBC batch | Use SEQUENCE strategy |

---

## Interview Questions & Model Answers

### Q1: How do you diagnose a connection pool exhaustion issue?

**Model Answer**: Connection pool exhaustion manifests as requests timing out waiting for a DB connection (`HikariCP` timeout after `connectionTimeout` = 30s by default). Diagnosis steps:

1. **Actuator metrics**: Check `hikaricp.connections.active`, `hikaricp.connections.pending`, `hikaricp.connections.timeout_total` — these show pool utilization.

2. **Slow query log**: Enable `leakDetectionThreshold: 60000` — HikariCP logs warnings if a connection is held > 60s, with a stack trace showing where it was acquired.

3. **SQL query profiling**: Enable `show-sql: true` temporarily (or use P6Spy proxy) to find slow queries holding connections.

**Root causes**:
- **Too small pool**: `maximum-pool-size` too low for workload → increase
- **Long transactions**: A transaction holds a connection from start to commit — external API calls inside transactions tie up connections for seconds
- **Connection leaks**: Code that gets a connection but doesn't return it (fixed by HikariCP leak detection)
- **Slow queries**: Unindexed table scan takes 10s → connection held 10s → pool exhausts under load

**Solution priorities**: Fix long transactions first (most common), then add indexes for slow queries, then tune pool size based on DB server capacity.

---

## Key Takeaways

- **HikariCP pool size**: `(core_count × 2) + effective_spindle_count` — not arbitrary
- **Virtual threads eliminate thread pool tuning** for I/O-bound work (Spring Boot 3.2+)
- **SEQUENCE over IDENTITY** for Hibernate batch inserts
- **flush() + clear()** after every batch in persistence context (prevent OOM)
- **leakDetectionThreshold** helps diagnose long-held connections
- **Parallel CompletableFuture** for independent service calls (latency = max, not sum)
- **GC flag**: `-XX:MaxRAMPercentage=75` trumps `-Xmx` in containers (adapts to limits)
- **JSON @JsonView** for selective field exposure without DTOs
- **CDS for production** (Boot 3.3+) — 30-50% startup improvement for serverless/K8s rolling deploys

---

## Further Reading

- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP#gear-configuration-knobs-baby)
- [Hibernate Performance Best Practices — Vlad Mihalcea](https://vladmihalcea.com/tutorials/hibernate/)
- [Spring Boot Performance Blog — Baeldung](https://www.baeldung.com/spring-boot-startup-speed)
- [Azul Systems — Optimizing Java for Containers](https://www.azul.com/resources/)
- "Optimizing Java" by Benjamin J Evans, James Gough, Chris Newland
