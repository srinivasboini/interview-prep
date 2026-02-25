# Spring Boot Version Features and Migration Guide

## Overview

Understanding Spring Boot's version evolution is critical for Staff/Principal Engineers — both for explaining the rationale behind current architectural choices and for planning migrations. The Jakarta EE 9+ namespace change in Spring Boot 3.0 was a major breaking change; the shift from `WebSecurityConfigurerAdapter` to `SecurityFilterChain` beans reflects Spring's evolution toward more testable, composable configuration. Virtual threads in 3.2 with Java 21 are arguably the most significant performance enhancement in Java's history.

This guide covers what changed in each version, why, and migration strategies — the "why" being what distinguishes principal-level answers from senior-level answers.

---

## Spring Boot 3.x: The Major Evolution

### 3.0 — The Paradigm Shift (2022-11-24)

**1. Jakarta EE 9+ Migration — javax → jakarta namespace**

This is the most impactful change:

```java
// Spring Boot 2.x (javax.*)
import javax.persistence.Entity;
import javax.persistence.Column;
import javax.servlet.http.HttpServletRequest;
import javax.validation.constraints.NotNull;
import javax.transaction.Transactional;

// Spring Boot 3.x (jakarta.*)
import jakarta.persistence.Entity;        // ← renamed
import jakarta.persistence.Column;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.constraints.NotNull;
import jakarta.transaction.Transactional;
```

**Why**: Oracle transferred Java EE to Eclipse Foundation in 2017. Eclipse Foundation couldn't use the `javax` namespace (Oracle's trademark). The new `jakarta` namespace is required for all future Jakarta EE features.

**Migration**: OpenRewrite provides automated migration:
```xml
<plugin>
    <groupId>org.openrewrite.maven</groupId>
    <artifactId>rewrite-maven-plugin</artifactId>
    <version>5.x.x</version>
    <configuration>
        <activeRecipes>
            <recipe>org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0</recipe>
        </activeRecipes>
    </configuration>
</plugin>
```

**2. Java 17 Baseline**

Spring Boot 3.x requires Java 17 minimum. This unlocks:
- Records (`record PaymentDTO(UUID id, BigDecimal amount) {}`)  
- Sealed classes
- Pattern matching for `instanceof`
- Text blocks
- Switch expressions with arrows

**3. Spring Framework 6.0**

- Ahead-of-Time (AOT) processing foundation
- Default path matching uses `PathPatternParser` (more efficient than `AntPathMatcher`)
- `ProblemDetail` for RFC 7807 error responses

**4. Native Image Support (GraalVM)**

```xml
<!-- Build a native executable -->
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

```bash
mvn -Pnative native:compile
# Produces: target/payment-service (native executable)
# Startup: ~50ms vs 3-5s for JVM
# Memory: ~50MB vs 300MB for JVM
```

**5. Micrometer Tracing replaces Spring Cloud Sleuth**

```xml
<!-- OLD (Spring Boot 2.x + Sleuth) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<!-- NEW (Spring Boot 3.x) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

**6. Spring Security 6.0 — WebSecurityConfigurerAdapter REMOVED**

```java
// ❌ Spring Boot 2.x (DEPRECATED in 2.7, REMOVED in 3.0)
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/api/**").authenticated()
            .and()
            .csrf().disable();
    }
}

// ✅ Spring Boot 3.x (SecurityFilterChain bean)
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/**").authenticated()
            )
            .build();
    }
}
```

---

### 3.1 — Testcontainers and Docker Compose (2023-05-18)

**1. Testcontainers at Development Time**

```java
// spring-boot-testcontainers dependency
@Bean
@ServiceConnection  // Auto-configures Spring properties from container
public PostgreSQLContainer<?> postgresContainer() {
    return new PostgreSQLContainer<>("postgres:15");
}

// @ServiceConnection: Automatically sets spring.datasource.url, username, password
// No need for @DynamicPropertySource!
```

