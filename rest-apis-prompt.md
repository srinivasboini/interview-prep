# Prompt: Create Comprehensive REST APIs Interview Preparation Guide

## Objective
Create an in-depth, interview-ready preparation guide on REST APIs that covers everything a Senior Java/Backend Developer (13+ years experience) needs to know for technical interviews at Staff/Principal Engineer level in enterprise banking/financial services. This guide should encompass REST fundamentals, HTTP protocol internals, API design best practices, security, versioning, Spring REST implementation, performance, testing, documentation, and real-world enterprise API architecture patterns.

## Target Audience Context
- **Experience Level**: Senior developer preparing for Staff/Principal Engineer roles
- **Current Knowledge**: 13+ years of enterprise Java experience in banking
- **Interview Focus**: Technical depth interviews at companies like JP Morgan, UBS, Goldman Sachs, etc.
- **Learning Style**: Prefers understanding "why" over "what", visual explanations, real-world enterprise context
- **Tech Stack**: Primarily Java/Spring Boot, but concepts should be framework-agnostic where applicable

## Content Requirements

### 1. Core Topics to Cover (MUST BE COMPREHENSIVE)

#### A. REST Fundamentals & Architectural Principles

**REST (Representational State Transfer) Core Concepts**:
- **What is REST**:
  - Roy Fielding's dissertation and the origin of REST
  - REST as an architectural style, NOT a protocol
  - REST vs SOAP vs GraphQL vs gRPC (detailed comparison)
  - Why REST became the dominant API paradigm

- **REST Architectural Constraints**:
  - **Client-Server**: Separation of concerns
  - **Statelessness**: No server-side session state, implications for scalability
  - **Cacheability**: Cache-Control headers, ETags, conditional requests
  - **Uniform Interface**: The four sub-constraints:
    - Resource identification (URIs)
    - Resource manipulation through representations
    - Self-descriptive messages
    - HATEOAS (Hypermedia as the Engine of Application State)
  - **Layered System**: Intermediary servers (proxies, gateways, load balancers)
  - **Code on Demand** (optional): Executable code transfer

- **Richardson Maturity Model**:
  - Level 0: The Swamp of POX (single URI, single HTTP method)
  - Level 1: Resources (multiple URIs, single HTTP method)
  - Level 2: HTTP Verbs (multiple URIs, proper HTTP methods)
  - Level 3: Hypermedia Controls (HATEOAS)
  - Why most "REST" APIs are actually Level 2
  - Practical implications for enterprise API design

- **Resources and Representations**:
  - Resource identification and URI design
  - Resource vs representation distinction
  - Content types (JSON, XML, HAL, JSON-LD)
  - Content negotiation (Accept and Content-Type headers)

#### B. HTTP Protocol Deep Dive (CRITICAL)

**HTTP Methods (Verbs)**:
- **GET**: Safe, idempotent, cacheable; query parameters; response codes
- **POST**: Non-idempotent, resource creation; request body semantics
- **PUT**: Idempotent, full resource replacement; create-or-update semantics
- **PATCH**: Partial updates; JSON Patch (RFC 6902) vs JSON Merge Patch (RFC 7396)
- **DELETE**: Idempotent, resource removal; soft delete patterns
- **HEAD**: Metadata retrieval without body
- **OPTIONS**: CORS preflight, API capability discovery
- **TRACE**: Diagnostic loop-back (security implications)
- **Safe vs Idempotent vs Cacheable** properties matrix
- **Method Selection Decision Tree** for API design

**HTTP Status Codes (Comprehensive)**:
- **1xx Informational**: 100 Continue, 101 Switching Protocols
- **2xx Success**:
  - 200 OK (general success)
  - 201 Created (resource creation with Location header)
  - 202 Accepted (asynchronous processing)
  - 204 No Content (successful with no response body)
  - 206 Partial Content (range requests)
- **3xx Redirection**:
  - 301 Moved Permanently
  - 302 Found
  - 304 Not Modified (conditional requests, caching)
  - 307 Temporary Redirect (preserves method)
  - 308 Permanent Redirect (preserves method)
- **4xx Client Errors**:
  - 400 Bad Request (validation errors)
  - 401 Unauthorized (authentication required)
  - 403 Forbidden (authorisation denied)
  - 404 Not Found
  - 405 Method Not Allowed
  - 406 Not Acceptable (content negotiation failure)
  - 408 Request Timeout
  - 409 Conflict (concurrent modification, optimistic locking)
  - 410 Gone (permanently removed)
  - 412 Precondition Failed (conditional request failure)
  - 415 Unsupported Media Type
  - 422 Unprocessable Entity (semantic validation)
  - 429 Too Many Requests (rate limiting)
- **5xx Server Errors**:
  - 500 Internal Server Error
  - 502 Bad Gateway
  - 503 Service Unavailable (maintenance, circuit breaker)
  - 504 Gateway Timeout
