# System Design - Complete Staff/Principal Engineer Interview Preparation Guide

Welcome to the definitive guide for mastering System Design interviews at the Staff, Principal, and Senior Engineering levels, with a specialized focus on highly regulated, high-throughput environments like Enterprise Banking, FinTech, and Trading platforms.

This guide moves beyond theoretical basics to deliver production-hardened architectural patterns, capacity planning (back-of-the-envelope math), advanced data modeling, and massive distributed systems case studies. 

## How to Use This Guide

*   **Study Path**: Read the files sequentially from Foundations (01) through Observability (16). These form the strict technical vocabulary and primitives needed to succeed.
*   **The Case Studies (17-24)**: These are masterclasses in applying the primitives. Do not memorize these designs; memorize the *trade-offs* made to achieve specific Non-Functional Requirements.
*   **Advanced Patterns (25-27)**: These files contain the "silver bullets" for answering extremely complex whiteboard questions (e.g., CQRS, Sagas, Active-Active Failovers).
*   **Interview Prompt**: For any question, always start with: (1) Clarify Requirements, (2) Back-of-the-Envelope Math, (3) High-Level Architecture, (4) Deep Dive / Bottleneck Analysis.

---

## Complete Table of Contents

### Part 1: Foundations & Distributed Systems Core
Understand the physics of networks, CAP theorem trade-offs, and how to estimate scale.

*   [**01. System Design Fundamentals**](./01-system-design-fundamentals.md)
    *   The Interview Framework (Requirements, Estimates, Design, Deep Dive).
    *   Back-of-the-Envelope Math (Latency numbers every programmer should know).
*   [**02. Distributed Systems Fundamentals**](./02-distributed-systems-fundamentals.md)
    *   CAP Theorem and PACELC.
    *   Network Partitions and The Fallacies of Distributed Computing.
*   [**03. Consistency and Consensus**](./03-consistency-and-consensus.md)
    *   Eventual vs. Strong Consistency.
    *   Quorum, Split-Brain, and Consensus algorithms (Raft, Paxos).
    *   Distributed Transactions (2PC vs. Sagas).

### Part 2: Scalability, Infrastructure & Data
How to scale out, balance traffic, cache aggressively, and store terabytes of structured/unstructured data.

*   [**04. Scalability Patterns**](./04-scalability-patterns.md)
    *   Horizontal Scaling vs. Vertical Scaling.
    *   Consistent Hashing and Load Balancing algorithms (L4 vs. L7).
*   [**05. Caching Strategies**](./05-caching-strategies.md)
    *   Read-Through, Write-Through, Write-Behind.
    *   Cache Invalidation, Cache Stampedes, and Distributed Locks (Redis).
*   [**06. Database Design and Storage**](./06-database-design-and-storage.md)
    *   Relational DBs, ACID compliance, and SQL Isolation Levels.
    *   B-Tree Indexing internals and Sharding strategies.
*   [**07. NoSQL and NewSQL Databases**](./07-nosql-and-newsql.md)
    *   Key-Value (Redis), Document (Mongo), Wide-Column (Cassandra/SSTables).
    *   Graph DBs (Neo4j) and NewSQL (Google Spanner / CockroachDB).

### Part 3: Asynchronous Architecture & APIs
Coupling, messaging, and exposing clean programmatic interfaces.

*   [**08. Messaging and Event-Driven Architecture**](./08-messaging-and-event-driven-architecture.md)
    *   Message Queues (RabbitMQ) vs. Event Streams (Kafka).
    *   Event Sourcing, CQRS, and the Transactional Outbox pattern.
*   [**09. Apache Kafka Deep Dive**](./09-kafka-deep-dive.md)
    *   Topics, Partitions, Consumer Groups, and Offset Management.
    *   Exactly-Once Semantics and High-Availability configurations.
*   [**10. API Design and Communication**](./10-api-design-and-communication.md)
    *   REST vs. GraphQL vs. gRPC (Protocol Buffers).
    *   WebSockets, Server-Sent Events (SSE).
    *   Idempotency implementation and API Gateways.

### Part 4: Microservices, Cloud-Native & Containerization
Building robust, stateless, disposable services in modern hybrid clouds.

*   [**11. Microservices Architecture**](./11-microservices-architecture.md)
    *   Monoliths to Microservices, Domain-Driven Design (DDD).
    *   Bounded Contexts and Strangler Fig Migration Pattern.
    *   Database-per-Service.
*   [**12. Microservices Patterns and Resilience**](./12-microservices-patterns-and-resilience.md)
    *   Circuit Breakers, Bulkheads, Retries (Exponential Backoff + Jitter).
    *   Blue-Green and Canary Deployment Strategies.
    *   Service Meshes (Istio) and the Sidecar Pattern.
