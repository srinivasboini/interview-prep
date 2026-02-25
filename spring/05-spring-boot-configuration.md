# Spring Boot Configuration Management

## Overview

Spring Boot's configuration system is one of the most powerful features of the framework. Configuration can come from 17+ property sources with a well-defined precedence order — environment variables beating application.yml, which beats @PropertySource files. Understanding this hierarchy is critical for debugging configuration issues in Kubernetes environments where multiple configuration sources interact.

In enterprise banking, configuration management is a security and compliance concern: secrets must never be in source code, feature flags must be toggleable without deployment, and environment-specific configurations must be audited.

---

## Property Source Hierarchy (17 Sources, Priority Order)

```
Highest Priority (wins)
  1.  Devtools global settings (~/.spring-boot-devtools.properties)
  2.  @TestPropertySource annotations
  3.  @SpringBootTest properties
  4.  Command line arguments (--server.port=9090)
  5.  Inline test properties (e.g., properties attribute in @SpringBootTest)
  6.  SPRING_APPLICATION_JSON (env var or system property)
  7.  ServletConfig init parameters
  8.  ServletContext init parameters
  9.  java:comp/env JNDI attributes
  10. System properties (System.getProperties())
  11. OS environment variables (SPRING_DATASOURCE_URL)
  12. Random value properties (${random.int})
  13. Profile-specific properties outside JAR (config/application-{profile}.yml)
  14. Profile-specific properties inside JAR (application-{profile}.yml)
  15. Application properties outside JAR (config/application.yml)
  16. Application properties inside JAR (application.yml)
  17. @PropertySource (on @Configuration classes)
  18. Default properties (SpringApplication.setDefaultProperties)
Lowest Priority
```

---

## @ConfigurationProperties (Type-Safe Configuration)

```java
// application.yml
payment:
  gateway:
    url: https://gateway.bank.com
    timeout-seconds: 30
    max-retries: 3
    api-key: ${PAYMENT_GATEWAY_API_KEY}  # From environment variable
    retry-on-status:
      - 502
      - 503
      - 504
  limits:
    daily-limit: 100000.00
    single-transaction-limit: 50000.00
    currencies:
      GBP: 100000
      USD: 120000
      EUR: 115000

# @ConfigurationProperties — type-safe, validated, IDE-autocomplete
@Configuration
@ConfigurationProperties(prefix = "payment")
@Validated  # Enable JSR-303 validation on config
public class PaymentConfiguration {
    
    @Valid
    @NotNull
    private Gateway gateway = new Gateway();
    
    @Valid
    @NotNull
    private Limits limits = new Limits();
    
    // ─── Nested configuration class ─────────────────────────────
    @Data
    public static class Gateway {
        @NotBlank
        private String url;
        
        @Min(1) @Max(300)
        private int timeoutSeconds = 30;
        
        @Min(0) @Max(10)
        private int maxRetries = 3;
        
        @NotBlank
        private String apiKey;
        
        private List<Integer> retryOnStatus = List.of(502, 503, 504);
    }
    
    @Data
    public static class Limits {
        @NotNull
        @DecimalMin("0.01")
        private BigDecimal dailyLimit = new BigDecimal("100000.00");
        
        @NotNull
        @DecimalMin("0.01")
        private BigDecimal singleTransactionLimit = new BigDecimal("50000.00");
        
        private Map<String, BigDecimal> currencies = new HashMap<>();
    }
}
```

### Using @ConfigurationProperties in Spring Boot 2.2+

```java
// Constructor binding (immutable, Java record-friendly)
@Configuration
@ConfigurationProperties(prefix = "payment.gateway")
public record PaymentGatewayProps(
    @NotBlank String url,
    @Min(1) int timeoutSeconds,
    @NotBlank String apiKey
) {
    // Spring automatically detects constructor-bound configuration
}

// Registering with @EnableConfigurationProperties
@SpringBootApplication
@EnableConfigurationProperties(PaymentGatewayProps.class)
public class Application { }

// Or using @ConfigurationPropertiesScan
@SpringBootApplication
@ConfigurationPropertiesScan  # Scans for all @ConfigurationProperties
public class Application { }
```

---

## Profile Management

```yaml
# application.yml — base configuration (all environments)
spring:
  application:
    name: payment-service
  jpa:
    open-in-view: false  # Always disable — performance anti-pattern

---
# application-local.yml — developer machine
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:postgresql://localhost:5432/paymentdb
    username: dev_user
    password: dev_password
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  kafka:
    bootstrap-servers: localhost:9092
logging:
  level:
    com.bank: DEBUG
    org.hibernate.SQL: DEBUG

---
# application-test.yml — CI/CD pipelines, integration tests
spring:
  config:
    activate:
      on-profile: test
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9094}

---
# application-production.yml — production Kubernetes
spring:
  config:
    activate:
      on-profile: production
  datasource:
    url: ${DATABASE_URL}
    hikari:
      maximum-pool-size: 20
  jpa:
    show-sql: false
  flyway:
    enabled: true
logging:
  level:
    root: WARN
    com.bank: INFO
```

### Profile Activation

```bash
# 1. Command line argument
java -jar app.jar --spring.profiles.active=production

# 2. Environment variable (Kubernetes/Docker)
SPRING_PROFILES_ACTIVE=production

# 3. In code
@Bean
@Profile("!production")   # Active when NOT production
public DataSource mockDataSource() { ... }

@Bean
@Profile({"local", "test"})  # Active in local OR test
public FakeEmailService fakeEmailService() { ... }

@Bean
@Profile("production")
public RealEmailService realEmailService() { ... }
```