- **Common status code mistakes** and correct usage patterns

**HTTP Headers (Essential Knowledge)**:
- **Request Headers**:
  - Authorization (Bearer, Basic, API Key)
  - Accept, Accept-Language, Accept-Encoding
  - Content-Type, Content-Length
  - If-Match, If-None-Match (ETags)
  - If-Modified-Since, If-Unmodified-Since
  - X-Request-ID, X-Correlation-ID (traceability)
  - Custom headers vs standard headers
- **Response Headers**:
  - Cache-Control, Expires, Pragma
  - ETag, Last-Modified
  - Location (for 201, 3xx responses)
  - Content-Type, Content-Length
  - X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
  - Retry-After (for 429, 503)
  - CORS headers (Access-Control-*)
- **Security Headers**:
  - Strict-Transport-Security (HSTS)
  - Content-Security-Policy
  - X-Content-Type-Options
  - X-Frame-Options
  - X-XSS-Protection (deprecated but interview-relevant)

**HTTP/1.1 vs HTTP/2 vs HTTP/3**:
- Protocol evolution and key differences
- Multiplexing, header compression, server push
- Impact on REST API design and performance
- Connection management and keep-alive
- gRPC's relationship to HTTP/2

#### C. RESTful API Design Best Practices (DEEP DIVE)

**URI Design Principles**:
- **Resource Naming Conventions**:
  - Nouns, not verbs (resources, not actions)
  - Plural vs singular resource names
  - Hierarchical resource relationships (nested URIs)
  - Resource naming patterns: `/api/v1/accounts/{accountId}/transactions`
  - URI path vs query parameters vs request body
  - Maximum URI length considerations
- **URI Anti-Patterns**:
  - Verb-based URIs (e.g., `/getAccount`, `/createUser`)
  - CRUD in URI (e.g., `/deleteOrder/123`)
  - Deep nesting beyond 2-3 levels
  - File extensions in URIs (e.g., `.json`, `.xml`)

**Request/Response Design**:
- **Request Body Design**:
  - JSON structure best practices
  - Flat vs nested payloads
  - Required vs optional fields
  - Default values and nullability
  - Bulk/batch operation patterns
- **Response Body Design**:
  - Envelope pattern (wrapping responses)
  - Direct response vs wrapper objects
  - Consistent error response format (RFC 7807 - Problem Details)
  - Metadata in responses (timestamps, request IDs)
  - Partial responses and field selection
- **Error Handling Design**:
  - Standardised error response format
  - Error codes vs HTTP status codes
  - Human-readable error messages
  - Machine-readable error codes
  - Validation error details (field-level errors)
  - Error response example structure

**Pagination**:
- **Offset-based Pagination**: `?page=2&size=20`
  - Pros, cons, and performance implications
  - Deep pagination problems
- **Cursor-based Pagination**: `?cursor=eyJpZCI6MTIzfQ==`
  - Opaque cursors and why they're better for large datasets
  - Consistency guarantees
- **Keyset Pagination**: `?after=lastId&limit=20`
  - Performance advantages with indexed columns
- **Pagination Metadata**:
  - Total count, page size, current page
  - Next/previous links (HATEOAS)
  - Link header (RFC 8288)
- **Pagination in enterprise banking**: Transaction history, audit logs, account listings

**Filtering, Sorting, and Searching**:
- **Filtering Patterns**:
  - Simple query parameters: `?status=active&type=savings`
  - Complex filters: `?filter=amount>1000 AND currency=USD`
  - LHS brackets: `?amount[gte]=1000`
  - RQL (Resource Query Language)
- **Sorting**: `?sort=createdAt:desc,amount:asc`
- **Searching**: Full-text search patterns, `?q=keyword`
- **Field Selection**: `?fields=id,name,balance` (sparse fieldsets)

**HATEOAS (Hypermedia)**:
- Why HATEOAS matters for API discoverability
- HAL (Hypertext Application Language)
- JSON:API specification
- Spring HATEOAS implementation
- Practical vs theoretical HATEOAS
- When HATEOAS adds value vs over-engineering

#### D. API Versioning Strategies (CRITICAL FOR INTERVIEWS)

**Versioning Approaches**:
- **URI Path Versioning**: `/api/v1/accounts`, `/api/v2/accounts`
  - Pros: Simple, visible, easy to route
  - Cons: URI pollution, not truly RESTful
- **Query Parameter Versioning**: `/api/accounts?version=2`
  - Pros: Optional, backward-compatible
  - Cons: Caching complications, easy to forget
- **Header Versioning**: `Accept: application/vnd.company.v2+json`
  - Pros: Clean URIs, proper content negotiation
  - Cons: Not visible in URLs, harder to test in browser