*   [**13. Cloud-Native Architecture**](./13-cloud-native-architecture.md)
    *   The Twelve-Factor App and Infrastructure as Code (Terraform).
    *   GitOps CI/CD Pipelines.
    *   BFF (Backend-For-Frontend) and Anti-Corruption Layers.
*   [**14. Kubernetes and Container Orchestration**](./14-kubernetes-and-container-orchestration.md)
    *   Control Plane vs. Data Plane.
    *   Pods, Deployments vs. StatefulSets, DaemonSets.
    *   Services (ClusterIP, NodePort) and Ingress Controllers.
    *   Liveness/Readiness Probes and Resource Limits preventing noisy neighbors.

### Part 5: Enterprise Security & Observability
Securing the perimeter and making 500 microservices debuggable.

*   [**15. Security Architecture**](./15-security-architecture.md)
    *   Zero Trust Architecture and mTLS.
    *   OAuth 2.0, OIDC, and JWT Token Management.
    *   Envelope Encryption (KMS) and PCI-DSS compliance.
*   [**16. Observability and Monitoring**](./16-observability-and-monitoring.md)
    *   The Three Pillars: Logs (ELK), Metrics (Prometheus/Grafana), Tracing (Jaeger/OTel).
    *   RED/USE Methodologies and Service Level Objectives (SLOs).

### Part 6: Staff-Level Case Studies
Applying the foundations to massive real-world problems.

*   [**17. Case Study: URL Shortener & Rate Limiter**](./17-case-study-url-shortener-and-rate-limiter.md)
    *   Base62 offline key generation and Token Bucket API protection.
*   [**18. Case Study: Chat Systems (WhatsApp) & Notifications**](./18-case-study-chat-and-notification-systems.md)
    *   Persistent Websockets, Presence mapping (Redis), and massive Async Fanout.
*   [**19. Case Study: Newsfeed (Instagram) & Distributed Search**](./19-case-study-newsfeed-and-search.md)
    *   Push vs. Pull Fanout (The Celebrity Problem) and Elasticsearch Inverted Indices.
*   [**20. Case Study: Distributed Cache & Unique ID Generator**](./20-case-study-distributed-cache-and-id-generator.md)
    *   Twitter Snowflake bitwise generation, Consistent Hashing Rings, and Bloom Filters.
*   [**21. Case Study: Payment Processing System**](./21-case-study-payment-processing-system.md)
    *   Strict Idempotency, Sagas, avoiding database deadlocks, and Double-Entry Ledgers.
*   [**22. Case Study: Fraud Detection System**](./22-case-study-fraud-detection-system.md)
    *   Real-time stream aggregation (Flink), Graph DB identity mapping (Neo4j), and ML Model Serving.
*   [**23. Case Study: Trading Platform & Matching Engine**](./23-case-study-trading-platform.md)
    *   Microsecond in-memory limit order books, LMAX Disruptor, and Event Sourcing fault tolerance.
*   [**24. Case Study: Banking Core Systems (Ledgers)**](./24-case-study-banking-core-systems.md)
    *   Asynchronous balance projections (CQRS), processing millions of End-of-Day interest calculations (Spark/MapReduce).

### Part 7: The Masterclass
Architectural patterns that definitively separate Senior developers from Staff/Principal leaders.

*   [**25. Advanced Distributed Patterns**](./25-advanced-distributed-patterns.md)
    *   Deep dives into CQRS, Sagas vs 2PC, Event Sourcing, and Transactional Outboxes.
*   [**26. Performance and Reliability**](./26-performance-and-reliability.md)
    *   Database connection pooling, thread exhaustion, Jitter, and Graceful Degradation.
*   [**27. Disaster Recovery and Multi-Region Architectures**](./27-disaster-recovery-and-multi-region.md)
    *   RPO/RTO calculation, Active-Active split-brain avoidance (Quorum), and cross-ocean Kafka replication.

---
## Final Interview Day Checklist

1.  **Stop and Clarify**: Never start drawing boxes immediately. Ask 3-5 clarification questions bounding the read/write load and target audience.
2.  **State your Assumptions loudly**: "I am assuming a 100:1 read-to-write ratio, giving us roughly 10,000 QPS."
3.  **Drive the Conversation**: A Staff engineer leads the meeting. Say "I will start with the API edge, then move to data storage, and finish with the asynchronous workers. Does that sound good?"
4.  **Embrace Trade-offs**: There are no perfect designs. Every time you choose Cassandra, you must explicitly state that you are sacrificing relational JOINs for blazing-fast writes. Every time you choose Kafka async fanout, you must state you accept eventual consistency.
5.  **Look for Bottlenecks**: Before the interviewer asks, proactively point out, "The weakest link here is the single Postgres writer node. If we cross 5,000 TPS, we will need to shard this by UserID."