**2. Docker Compose Support**

```yaml
# compose.yaml in project root
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: paymentdb
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

```bash
# Spring Boot automatically starts Docker Compose on application startup!
spring.docker.compose.enabled: true
spring.docker.compose.lifecycle-management: start_and_stop
```

**3. SSL Bundle Support**

```yaml
spring:
  ssl:
    bundle:
      jks:
        server:
          key:
            alias: server
          keystore:
            location: classpath:server.jks
            password: ${KEYSTORE_PASSWORD}
            type: JKS
server:
  ssl:
    bundle: server
```

---

### 3.2 — Virtual Threads and RestClient (2023-11-23)

**1. Virtual Threads (Java 21, Project Loom)**

```yaml
# Enable virtual threads for ALL Spring-managed threads:
spring:
  threads:
    virtual:
      enabled: true
```

This setting:
- Embedded Tomcat uses one virtual thread per request (vs fixed platform thread pool)
- `@Async` methods run on virtual threads  
- Scheduled tasks run on virtual threads
- SimpleAsyncTaskExecutor backed by virtual threads

**Performance impact**: Web applications with I/O-bound workloads can handle significantly more concurrent requests without the complexity of reactive programming.

**2. RestClient — The RestTemplate Replacement**

```java
// LEGACY: RestTemplate (Spring 3.0, synchronous, still supported but not recommended)
RestTemplate template = new RestTemplate();
ResponseEntity<Payment> response = template.getForEntity("/payments/{id}", Payment.class, id);

// LEGACY REACTIVE: WebClient (Spring 5.0, reactive, works in both MVC and WebFlux)
WebClient client = WebClient.create("http://payment-service");
Payment payment = client.get().uri("/payments/{id}", id).retrieve().bodyToMono(Payment.class).block();

// NEW: RestClient (Spring 6.1 / Boot 3.2, synchronous, modern API)
RestClient restClient = RestClient.builder()
    .baseUrl("http://payment-service")
    .defaultHeader("Accept", "application/json")
    .build();

Payment payment = restClient.get()
    .uri("/payments/{id}", id)
    .retrieve()
    .body(Payment.class);

// RestClient with error handling
Payment payment = restClient.get()
    .uri("/payments/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
        throw new PaymentNotFoundException(id);
    })
    .onStatus(HttpStatusCode::is5xxServerError, (request, response) -> {
        throw new PaymentServiceException("Upstream error");
    })
    .body(Payment.class);
```

**3. JdbcClient — Fluent JDBC API**

```java
// OLD: JdbcTemplate (verbose)
List<Payment> payments = jdbcTemplate.query(
    "SELECT * FROM payments WHERE account_id = ? AND status = ?",
    new Object[]{accountId, status},
    paymentRowMapper
);

// NEW: JdbcClient (Spring 6.1 / Boot 3.2, fluent)
List<Payment> payments = jdbcClient.sql(
    "SELECT * FROM payments WHERE account_id = :accountId AND status = :status")
    .param("accountId", accountId)
    .param("status", status.name())
    .query(Payment.class)
    .list();

// Single result:
Optional<Payment> payment = jdbcClient.sql("SELECT * FROM payments WHERE id = :id")
    .param("id", paymentId)
    .query(Payment.class)
    .optional();

// Update:
int updated = jdbcClient.sql("UPDATE payments SET status = :status WHERE id = :id")
    .param("status", newStatus.name())
    .param("id", paymentId)
    .update();
```

---

### 3.3 — CDS and Structured Logging (2024-05-23)

**1. Class Data Sharing (CDS)**

```bash
# Training run: profile class loading
java -Dspring.context.exit=onRefresh -jar payment-service.jar
java -XX:DumpLoadedClassList=classes.lst -jar payment-service.jar

# Create shared archive
java -Xshare:dump -XX:SharedClassListFile=classes.lst \
     -XX:SharedArchiveFile=app.jsa -jar payment-service.jar