- **Media Type Versioning (Content Negotiation)**: Custom media types
  - Pros: Most RESTful approach
  - Cons: Complex, harder tooling support
- **Version Comparison Matrix**: When to use which approach
- **Enterprise Banking Perspective**: Why path versioning dominates in practice

**Versioning Best Practices**:
- Semantic versioning for APIs
- Backward compatibility contracts
- Deprecation strategies and sunset policies
- API changelog management
- Consumer-driven contract testing
- Breaking vs non-breaking changes

**API Lifecycle Management**:
- API design phase
- Development and testing
- Publishing and documentation
- Monitoring and analytics
- Deprecation and retirement
- Sunset headers (RFC 8594)

#### E. API Security (EXTENSIVE COVERAGE)

**Authentication Mechanisms**:
- **Basic Authentication**:
  - Base64 encoding (NOT encryption)
  - When to use (internal services, development only)
  - Always require HTTPS
- **API Key Authentication**:
  - Header-based vs query parameter-based
  - Key management and rotation
  - Rate limiting per API key
- **OAuth 2.0** (DEEP DIVE):
  - **Authorisation Grant Types**:
    - Authorization Code (with PKCE for public clients)
    - Client Credentials (service-to-service)
    - Resource Owner Password Credentials (legacy, discouraged)
    - Implicit (deprecated, replaced by Authorization Code + PKCE)
    - Device Authorization Grant
  - **Token Types**: Access tokens, refresh tokens
  - **Token Formats**: Opaque vs JWT (self-contained)
  - **Scopes and Permissions**: Fine-grained access control
  - **Token Lifecycle**: Issuance, validation, refresh, revocation
  - **OAuth 2.0 Flows Diagrams**: Sequence diagrams for each grant type
- **JWT (JSON Web Tokens)**:
  - Structure: Header, Payload, Signature
  - Claims: Registered (iss, sub, aud, exp, nbf, iat, jti), public, private
  - Signing algorithms: HMAC (HS256), RSA (RS256), ECDSA (ES256)
  - JWT validation process
  - JWT vs opaque tokens trade-offs
  - JWT security best practices (short expiry, audience validation)
  - Common JWT vulnerabilities (alg:none, key confusion)
- **OpenID Connect (OIDC)**:
  - OAuth 2.0 extension for authentication
  - ID tokens vs access tokens
  - UserInfo endpoint
  - Standard scopes (openid, profile, email)
- **Mutual TLS (mTLS)**:
  - Client certificate authentication
  - Zero-trust architecture patterns
  - Certificate management in enterprise

**Authorisation Patterns**:
- **RBAC** (Role-Based Access Control)
- **ABAC** (Attribute-Based Access Control)
- **PBAC** (Policy-Based Access Control)
- **Scope-based access control** (OAuth 2.0 scopes)
- **Resource-level permissions**
- **Authorisation in enterprise banking**: Customer data access, regulatory compliance

**API Security Best Practices**:
- Input validation and sanitisation
- SQL injection prevention in APIs
- XSS prevention (output encoding)
- CSRF protection for APIs (SameSite cookies, tokens)
- Rate limiting and throttling
- IP allowlisting/blocklisting
- Request size limits
- HTTPS everywhere (TLS 1.2+ enforcement)
- CORS (Cross-Origin Resource Sharing) configuration
- Security headers
- API gateway security features
- OWASP API Security Top 10 (2023)
- Data masking and PII handling in banking APIs
- Audit logging for compliance (SOX, PCI-DSS)

#### F. REST API Implementation with Spring Boot (PRACTICAL)

**Spring MVC REST Controllers**:
- `@RestController` vs `@Controller` + `@ResponseBody`
- `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping`
- `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader`
- `@ResponseStatus`
- Request/response entity mapping
- Content negotiation configuration

**Exception Handling in Spring REST**:
- `@ExceptionHandler`
- `@ControllerAdvice` / `@RestControllerAdvice`
- `ResponseStatusException`
- Custom error response structures
- Global exception handling patterns
- Problem Details (RFC 7807) with Spring 6+
- Validation error handling (`@Valid`, `BindingResult`)

**Request Validation**:
- Bean Validation (JSR 380 / Jakarta Validation)
- `@Valid` and `@Validated`
- Built-in constraints (`@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Pattern`, `@Email`)
- Custom validators
- Cross-field validation
- Group validation
- Method-level validation

**Content Negotiation**:
- JSON (Jackson) configuration
  - Custom serialisers/deserialisers
  - `@JsonProperty`, `@JsonIgnore`, `@JsonFormat`
  - Date/time serialisation
  - Null handling strategies
  - Polymorphic type handling (`@JsonTypeInfo`)
- XML support (JAXB, Jackson XML)
- Custom media types
- `HttpMessageConverter` customisation

