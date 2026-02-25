# Spring Caching, Scheduling, and Async Processing

## Overview

Caching, scheduling, and asynchronous processing are three orthogonal but related performance patterns in Spring Boot applications. Caching reduces redundant computation and database load. Scheduling enables time-based and periodic task execution. Async processing decouples work from the request thread to improve response times and throughput.

In enterprise banking, these patterns are ubiquitous: FX rates are cached (they don't change per request), end-of-day processing runs on a schedule, compliance reports are generated asynchronously, and payment notifications are sent on a background thread.

---

## Spring Caching Abstraction

### The @EnableCaching Foundation

```java
@Configuration
@EnableCaching
public class CachingConfig {
    
    // ─── In-memory cache (development/simple use cases) ──────────────
    @Bean
    @Profile("!production")
    public CacheManager localCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats()  // Enable statistics
        );
        return manager;
    }
    
    // ─── Redis distributed cache (production) ─────────────────────────
    @Bean
    @Profile("production")
    public CacheManager redisCacheManager(RedisConnectionFactory connectionFactory) {
        RedisSerializer<Object> valueSerializer = new GenericJackson2JsonRedisSerializer(objectMapper());
        
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))           // Default TTL
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(valueSerializer))
            .disableCachingNullValues();                 // Don't cache nulls (usually)
        
        // Per-cache TTL configuration
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("fxRates", 
            defaultConfig.entryTtl(Duration.ofMinutes(5)));
        cacheConfigurations.put("accountBal", 
            defaultConfig.entryTtl(Duration.ofSeconds(30)));
        cacheConfigurations.put("fraudRules", 
            defaultConfig.entryTtl(Duration.ofHours(1)));
        cacheConfigurations.put("staticData",
            defaultConfig.entryTtl(Duration.ofHours(24)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .transactionAware()  // Synchronize with Spring transactions
            .build();
    }
    
    // Multi-level cache: local (Caffeine) + distributed (Redis)
    @Bean
    @Profile("production")
    public CacheManager multiLevelCacheManager(RedisConnectionFactory redisConnectionFactory) {
        return new MultiLevelCacheManager(
            caffeineCacheManager(),  // L1: local, fast
            redisCacheManager(redisConnectionFactory)  // L2: distributed, shared
        );
    }
}
```

### Caching Annotations in Practice

```java
@Service
public class FxRateService {
    
    private final FxRateRepository fxRateRepository;
    private final ExternalFxProvider externalProvider;
    
    // ─── @Cacheable: Return cached value, or compute and cache ────────
    // Key: "fxRates::GBP-USD" (cacheName::key)
    @Cacheable(
        value = "fxRates",
        key = "#baseCurrency + '-' + #quoteCurrency",
        condition = "#baseCurrency != null",  // Don't cache if null input
        unless = "#result == null"             // Don't cache null results
    )
    public BigDecimal getFxRate(String baseCurrency, String quoteCurrency) {
        log.info("Cache MISS: loading FX rate {}/{}", baseCurrency, quoteCurrency);
        return externalProvider.fetchRate(baseCurrency, quoteCurrency);
        // Result is cached after this method returns
    }
    
    // ─── SpEL in cache key ────────────────────────────────────────────
    @Cacheable(
        value = "accountBalance",
        key = "#account.id + ':' + #account.currency",
        sync = true  // Prevent cache stampede: only one thread computes, others wait
    )
    public Money getBalance(Account account) {
        return accountRepository.calculateBalance(account.getId());
    }
    
    // ─── @CachePut: Always update cache, run method ────────────────────
    @CachePut(
        value = "fxRates",
        key = "#baseCurrency + '-' + #quoteCurrency"
    )
    @Scheduled(fixedRate = 60_000)
    public Map<String, BigDecimal> refreshFxRates() {
        // Always executed and result always stored in cache
        return externalProvider.fetchAllRates();
    }
    
    // ─── @CacheEvict: Remove entries from cache ────────────────────────
    @CacheEvict(
        value = "fxRates",
        key = "#baseCurrency + '-' + #quoteCurrency"
    )
    public void invalidateRate(String baseCurrency, String quoteCurrency) {
        // Evicts specific cache entry — next call will recompute
    }
    
    @CacheEvict(value = "fxRates", allEntries = true)
    @Scheduled(cron = "0 0 1 * * *")  // Every day at 01:00
    public void clearAllFxRates() {
        log.info("Clearing all FX rate cache entries");
        // Evicts ENTIRE fxRates cache
    }
    
    // ─── @Caching: Multiple cache operations on one method ────────────
    @Caching(
        cacheable = @Cacheable("accountSummary"),
        put = @CachePut("accountDetails"),
        evict = @CacheEvict(value = "accountList", allEntries = true)
    )
    public AccountDetails getFullDetails(UUID accountId) {
        return accountRepository.findFullDetails(accountId);
    }
}
```

### Custom Cache Key Generator

```java
// For complex keys that can't be expressed with SpEL
@Component("bankingCacheKeyGenerator")
public class BankingCacheKeyGenerator implements KeyGenerator {
    
    @Override
    public Object generate(Object target, Method method, Object... params) {
        StringBuilder key = new StringBuilder();
        key.append(target.getClass().getSimpleName()).append(":");
        key.append(method.getName()).append(":");
        
        for (Object param : params) {
            if (param instanceof Auditable a) {
                key.append(a.getId());
            } else if (param instanceof String s) {
                key.append(s.toLowerCase());  // Normalise
            } else {
                key.append(param);
            }
            key.append("_");
        }
        
        return key.toString();
    }
}

@Cacheable(value = "reports", keyGenerator = "bankingCacheKeyGenerator")
public Report generateReport(Account account, DateRange period) { ... }
```

### Cache Stampede Prevention

```java
// Without sync=true: multiple threads all miss cache simultaneously
// → all compute → database overloaded → performance degradation
@Cacheable(value = "balances", key = "#accountId", sync = true)
public BigDecimal getBalance(UUID accountId) {
    // sync=true: Only ONE thread calls this method for a given key
    // Other threads wait for the result — prevents thundering herd
    return accountRepository.calculateBalance(accountId);
}
```

---

## Spring Scheduling

```java
@Configuration
@EnableScheduling
public class SchedulingConfig {
    
    // Custom thread pool for scheduled tasks
    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setErrorHandler(t -> log.error("Scheduled task error: {}", t.getMessage(), t));
        scheduler.initialize();
        return scheduler;
    }
}

@Component
public class BankingScheduledTasks {
    
    // ─── Fixed Rate: run every 1 minute (even if previous execution takes longer) ──
    @Scheduled(fixedRate = 60_000, initialDelay = 30_000)
    public void syncFxRates() {
        log.info("Syncing FX rates...");
        fxRateService.refreshAll();
    }
    
    // ─── Fixed Delay: wait 30 seconds AFTER previous execution completes ─────────
    @Scheduled(fixedDelay = 30_000)
    public void processOutboxEvents() {
        log.info("Processing outbox events...");
        outboxRelay.relayPendingEvents();
    }
    
    // ─── Cron expressions: Complex scheduling ────────────────────────────────────
    // Cron format: second minute hour day-of-month month day-of-week
    
    @Scheduled(cron = "0 0 22 * * MON-FRI", zone = "Europe/London")
    public void endOfDayProcessing() {
        log.info("Starting end-of-day processing...");
        eodService.processAllAccounts();
    }
    
    @Scheduled(cron = "0 0 1 1 * *")  // First day of each month at 01:00
    public void monthlyStatements() {
        statementService.generateMonthlyStatements();
    }
    
    @Scheduled(cron = "0 */15 9-17 * * MON-FRI")  // Every 15 min, 9am-5pm, weekdays
    public void refreshLiquidityPositions() {
        liquidityService.refreshAll();
    }
    
    // ─── External configuration of schedule ──────────────────────────────────────
    @Scheduled(cron = "${app.scheduling.eod.cron:0 0 22 * * MON-FRI}")
    public void configurableEod() {
        // Schedule defined in config — easily changed per environment
    }
}

// Common cron expression cheat sheet:
// "0 0 12 * * MON-FRI"     = 12:00 PM on weekdays
// "0 */5 * * * *"          = Every 5 minutes
// "0 0 0 L * *"            = Last day of every month at midnight
// "0 0 6 * * SAT,SUN"      = 6am on weekends
// "0 0/30 8-18 * * MON-FRI"= Every 30min, 8am-6pm, Mon-Fri
```

### Distributed Scheduling with ShedLock

When multiple instances of a service run, scheduled tasks would execute on ALL instances simultaneously. ShedLock ensures only ONE instance runs each task:

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class ShedLockConfig {
    
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()  // Use DB time for consistency across servers
                .build()
        );
    }
}