### Kubernetes ConfigMap Integration

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: payment-service
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            # Secrets from Kubernetes Secrets (not ConfigMap!)
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: database-url
            - name: PAYMENT_GATEWAY_API_KEY
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: gateway-api-key
          # Volume-mount ConfigMap as config files
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: payment-service-config
```

```yaml
# Override specific properties via volume-mounted file
# /app/config/application.yml (overrides embedded application.yml)
payment:
  limits:
    daily-limit: 250000.00

spring:
  kafka:
    bootstrap-servers: kafka-prod.internal:9092
```

---

## @Value vs @ConfigurationProperties

| Aspect | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| **Type safety** | No (string injection) | Yes (typed binding) |
| **Validation** | Manual | JSR-303 via `@Validated` |
| **Relaxed binding** | Partial | Full (camelCase, kebab-case, SCREAMING_SNAKE_CASE all work) |
| **IDE support** | Limited | Full autocomplete with metadata |
| **Refactoring** | Risky (string keys) | Safe (Java field names) |
| **Complex types** | awkward | Natural (Maps, Lists, nested objects) |
| **Default values** | `${prop:default}` | Field initializer |
| **Use when** | 1-2 simple values | Feature configuration classes |

```java
// @Value — appropriate for simple, standalone values
@Component
public class HealthCheckService {
    
    @Value("${app.health.check.interval:30}")
    private int checkIntervalSeconds;
    
    @Value("${spring.application.name}")
    private String applicationName;
    
    @Value("${ENVIRONMENT:local}")  // OS env var with default
    private String environment;
}
```

---

## Spring Cloud Config Client

```yaml
# application.yml
spring:
  config:
    import:
      - "configserver:http://config-server.internal:8888"
      - "vault://secret/payment-service"   # HashiCorp Vault import
  cloud:
    config:
      fail-fast: true          # Fail startup if config server unreachable
      retry:
        initial-interval: 1000
        max-attempts: 6
      label: main              # Git branch/tag to use
      profile: ${spring.profiles.active}
```

### @RefreshScope for Dynamic Updates

```java
// Bean is re-created when /actuator/refresh or Spring Cloud Bus refresh event fires
@RefreshScope
@Component
public class DynamicFeatureFlags {
    
    @Value("${feature.new-payment-ui.enabled:false}")
    private boolean newPaymentUiEnabled;
    
    @Value("${feature.fraud-ml-model.enabled:false}")
    private boolean fraudMlEnabled;
    
    @Value("${limits.daily-payment-limit:50000}")
    private BigDecimal dailyLimit;
}
```

---

## Interview Questions & Model Answers

### Q1: What is the order of property source resolution in Spring Boot?

**Model Answer**: Spring Boot consults 17+ property sources in a specific priority order. The key hierarchy to remember:

1. **Highest**: Command-line arguments (`--server.port=9090`)
2. **OS environment variables** (`SPRING_DATASOURCE_URL`) — critical for container deployments
3. **JVM system properties** (`-Dserver.port=9090`)
4. **External config files** (`config/application.yml` next to JAR)
5. **Embedded config files** (`application-{profile}.yml` inside JAR, then `application.yml`)
6. **@PropertySource** (lowest among application-managed sources)

In Kubernetes: environment variables from `envFrom` (ConfigMap/Secret) override the embedded `application.yml`. This is by design — you deploy one JAR and configure it differently per environment via environment variables, without rebuilding.

**Banking security implication**: Never embed database passwords or API keys in `application.yml`. They should come from environment variables backed by Kubernetes Secrets or Vault — sources higher in the hierarchy than the code artifact.

---

### Q2: Why prefer @ConfigurationProperties over @Value?

**Model Answer**: `@Value` is acceptable for 1-2 simple properties scattered across different beans. But for related configuration (all properties for a payment gateway, all feature flags), `@ConfigurationProperties` is superior:

1. **Type safety**: Missing or misspelled properties fail at startup (with `@Validated`), not at runtime.
2. **Relaxed binding**: Spring accepts `PAYMENT_GATEWAY_URL`, `payment.gateway.url`, or `payment-gateway-url` as equivalent — essential for Kubernetes where env var names use `_` not `.`.
3. **IDE support**: With the metadata annotation processor, IDEs provide autocomplete for YAML keys matching `@ConfigurationProperties` classes.
4. **Refactoring**: Renaming a field is a compile-time change; renaming `@Value("${some.key}")` string is invisible to the compiler.
5. **Testability**: You can instantiate `@ConfigurationProperties` classes directly in tests without Spring context.

```java
// Test with @ConfigurationProperties: no Spring needed
var config = new PaymentGatewayProps("https://test.gateway.com", 5, "test-key");
var service = new PaymentService(config);
```

---

## Key Takeaways

- **Environment variables beat application.yml** — use for secrets and environment-specific config in K8s
- **@ConfigurationProperties for groups of config** — type-safe, validated, IDE-supported
- **@Value for isolated single values** — avoid for complex or secret configuration
- **spring.profiles.active defines the active profile** — set via env var in Kubernetes
- **@Profile("!production") for dev-only beans** — conditional bean registration based on profile
- **@RefreshScope for dynamic updates** — bean re-created when config changes, requires Config Server

---

## Further Reading

- [Spring Boot Externalized Configuration Reference](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Spring Boot Profiles Documentation](https://docs.spring.io/spring-boot/reference/features/profiles.html)
- [Baeldung — @ConfigurationProperties Guide](https://www.baeldung.com/configuration-properties-in-spring-boot)
