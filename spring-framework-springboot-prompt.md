# Prompt: Create Comprehensive Spring Framework & Spring Boot Interview Preparation Guide

## Objective
Create an in-depth, interview-ready preparation guide on Spring Framework, Spring Boot, and the broader Spring ecosystem that covers everything a Senior Java Developer (13+ years experience) needs to know for technical interviews at Staff/Principal Engineer level in enterprise banking/financial services.

## Target Audience Context
- **Experience Level**: Senior developer preparing for Staff/Principal Engineer roles
- **Current Knowledge**: 13+ years of enterprise Java experience in banking
- **Interview Focus**: Technical depth interviews at companies like JP Morgan, UBS, Goldman Sachs, Barclays, HSBC, etc.
- **Learning Style**: Prefers understanding "why" over "what", visual explanations, real-world enterprise context
- **Tech Stack Context**: Spring Boot microservices deployed on Azure/AWS, REST APIs, Kafka, PostgreSQL, Hibernate

## Content Requirements

### 1. Core Topics to Cover (MUST BE COMPREHENSIVE)

#### A. Spring Framework Core (DEEP DIVE)

**Inversion of Control (IoC) and Dependency Injection (DI)**:
- **IoC Container**:
  - What is IoC and why it matters
  - BeanFactory vs ApplicationContext
  - ApplicationContext hierarchy and types (AnnotationConfigApplicationContext, ClassPathXmlApplicationContext, GenericWebApplicationContext)
  - Container lifecycle: startup, refresh, shutdown
  - ApplicationContext events and listeners
  - Eager vs lazy initialization of beans

