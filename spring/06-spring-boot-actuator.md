# Spring Boot Actuator and Production Readiness

## Overview

Spring Boot Actuator exposes production-ready monitoring endpoints that provide the operational visibility required to run services confidently in production. Kubernetes probes rely on Actuator's `/health/liveness` and `/health/readiness` endpoints. APM tools scrape `/actuator/prometheus` for metrics. Operations teams dynamically change log levels via `/actuator/loggers` without deployments.

Beyond out-of-the-box endpoints, Actuator is extensible — custom health indicators express your service's health in terms of domain-specific dependencies (payment gateway availability, fraud service status), and custom metrics expose business KPIs alongside infrastructure metrics.

---

## Health Indicators

### Built-in Health Indicators

Spring Boot automatically configures health indicators for detected dependencies:

| Indicator | Condition | Checks |
|---|---|---|
| `DataSourceHealthIndicator` | DataSource in context | Executes `SELECT 1` |
| `RedisHealthIndicator` | Redis in context | `PING` command |
| `KafkaHealthIndicator` | Kafka in context | Gets admin client metadata |
| `DiskSpaceHealthIndicator` | Always | Available disk space |
| `LivenessStateHealthIndicator` | Always | JVM liveness state |
| `ReadinessStateHealthIndicator` | Always | Readiness state |

### Custom Health Indicators

```java
// Domain-specific health indicator
@Component
public class PaymentGatewayHealthIndicator extends AbstractHealthIndicator {
    
    private final PaymentGatewayClient client;
    private final PaymentGatewayProperties props;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        try {
            PingResponse ping = client.ping(Duration.ofSeconds(3));
            
            builder.up()
                .withDetail("gateway.url", props.getUrl())
                .withDetail("gateway.latency_ms", ping.getLatencyMs())
                .withDetail("gateway.version", ping.getVersion())
                .withDetail("gateway.region", ping.getRegion());
                
        } catch (SocketTimeoutException e) {
            builder.down()
                .withDetail("error", "Connection timeout after 3s")
                .withDetail("gateway.url", props.getUrl())
                .withException(e);
        } catch (Exception e) {
            builder.unknown()  // Don't mark DOWN for transient issues
                .withDetail("error", e.getMessage());
        }
    }
}

@Component
public class FraudServiceHealthIndicator extends AbstractHealthIndicator {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("fraud-service");
        CircuitBreaker.State state = cb.getState();
        
        if (state == CircuitBreaker.State.CLOSED) {
            builder.up()
                .withDetail("circuit.state", state)
                .withDetail("failure.rate", cb.getMetrics().getFailureRate() + "%")
                .withDetail("slow.call.rate", cb.getMetrics().getSlowCallRate() + "%");
        } else if (state == CircuitBreaker.State.HALF_OPEN) {
            builder.unknown()  // Recovering state
                .withDetail("circuit.state", state)
                .withDetail("permitted.test.calls", cb.getMetrics().getNumberOfBufferedCalls());
        } else {  // OPEN
            builder.down()
                .withDetail("circuit.state", state)
                .withDetail("failure.rate", cb.getMetrics().getFailureRate() + "%");
        }
    }
}
```

### Kubernetes Probe Configuration

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true         # Enables /health/liveness and /health/readiness
      group:
        liveness:
          include: livenessState,diskSpace
          # Liveness: Is the JVM alive? (Kubernetes restarts if this fails)
        readiness:
          include: readinessState,db,redis,kafka,paymentGateway
          # Readiness: Can the service handle requests? (Traffic is redirected if this fails)
```

```yaml
# Kubernetes deployment: using Actuator probes
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: payment-service
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 9090      # Management port (separate from API port)
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 9090
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3
```

### Graceful Shutdown Integration

```yaml
server:
  shutdown: graceful              # Drain in-flight requests before shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # Max time to wait for active requests to complete
```

---

## Custom Endpoints

```java
// Create a custom Actuator endpoint
@Component
@Endpoint(id = "payment-status")       # Results in /actuator/payment-status
public class PaymentStatusEndpoint {
    
    private final PaymentRepository paymentRepository;
    private final CircuitBreakerRegistry cbRegistry;
    
    @ReadOperation                      # HTTP GET
    public Map<String, Object> paymentSystemStatus() {
        return Map.of(
            "pending_payments", paymentRepository.countByStatus(PaymentStatus.PENDING),
            "circuit_breaker_state", cbRegistry.circuitBreaker("payment-gateway").getState(),
            "last_successful_payment", paymentRepository.findLastSuccessful().map(Payment::getId).orElse(null)
        );
    }
    
    @WriteOperation                      # HTTP POST
    public Map<String, String> resetMetrics() {
        // Reset internal metrics
        metricsService.resetAll();
        return Map.of("result", "metrics reset successfully");
    }
    
    @DeleteOperation                      # HTTP DELETE
    public Map<String, String> clearPendingPayments(@Selector String status) {
        int count = paymentRepository.deleteByStatus(PaymentStatus.valueOf(status));
        return Map.of("deleted", String.valueOf(count));
    }
}
```

---

## Actuator Security

```java
@Bean
public SecurityFilterChain managementSecurityFilterChain(HttpSecurity http) throws Exception {
    return http
        .requestMatcher(EndpointRequest.toAnyEndpoint())  // Only matches management endpoints
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(EndpointRequest.to(HealthEndpoint.class, InfoEndpoint.class))
                .permitAll()                               // Health & info: public
            .requestMatchers(EndpointRequest.to(PrometheusEndpoint.class))
                .hasIpAddress("10.0.0.0/8")              // Prometheus: internal network only
            .anyRequest()
                .hasRole("ADMIN")                          // Everything else: admin only
        )
        .httpBasic(Customizer.withDefaults())
        .build();
}
```

---

## Interview Questions & Model Answers

### Q1: How does Spring Boot's liveness vs readiness probe differ?

**Model Answer**: These map directly to Kubernetes health probe semantics:

**Liveness** (`/health/liveness`): Is the application process alive and not in a broken state? A failed liveness probe causes Kubernetes to **restart the pod**. This should be lightweight — checking fundamental JVM health, not external dependencies. It should only fail if the application is in an unrecoverable state (deadlock, OOM, fatal error).

**Readiness** (`/health/readiness`): Is the application ready to handle traffic? A failed readiness probe causes Kubernetes to **remove the pod from the Service load balancer** (stop sending new requests) WITHOUT restarting it. This is appropriate during startup (before caches are warm), maintenance windows, or when a critical dependency (database, payment gateway) is temporarily unavailable.

**Wrong configuration**: Including database health in liveness causes the pod to restart every time the DB has a blip — cascading restarts amplify a database issue into a complete service outage. Database health belongs in readiness.

---

## Key Takeaways

- **Liveness: JVM alive? → pod restarts if fails**
- **Readiness: traffic ready? → removed from load balancer if fails**  
- **Never put external dependencies in liveness** — causes restart cascades
- **Custom health indicators**: extend `AbstractHealthIndicator` for domain health
- **Separate management port** (9090) from API port (8080) — don't expose to external traffic
- **Graceful shutdown**: `server.shutdown=graceful` + lifecycle timeout for zero-downtime deploys

---

## Further Reading

- [Spring Boot Actuator Reference](https://docs.spring.io/spring-boot/reference/actuator/index.html)
- [Kubernetes Health Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Baeldung — Spring Boot Actuator](https://www.baeldung.com/spring-boot-actuators)
