# Spring Observability, Monitoring, and Production Readiness

## Overview

Production-ready Spring Boot applications require comprehensive observability — the ability to understand what's happening inside the system from its external outputs: metrics, logs, and traces. Spring Boot 3.x introduced Micrometer Observation API as the unifying abstraction, integrating metrics, tracing, and logging into a single API.

For enterprise banking, observability is critical for SLA compliance, incident response, and regulatory reporting. Dashboards must show transaction throughput, error rates, and latency percentiles. Distributed traces must correlate a single payment request across 5+ microservices. Structured logs must feed into SIEM systems for security monitoring.

---

## Micrometer Metrics

### Meter Types

```java
@Service
public class PaymentMetricsService {
    
    private final MeterRegistry meterRegistry;
    
    // Counter: Total payments processed (ever-increasing)
    private final Counter paymentCounter;
    private final Counter paymentFailureCounter;
    
    // Timer: Payment processing duration
    private final Timer paymentProcessingTimer;
    
    // Gauge: Current in-flight payments (can go up and down)
    private final AtomicInteger inFlightPayments = new AtomicInteger(0);
    
    // DistributionSummary: Payment amounts (statistical distribution)
    private final DistributionSummary paymentAmountSummary;
    
    public PaymentMetricsService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // Counter — dimensional metrics (tagged)
        this.paymentCounter = Counter.builder("payments.processed")
            .description("Total number of payments processed")
            .tag("service", "payment")
            .register(meterRegistry);
        
        this.paymentFailureCounter = Counter.builder("payments.failures")
            .description("Total number of payment failures")
            .register(meterRegistry);
        
        // Timer — automatically tracks count, total time, max time
        this.paymentProcessingTimer = Timer.builder("payment.processing.duration")
            .description("Payment processing time")
            .sla(Duration.ofMillis(100), Duration.ofMillis(500), Duration.ofSeconds(1))
            .percentilePrecision(2)
            .publishPercentiles(0.5, 0.90, 0.95, 0.99)   // P50, P90, P95, P99
            .register(meterRegistry);
        
        // Gauge — register with supplier, auto-updated
        Gauge.builder("payments.inflight", inFlightPayments, AtomicInteger::get)
            .description("Currently processing payments")
            .register(meterRegistry);
        
        // DistributionSummary — statistical distribution of amounts
        this.paymentAmountSummary = DistributionSummary.builder("payment.amount")
            .description("Distribution of payment amounts in GBP")
            .baseUnit("GBP")
            .minimumExpectedValue(1.0)
            .maximumExpectedValue(1_000_000.0)
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
    }
    
    @Around("execution(* com.bank.payment.PaymentService.process(..))")
    public Object recordPaymentMetrics(ProceedingJoinPoint pjp) throws Throwable {
        inFlightPayments.incrementAndGet();
        
        return paymentProcessingTimer.recordCallable(() -> {
            try {
                Object result = pjp.proceed();
                paymentCounter.increment();
                // Tag with specific attributes
                meterRegistry.counter("payments.success", 
                    "gateway", getGatewayName(),
                    "currency", getCurrency()).increment();
                return result;
            } catch (Exception e) {
                paymentFailureCounter.increment();
                meterRegistry.counter("payments.error",
                    "type", e.getClass().getSimpleName(),
                    "reason", getErrorReason(e)).increment();
                throw e;
            } finally {
                inFlightPayments.decrementAndGet();
            }
        });
    }
    
    // On custom types recorded as payment goes through
    public void recordPaymentAmount(BigDecimal amount) {
        paymentAmountSummary.record(amount.doubleValue());
    }
}
```

### @Timed and @Counted Annotations

```java
// Convenient annotation-driven metrics (requires Micrometer aspect)
@Service
public class AccountService {
    
    @Timed(
        value = "account.balance.lookup",
        description = "Time to look up account balance",
        percentiles = {0.5, 0.95, 0.99},
        histogram = true
    )
    public BigDecimal getBalance(UUID accountId) {
        return accountRepository.calculateBalance(accountId);
    }
    
    @Counted(
        value = "account.operations",
        description = "Number of account operations",
        recordFailuresOnly = false
    )
    public void createAccount(Account account) {
        accountRepository.save(account);
    }
}
```

### Prometheus Integration

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
        payment.processing.duration: true
      slo:
        payment.processing.duration: 100ms,500ms,1000ms
      percentiles:
        payment.processing.duration: 0.5,0.90,0.95,0.99
  endpoints:
    web:
      exposure:
        include: prometheus,health,info,metrics