# Production run with CDS archive (30-50% faster startup)
java -XX:SharedArchiveFile=app.jsa -jar payment-service.jar
```

**2. Built-in Structured Logging**

```yaml
logging:
  structured:
    format:
      console: ecs    # Elastic Common Schema (ECS)
      # Options: logstash, ecs, gelf
```

---

### 3.4 — Latest Enhancements (2024-11-21)

**1. MockMvcTester (New Testing API)**

```java
// OLD MockMvc (Spring Boot 3.3 and earlier)
mockMvc.perform(get("/api/payments/{id}", id))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.status").value("COMPLETED"));

// NEW MockMvcTester (Boot 3.4, AssertJ-based, more expressive)
@Autowired
private MockMvcTester mockMvcTester;

void testGetPayment() {
    assertThat(mockMvcTester.get().uri("/api/payments/{id}", id))
        .hasStatusOk()
        .bodyJson()
        .extractingPath("$.status").isEqualTo("COMPLETED");
}
```

**2. @Fallback Beans**

```java
// Simpler than @ConditionalOnMissingBean for optional fallbacks
@Component
@Fallback  // Used when no other PaymentGateway bean is registered
public class DefaultPaymentGateway implements PaymentGateway {
    // Fallback implementation (e.g., test stub, in-memory mock)
}

@Component
@ConditionalOnProperty("payment.gateway.stripe.enabled")
public class StripePaymentGateway implements PaymentGateway {
    // Real implementation — if present, @Fallback bean is NOT used
}
```

---

## Spring Boot 2.x → 3.x Migration Guide

### Step-by-Step Migration Checklist

```bash
# Step 1: Ensure Java 17+ compatibility
mvn versions:set -DnewVersion=3.2.0
mvn versions:update-parent
# Check: Java 17 minimum; record classes work; sealed classes optional

# Step 2: Run OpenRewrite for automated migrations
mvn rewrite:run -Drewrite.activeRecipes=org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_0

# Step 3: Apply Spring Security migration
mvn rewrite:run -Drewrite.activeRecipes=org.openrewrite.java.spring.security6.UpgradeSpringSecurity_6_0
```

### Migration Risk Areas

| Area | 2.x | 3.x | Risk |
|---|---|---|---|
| Namespace | `javax.*` | `jakarta.*` | 🔴 HIGH |
| Web Security | `WebSecurityConfigurerAdapter` | `SecurityFilterChain` bean | 🔴 HIGH |
| URL matching | `antMatchers()` | `requestMatchers()` | 🟡 MEDIUM |
| Tracing | Spring Cloud Sleuth | Micrometer Tracing | 🟡 MEDIUM |
| Auto-config registry | `spring.factories` | `AutoConfiguration.imports` | 🟡 MEDIUM |
| HTTP clients | RestTemplate | RestClient, HTTP Exchange | 🟢 LOW (not breaking) |
| Data properties | `spring.data.jpa.*` | Mostly same, some renamed | 🟢 LOW |

### Key Property Changes

```yaml
# Spring Boot 2.x
spring:
  mvc:
    pathmatch:
      use-suffix-pattern: true         # REMOVED in 3.x
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: xxx
  
# Spring Boot 3.x
spring:
  threads:
    virtual:
      enabled: true                    # NEW
  docker:
    compose:
      enabled: true                    # NEW
  autoconfigure:
    exclude:                           # Still works, different property names for some
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## Spring Boot 2.x Key Feature Timeline