**Spring HATEOAS**:
- `RepresentationModel`, `EntityModel`, `CollectionModel`
- `WebMvcLinkBuilder`
- HAL format support
- Link relations

**Spring Data REST**:
- Repository-based REST endpoints
- Customisation and projections
- When to use vs custom controllers
- Limitations and trade-offs

**Reactive REST APIs (Spring WebFlux)**:
- Reactive vs imperative REST controllers
- `Mono` and `Flux` return types
- Non-blocking I/O benefits
- WebClient for consuming REST APIs
- When to choose WebFlux over MVC
- Performance implications

#### G. API Performance & Optimization (CRITICAL)

**Caching Strategies**:
- **HTTP Caching**:
  - Cache-Control directives (public, private, no-cache, no-store, max-age, s-maxage)
  - ETag-based caching (strong vs weak ETags)
  - Conditional requests (If-None-Match, If-Modified-Since)
  - Cache invalidation strategies
  - Vary header for content negotiation
- **Application-Level Caching**:
  - Redis/Memcached for API response caching
  - Spring Cache abstraction (`@Cacheable`, `@CacheEvict`, `@CachePut`)
  - Cache-aside, read-through, write-through, write-behind patterns
  - Cache warming strategies
  - TTL and eviction policies
- **CDN Caching**: Edge caching for API responses
- **Caching in banking APIs**: Trade-offs between freshness and performance

**Rate Limiting & Throttling**:
- Token bucket algorithm
- Leaky bucket algorithm
- Fixed window vs sliding window counters
- Rate limiting headers (X-RateLimit-*)
- Client-based vs API-based vs user-based limits
- Spring Boot rate limiting implementations
- API gateway rate limiting (Kong, Apigee, AWS API Gateway)
- Rate limiting in banking: Per-customer, per-API, per-partner

**Compression**:
- gzip and Brotli compression
- Content-Encoding and Accept-Encoding headers
- When to compress (text-based responses > threshold size)
- Spring Boot compression configuration

**Connection Management**:
- Connection pooling (HikariCP, Apache HttpClient pools)
- Keep-alive connections
- Timeouts: connect, read, write
- Circuit breaker pattern (Resilience4j)
- Retry strategies with exponential backoff
- Bulkhead pattern

**Asynchronous Processing**:
- Long-running operations pattern (202 Accepted + polling)
- Webhooks for event notification
- Server-Sent Events (SSE) for real-time updates
- WebSockets vs REST for real-time data
- Async request processing in Spring (`@Async`, `DeferredResult`, `CompletableFuture`)

**Payload Optimization**:
- Field selection/sparse fieldsets
- Response compression
- Pagination to limit response size
- JSON serialisation performance (Jackson vs Gson vs JSON-B)
- Protocol Buffers for internal services

#### H. API Testing (COMPREHENSIVE)

**Unit Testing REST APIs**:
- MockMvc for Spring MVC controllers
- `@WebMvcTest` for slice testing
- Mocking service layers
- Request/response verification
- JSON path assertions

**Integration Testing**:
- `@SpringBootTest` with `TestRestTemplate`
- `WebTestClient` for reactive APIs
- Test containers for database dependencies
- WireMock for external service simulation
- MockServer for HTTP mocking

**Contract Testing**:
- Consumer-Driven Contract Testing (CDC)
- Spring Cloud Contract
- Pact framework
- Contract testing in microservices architectures
- Provider vs consumer tests

**Performance Testing**:
- Load testing with JMeter, Gatling, k6
- Stress testing and breaking point identification
- Latency percentiles (p50, p95, p99)
- Throughput measurement
- Connection pool exhaustion testing
- API benchmark methodologies

**Security Testing**:
- OWASP ZAP for API security scanning
- Authentication/authorisation testing
- Fuzzing API endpoints
- SQL injection and XSS testing
- Rate limit testing

**API Testing Best Practices**:
- Test pyramid for APIs (unit > integration > E2E)
- Idempotency testing
- Boundary value testing
- Error scenario testing
- Test data management

#### I. API Documentation (IMPORTANT)

**OpenAPI Specification (Swagger)**:
- OpenAPI 3.0/3.1 specification structure
- Defining paths, operations, parameters, request bodies, responses
- Schema definitions and reusable components
- Security scheme definitions
- Example values and descriptions
- Code generation from OpenAPI specs (server stubs, client SDKs)

**Spring Integration**:
- SpringDoc OpenAPI (springdoc-openapi-starter-webmvc-ui)
- Swagger annotations (`@Operation`, `@Schema`, `@Parameter`, `@ApiResponse`)
- Generating OpenAPI specs from code (code-first approach)
- Generating code from OpenAPI specs (API-first/design-first approach)
- API-first vs code-first trade-offs
- Customising Swagger UI