```

---

## Structured Logging

### Logback Configuration with JSON

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="production">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <customFields>
                    {"service":"payment-service","version":"${APP_VERSION}","env":"production"}
                </customFields>
                <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                    <maxDepthPerThrowable>30</maxDepthPerThrowable>
                    <maxLength>8192</maxLength>
                    <shortenedClassNameLength>35</shortenedClassNameLength>
                    <rootCauseFirst>true</rootCauseFirst>
                </throwableConverter>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
        </root>
    </springProfile>
    
    <springProfile name="local,test">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] [%X{traceId:--}] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

```java
// Structured logging with MDC (Mapped Diagnostic Context)
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-ID"))
            .orElseGet(() -> UUID.randomUUID().toString());
        
        String userId = extractUserId(request);
        
        // MDC is automatically included in log output (works with Logback/Log4j2)
        MDC.put("correlationId", correlationId);
        MDC.put("userId", userId);
        MDC.put("requestUri", request.getRequestURI());
        MDC.put("httpMethod", request.getMethod());
        MDC.put("clientIp", getClientIp(request));
        
        response.setHeader("X-Correlation-ID", correlationId);
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
            Object handler, Exception ex) {
        MDC.clear();  // CRITICAL: Prevent MDC leakage in thread pools
    }
}

// Usage: JSON log entry automatically includes MDC values
// {"timestamp":"2024-01-15T10:30:45Z","level":"INFO","message":"Processing payment",
//  "correlationId":"abc-123","userId":"user456","service":"payment-service",
//  "traceId":"6bef47302b62148a","spanId":"2279ac96e84e3b5f"}
```

### Spring Boot 3.3+ Structured Logging

```yaml
# Spring Boot 3.3+: built-in structured logging support
logging:
  structured:
    format:
      console: ecs  # or logstash, gelf
  pattern:
    correlation: "[${spring.application.name:}, %X{traceId:-}, %X{spanId:-}]"
```

---

## Distributed Tracing with Micrometer Tracing

```java
// Spring Boot 3.x: Micrometer Tracing (replaces Spring Cloud Sleuth)
@Service
public class PaymentService {
    
    private final Tracer tracer;  // Micrometer Tracer (wraps Brave or OpenTelemetry)
    
    @NewSpan("payment-processing")
    public Payment processPayment(@SpanTag("payment.type") String type, PaymentRequest request) {
        
        // Create child spans for specific operations
        Span fraudCheckSpan = tracer.nextSpan().name("fraud-check").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(fraudCheckSpan)) {
            FraudAssessment assessment = fraudService.assess(request);
            fraudCheckSpan.tag("fraud.risk", assessment.getRisk().name());
        } finally {
            fraudCheckSpan.end();
        }
        
        // Baggage propagation (context that flows with the trace)
        BaggageField userId = BaggageField.create("userId");
        userId.updateValue(request.getUserId());
        
        return processInternal(request);
    }
}
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 0.1   # Sample 10% in production (adjust based on volume)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
  
# Dependencies:
# io.micrometer:micrometer-tracing-bridge-otel
# io.opentelemetry:opentelemetry-exporter-zipkin
```

---

## Spring Boot Actuator Production Configuration

```yaml
management:
  server:
    port: 9090                              # Separate management port
    base-path: /management                  # Hard to guess, separate from API

  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers
        # NEVER: beans, env, configprops, heapdump in production (security risk!)

  endpoint:
    health:
      show-details: when-authorized
      show-components: always
      probes:
        enabled: true                        # /health/liveness, /health/readiness
      group:
        liveness:
          include: livenessState,diskSpace
        readiness:
          include: readinessState,db,redis,kafka

    loggers:
      enabled: true                          # Dynamic log level changes

    info:
      enabled: true

  info:
    env:
      enabled: true
    git:
      mode: full                             # Include git commit info in /actuator/info
    build:
      enabled: true

info:
  application:
    name: payment-service
    version: "@project.version@"            # Maven placeholder
    environment: ${ENVIRONMENT:local}
```

```java
// Custom health indicator for critical external dependencies
@Component
public class PaymentGatewayHealthIndicator extends AbstractHealthIndicator {
    