- **Dependency Injection Mechanisms**:
  - Constructor injection (preferred approach and why)
  - Setter injection
  - Field injection (why it's discouraged)
  - Method injection (lookup method injection)
  - @Autowired, @Inject (JSR-330), @Resource (JSR-250) differences
  - Injection of collections (List, Set, Map)
  - @Qualifier and custom qualifiers
  - @Primary for default bean selection
  - Optional dependencies and @Nullable

- **Bean Definition and Registration**:
  - @Component, @Service, @Repository, @Controller stereotypes
  - @Bean methods in @Configuration classes
  - @ComponentScan and its filters (includeFilters, excludeFilters)
  - Programmatic bean registration (BeanDefinitionRegistryPostProcessor)
  - FactoryBean interface
  - Conditional bean registration (@Conditional, @ConditionalOnProperty, @ConditionalOnClass, etc.)

**Bean Lifecycle (CRITICAL)**:
- **Complete Bean Lifecycle Flow**:
  - Bean instantiation
  - Populate properties (dependency injection)
  - BeanNameAware, BeanFactoryAware, ApplicationContextAware callbacks
  - BeanPostProcessor.postProcessBeforeInitialization
  - @PostConstruct (JSR-250)
  - InitializingBean.afterPropertiesSet
  - Custom init-method
  - BeanPostProcessor.postProcessAfterInitialization
  - Bean is ready for use
  - @PreDestroy (JSR-250)
  - DisposableBean.destroy
  - Custom destroy-method

- **Bean Scopes**:
  - singleton (default): One instance per IoC container
  - prototype: New instance every time requested
  - request: One instance per HTTP request (web only)
  - session: One instance per HTTP session (web only)
  - application: One instance per ServletContext (web only)
  - websocket: One instance per WebSocket session
  - Custom scopes: Creating and registering custom scopes
  - Scoped proxy: @Scope(proxyMode) for injecting narrower-scoped beans into wider-scoped beans

**Spring Configuration Models**:
- **Java-based Configuration**:
  - @Configuration and full vs lite mode
  - @Bean method semantics (inter-bean references, proxying via CGLIB)
  - @Import for composing configurations
  - @ImportResource for XML integration
  - @PropertySource and Environment abstraction
  - @Profile for environment-specific beans

- **Annotation-based Configuration**:
  - Component scanning
  - Stereotype annotations
  - Meta-annotations and composed annotations

- **XML-based Configuration** (legacy but interview-relevant):
  - Namespace handlers
  - Bean definition XML schema
  - Why the industry moved away from XML

**Spring AOP (Aspect-Oriented Programming)**:
- **AOP Concepts**:
  - Cross-cutting concerns (logging, security, transaction management)
  - Aspect, Join Point, Advice, Pointcut, Weaving
  - Compile-time vs load-time vs runtime weaving

- **Spring AOP Implementation**:
  - Proxy-based AOP (JDK dynamic proxy vs CGLIB proxy)
  - @Aspect annotation
  - Advice types: @Before, @After, @AfterReturning, @AfterThrowing, @Around
  - Pointcut expressions (execution, within, args, @annotation, bean)
  - Pointcut designators and combining pointcuts
  - Introduction (inter-type declarations)
  - AOP proxying limitations (self-invocation problem)
  - @EnableAspectJAutoProxy

- **AOP Use Cases in Enterprise**:
  - Declarative transaction management (@Transactional)
  - Security method interceptors (@PreAuthorize, @Secured)
  - Caching (@Cacheable)
  - Custom audit logging
  - Performance monitoring
  - Retry logic

**Spring Events and Event-Driven Architecture**:
- ApplicationEvent and ApplicationEventPublisher
- @EventListener and @TransactionalEventListener
- Asynchronous event handling (@Async)
- Custom events in enterprise applications
- Event ordering with @Order

**Spring Expression Language (SpEL)**:
- SpEL syntax and operators
- Using SpEL in annotations (@Value, @ConditionalOnExpression)
- Bean references, method invocations, operators
- Collection selection and projection
- Ternary and Elvis operators

**Spring Resource Abstraction**:
- Resource interface and implementations (ClassPathResource, FileSystemResource, UrlResource)
- ResourceLoader and ResourcePatternResolver
- Resource loading patterns

---

#### B. Spring Boot (EXTENSIVE COVERAGE)

**Spring Boot Fundamentals**:
- **What is Spring Boot and Why**:
  - Convention over configuration philosophy
  - Opinionated defaults
  - How Spring Boot simplifies Spring application development
  - Spring Boot vs Spring Framework (they are NOT competing)

- **Auto-Configuration Deep Dive**:
  - How auto-configuration works internally
  - @EnableAutoConfiguration and @SpringBootApplication
  - spring.factories / AutoConfiguration.imports (Spring Boot 3.x)
  - @Conditional annotations family:
    - @ConditionalOnClass / @ConditionalOnMissingClass
    - @ConditionalOnBean / @ConditionalOnMissingBean
    - @ConditionalOnProperty
    - @ConditionalOnResource
    - @ConditionalOnWebApplication / @ConditionalOnNotWebApplication
    - @ConditionalOnExpression
  - Ordering auto-configuration (@AutoConfigureBefore, @AutoConfigureAfter, @AutoConfigureOrder)
  - Disabling specific auto-configurations (exclude attribute)
  - Writing custom auto-configuration (creating your own starter)
  - Debug auto-configuration: --debug flag, ConditionEvaluationReport

- **Spring Boot Starters**:
  - What are starters and how they work
  - Common starters (spring-boot-starter-web, -data-jpa, -security, -test, -actuator, -validation, -cache, -amqp, -kafka)
  - Creating custom starters
  - Starter naming conventions (spring-boot-starter-* vs custom-spring-boot-starter)

- **Externalized Configuration (CRITICAL)**:
  - Property source hierarchy and precedence (17 levels):
    1. Default properties
    2. @PropertySource annotations
    3. application.properties / application.yml
    4. Profile-specific properties (application-{profile}.properties)
    5. OS environment variables
    6. Java system properties
    7. JNDI attributes
    8. Command-line arguments
    9. Test properties
    10. @TestPropertySource
    11. DevTools global settings
  - @Value annotation
  - @ConfigurationProperties (type-safe configuration)
  - @ConfigurationPropertiesScan
  - Relaxed binding rules
  - Configuration validation with @Validated and JSR-303
  - Profiles (@Profile, spring.profiles.active, spring.profiles.include, profile groups in Boot 2.4+)
  - YAML vs properties files
  - Multi-document files (--- separator)
  - Importing additional configuration files (spring.config.import)
  - Config data from external sources (Config Server, Vault, Consul)
  - Environment variables mapping conventions

- **Embedded Servers**:
  - Tomcat (default), Jetty, Undertow, Netty (for WebFlux)
  - How embedded server auto-configuration works
  - Configuring embedded servers (port, context path, SSL, HTTP/2)
  - Programmatic server customization (WebServerFactoryCustomizer)
  - Graceful shutdown (Spring Boot 2.3+)

- **Spring Boot Actuator (IMPORTANT)**:
  - Production-ready features overview
  - Built-in endpoints:
    - /health (liveness and readiness probes for Kubernetes)
    - /info
    - /metrics (Micrometer integration)
    - /env
    - /configprops
    - /beans
    - /mappings
    - /loggers (dynamic log level change)
    - /threaddump, /heapdump
    - /scheduledtasks
    - /flyway, /liquibase
    - /prometheus
  - Health indicators (custom health checks, composite health)
  - Health groups for Kubernetes (liveness, readiness)
  - Custom endpoints (@Endpoint, @ReadOperation, @WriteOperation)
  - Securing actuator endpoints
  - Actuator and Micrometer metrics
  - Exposing endpoints (include/exclude configuration)

- **Spring Boot DevTools**:
  - Automatic restart
  - LiveReload
  - Property defaults for development
  - Remote debugging support

- **Spring Boot CLI and Initializr**:
  - Spring Initializr (start.spring.io) for project bootstrapping
  - Spring Boot CLI usage

---

#### C. Spring MVC and Web Layer (DEEP DIVE)

**Spring MVC Architecture**:
- **DispatcherServlet**:
  - Front controller pattern
  - Request processing flow (complete lifecycle)
  - DispatcherServlet initialization and configuration
  - Multiple DispatcherServlets
  - Servlet container integration

- **Request Processing Pipeline**:
  - HandlerMapping: URL to handler resolution
  - HandlerAdapter: Handling different handler types
  - HandlerInterceptor: Pre/post processing
  - ViewResolver: Logical view to physical view
  - ExceptionResolver: Exception handling chain
  - LocaleResolver, ThemeResolver

- **Controller Layer**:
  - @Controller vs @RestController
  - @RequestMapping (method, path, params, headers, consumes, produces)
  - HTTP method annotations: @GetMapping, @PostMapping, @PutMapping, @DeleteMapping, @PatchMapping
  - Path variables (@PathVariable)
  - Request parameters (@RequestParam)
  - Request body (@RequestBody)
  - Response body (@ResponseBody)
  - ResponseEntity for full response control
  - @ModelAttribute
  - @SessionAttributes

- **Data Binding and Validation**:
  - @Valid and @Validated
  - BindingResult
  - Custom validators
  - JSR-303/380 Bean Validation (Jakarta Validation)
  - @NotNull, @NotBlank, @Size, @Min, @Max, @Pattern, @Email
  - Custom constraint annotations
  - Validation groups
  - Method-level validation

- **Exception Handling (IMPORTANT)**:
  - @ExceptionHandler (controller-level)
  - @ControllerAdvice / @RestControllerAdvice (global)
  - ResponseStatusException (Spring 5+)
  - @ResponseStatus annotation
  - ProblemDetail (RFC 7807) support (Spring 6 / Boot 3)
  - Custom error responses for REST APIs
  - Error handling best practices in microservices

- **Content Negotiation**:
  - Accept header-based
  - URL extension-based (deprecated)
  - Parameter-based
  - Producing JSON (Jackson), XML (JAXB/Jackson XML)
  - Custom HttpMessageConverters
  - Jackson ObjectMapper customization

- **REST API Design with Spring**:
  - RESTful API best practices
  - HATEOAS (Spring HATEOAS)
  - API versioning strategies (URI, header, parameter)
  - Pagination (Pageable, Page, Slice)
  - Sorting
  - Filtering
  - API documentation (SpringDoc / OpenAPI 3)

**Spring WebFlux (Reactive Web)**:
- **Reactive Programming Foundation**:
  - Project Reactor fundamentals (Mono, Flux)
  - Reactive Streams specification
  - Backpressure handling
  - Schedulers and threading model

- **WebFlux Architecture**:
  - Annotated controllers (similar to Spring MVC)
  - Functional endpoints (RouterFunction, HandlerFunction)
  - Netty as default server
  - WebClient (reactive HTTP client replacing RestTemplate)
  - Server-Sent Events (SSE)
  - WebSocket support

- **When to Use WebFlux vs MVC**:
  - Decision matrix
  - Performance characteristics
  - Threading model differences
  - Blocking vs non-blocking I/O
  - Enterprise use cases (real-time pricing, streaming data)

**HTTP Clients**:
- RestTemplate (legacy, synchronous)
- WebClient (reactive, non-blocking)
- RestClient (Spring 6.1+ / Boot 3.2+, synchronous declarative)
- HTTP Interface clients (@HttpExchange - Spring 6+)
- Feign Client (Spring Cloud OpenFeign)
- Connection pooling and timeout configuration
- Retry and circuit breaker integration

---

#### D. Spring Data and Data Access (EXTENSIVE)

**Spring Data JPA**:
- **Repository Abstraction**:
  - Repository, CrudRepository, JpaRepository, PagingAndSortingRepository
  - Query methods (method name derivation)
  - @Query (JPQL and native SQL)
  - Named queries
  - Projections (interface-based, class-based, dynamic)
  - Specifications for dynamic queries (JPA Criteria API)
  - Querydsl integration
  - Custom repository implementations

- **Entity Management**:
  - @Entity, @Table, @Column, @Id, @GeneratedValue
  - Entity relationships (@OneToOne, @OneToMany, @ManyToOne, @ManyToMany)
  - FetchType.LAZY vs FetchType.EAGER (N+1 problem)
  - Cascade types (PERSIST, MERGE, REMOVE, REFRESH, DETACH, ALL)
  - Orphan removal
  - @Embeddable and @Embedded
  - @MappedSuperclass
  - Inheritance strategies (SINGLE_TABLE, TABLE_PER_CLASS, JOINED)
  - Entity lifecycle callbacks (@PrePersist, @PostPersist, @PreUpdate, etc.)

- **Transaction Management (CRITICAL)**:
  - @Transactional annotation deep dive
  - Transaction propagation types (REQUIRED, REQUIRES_NEW, NESTED, SUPPORTS, NOT_SUPPORTED, MANDATORY, NEVER)
  - Transaction isolation levels (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE)
  - Rollback rules (rollbackFor, noRollbackFor)
  - Read-only transactions and performance implications
  - Transaction management internals (PlatformTransactionManager, proxy-based)
  - Programmatic transaction management (TransactionTemplate)
  - Self-invocation problem with @Transactional (AOP proxy limitation)
  - Distributed transactions and sagas
  - @TransactionalEventListener for post-commit actions

- **JPA Performance Optimization**:
  - N+1 query problem and solutions (JOIN FETCH, @EntityGraph, @BatchSize)
  - Batch insert/update
  - Second-level cache (Ehcache, Redis)
  - Query result caching
  - Connection pooling (HikariCP - default in Spring Boot)
  - Hibernate statistics and logging
  - Read replicas configuration
  - Pagination best practices

- **Database Migrations**:
  - Flyway integration with Spring Boot
  - Liquibase integration with Spring Boot
  - Migration strategies and best practices
  - Baseline migrations

**Spring Data JDBC**:
- Differences from Spring Data JPA
- Aggregate root concept
- When to use JDBC over JPA
- Simpler object-relational mapping

**Spring Data with NoSQL**:
- Spring Data Redis (caching, session management)
- Spring Data MongoDB
- Spring Data Elasticsearch
- Repository abstraction consistency across data stores

**JDBC and JdbcTemplate**:
- JdbcTemplate and NamedParameterJdbcTemplate
- RowMapper and ResultSetExtractor
- SimpleJdbcInsert and SimpleJdbcCall
- Batch operations
- When to use JdbcTemplate over JPA

---

#### E. Spring Security (DEEP DIVE)

**Security Fundamentals**:
- **Authentication Architecture**:
  - SecurityFilterChain
  - AuthenticationManager and AuthenticationProvider
  - UserDetailsService and UserDetails
  - PasswordEncoder (BCrypt, SCrypt, Argon2)
  - Authentication object and SecurityContext
  - SecurityContextHolder strategies (ThreadLocal, InheritableThreadLocal, Global)

- **Filter Chain Architecture**:
  - DelegatingFilterProxy
  - FilterChainProxy
  - SecurityFilterChain (Spring Security 5.7+ / 6.x configuration style)
  - Built-in filters and their order
  - Custom filters
  - OncePerRequestFilter

- **Authorization**:
  - Role-based access control (RBAC)
  - Authority-based access control
  - @PreAuthorize, @PostAuthorize, @PreFilter, @PostFilter (method-level security)
  - @Secured and @RolesAllowed
  - HTTP security (requestMatchers, authorizeHttpRequests)
  - Expression-based access control (hasRole, hasAuthority, hasPermission)
  - Custom AccessDecisionManager / AuthorizationManager (Spring Security 6+)

- **Modern Spring Security Configuration (Boot 3.x / Security 6.x)**:
  - SecurityFilterChain bean-based configuration
  - Lambda DSL
  - Removing WebSecurityConfigurerAdapter (deprecated in 5.7, removed in 6.0)
  - requestMatchers vs antMatchers migration

**Authentication Mechanisms**:
- Form-based authentication
- HTTP Basic authentication
- JWT (JSON Web Token) authentication
  - JWT structure (header, payload, signature)
  - Token generation and validation
  - Refresh token strategy
  - Stateless authentication
- OAuth 2.0 and OpenID Connect
  - Authorization Code flow
  - Client Credentials flow
  - Resource Server configuration
  - @EnableResourceServer / jwt() decoder configuration
  - Spring Authorization Server
- SAML 2.0
- LDAP authentication
- Multi-factor authentication patterns
- Remember-Me authentication
- Session management (concurrent sessions, session fixation protection)

**CORS and CSRF**:
- Cross-Origin Resource Sharing configuration
- CSRF protection and when to disable (stateless APIs)
- SameSite cookie attributes

**Security Best Practices in Banking**:
- Securing REST APIs
- API key and mutual TLS (mTLS) authentication
- Rate limiting and throttling
- Input validation and sanitization
- Audit logging for security events
- Secret management (Spring Cloud Vault, Azure Key Vault)
- OWASP Top 10 mitigations with Spring Security
- PCI-DSS compliance considerations

---

#### F. Spring Boot Testing (COMPREHENSIVE)

**Testing Hierarchy**:
- Unit tests vs integration tests vs end-to-end tests
- Test pyramid in Spring Boot applications

**Unit Testing**:
- @ExtendWith(MockitoExtension.class)
- @Mock, @InjectMocks, @Spy
- Mockito fundamentals (when/thenReturn, verify, argument captors)
- Testing services, controllers, repositories independently
- AssertJ assertions

**Spring Boot Test Support**:
- @SpringBootTest (full application context)
- @WebMvcTest (controller layer only)
- @DataJpaTest (JPA repository layer)
- @WebFluxTest (WebFlux controllers)
- @JsonTest (JSON serialization/deserialization)
- @RestClientTest
- @JdbcTest
- Test slices concept and why they matter

**Integration Testing**:
- @SpringBootTest with TestRestTemplate / WebTestClient
- @AutoConfigureMockMvc and MockMvc
- TestContainers for database/Kafka/Redis integration tests
- @DynamicPropertySource for container-based tests
- @Sql and @SqlGroup for test data setup
- Test profiles and configuration

**Test Configuration**:
- @TestConfiguration
- @MockBean and @SpyBean (and their deprecation path in Boot 3.4+)
- @Import for test-specific beans
- Test properties (application-test.yml)
- @ActiveProfiles("test")

**Contract Testing**:
- Spring Cloud Contract
- Consumer-driven contract testing
- Pact framework integration

---

#### G. Spring Cloud and Microservices Patterns (CRITICAL FOR INTERVIEWS)

**Service Discovery**:
- Spring Cloud Netflix Eureka (legacy but interview-relevant)
- Spring Cloud Consul
- Spring Cloud Kubernetes (native service discovery)

**API Gateway**:
- Spring Cloud Gateway (reactive)
- Route predicates and filters
- Rate limiting, circuit breaking at gateway level
- Request/response transformation
- Authentication at gateway level

**Configuration Management**:
- Spring Cloud Config Server and Client
- Config refresh and bus refresh
- Vault integration for secrets
- Consul KV store
- Spring Cloud Kubernetes ConfigMap/Secret integration

**Resilience Patterns**:
- **Circuit Breaker** (Resilience4j):
  - States: CLOSED, OPEN, HALF_OPEN
  - Configuration parameters (failure rate threshold, wait duration, sliding window)
  - Fallback methods
  - @CircuitBreaker annotation
- **Retry** (Resilience4j):
  - @Retry annotation
  - Exponential backoff
  - Retry on specific exceptions
- **Rate Limiter** (Resilience4j):
  - @RateLimiter annotation
  - Configuration options
- **Bulkhead** (Resilience4j):
  - Semaphore-based vs thread pool-based
  - @Bulkhead annotation
- **Time Limiter**:
  - @TimeLimiter for timeout control

**Distributed Tracing and Observability**:
- Micrometer Tracing (replacing Spring Cloud Sleuth)
- Trace ID and Span ID propagation
- Zipkin / Jaeger integration
- Micrometer metrics
- Prometheus and Grafana dashboards
- OpenTelemetry integration (Spring Boot 3+)
- Log correlation with trace IDs (MDC)

**Inter-Service Communication**:
- Synchronous: RestClient, WebClient, OpenFeign
- Asynchronous: Kafka, RabbitMQ, Event-driven architecture
- gRPC with Spring Boot
- Service mesh considerations (Istio, Linkerd)

---

#### H. Spring Messaging and Event-Driven Architecture

**Apache Kafka with Spring**:
- **Spring Kafka**:
  - KafkaTemplate for producing messages
  - @KafkaListener for consuming messages
  - Consumer groups and partition assignment
  - Error handling (DefaultErrorHandler, DeadLetterPublishingRecoverer, retry topics)
  - Exactly-once semantics
  - Kafka transactions with Spring @Transactional
  - Schema Registry and Avro/JSON Schema
  - KafkaStreams integration
  - Kafka consumer concurrency configuration
  - Manual vs automatic acknowledgment
  - Batch listener

- **Spring AMQP (RabbitMQ)**:
  - RabbitTemplate
  - @RabbitListener
  - Exchange types (direct, topic, fanout, headers)
  - Dead letter exchanges
  - Message acknowledgment
  - Retry and error handling

- **Spring JMS**:
  - JmsTemplate
  - @JmsListener
  - ActiveMQ / IBM MQ integration

**Event-Driven Patterns**:
- Event sourcing with Spring
- CQRS pattern implementation
- Saga pattern (orchestration vs choreography)
- Outbox pattern for reliable messaging
- Idempotent consumers

---

#### I. Spring Caching

**Caching Abstraction**:
- @EnableCaching
- @Cacheable, @CachePut, @CacheEvict, @Caching
- Cache managers (ConcurrentMapCacheManager, EhCache, Caffeine, Redis)
- SpEL in cache annotations (key, condition, unless)
- Custom key generators
- Cache synchronization

**Distributed Caching**:
- Redis as distributed cache (Spring Data Redis)
- Cache invalidation strategies
- Cache-aside, read-through, write-through, write-behind patterns
- TTL and eviction policies
- Multi-level caching (local + distributed)

---

#### J. Spring Scheduling and Async Processing

**Task Scheduling**:
- @EnableScheduling and @Scheduled
- Fixed rate vs fixed delay vs cron expressions
- TaskScheduler abstraction
- Dynamic scheduling
- Distributed scheduling (ShedLock, Quartz with Spring)

**Async Processing**:
- @EnableAsync and @Async
- Custom TaskExecutor configuration
- CompletableFuture integration
- Exception handling in async methods
- @Async limitations (self-invocation, proxy-based)
- Virtual threads integration (Spring Boot 3.2+ with Java 21)

---

#### K. Spring Boot Version-Specific Features and Migration

**Spring Boot 2.x Key Features**:
- Spring Boot 2.0: Spring Framework 5, Reactive support, Kotlin support
- Spring Boot 2.1: Java 11 support, lazy initialization
- Spring Boot 2.2: Immutable @ConfigurationProperties
- Spring Boot 2.3: Graceful shutdown, Liveness/Readiness probes, Buildpacks
- Spring Boot 2.4: Config file processing changes, startup endpoints
- Spring Boot 2.5: Java 16 support, enhanced Docker image building
- Spring Boot 2.6: Pathpattern-based path matching by default
- Spring Boot 2.7: Auto-configuration registration changes

**Spring Boot 3.x Key Features (CRITICAL)**:
- Spring Boot 3.0:
  - Jakarta EE 9+ (javax → jakarta namespace migration)
  - Java 17 baseline
  - Spring Framework 6.0
  - Native image support with GraalVM (Spring Native)
  - Micrometer Observation API
  - Micrometer Tracing (replacing Sleuth)
  - ProblemDetail for error responses (RFC 7807)
  - Removal of deprecated code
  - HttpMessageConverters refactoring
- Spring Boot 3.1:
  - Docker Compose support
  - Testcontainers at development time
  - SSL bundle support
  - Service connection abstraction
- Spring Boot 3.2:
  - Virtual threads support (Java 21)
  - JdbcClient
  - RestClient
  - SSL bundle reloading
  - Observability improvements
- Spring Boot 3.3:
  - CDS (Class Data Sharing) support
  - Structured logging
  - Service connection improvements
- Spring Boot 3.4 (Latest):
  - Enhanced observability
  - Continued virtual threads improvements
  - MockMvcTester
  - @Fallback beans

**Migration Guide (2.x → 3.x)**:
- javax → jakarta migration steps
- Spring Security 5 → 6 changes
- Configuration property changes
- Testing changes
- Third-party library compatibility
- Automated migration tools (OpenRewrite recipes)

---

#### L. Spring Native and GraalVM

**Native Images with Spring**:
- AOT (Ahead-of-Time) compilation
- Spring AOT processing
- Native image build configuration
- Reflection and proxy configuration for native images
- Build tools (Maven native profile, Gradle native plugin)
- Performance characteristics (startup time, memory footprint)
- Limitations and unsupported features
- Cloud-native deployment benefits (containers, serverless)

---

#### M. Spring Observability and Production Readiness

**Micrometer Metrics**:
- Meter types (Counter, Gauge, Timer, DistributionSummary, LongTaskTimer)
- Meter registries (Prometheus, Datadog, New Relic, CloudWatch)
- Custom metrics
- @Timed and @Counted annotations
- Dimensional metrics and tags
- SLOs (Service Level Objectives) with Micrometer

**Logging**:
- Spring Boot default logging (Logback)
- SLF4J facade
- Log levels and configuration
- Structured logging (JSON format for ELK/Splunk)
- MDC (Mapped Diagnostic Context) with trace correlation
- Log4j2 integration
- Async logging for performance

**Distributed Tracing (Detailed)**:
- Micrometer Tracing with Brave or OpenTelemetry
- W3C Trace Context propagation
- Instrumentation (auto vs manual)
- Baggage propagation
- Sampling strategies

---

#### N. Advanced Spring Topics

**Spring Modulith**:
- Modular monolith architecture
- Module boundaries and verification
- Application events for inter-module communication
- Documenting application structure
- When monolith-first makes sense

**Spring HATEOAS**:
- Hypermedia-driven REST APIs
- RepresentationModel, EntityModel, CollectionModel
- Link and Affordance
- HAL, HAL-Forms media types

**Spring Batch**:
- Job, Step, Chunk, Tasklet concepts
- ItemReader, ItemProcessor, ItemWriter
- Job scheduling and monitoring
- Fault tolerance (skip, retry, restart)
- Partitioning for parallel batch processing
- Use cases in banking (end-of-day processing, report generation, ETL)

**Spring Profiles and Multi-Tenancy**:
- Profile-based configuration for environments
- Multi-tenant architecture patterns with Spring
- Database per tenant vs schema per tenant vs shared schema
- Tenant resolution strategies

**Spring Integration** (brief coverage):
- Enterprise Integration Patterns (EIP)
- Message channels, transformers, routers
- Adapters (file, FTP, JDBC, JMS, Kafka)

---

### 2. Format Requirements

Each major topic section MUST include:

```markdown
# Topic Name

## Overview (2-3 paragraphs)
- What is it and why it matters
- Why interviewers ask about this
- Real-world relevance in enterprise banking systems

## Foundational Concepts
- Start with basics, build up complexity
- Define all terms clearly
- Address common misconceptions

## Technical Deep Dive
- Detailed explanations with multiple subsections
- Step-by-step process flows
- Internal implementation details (e.g., how @Transactional works under the hood)

## Visual Representations (MANDATORY)
- Architecture diagrams using Mermaid
- Request flow diagrams
- Sequence diagrams for complex interactions (e.g., Spring Security filter chain)
- Component relationship diagrams
- Before/after comparison diagrams
- Configuration hierarchy diagrams

## Code Examples (Where Relevant)
- Minimal, focused code snippets
- Heavy annotations explaining the "why"
- Show what happens under the hood
- Include both correct and incorrect patterns
- Real-world enterprise examples (not toy code)
- Spring Boot 3.x / Java 17+ syntax

## Configuration Examples
- application.yml / application.properties snippets
- Annotated configuration with explanations
- Environment-specific configuration patterns
- Production-grade configuration examples

## Interview Questions & Model Answers
- 10-15 common interview questions for this topic
- Detailed answers explaining the concept
- Follow-up questions interviewers might ask
- How to structure your answer in an interview
- Gotcha questions and tricky scenarios

## Real-World Enterprise Scenarios
- How this applies in banking/financial services
- Microservices architecture examples
- High-availability and resilience patterns
- Performance implications in production
- Regulatory compliance considerations (PCI-DSS, SOX, GDPR)

## Common Pitfalls & Best Practices
- What NOT to do
- Production war stories (lessons learned)
- Performance anti-patterns
- Debugging tips
- Code review red flags

## Comparison Tables
- When to use X vs Y
- Trade-offs between approaches
- Performance characteristics comparison
- Version-specific differences

## Key Takeaways (Bullet Points)
- 5-7 critical points to remember
- Facts and numbers
- Quick reference for last-minute review

## Further Reading
- Official Spring documentation links
- Spring blog references
- Authoritative books ("Spring in Action", "Cloud Native Spring in Action")
- GitHub examples and reference architectures
```

### 3. Research Requirements

**YOU MUST**:
1. **Search official documentation**:
   - Spring Framework reference documentation (spring.io)
   - Spring Boot reference documentation
   - Spring Security reference documentation
   - Spring Data JPA documentation
   - Spring Cloud documentation
   - Spring for Apache Kafka documentation
   - Micrometer documentation

2. **Reference authoritative sources**:
   - "Spring in Action" by Craig Walls (6th Edition for Spring Boot 3)
   - "Cloud Native Spring in Action" by Thomas Vitale
   - "Spring Security in Action" by Laurentiu Spilca
   - "Reactive Spring" by Josh Long
   - Spring blog (spring.io/blog)
   - Baeldung (for practical examples)
   - Official Spring guides (spring.io/guides)

3. **Include version-specific information**:
   - Specify Spring Boot versions (2.7, 3.0, 3.1, 3.2, 3.3, 3.4)
   - Specify Spring Framework versions (5.x vs 6.x)
   - Specify Spring Security versions (5.x vs 6.x)
   - Highlight what changed between versions
   - Note deprecated features and migration paths

4. **Provide factual information**:
   - Release dates for versions
   - Default configuration values
   - Default thread pool sizes
   - Connection pool defaults (HikariCP)
   - Actuator endpoint paths
   - Performance characteristics with data where available

### 4. Special Requirements

#### Diagrams
- **MUST create Mermaid diagrams** for:
  - Spring IoC container and bean lifecycle
  - Spring MVC request processing flow
  - Spring Security filter chain
  - Spring Boot auto-configuration process
  - Spring Data JPA architecture
  - Transaction propagation behavior
  - Spring Cloud microservices architecture
  - Circuit breaker state machine
  - Kafka consumer/producer architecture
  - Event-driven architecture patterns
  - Spring Boot starter dependency hierarchy
  - Spring Actuator architecture
  - OAuth2/JWT authentication flow
  - Saga pattern (orchestration and choreography)
  - Spring Boot 2.x vs 3.x architecture differences

#### Interview Focus
- After each major section, include "What Interviewers Look For"
- Provide both surface-level and deep technical answers
- Show how to explain complex topics simply (elevator pitch style)
- Include follow-up questions and how to handle them
- Cover tricky interview scenarios (e.g., "Why doesn't @Transactional work on private methods?")
- Include system design questions involving Spring Boot microservices

#### Enterprise Banking Context
- Relate concepts to high-volume transaction processing systems
- Discuss microservices patterns for banking domains (payments, accounts, risk, compliance)
- Security considerations for financial applications (PCI-DSS, SOX)
- Event-driven architecture for real-time fraud detection
- API gateway patterns for channel integration (mobile, web, partner APIs)
- Database per microservice pattern for banking domains
- Audit trail and regulatory requirements
- High availability and disaster recovery patterns

### 5. Structure and Organization

Create the guide as separate, well-organized markdown files:

```
spring/
├── 01-spring-core-ioc-di.md
├── 02-bean-lifecycle-and-scopes.md
├── 03-spring-aop.md
├── 04-spring-boot-fundamentals.md
├── 05-spring-boot-auto-configuration.md
├── 06-spring-boot-externalized-configuration.md
├── 07-spring-mvc-deep-dive.md
├── 08-spring-webflux-reactive.md
├── 09-spring-data-jpa.md
├── 10-transaction-management.md
├── 11-spring-security-authentication.md
├── 12-spring-security-authorization-oauth2-jwt.md
├── 13-spring-boot-testing.md
├── 14-spring-cloud-microservices-patterns.md
├── 15-resilience-patterns-circuit-breaker.md
├── 16-spring-kafka-messaging.md
├── 17-spring-caching.md
├── 18-spring-scheduling-async.md
├── 19-spring-observability-monitoring.md
├── 20-spring-batch.md
├── 21-spring-native-graalvm.md
├── 22-spring-boot-version-features-migration.md
├── 23-advanced-topics-modulith-hateoas.md
└── 24-spring-interview-master-guide.md
```

The master guide (24-spring-interview-master-guide.md) should have this structure:

```markdown
# Spring Framework & Spring Boot - Complete Interview Preparation Guide

## Table of Contents
[Detailed TOC with links to all sections]

## How to Use This Guide
- Study path for interview preparation
- Quick reference sections
- Priority topics by interview type (coding, system design, deep dive)

## Part 1: Spring Framework Core
### 1.1 IoC Container and Dependency Injection
### 1.2 Bean Lifecycle and Scopes
### 1.3 Configuration Models (Java, Annotation, XML)
### 1.4 Spring AOP
### 1.5 Spring Events and SpEL

## Part 2: Spring Boot Fundamentals
### 2.1 Auto-Configuration Internals
### 2.2 Starters and Dependency Management
### 2.3 Externalized Configuration
### 2.4 Embedded Servers
### 2.5 Actuator and Production Readiness
### 2.6 DevTools and Developer Experience

## Part 3: Web Layer
### 3.1 Spring MVC Architecture and Request Flow
### 3.2 RESTful API Design
### 3.3 Exception Handling and Validation
### 3.4 Content Negotiation and Serialization
### 3.5 Spring WebFlux and Reactive Programming
### 3.6 HTTP Clients (RestClient, WebClient, OpenFeign)

## Part 4: Data Access
### 4.1 Spring Data JPA Repository Abstraction
### 4.2 Entity Management and Relationships
### 4.3 Transaction Management Deep Dive
### 4.4 JPA Performance Optimization (N+1, Caching, Batching)
### 4.5 Database Migrations (Flyway, Liquibase)
### 4.6 JdbcTemplate and JdbcClient

## Part 5: Spring Security
### 5.1 Security Architecture and Filter Chain
### 5.2 Authentication Mechanisms
### 5.3 JWT and OAuth 2.0 Authentication
### 5.4 Method-Level Security and Authorization
### 5.5 CORS, CSRF, and Security Headers
### 5.6 Security Best Practices for Financial Services

## Part 6: Testing
### 6.1 Unit Testing with Mockito
### 6.2 Spring Boot Test Slices
### 6.3 Integration Testing with TestContainers
### 6.4 MockMvc and WebTestClient
### 6.5 Contract Testing

## Part 7: Microservices and Spring Cloud
### 7.1 Service Discovery and API Gateway
### 7.2 Configuration Management
### 7.3 Resilience Patterns (Circuit Breaker, Retry, Bulkhead)
### 7.4 Inter-Service Communication
### 7.5 Distributed Tracing and Observability
### 7.6 Event-Driven Microservices

## Part 8: Messaging and Event-Driven Architecture
### 8.1 Spring Kafka Deep Dive
### 8.2 RabbitMQ with Spring AMQP
### 8.3 Event Sourcing and CQRS
### 8.4 Saga and Outbox Patterns

## Part 9: Caching, Scheduling, and Async
### 9.1 Spring Caching Abstraction
### 9.2 Distributed Caching with Redis
### 9.3 Task Scheduling
### 9.4 Async Processing and Virtual Threads

## Part 10: Production Readiness
### 10.1 Micrometer Metrics and Monitoring
### 10.2 Logging and Structured Logging
### 10.3 Health Checks and Kubernetes Probes
### 10.4 Spring Native and GraalVM
### 10.5 Performance Tuning

## Part 11: Spring Boot Versions and Migration
### 11.1 Spring Boot 2.x Feature Timeline
### 11.2 Spring Boot 3.x Feature Timeline
### 11.3 Migration Guide (2.x → 3.x)
### 11.4 Jakarta EE Migration

## Part 12: Advanced Topics
### 12.1 Spring Modulith
### 12.2 Spring Batch
### 12.3 Spring HATEOAS
### 12.4 Multi-Tenancy Patterns

## Part 13: Interview Preparation
### 13.1 Top 100 Spring Boot Interview Questions
### 13.2 System Design Questions with Spring Boot
### 13.3 Live Coding Challenges (REST API, Security Config, etc.)
### 13.4 Troubleshooting Scenarios
### 13.5 How to Discuss Spring Topics in Interviews
### 13.6 Architect-Level Spring Questions

## Appendices
### Appendix A: Spring Boot Properties Quick Reference
### Appendix B: Spring Security Configuration Cheat Sheet
### Appendix C: Spring Boot Starter Dependency Matrix
### Appendix D: @Transactional Propagation Quick Reference
### Appendix E: Resilience4j Configuration Reference
### Appendix F: Spring Boot 2.x vs 3.x Comparison Matrix
### Appendix G: Recommended Reading and Resources
```

### 6. Quality Standards

The guide MUST be:
- ✅ **Comprehensive**: Cover 100% of Spring Framework and Spring Boot interview topics
- ✅ **Accurate**: All technical details verified against official Spring documentation
- ✅ **Visual**: At least 40+ diagrams throughout
- ✅ **Interview-ready**: Every section has interview Q&A
- ✅ **Enterprise-focused**: Real-world banking/financial context
- ✅ **Version-aware**: Clear about Spring Boot 2.x vs 3.x differences
- ✅ **Deep**: Go beyond surface-level explanations (explain internals, proxy mechanisms, auto-configuration flow)
- ✅ **Practical**: Include configuration examples, troubleshooting, monitoring
- ✅ **Production-grade**: Cover resilience, observability, security, performance tuning

### 7. Expected Length

This should be a **substantial, comprehensive guide**:
- Estimated length: 25,000-40,000 words across all files
- 40-50+ Mermaid diagrams
- 200+ interview questions with detailed answers
- Multiple code and configuration examples per major section
- Comparison tables for key decision points
- Real-world architecture diagrams for banking microservices

### 8. Success Criteria

When complete, a reader should be able to:
1. Explain Spring IoC, DI, and AOP to both junior developers and senior architects
2. Design and implement Spring Boot microservices for enterprise banking systems
3. Configure Spring Security for JWT/OAuth2 authentication in REST APIs
4. Tune Spring Data JPA for high-performance data access
5. Implement resilience patterns (circuit breaker, retry, bulkhead) with confidence
6. Set up production-grade observability (metrics, tracing, logging)
7. Choose between Spring MVC and WebFlux with clear justification
8. Answer any Spring-related question in a technical interview with confidence
9. Discuss Spring Boot version migration strategies and trade-offs
10. Design event-driven architectures using Spring Kafka
11. Explain how Spring Boot auto-configuration works internally
12. Troubleshoot common Spring Boot production issues

## Additional Instructions

- Use British English spellings where applicable
- Include version numbers for all features (e.g., "Introduced in Spring Boot 3.0")
- Cite sources in footnotes or references section
- If certain information is not available or outdated, clearly state this
- When discussing deprecated features (like WebSecurityConfigurerAdapter), explain why and what replaced them
- Include both theoretical knowledge and practical configuration examples
- Show evolution of features (e.g., how Spring Security configuration evolved from XML to Java config to lambda DSL)
- Emphasize Spring Boot 3.x as the current standard, but cover 2.x for migration context
- All code examples should use Java 17+ syntax and Spring Boot 3.x APIs unless showing migration comparisons
- Cover Kotlin-specific Spring features briefly (many enterprise teams use Kotlin with Spring)

## Interview Question Categories to Include

For each major topic, include questions in these categories:
1. **Foundational**: Basic concepts (Junior level)
2. **Intermediate**: Practical application (Mid-level)
3. **Advanced**: Deep technical understanding (Senior level)
4. **Tricky**: Edge cases and gotchas (Staff/Principal level)
5. **Scenario-based**: Real-world problem solving (Architect level)
6. **System Design**: Architecture decisions involving Spring (Staff+ level)

## Code Example Requirements

Every code example MUST:
- Be compilable and runnable with Spring Boot 3.x
- Include relevant import statements
- Have inline comments explaining the "why"
- Show both what works and common mistakes
- Include expected behavior or output
- Use realistic entity/service names from banking domain (Account, Transaction, Payment, etc.)
- Follow Spring best practices and conventions
- Use constructor injection (not field injection)
- Use records where applicable (Java 17+)

## Configuration Example Requirements

Every configuration example MUST:
- Use application.yml format (with equivalent .properties comment)
- Include sensible defaults and explain why
- Show environment-specific overrides
- Cover both development and production configurations
- Include comments on security implications of settings
- Reference official documentation for each property

## Begin!

Start by researching official Spring documentation (spring.io), Spring blog posts, and authoritative Spring books. Then create the comprehensive guide following the structure and requirements above.

**Remember**: This is for interview preparation at senior/principal level - depth and accuracy are critical. Don't skip details. Every section should be thorough enough that the reader could teach it to someone else and ace technical interviews at top financial institutions.

Focus on:
- **Why** things work the way they do (e.g., why proxy-based AOP has limitations)
- **How** things work internally (e.g., how auto-configuration resolves conditions)
- **When** to use different approaches (e.g., WebFlux vs MVC, JPA vs JdbcTemplate)
- **Trade-offs** between options (e.g., circuit breaker vs retry vs bulkhead)
- **Evolution** of features across Spring Boot versions
- **Real-world** application in enterprise banking microservices
- **Interview** strategies and model answers
- **Production** considerations (monitoring, scaling, resilience)