@Component
public class DistributedScheduledTasks {
    
    @Scheduled(cron = "0 0 22 * * MON-FRI")
    @SchedulerLock(
        name = "eodProcessing",           // Lock name (must be unique)
        lockAtLeastFor = "1m",            // Hold lock for at least 1 minute
        lockAtMostFor = "30m"             // Release lock after 30min even if crashed
    )
    public void endOfDayProcessing() {
        // Only ONE instance across the cluster will execute this at a time
        log.info("Running EOD processing on instance: {}", instanceId);
        eodService.processAll();
    }
}
```

```sql
-- ShedLock requires a database table:
CREATE TABLE shedlock (
    name         VARCHAR(64) NOT NULL,
    lock_until   TIMESTAMP NOT NULL,
    locked_at    TIMESTAMP NOT NULL,
    locked_by    VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

---

## @Async Processing

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    // Custom thread pool for async tasks
    @Bean(name = "asyncExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-task-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // Handle uncaught exceptions in async tasks
        executor.initialize();
        return executor;
    }
    
    // Separate high-priority pool for payment processing
    @Bean(name = "paymentAsyncExecutor")
    public Executor paymentAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(50);    // Smaller queue — fail faster
        executor.setThreadNamePrefix("payment-async-");
        executor.initialize();
        return executor;
    }
    
    // Async exception handler
    @Bean
    public AsyncUncaughtExceptionHandler asyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> {
            log.error("Async exception in method: {}.{}({})", 
                method.getDeclaringClass().getSimpleName(),
                method.getName(),
                Arrays.toString(params),
                throwable);
            alertingService.notifyAsyncFailure(throwable, method);
        };
    }
}