    private final PaymentGatewayClient client;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        try {
            PingResult result = client.ping(Duration.ofSeconds(3));
            
            if (result.isHealthy()) {
                builder.up()
                    .withDetail("gateway.latency_ms", result.getLatencyMs())
                    .withDetail("gateway.version", result.getVersion())
                    .withDetail("gateway.region", result.getRegion());
            } else {
                builder.down()
                    .withDetail("reason", result.getMessage());
            }
        } catch (TimeoutException e) {
            builder.down()
                .withDetail("reason", "timeout")
                .withDetail("timeout_ms", 3000)
                .withException(e);
        }
    }
}

// Readiness indicator — can be explicitly set
@Component
public class StartupReadinessIndicator implements ApplicationListener<ApplicationReadyEvent> {
    
    private final ApplicationAvailability availability;
    private final CacheWarmingService cacheService;
    
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        try {
            cacheService.warmCriticalCaches();
            // Mark as ready only after caches are warm
            // Until this point, Kubernetes readiness probe returns 503
            log.info("Application ready. Caches warm, accepting traffic.");
        } catch (Exception e) {
            log.error("Cache warming failed — application not ready", e);
            // Stays in ReadinessState.REFUSING_TRAFFIC
        }
    }
}
```

---

## Interview Questions & Model Answers

### Q1: What is the difference between metrics, logs, and traces?

**Model Answer**: The three pillars of observability serve different purposes:

**Metrics**: Numerical time-series data aggregated over time (counts, gauges, timers). Excellent for dashboards, alerts, and SLA monitoring. Example: payment success rate over the last 1 hour, P99 latency trend. Storage-efficient because data is aggregated, not raw. Tools: Prometheus, Grafana, Micrometer.

**Logs**: Timestamped text records of individual events. Best for debugging specific incidents — "what happened at 14:32:45 for user X?" Not suitable for aggregation-heavy analysis (expensive storage for high-volume systems). Tools: Logback, ELK Stack, Splunk.

**Traces**: Correlated sequences of events (spans) representing a single request's journey through distributed systems. Essential for understanding latency in microservices. A trace ID ties together: gateway span → payment service span → database span → Kafka span. Tools: Zipkin, Jaeger, OpenTelemetry.

In practice: use metrics for alerting, traces for identifying *where* time is spent, and logs for debugging *what* happened at a specific point.

---

### Q2: How does Micrometer Tracing work in Spring Boot 3.x?

**Model Answer**: Spring Boot 3.x uses Micrometer Tracing (which replaced Spring Cloud Sleuth) as an abstraction over distributed tracing implementations (OpenTelemetry, Brave/Zipkin).

The integration works in three layers:

1. **Micrometer Observation API**: Business code creates "observations" — each observation can produce both metrics AND traces. `@NewSpan`, `@ContinueSpan` annotations create trace spans.

2. **Bridge layer**: `micrometer-tracing-bridge-otel` (for OpenTelemetry) or `micrometer-tracing-bridge-brave` (for Zipkin) connects Micrometer observations to the underlying tracing implementation.

3. **Instrumentation**: Spring auto-configures HTTP server/client instrumentation, Spring Data, and Kafka listeners to automatically create and propagate spans.

Trace context propagation uses standard headers (W3C Trace Context: `traceparent`, `tracestate`) which are extracted from incoming HTTP requests and injected into outgoing calls, enabling cross-service trace correlation.

The `traceId` and `spanId` are automatically added to MDC (Mapped Diagnostic Context), so they appear in every log entry without additional code — enabling log-to-trace correlation.

---

## Key Takeaways

- **Three pillars**: Metrics (numbers over time), Logs (events), Traces (request journeys)
- **MeterRegistry for metrics**: Counter (ever-incr), Gauge (current), Timer (duration), Summary (distribution)
- **Dimensional/tagged metrics**: Always tag with meaningful dimensions (gateway, currency, region)
- **Structured JSON logging**: Use Logstash encoder; always clear MDC in request lifecycle end
- **MDC for log correlation**: traceId/spanId auto-added by Micrometer Tracing
- **Micrometer Tracing replaces Spring Cloud Sleuth** in Spring Boot 3.x
- **Actuator security**: Never expose `env`, `heapdump`, `beans` in production without auth
- **Custom health indicators**: Essential for K8s readiness/liveness probe reliability

---

## Further Reading

- [Micrometer Documentation](https://micrometer.io/docs)
- [Spring Boot Observability Reference](https://docs.spring.io/spring-boot/reference/actuator/observability.html)
- [Spring Boot Actuator Reference](https://docs.spring.io/spring-boot/reference/actuator/index.html)
- [OpenTelemetry Spring Boot Integration](https://opentelemetry.io/docs/languages/java/automatic/spring-boot/)