**Documentation Best Practices**:
- API-first design methodology
- Example requests and responses
- Error response documentation
- Authentication/authorisation documentation
- Rate limiting documentation
- Changelog and versioning notes
- SDK and client library documentation

#### J. API Gateway & Microservices Patterns (ARCHITECTURE)

**API Gateway Pattern**:
- What is an API Gateway and why it's needed
- Gateway responsibilities: routing, composition, protocol translation
- API Gateway vs reverse proxy vs load balancer
- Popular gateways: Spring Cloud Gateway, Kong, Apigee, AWS API Gateway, Azure API Management
- Gateway patterns: routing, aggregation, offloading

**Microservices Communication Patterns**:
- Synchronous REST vs asynchronous messaging
- Service discovery (Eureka, Consul, Kubernetes)
- Client-side vs server-side load balancing
- Circuit breaker pattern (Resilience4j, Hystrix legacy)
- Retry and timeout patterns
- Saga pattern for distributed transactions
- API composition/aggregation pattern

**Backend for Frontend (BFF) Pattern**:
- Separate API layers for different clients (web, mobile, third-party)
- API tailoring and aggregation
- When BFF makes sense vs over-engineering

**Service Mesh**:
- Istio, Linkerd for service-to-service communication
- mTLS between services
- Traffic management and observability
- Sidecar proxy pattern

#### K. API Observability & Monitoring

**Logging**:
- Structured logging for APIs (JSON format)
- Correlation IDs and distributed tracing
- Request/response logging (sanitising sensitive data)
- Log levels for API operations
- ELK stack (Elasticsearch, Logstash, Kibana) integration

**Metrics**:
- RED metrics: Rate, Errors, Duration
- API-specific metrics: latency percentiles, error rates, throughput
- Micrometer + Prometheus integration
- Grafana dashboards for API health
- Custom business metrics (transactions per second, etc.)

**Distributed Tracing**:
- OpenTelemetry for distributed tracing
- Trace context propagation (W3C Trace Context)
- Jaeger, Zipkin for trace visualisation
- Span creation and context propagation in Spring Boot

**Health Checks and Readiness**:
- Spring Boot Actuator endpoints
- Liveness vs readiness probes
- Custom health indicators
- Dependency health checks

#### L. Advanced REST API Topics

**Idempotency**:
- Why idempotency matters in distributed systems
- Idempotency keys (Idempotency-Key header)
- Implementation patterns (database-backed, cache-backed)
- Idempotency in payment processing (critical for banking)
- At-least-once vs exactly-once semantics

**Concurrency Control**:
- Optimistic locking with ETags (If-Match header)
- Pessimistic locking patterns
- Conflict resolution strategies (409 Conflict)
- Last-write-wins vs first-write-wins
- Optimistic concurrency in banking transactions

**Bulk/Batch Operations**:
- Batch request patterns (single endpoint for multiple operations)
- Async batch processing with job status endpoints
- Partial success handling
- Transaction boundaries in batch operations

**File Upload/Download APIs**:
- Multipart file uploads
- Streaming large files
- Resumable uploads
- Pre-signed URLs (S3 pattern)
- Content-Disposition header

**Webhooks**:
- Webhook registration and management
- Webhook delivery guarantees
- Retry policies for failed deliveries
- Webhook security (HMAC signatures, mutual TLS)
- Webhook vs polling trade-offs