```
Spring Boot 2.0 (March 2018):    Spring Framework 5, Reactive/WebFlux, Actuator redesign
Spring Boot 2.1 (October 2018):  Java 11 support, lazy initialization, Bean Overriding default
Spring Boot 2.2 (October 2019):  Immutable @ConfigurationProperties, Java 13 support
Spring Boot 2.3 (May 2020):      Graceful shutdown, Liveness/Readiness probes, Docker layer builds
Spring Boot 2.4 (November 2020): Config file processing overhaul, Volume-mounted configs
Spring Boot 2.5 (May 2021):      Java 16 records, enhanced Docker image building
Spring Boot 2.6 (November 2021): PathPatternParser default, Spring MVC circular path warning
Spring Boot 2.7 (May 2022):      Auto-configuration annotation changes (begins 3.x migration)
```

---

## Interview Questions & Model Answers

### Q1: Why was the javax → jakarta namespace migration necessary?

**Model Answer**: Oracle donated Java EE to the Eclipse Foundation in 2017, creating the Jakarta EE project. However, Oracle retained the trademark on the `javax` namespace. Eclipse Foundation could continue using existing `javax` APIs unchanged, but could not add new features or APIs under the `javax` namespace.

To enable future evolution of the platform (Jakarta EE 10, 11...), all new features must use the `jakarta.*` namespace. Jakarta EE 9 was effectively just a namespace rename — no new features, just `javax` → `jakarta` to establish the new baseline.

**Impact for Spring**: Spring Framework 6 (and Spring Boot 3) adopted Jakarta EE 9+ as the minimum requirement. All Spring Framework APIs that depended on `javax.servlet`, `javax.persistence`, `javax.validation` etc. were updated to use `jakarta.*`.

**For banking migration teams**: This is a physical code change — every import must be updated. OpenRewrite automates ~90% of this. The remaining 10% are typically in generated code, configuration files, or custom code that directly references `javax` classes.

---

### Q2: What are the key advantages of RestClient over RestTemplate?

**Model Answer**: RestClient (Spring 6.1 / Boot 3.2) addresses RestTemplate's long-standing limitations:

1. **Modern fluent API**: Builder-style API with method chaining — clearer and less error-prone than RestTemplate's many overloaded methods.

2. **Proper error handling**: `onStatus()` lets you handle specific HTTP status codes declaratively. RestTemplate's `ResponseErrorHandler` is global and complex to override correctly.

3. **Type-safe**: Generic types work correctly with parametrized types. RestTemplate required `ParameterizedTypeReference` for `List<Payment>` — ugly and error-prone.

4. **HTTP interface support**: RestClient underpins `@HttpExchange` declarative HTTP clients (similar to Feign), which are generated proxies from interfaces.

5. **Virtual thread friendly**: Unlike WebClient (reactor), RestClient is synchronous — perfect for blocking workloads on virtual threads without reactive complexity.

RestTemplate remains fully supported but is feature-frozen. For new code: RestClient for simple synchronous clients, WebClient if you genuinely need reactive streaming.

---

## Key Takeaways

- **javax → jakarta is THE Spring Boot 3.0 migration** — physical code change, OpenRewrite helps
- **Java 17 required for Boot 3.x** — unlocks records, sealed classes, text blocks
- **`WebSecurityConfigurerAdapter` REMOVED** — use `SecurityFilterChain @Bean` pattern
- **Virtual threads in 3.2** with `spring.threads.virtual.enabled=true` — massive scalability improvement
- **RestClient replaces RestTemplate** for synchronous HTTP clients (Boot 3.2+)
- **JdbcClient for fluent JDBC** (Boot 3.2+) — simpler than JdbcTemplate, no ORM overhead
- **Micrometer Tracing replaces Spring Cloud Sleuth** — different dependency, similar functionality
- **OpenRewrite automates most migrations** — run recipes for javax→jakarta, Security 5→6, etc.

---

## Further Reading

- [Spring Boot 3.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Release-Notes)
- [Spring Boot 3.2 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Release-Notes)
- [OpenRewrite Spring Boot Migration](https://docs.openrewrite.org/recipes/java/spring/boot3)
- [Spring Boot Migration Guide](https://docs.spring.io/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide.html)
- [Jakarta EE 10 Overview](https://jakarta.ee/release/10/)
