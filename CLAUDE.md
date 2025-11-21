# Interview Preparation Repository - Claude Code Guide

## Project Overview

This repository is a comprehensive interview preparation resource for a Senior Java Developer with 13+ years of enterprise banking experience. The focus is on creating a **knowledge base with minimal code but maximum conceptual clarity** through explanations, diagrams, and visual aids.

## Primary Focus Areas

### Core Technologies
- **Java** (13+ years experience, enterprise-level patterns)
- **Spring Framework** (Spring Boot, Spring Security, Spring Data, Spring Cloud)
- **Cloud Platforms** (Azure - currently pursuing AZ-204, AWS - certified)
- **Apache Kafka** (messaging, event-driven architecture, log compaction)
- **Data Structures & Algorithms** (interview-focused problems and solutions)

### Enterprise Context
- Banking/Financial services domain knowledge
- Microservices architecture patterns
- API management (Apigee)
- Database technologies (PostgreSQL, Azure PostgreSQL Flexible Server)
- Enterprise authentication (SiteMinder, LDAP, OAuth2, JWT)
- Cloud migration strategies (PCF to Azure)
- Clean Architecture (ports and adapters pattern)

## Repository Philosophy

### What This Repo IS
- ✅ Interview preparation guide with deep conceptual explanations
- ✅ Documentation-first approach with diagrams and visual aids
- ✅ Real-world examples from enterprise banking environments
- ✅ Comprehensive coverage of "why" and "how" for each topic
- ✅ Structured learning path for interview scenarios
- ✅ Reference architecture patterns and best practices

### What This Repo IS NOT
- ❌ Full production codebases or complete applications
- ❌ Copy-paste code snippets without context
- ❌ Generic tutorials available elsewhere
- ❌ Boilerplate-heavy implementations

## Content Structure Guidelines

When creating or organizing content in this repository, follow these principles:

### 1. Documentation Format
Each topic should include:

```
# Topic Name

## Overview
- Brief 2-3 sentence description
- Why this matters in interviews
- Real-world use cases

## Key Concepts
- Fundamental principles explained clearly
- Common misconceptions addressed
- Enterprise considerations

## Technical Deep Dive
- Detailed explanations with examples
- Architecture diagrams (Mermaid, PlantUML, or ASCII)
- Sequence diagrams for flows
- Code snippets (minimal, focused on concept illustration)

## Interview Questions & Answers
- Common interview questions
- How to approach answering them
- Key points to emphasize

## Gotchas & Best Practices
- Common pitfalls
- Production considerations
- Performance implications

## References
- Official documentation links
- Relevant articles or papers
```

### 2. Code Guidelines
When code is necessary:
- **Minimal**: Only what's needed to illustrate the concept
- **Annotated**: Heavy comments explaining the "why"
- **Realistic**: Based on enterprise patterns, not toy examples
- **Compilable**: Should be runnable if someone wants to test

### 3. Diagram Expectations
Prefer visual explanations:
- **Mermaid diagrams** for flowcharts, sequence diagrams, and system architecture
- **PlantUML** for detailed UML diagrams
- **ASCII art** for simple illustrations
- **Markdown tables** for comparisons

## Topic Coverage

### Java & JVM
- Core Java concepts (collections, concurrency, streams, generics)
- JVM internals (memory model, GC, classloading)
- Design patterns (especially enterprise patterns)
- Multithreading and concurrent programming
- Performance optimization

### Spring Ecosystem
- Spring Boot (autoconfiguration, starters, profiles)
- Spring Security (authentication flows, JWT, OAuth2, RBAC)
- Spring Data JPA (repositories, queries, transactions)
- Spring Cloud (config, discovery, circuit breakers)
- Testing (JUnit, Mockito, integration tests)

### Cloud & Infrastructure
- Azure services (App Service, PostgreSQL, Key Vault, UAMI, Private Endpoints)
- AWS fundamentals
- Infrastructure as Code (Ansible, Terraform concepts)
- Containerization (Docker basics)
- CI/CD patterns

### Kafka & Messaging
- Kafka architecture and internals
- Producers, consumers, and consumer groups
- Topic design and partitioning strategies
- Log compaction
- Exactly-once semantics
- Integration patterns with Spring

### Databases
- PostgreSQL internals and optimization
- Liquibase/Flyway migration strategies
- Connection pooling
- Transaction management
- Azure PostgreSQL Flexible Server specifics

### System Design
- Microservices patterns
- API design (REST, versioning, pagination)
- Authentication and authorization architectures
- Event-driven architecture
- Distributed systems concepts (CAP theorem, eventual consistency)
- Observability (OpenTelemetry, Micrometer, logging strategies)

### Data Structures & Algorithms
- Common interview patterns
- Time/space complexity analysis
- Problem-solving approaches
- Java-specific implementations

## How Claude Code Should Assist