**API Evolution and Backward Compatibility**:
- Additive changes (safe)
- Breaking changes and how to avoid them
- Tolerant reader pattern
- Robustness principle (Postel's Law)
- Consumer-driven contracts

**REST vs Alternatives**:
- **GraphQL**: Query flexibility, over-fetching/under-fetching, N+1 problem
- **gRPC**: Binary protocol, streaming, code generation, protobuf
- **WebSockets**: Full-duplex communication, real-time
- **Server-Sent Events**: One-way real-time updates
- **Message Queues** (Kafka, RabbitMQ): Async communication
- When to choose REST vs alternatives decision matrix

#### M. Enterprise Banking API Patterns

**Open Banking APIs**:
- PSD2 (Payment Services Directive 2) compliance
- Open Banking standards (UK Open Banking, Berlin Group)
- Account Information Service (AIS)
- Payment Initiation Service (PIS)
- Strong Customer Authentication (SCA)

**Banking API Design Patterns**:
- Account inquiry APIs
- Payment processing APIs (single, bulk, scheduled)
- Transaction history APIs (filtering, pagination, date ranges)
- Reference data APIs (currency codes, country codes, branch data)
- Regulatory reporting APIs

**Compliance and Regulatory Requirements**:
- PCI-DSS compliance for payment APIs
- GDPR and data privacy in APIs (data minimisation, right to erasure)
- SOX compliance for financial data APIs
- Audit trail requirements
- Data residency and sovereignty
- KYC (Know Your Customer) API flows

**Enterprise Integration Patterns**:
- API-led connectivity
- System of Record (SOR) integration
- Legacy system wrapping (anti-corruption layer)
- Event-driven architecture with REST triggers
- CQRS (Command Query Responsibility Segregation) with REST

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
- Step-by-step explanations
- Internal implementation details where relevant

## Visual Representations (MANDATORY)
- Architecture diagrams using Mermaid
- Sequence diagrams for API flows
- Flowcharts for decision processes
- Comparison matrices (tables)
- Request/response flow diagrams

## Code Examples (MANDATORY for Spring Boot topics)
- Minimal, focused code snippets (Java/Spring Boot)
- Heavy annotations explaining the "why"
- Show both correct and incorrect patterns
- Real-world enterprise examples (banking context)
- Include curl/httpie command examples for API calls

## Interview Questions & Model Answers
- 10-15 common interview questions for this topic
- Detailed answers explaining the concept
- Follow-up questions interviewers might ask
- How to structure your answer in an interview
- Gotcha questions and tricky scenarios

## Real-World Enterprise Scenarios
- How this applies in banking/financial services
- Performance implications in production
- When to use each approach
- Integration with microservices architecture
- Regulatory implications where applicable

## Common Pitfalls & Best Practices
- What NOT to do
- Production war stories (lessons learned)
- Performance anti-patterns
- Security anti-patterns
- Debugging tips
- Code review red flags

## Comparison Tables
- When to use X vs Y
- Trade-offs between approaches
- Performance characteristics comparison
- Security implications comparison

## Key Takeaways (Bullet Points)
- 5-7 critical points to remember
- Facts and numbers
- Quick reference for last-minute review

## Further Reading
- Official specification links (RFCs, OpenAPI)
- Spring Framework documentation
- OWASP API Security resources
- Enterprise architecture resources
- Authoritative blog posts and books
```

### 3. Research Requirements

**YOU MUST**:
1. **Search official specifications and RFCs**:
   - RFC 7230-7237 (HTTP/1.1)
   - RFC 9110-9114 (HTTP Semantics, HTTP/2, HTTP/3)
   - RFC 6749 (OAuth 2.0)
   - RFC 7519 (JWT)
   - RFC 7807 (Problem Details for HTTP APIs)
   - RFC 8288 (Web Linking)
   - RFC 8594 (Sunset Header)
   - RFC 6902 (JSON Patch)
   - RFC 7396 (JSON Merge Patch)
   - OpenAPI Specification 3.0/3.1

2. **Reference authoritative sources**:
   - "RESTful Web APIs" by Leonard Richardson & Mike Amundsen
   - "API Design Patterns" by JJ Geewax (Manning)
   - "REST API Design Rulebook" by Mark Massé (O'Reilly)
   - Roy Fielding's original dissertation
   - Spring Framework official documentation
   - OWASP API Security Top 10
   - Martin Fowler's articles on REST, microservices
   - Richardson Maturity Model (Martin Fowler's article)

3. **Include version-specific information**:
   - Spring Boot 2.x vs 3.x differences
   - Jakarta EE vs Java EE namespace migration
   - HTTP/1.1 vs HTTP/2 vs HTTP/3 evolution
   - OpenAPI 2.0 (Swagger) vs 3.0 vs 3.1

4. **Provide factual information**:
   - RFC numbers and dates
   - HTTP status code ranges and semantics
   - Performance benchmarks where available
   - Security standards compliance details

### 4. Special Requirements

#### Diagrams
- **MUST create Mermaid diagrams** for:
  - REST architectural constraints overview
  - Richardson Maturity Model levels
  - HTTP request/response lifecycle
  - OAuth 2.0 flow (all grant types)
  - JWT structure and validation flow
  - API Gateway architecture
  - Microservices communication patterns
  - Caching strategy decision tree
  - Rate limiting algorithms
  - Circuit breaker state machine
  - API versioning strategy comparison
  - Pagination strategy comparison
  - API testing pyramid
  - CORS preflight flow
  - Webhook delivery flow
  - Open Banking API flow

#### Interview Focus
- After each major section, include "What Interviewers Look For"
- Provide both surface-level and deep technical answers
- Show how to explain complex topics simply (whiteboard-friendly explanations)
- Include follow-up questions and how to handle them
- Cover tricky interview scenarios (e.g., "Design an API for a payment system")
- Include system design questions involving REST APIs

#### Enterprise Banking Context
- Relate concepts to high-volume transaction processing
- Discuss API security in the context of financial regulations
- Payment processing API design patterns
- Account management API patterns
- Real-time vs batch processing API patterns
- Multi-tenant API design for different business units
- SLA requirements for banking APIs (99.99% uptime)
- Disaster recovery and API failover patterns

### 5. Structure and Organization

Create the guide as separate, well-organized markdown files:

```
rest-apis/
├── 01-rest-fundamentals-and-principles.md
├── 02-http-protocol-deep-dive.md
├── 03-api-design-best-practices.md
├── 04-api-versioning-strategies.md
├── 05-api-security-authentication-authorisation.md
├── 06-spring-boot-rest-implementation.md
├── 07-api-performance-and-caching.md
├── 08-api-testing-strategies.md
├── 09-api-documentation-openapi.md
├── 10-api-gateway-and-microservices.md
├── 11-api-observability-and-monitoring.md
├── 12-advanced-rest-topics.md
├── 13-enterprise-banking-api-patterns.md
├── 14-rest-vs-alternatives.md
├── 15-rest-api-interview-master-guide.md
└── 16-api-design-cheat-sheet.md
```

The master guide (15-rest-api-interview-master-guide.md) should have this structure:

```markdown
# REST APIs - Complete Interview Preparation Guide

## Table of Contents
[Detailed TOC with links to all sections]

## How to Use This Guide
- Study path for interview preparation
- Quick reference sections
- Topic prioritisation for banking interviews

## Part 1: REST Foundations
### 1.1 REST Architectural Style and Constraints
### 1.2 Richardson Maturity Model
### 1.3 Resources and Representations
### 1.4 Content Negotiation

## Part 2: HTTP Protocol Mastery
### 2.1 HTTP Methods (Verbs) - Properties and Usage
### 2.2 HTTP Status Codes - Complete Reference
### 2.3 HTTP Headers - Essential Knowledge
### 2.4 HTTP/1.1 vs HTTP/2 vs HTTP/3

## Part 3: API Design Mastery
### 3.1 URI Design Principles
### 3.2 Request/Response Design
### 3.3 Error Handling Design (RFC 7807)
### 3.4 Pagination Strategies
### 3.5 Filtering, Sorting, and Searching
### 3.6 HATEOAS and Hypermedia

## Part 4: API Versioning
### 4.1 Versioning Approaches Comparison
### 4.2 Backward Compatibility
### 4.3 API Lifecycle Management
### 4.4 Deprecation and Sunset Strategies

## Part 5: API Security
### 5.1 Authentication Mechanisms (Basic, API Key, OAuth 2.0, JWT)
### 5.2 OAuth 2.0 Deep Dive (All Grant Types)
### 5.3 JWT Structure, Validation, and Security
### 5.4 Authorisation Patterns (RBAC, ABAC)
### 5.5 API Security Best Practices (OWASP Top 10)
### 5.6 CORS Configuration
### 5.7 Banking-Specific Security (PCI-DSS, GDPR)

## Part 6: Spring Boot REST Implementation
### 6.1 REST Controllers and Annotations
### 6.2 Exception Handling (@ControllerAdvice)
### 6.3 Request Validation (Bean Validation)
### 6.4 Content Negotiation and Jackson Configuration
### 6.5 Spring HATEOAS
### 6.6 Reactive REST APIs (WebFlux)

## Part 7: API Performance
### 7.1 HTTP Caching Strategies
### 7.2 Application-Level Caching
### 7.3 Rate Limiting and Throttling
### 7.4 Compression and Payload Optimization
### 7.5 Async Processing Patterns
### 7.6 Connection Management and Resilience

## Part 8: API Testing
### 8.1 Unit Testing with MockMvc
### 8.2 Integration Testing
### 8.3 Contract Testing (CDC, Pact)
### 8.4 Performance Testing
### 8.5 Security Testing

## Part 9: API Documentation
### 9.1 OpenAPI Specification (3.0/3.1)
### 9.2 SpringDoc Integration
### 9.3 API-First vs Code-First Design

## Part 10: API Architecture Patterns
### 10.1 API Gateway Pattern
### 10.2 Microservices Communication
### 10.3 BFF (Backend for Frontend) Pattern
### 10.4 Service Mesh

## Part 11: API Observability
### 11.1 Structured Logging and Correlation IDs
### 11.2 Metrics (RED, Micrometer, Prometheus)
### 11.3 Distributed Tracing (OpenTelemetry)
### 11.4 Health Checks and Actuator

## Part 12: Advanced Topics
### 12.1 Idempotency in Distributed Systems
### 12.2 Concurrency Control (Optimistic Locking)
### 12.3 Bulk/Batch Operations
### 12.4 Webhooks
### 12.5 API Evolution and Backward Compatibility

## Part 13: Enterprise Banking API Patterns
### 13.1 Open Banking APIs (PSD2)
### 13.2 Payment Processing APIs
### 13.3 Regulatory Compliance (PCI-DSS, GDPR, SOX)
### 13.4 Enterprise Integration Patterns

## Part 14: REST vs Alternatives
### 14.1 REST vs GraphQL
### 14.2 REST vs gRPC
### 14.3 REST vs WebSockets
### 14.4 Technology Selection Decision Matrix

## Part 15: Interview Preparation
### 15.1 Top 100 REST API Interview Questions
### 15.2 System Design Questions Involving APIs
### 15.3 API Design Exercise: Payment System
### 15.4 Troubleshooting Scenarios
### 15.5 How to Discuss REST Topics in Interviews

## Appendices
### Appendix A: HTTP Status Code Quick Reference
### Appendix B: HTTP Header Quick Reference
### Appendix C: REST API Design Checklist
### Appendix D: API Security Checklist
### Appendix E: Common REST Anti-Patterns
### Appendix F: RFC Quick Reference
### Appendix G: Recommended Reading and Resources
```

### 6. Quality Standards

The guide MUST be:
- ✅ **Comprehensive**: Cover 100% of REST API interview topics
- ✅ **Accurate**: All technical details verified against RFCs and official specs
- ✅ **Visual**: At least 30+ diagrams throughout
- ✅ **Interview-ready**: Every section has interview Q&A
- ✅ **Enterprise-focused**: Real-world banking/financial context throughout
- ✅ **Security-focused**: Deep coverage of API security (critical for banking)
- ✅ **Practical**: Include Spring Boot code examples and curl commands
- ✅ **Up-to-date**: Cover latest Spring Boot 3.x, HTTP/3, OpenAPI 3.1
- ✅ **Deep**: Go beyond surface-level explanations into internals

### 7. Expected Length

This should be a **substantial, comprehensive guide**:
- Estimated length: 25,000-35,000 words across all files
- 30-40+ Mermaid diagrams
- 150+ interview questions with detailed answers
- Multiple code examples per major section (Java/Spring Boot)
- curl/httpie command examples for all API patterns
- Comparison tables for key decision points
- Security checklists and best practice summaries

### 8. Success Criteria

When complete, a reader should be able to:
1. Design RESTful APIs following industry best practices and REST constraints
2. Implement secure, performant REST APIs using Spring Boot
3. Choose appropriate authentication/authorisation mechanisms for different scenarios
4. Design pagination, filtering, and versioning strategies with clear justification
5. Answer any REST API question in a technical interview with confidence
6. Debug and troubleshoot REST API issues in production
7. Design banking-specific APIs compliant with regulatory requirements
8. Compare REST with alternative technologies and justify technology choices
9. Implement comprehensive API testing strategies
10. Design API architectures for microservices with proper gateway, security, and observability patterns

## Additional Instructions

- Use British English spellings where applicable
- Include RFC numbers for all referenced specifications
- Cite sources in footnotes or references section
- If certain information is not available or outdated, clearly state this
- When discussing deprecated approaches, explain why and what replaced them
- Include both theoretical knowledge and practical code examples
- Show curl command examples alongside Java/Spring Boot code
- Include request/response examples for every API pattern
- Focus on security throughout (critical for banking interviews)
- Differentiate between what's "truly RESTful" vs "pragmatic REST"

## Interview Question Categories to Include

For each major topic, include questions in these categories:
1. **Foundational**: Basic concepts (e.g., "What are the HTTP methods?")
2. **Intermediate**: Practical application (e.g., "How would you handle pagination?")
3. **Advanced**: Deep technical understanding (e.g., "Explain the difference between 401 and 403")
4. **Tricky**: Edge cases and gotchas (e.g., "Is PUT always idempotent? What about auto-generated fields?")
5. **Scenario-based**: Real-world problem solving (e.g., "Design a payment processing API")
6. **Architecture-level**: System design involving APIs (e.g., "Design an API gateway for a banking platform")

## Code Example Requirements

Every code example MUST:
- Be compilable and runnable (Spring Boot 3.x compatible)
- Include import statements if not obvious
- Have inline comments explaining the "why"
- Show both what works and common mistakes
- Include corresponding curl commands for testing
- Use realistic variable names (banking domain: accounts, transactions, payments)
- Follow Spring Boot best practices
- Include proper error handling

## Begin!

Start by researching official RFCs (HTTP, OAuth 2.0, JWT), OWASP API Security guidelines, Spring Framework documentation, and authoritative REST API design resources. Then create the comprehensive guide following the structure and requirements above.

**Remember**: This is for interview preparation at senior/principal level in banking - depth, accuracy, and security awareness are critical. Don't skip details. Every section should be thorough enough that the reader could design, build, secure, and scale production-grade REST APIs for enterprise banking systems and ace technical interviews at top financial institutions.

Focus on:
- **Why** REST design decisions matter
- **When** to use different approaches (with trade-offs)
- **How** to implement securely in Spring Boot
- **Security** implications at every layer
- **Enterprise** patterns for banking/financial services
- **Performance** at scale (millions of transactions)
- **Interview** strategies and model answers