@Service
public class NotificationService {
    
    // ─── Fire-and-forget async ────────────────────────────────────────
    @Async("asyncExecutor")
    public void sendPaymentConfirmation(Payment payment) {
        // Runs on asyncExecutor thread pool
        // Caller doesn't wait for this to complete
        emailProvider.send(EmailTemplate.PAYMENT_CONFIRMATION, payment);
        smsProvider.send(payment.getPhoneNumber(), buildSmsMessage(payment));
    }
    
    // ─── Async with CompletableFuture return ──────────────────────────
    @Async("paymentAsyncExecutor")
    public CompletableFuture<NotificationResult> sendNotificationAsync(Payment payment) {
        try {
            NotificationResult result = emailProvider.sendAndWait(payment);
            return CompletableFuture.completedFuture(result);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
    
    // ─── Combining async results ──────────────────────────────────────
    public PaymentConfirmation sendAllNotifications(Payment payment) {
        CompletableFuture<NotificationResult> emailFuture = sendEmailAsync(payment);
        CompletableFuture<NotificationResult> smsFuture = sendSmsAsync(payment);
        CompletableFuture<NotificationResult> pushFuture = sendPushAsync(payment);
        
        // Wait for all to complete
        return CompletableFuture.allOf(emailFuture, smsFuture, pushFuture)
            .thenApply(v -> PaymentConfirmation.builder()
                .emailSent(emailFuture.join().isSuccess())
                .smsSent(smsFuture.join().isSuccess())
                .pushSent(pushFuture.join().isSuccess())
                .build())
            .join();  // Block calling thread until all done
    }
}
```

### Virtual Threads (Spring Boot 3.2+ / Java 21)

```yaml
# Enable virtual threads — replaces traditional thread pools for I/O-bound work
spring:
  threads:
    virtual:
      enabled: true  # All @Async, scheduled tasks, and server threads use virtual threads
```

```java
// With virtual threads enabled, you can dramatically simplify async patterns
// Virtual threads are lightweight — create millions without the thread pool overhead

@Service
public class ModernAsyncPaymentService {
    
    // With virtual threads: each @Async call creates a new virtual thread
    // No need to pre-configure thread pool sizes!
    @Async
    public CompletableFuture<FraudAssessment> assessFraudAsync(Payment payment) {
        // This blocks waiting for external service
        // With virtual threads: blocking is cheap! The carrier thread is NOT blocked.
        FraudAssessment assessment = fraudServiceClient.assess(payment);  // Blocking HTTP call
        return CompletableFuture.completedFuture(assessment);
    }
    
    // Practical impact: Replace reactive (WebFlux) complexity with simple blocking code
    // that scales just as well via virtual threads
}
```

---

## Interview Questions & Model Answers

### Q1: How does Spring's @Cacheable work internally?

**Model Answer**: `@Cacheable` is implemented via Spring AOP proxy. When `@EnableCaching` is declared, Spring registers `CacheInterceptor` as a `BeanPostProcessor` that wraps all beans with `@Cacheable`/`@CacheEvict`/`@CachePut` methods in a proxy.

When the proxy intercepts a `@Cacheable` method call:
1. It generates the cache key (from the `key` SpEL expression, or the default `KeyGenerator`)
2. Looks up the key in the configured `CacheManager` + named cache
3. If **cache hit**: returns cached value WITHOUT calling the actual method
4. If **cache miss**: calls the actual method, stores the result in the cache, returns it

**Critical gotcha**: Self-invocation. Just like `@Transactional`, if you call a `@Cacheable` method from another method in the SAME bean (`this.getCachedData()`), the call bypasses the proxy — the cache is NOT consulted. The solution is the same: separate the cached method into another bean.

Also: `@Cacheable` only caches per-class-instance, not per-application. With the default `ConcurrentMapCacheManager`, each JVM instance has its own cache. For distributed caching (multiple service instances), use Redis via `RedisCacheManager`.

---

### Q2: When is @Scheduled fixedRate vs fixedDelay appropriate?

**Model Answer**: The difference is how the delay is measured:

**`fixedRate = 60000`**: Scheduled 60 seconds from the START of the last execution. If the task takes 50 seconds, the next execution starts 10 seconds after it finishes. If the task takes 90 seconds (longer than the rate), the next execution starts immediately after (no backlog by default — single-threaded).

**`fixedDelay = 60000`**: Waits 60 seconds from the END of the last execution. Always 60 seconds between finish and next start.

**When to use each**:
- `fixedRate`: Use when you want consistent "heartbeat" - e.g., metrics collection that should happen every minute regardless of processing time.
- `fixedDelay`: Use when you want a guaranteed rest period between executions - e.g., polling an external system where you want to avoid overloading it, or processing outbox events where you want spacing between batches.

For distributed systems: use ShedLock with either, plus `@SchedulerLock`, to ensure only one instance runs simultaneously.

---

### Q3: What are virtual threads and how do they change async programming?

**Model Answer**: Virtual threads (Java 21, Project Loom) are lightweight threads managed by the JVM, not the OS. Traditional (platform) threads are expensive to create (each maps to an OS thread, ~1MB stack). Virtual threads have tiny stacks (starts at ~1KB), can be created by the millions, and when they block (I/O, Lock.lock()), the underlying carrier thread is released and can serve other virtual threads.

**Impact on Spring**: With `spring.threads.virtual.enabled=true`:
- The embedded Tomcat uses virtual threads (instead of fixed thread pool)
- `@Async` executes on virtual threads (instead of ThreadPoolTaskExecutor)
- Scheduled tasks run on virtual threads
- HikariCP connections run on virtual threads (with some caveats)

**Programming model**: You can write simple, synchronous/blocking code that scales as well as reactive (WebFlux) — without the complexity of Flux/Mono, flatMap chains, or thread switching. This is revolutionary: instead of `WebClient` flatMap chains, you can use blocking `RestClient` calls on a virtual thread.

**Key limitation**: Synchronized blocks (not ReentrantLock) "pin" the carrier thread — avoid `synchronized` with virtual threads. Use `ReentrantLock` instead.

---

## Key Takeaways

- **@Cacheable bypasses cache on self-invocation** — same proxy limitation as @Transactional
- **Use SpEL for cache keys** — `#entity.id`, `#param.toString()`, composite keys with `+`
- **`sync = true` prevents cache stampede** — only one thread computes for given key
- **Redis for distributed caching** — Caffeine for single-node, Redis for multiple instances
- **ShedLock for distributed scheduling** — prevents duplicate execution across instances
- **`fixedRate` vs `fixedDelay`** — rate from START, delay from END of last execution
- **Virtual threads (Java 21 + Boot 3.2)** — replace thread pool tuning; blocking is cheap
- **@Async limitations**: self-invocation bypasses proxy, no transaction propagation across thread boundaries
- **Multi-level cache**: Caffeine (L1, fast) + Redis (L2, shared across instances)

---

## Further Reading

- [Spring Cache Abstraction Reference](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [Spring Boot Virtual Threads Support](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.virtual-threads)
- [ShedLock GitHub](https://github.com/lukas-krecan/ShedLock)
- [Baeldung — Spring Caching Tutorial](https://www.baeldung.com/spring-cache-tutorial)