### When Creating New Content
1. **Ask clarifying questions** about the specific topic and interview angle
2. **Propose a structure** based on the guidelines above
3. **Create diagrams** using Mermaid or PlantUML when helpful
4. **Provide enterprise context** - relate to banking/financial services when relevant
5. **Include real-world scenarios** from my experience (UBS, JP Morgan contexts)

### When Explaining Concepts
- Start with fundamentals, then go deeper
- Use analogies related to banking/financial systems when helpful
- Highlight what interviewers are really looking for
- Compare different approaches with pros/cons
- Reference official documentation and authoritative sources

### When Writing Code Examples
- Keep it minimal and focused
- Use Spring Boot 3.x+ patterns
- Include JavaDoc comments explaining the concept
- Show enterprise-grade patterns (not just "Hello World")
- Demonstrate clean architecture principles when relevant

### When Creating Diagrams
- Use Mermaid for most diagrams (natively supported in Markdown)
- Create sequence diagrams for authentication flows, API interactions
- Create architecture diagrams for system design questions
- Use class diagrams for design pattern explanations
- Include component diagrams for microservices architecture

## Interview Preparation Context

### Target Roles
- Senior Java Developer
- Staff Engineer / Principal Engineer
- Architect (Solution/Application)
- Technical Lead positions in enterprise banking

### Interview Types to Prepare For
1. **Technical Depth**: Deep Java, Spring, and Cloud knowledge
2. **System Design**: Microservices, distributed systems, scalability
3. **Problem Solving**: DSA, algorithmic thinking
4. **Experience-Based**: Past projects, architectural decisions, tradeoffs
5. **Domain Knowledge**: Banking systems, compliance, security

### Communication Style
- I prefer understanding **foundational concepts** over memorization
- I value **why** something works, not just how
- I appreciate **visual representations** of complex systems
- I like **real-world examples** that relate to my experience

## Current Focus Areas

### High Priority
- Azure certification (AZ-204) preparation
- Spring Boot 3.x patterns and best practices
- Kafka architecture and design patterns
- System design for banking applications
- Clean architecture implementation

### Medium Priority
- Data structures and algorithms practice
- Design patterns in enterprise context
- PostgreSQL optimization
- Microservices patterns

## Project Organization

```
/
├── java/
│   ├── core/           # Core Java concepts
│   ├── concurrency/    # Multithreading, concurrent collections
│   └── jvm/            # JVM internals, GC, performance
├── spring/
│   ├── boot/           # Spring Boot specifics
│   ├── security/       # Authentication, authorization
│   ├── data/           # Spring Data JPA
│   └── cloud/          # Spring Cloud components
├── cloud/
│   ├── azure/          # Azure services and patterns
│   └── aws/            # AWS fundamentals
├── kafka/              # Kafka architecture and patterns
├── databases/          # Database concepts and optimization
├── system-design/      # Architecture patterns, distributed systems
├── algorithms/         # DSA problems and solutions
└── diagrams/           # Standalone diagrams and architecture docs
```

## Examples of What Good Content Looks Like

### Good Example - Kafka Log Compaction
```markdown
# Kafka Log Compaction

## Overview
Log compaction in Kafka ensures that consumers can restore state by 
replaying messages, keeping only the latest value for each key. Critical 
for building event-sourced systems and maintaining compacted changelog topics.

## When to Use
- Maintaining current state of entities (e.g., contract status)
- Database CDC (Change Data Capture) use cases
- Microservices needing to rebuild state from event log

## How It Works
[Mermaid diagram showing before/after compaction]

## Key Configuration
- cleanup.policy=compact
- min.cleanable.dirty.ratio
- segment.ms considerations

## Interview Points
- Explain difference between deletion and compaction
- When would you choose compaction over time-based retention?
- How does this affect consumer design?
...
```

### Poor Example - Generic Tutorial
```markdown
# Introduction to Kafka

Kafka is a distributed streaming platform. Here's how to install it:
[generic installation steps...]
```

## Summary

This repository serves as my personalized interview preparation guide. Claude Code should help me build a comprehensive, visual, and conceptually deep knowledge base that reflects enterprise-level understanding of Java, Spring, Cloud technologies, and system design - all contextualized for senior engineering roles in banking and financial services.

**Remember**: Maximum explanation, minimum code. Every topic should be interview-ready with clear explanations, diagrams, and real-world context.
- dont create unncecssary files the progress summary md files.. you only need to create the interview guide that is enough
- Be Concise - Focus on interview-ready content, not exhaustive reference material (6,000-8,000 words max,
  not 20,000+)
- Minimal Code - Only essential examples with heavy annotations explaining "why", not full implementations
- NO Multiple Agents - Work on ONE file at a time, complete it, then move to the next
- Direct Work - Create files directly myself instead of launching Task agents that may fail or timeout
- Token Efficiency - Keep responses focused and complete tasks within reasonable token limits
- One Task at a Time - Finish file #1 completely before starting file #2