# REST APIs - Complete Interview Preparation Guide

## Overview

Welcome to the **Master Preparation Guide for REST APIs**. This resource is explicitly tailored for Senior, Staff, and Principal Engineers preparing for grueling technical interviews and system design rounds at Tier-1 tech environments and global financial institutions.

This guide transcends memorizing standard HTTP verbs. It is architected to rapidly instill the ability to defend complex design decisions, debate performance trade-offs under high concurrency, enforce unyielding security topologies, and design enterprise microservice networks successfully.

---

## Table of Contents

### Part 1: Foundations & Architecture
- [01. REST Fundamentals & Architectural Principles](./01-rest-fundamentals-and-principles.md)
  - Core Constraints (Statelessness, Uniform Interface, Layered System), Richardson Maturity Model.
- [02. HTTP Protocol Deep Dive](./02-http-protocol-deep-dive.md)
  - Method Idempotency vs Safety, Granular Status Codes (409, 412, 422), Protocol Evolution (HTTP/2 & HTTP/3).

### Part 2: Implementation & Design
- [03. API Design Best Practices](./03-api-design-best-practices.md)
  - Proper URI Naming, Pagination comparisons (Offset vs Cursor), Advanced Filtering.
- [04. API Versioning Strategies](./04-api-versioning-strategies.md)
  - Path vs Header versioning, Sunset semantics, and Deprecation RFCs.
- [06. Spring Boot REST Implementation](./06-spring-boot-rest-implementation.md)
  - `@RestController` Internals, Global Exception Handling (RFC 7807), Bean Validation, Reactive WebFlux.

### Part 3: Security & Performance
- [05. API Security: Authentication & Authorisation](./05-api-security-authentication-authorisation.md)
  - OAuth 2.0 PKCE, Deep-dive JWT Vulnerabilities, Mutual TLS (mTLS), OWASP API Top 10.
- [07. API Performance & Optimization](./07-api-performance-and-caching.md)
  - `Cache-Control`, ETag Conditional Revalidation, Rate Limiting Algorithms, Connection Pool resilience.
- [11. API Observability & Monitoring](./11-api-observability-and-monitoring.md)
  - RED Metrics, Distributed Tracing (Correlation/Trace IDs), OpenTelemetry, Log Masking Compliance.

### Part 4: Advanced Validation & Networking
- [08. API Testing Strategies](./08-api-testing-strategies.md)
  - The Shift-Left Paradigm, Consumer-Driven Contract Testing (Pact), TestContainers, WireMock.
- [09. API Documentation (OpenAPI)](./09-api-documentation-openapi.md)
  - OpenAPI Spec 3.x, API-First Design patterns, Auto-generation limits.
- [10. API Gateway & Microservices Patterns](./10-api-gateway-and-microservices.md)
  - BFF (Backend-For-Frontend) topology, Resilience4j (Circuit Breakers/Retries), Service Mesh offloading.
- [12. Advanced REST Topics](./12-advanced-rest-topics.md)
  - Implementing rigorous Idempotency Keys, Optimistic DB Locking, Asynchronous Bulk Operations (202 Accepted).

### Part 5: Specialized Architectural Paradigms
- [13. Enterprise Banking API Patterns](./13-enterprise-banking-api-patterns.md)
  - Open Banking (PSD2), Anti-Corruption Layers integrating legacy Mainframes, Audit Ledgers.
- [14. REST vs. Alternatives](./14-rest-vs-alternatives.md)
  - Pragmatic benchmarking between REST, GraphQL, gRPC, and WebSockets.
- [16. REST API Design Cheat Sheet](./16-api-design-cheat-sheet.md)
  - High-velocity summary sheet for rapid pre-interview recall.

---

## How to Utilize This Guide for Preparation

### 1. Master "The Why"
Architects do not interview for syntax recall. Interviewers will present a flawed design and ask you to critique it. Focus acutely on the `"Common Pitfalls & Best Practices"` and `"Real-World Enterprise Scenarios"` sections embedded within each chapter.

### 2. Whiteboard Emulation
The provided **Mermaid Diagrams** graphically synthesize complex flows (e.g., The OAuth2 Authorization Code sequence, Idempotency Flows, Circuit Breaker states). Practice replicating these logic flows manually; articulating a solution visually immediately establishes Senior credibility.

### 3. Interview Prioritization
If time constraints apply, ruthlessly prioritize mastering these domains first:
1.  **Security Architecture** (OAuth2/JWT/mTLS) - Section 05.
2.  **System Reliability & Idempotency** - Sections 10 & 12.
3.  **Optimization at Scale** - Section 07.
4.  **Protocol Semantics** - Section 02.

---

## The "Critical Core 10" Interview Questions

Regardless of the organization, you are intensely likely to encounter variations of these defining questions:

1.  **Protocol Knowledge**: Why is returning an HTTP `200 OK` regarding a validation failure an anti-pattern? Elaborate on the differences between `400`, `401`, `403`, and `422`.
2.  **Scale & Locking**: If two automated systems simultaneously update a user's ledger, how do you prevent data pollution utilizing solely HTTP Headers and Optimistic Controls?
3.  **Systems Integrity**: Describe precisely how to implement mathematical Idempotency using HTTP Headers to ensure a retried POST request never executes a duplicate payment.
4.  **Security Modeling**: Explain the Algorithm Confusion vulnerability affecting JWTs, and why standard OAuth 2.0 Bearer tokens are occasionally deemed inadequate for High-Value enterprise APIs.
5.  **Microservice Topology**: Discuss the concrete operational differences between deploying an API Edge Gateway versus integrating a Service Mesh (Sidecar Proxy).
6.  **Alternative Scoping**: Under what specific architectural or client-side conditions would you actively advocate to abandon REST and integrate an internal GraphQL server? When would you mandate gRPC?
7.  **Asynchronous Execution**: How do you architect a REST endpoint tasked with parsing an uploaded CSV resulting in 20 minutes of continuous server processing without timing out load-balancer connections?
8.  **Resilience**: Walk through the exact state transitions of the Circuit Breaker pattern. Explain the "Retry Storm" phenomenon.
9.  **Scale Pagination**: Why does standard database `OFFSET / LIMIT` pagination catastrophically degrade when rendering the 500th page of a dataset, and what modern pattern resolves this?
10. **Versioning Evolution**: What is the safest implementation strategy to execute a Breaking Change to a highly-consumed external corporate API without disrupting active native mobile-app consumers?

---

*Proceed to [Chapter 1: Foundations & Architecture](./01-rest-fundamentals-and-principles.md) to begin the curriculum.*
