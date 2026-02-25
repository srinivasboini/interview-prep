# API Design and Communication Protocols

## Overview

In modern distributed systems, the Application Programming Interface (API) is the foundational contract between services, and between the backend and the outside world. Poor API design creates tight coupling, makes versioning a nightmare, and exposes internal implementation details to clients. Furthermore, using the wrong communication protocol (e.g., using HTTP/1.1 REST for a real-time trading ticker instead of WebSockets) can cripple system performance.

For a Staff/Principal Engineer, API design is not just about choosing verbs like `GET` or `POST`. It involves high-level architectural decisions regarding synchronous vs. asynchronous communication, dealing with partial failures, and managing API Gateways at the edge of the network. In an enterprise banking environment, your APIs must be strictly versioned, highly secure, and—most importantly—**Idempotent**, ensuring that network retries do not result in double-charging a customer's account.

Interviewers will expect you to comfortably contrast REST, GraphQL, and gRPC, specifically highlighting the performance characteristics, payload sizes, and connection management trade-offs of each.

## Foundational Concepts

### RESTful API Design Principles

Representational State Transfer (REST) is the architectural style of the web. 
*   **Resources**: Everything is a noun (e.g., `/accounts`, `/transactions`).
*   **Statelessness**: The server stores no client context between requests. Every request must contain all the information necessary to understand it (e.g., an Auth token). This is what enables horizontal scaling of API servers.
*   **HTTP Verbs**:
    *   `GET` (Read, Idempotent, Safe)
    *   `POST` (Create, Not Idempotent)
    *   `PUT` (Update entire resource, Idempotent)
    *   `PATCH` (Partial update, typically Not Idempotent, though can be designed to be)
    *   `DELETE` (Remove, Idempotent)
*   **HATEOAS**: (Hypermedia as the Engine of Application State). Responses include links to related actions to guide the client. Rarely implemented fully in practice due to complexity, but prized in academia.

### The Problem with REST (Why GraphQL and gRPC exist)

1.  **Over-fetching**: A mobile app needs a user's name and balance. A `GET /users/123` REST call returns the name, balance, address, transaction history, and 50 other fields (bloating the payload and killing mobile data plans).
2.  **Under-fetching (N+1 Problem)**: To render a dashboard, the client must call `GET /users/123`, wait, then call `GET /users/123/accounts`, wait, then call `GET /accounts/456/transactions`. Multiple network round-trips destroy latency.
3.  **JSON Serialization Overhead**: Parsing human-readable JSON strings into memory objects is computationally expensive for high-throughput backend-to-backend communication.

## Technical Deep Dive: Communication Protocols

### 1. GraphQL
Developed by Facebook to solve the over/under-fetching problems for mobile clients.
*   **How it works**: The client sends a specific query string detailing *exactly* which fields it wants. The server responds with only those fields in a single round-trip.
*   **Strengths**: Extreme flexibility for frontend teams. Eliminates over-fetching. A single endpoint (`/graphql`) replaces dozens of REST endpoints.
*   **Weaknesses**: Caching is brutally hard (you can't cache based on the URL anymore). Malicious or complex queries can DoS the database (e.g., a query asking for User -> Friends -> Friends -> Friends). Requires complex rate-limiting based on query cost analysis.
*   **Banking Use Case**: BFF (Backend-for-Frontend) layers aggregating multiple internal microservices into a single, tailored response for the mobile banking app.

### 2. gRPC
Developed by Google. A high-performance Remote Procedure Call (RPC) framework.
*   **How it works**: Uses Protocol Buffers (protobuf) instead of JSON. Protobuf is a strongly typed, strictly schematized binary format. Uses HTTP/2 underneath, enabling multiplexing (sending multiple requests concurrently over a single TCP connection).
*   **Strengths**: Blazing fast. Payload sizes are drastically smaller than JSON (no field names transmitted, just binary values based on position). Strongly typed contracts ensure backward compatibility. Supports unary (1:1), server-streaming, client-streaming, and bidirectional streaming.
*   **Weaknesses**: Not human-readable (debugging requires tools to deserialize). Browser support is poor (requires gRPC-Web proxies).
*   **Banking Use Case**: Internal service-to-service communication. e.g., The Payment API calling the Fraud Check microservice synchronously.

### 3. WebSockets and Server-Sent Events (SSE)
Standard HTTP is client-pull (the client must ask for data). In finance, we often need server-push (the server pushes data down when an event happens).
*   **WebSockets**: A persistent, bidirectional, full-duplex TCP connection. The client and server can send messages to each other at any time. Ideal for low-latency live trading platforms or chat apps. Hard to scale because load balancers must maintain millions of open connections, heavily consuming memory.
*   **Server-Sent Events (SSE)**: A persistent, unidirectional connection (Server -> Client) over standard HTTP. Ideal for live dashboards or notifications where the client only needs to receive updates, but doesn't need to stream data rapidly back to the server. Easier to scale and route than WebSockets.

### API Gateways

An API Gateway sits between external clients and your internal microservices. In an enterprise, exposing 50 internal microservices directly to the public internet is a security and operational disaster.

**Gateway Responsibilities:**
*   **Routing**: `/api/users` goes to Service A; `/api/payments` goes to Service B.
*   **Authentication/Authorization**: Validating JWT tokens or OIDC flows at the edge, rejecting unauthenticated traffic before it hits internal networks.
*   **Rate Limiting / Throttling**: Protecting backend databases from DDoS or abusive scraping.
*   **SSL Termination**: Decrypting HTTPS traffic at the edge, reducing CPU load on internal services (internally, traffic might be HTTP or mTLS).
*   **Aggregation (BFF)**: Calling three internal services in parallel and merging them into one JSON response for the client.

## Visual Representations

### REST vs GraphQL vs gRPC

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#E3F2FD', 'edgeLabelBackground':'#FFF'}}}%%
flowchart LR
    classDef client fill:#E1BEE7,stroke:#8E24AA,stroke-width:2px;
    classDef rest fill:#C8E6C9,stroke:#388E3C,stroke-width:2px;
    classDef graph fill:#FFCCBC,stroke:#E64A19,stroke-width:2px;
    classDef grpc fill:#B3E5FC,stroke:#0288D1,stroke-width:2px;

    Client1[Mobile App]:::client
    Client2[Web App]:::client
    Client3[Internal Svc A]:::client

    subgraph REST (JSON over HTTP/1.1)
        REST_API[3 Round-trips \n 1. /users/123 \n 2. /accounts/45 \n 3. /txns]:::rest
    end

    subgraph GraphQL (JSON over HTTP/1.1)
        Graph_API[1 Round-trip \n POST /graphql \n {user{id, accounts{txns}}}]:::graph
    end

    subgraph gRPC (Protobuf over HTTP/2)
        gRPC_API[Multiplexed Stream \n Binary Payload \n (Microsecond Latency)]:::grpc
    end

    Client1 -.->|Under/Over fetching| REST_API
    Client2 -.->|Tailored Payload| Graph_API
    Client3 -.->|High Throughput Backend| gRPC_API
```

### API Gateway Architecture (Enterprise Banking)

```mermaid
flowchart TD
    Client[Mobile/Web Client]
    
    subgraph DMZ (Public Facing)
        WAF[Web Application Firewall \n (DDoS Protection)]
        Gateway[API Gateway \n Kong / Azure APIM]
    end
    
    subgraph Internal Network (Private)
        AuthSvc[Identity/Auth Service \n (OAuth2 / OIDC)]
        UserSvc[User Microservice]
        PaySvc[Payment Microservice]
        LegacyCore[(Legacy Mainframe)]
    end

    Client -->|HTTPS| WAF
    WAF --> Gateway
    
    Gateway -->|1. Validate JWT Token| AuthSvc
    AuthSvc -->|2. HTTP 200 OK| Gateway
    
    Gateway -->|3. Route /users| UserSvc
    Gateway -->|3. Route /payments| PaySvc
    
    PaySvc -->|gRPC/TCP| LegacyCore
    
    %% Gateway Policies
    note right of Gateway: Gateway Policies:<br/>1. Rate Limiting (100 req/min)<br/>2. SSL Termination<br/>3. IP Whitelisting<br/>4. Request Logging
```

## Code/Configuration Examples

### Practical Idempotency Implementation (REST API)

In banking, network timeouts happen. The client must retry. The server must handle the retry without charging the client twice. 

```java
@RestController
@RequestMapping("/v1/transfers")
public class TransferController {

    private final TransferService transferService;

    // The client MUST generate a unique UUID and send it in this header
    @PostMapping
    public ResponseEntity<TransferResponse> createTransfer(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody TransferRequest request) {

        // 1. Check if we've seen this key before (Using Redis or DB)
        Optional<TransferResponse> existingResponse = transferService.checkIdempotency(idempotencyKey);
        
        if (existingResponse.isPresent()) {
            // 2. Return the EXACT SAME historical response. Do not execute logic.
            return ResponseEntity.ok(existingResponse.get());
        }

        // 3. Store the key in 'PENDING' state to prevent concurrent stampedes
        boolean lockAcquired = transferService.lockKey(idempotencyKey);
        if (!lockAcquired) {
            return ResponseEntity.status(409).body("Transfer already in progress");
        }

        try {
            // 4. Execute the business logic (ACID Transaction)
            TransferResponse newResponse = transferService.executeTransfer(request);
            
            // 5. Store the final successful response against the key permanently
            transferService.storeIdempotentResponse(idempotencyKey, newResponse);
            
            return ResponseEntity.status(201).body(newResponse);
        } catch (Exception e) {
            // 6. If it fails (e.g., insufficient funds), store the failure or unlock
            transferService.unlockKey(idempotencyKey);
            throw e;
        }
    }
}
```

### Protocol Buffers (gRPC) Schema Example

Notice there are no field names in the binary wire transmission. The schema defines the position (`= 1`, `= 2`). This is why it is so much smaller and faster than JSON.

```protobuf
syntax = "proto3";
package bank.payments.v1;

// The RPC Service Definition
service PaymentService {
  // Unary (1:1) synchronous call
  rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
  
  // Server Streaming (e.g., exporting a monthly statement report row by row)
  rpc StreamStatement (StatementRequest) returns (stream Transaction);
}

// Strongly typed messages
message PaymentRequest {
  string idempotency_key = 1;
  string source_account_id = 2;
  string target_account_id = 3;
  double amount = 4;
  string currency_code = 5;
}

message PaymentResponse {
  string transaction_id = 1;
  enum Status {
    UNKNOWN = 0;
    SUCCESS = 1;
    INSUFFICIENT_FUNDS = 2;
    BLOCKED_BY_FRAUD = 3;
  }
  Status status = 2;
}
```

## Interview Questions & Model Answers

**Q1: Contrast REST and gRPC. When would you choose one over the other in a microservices architecture?**
*Answer*: I would use REST (JSON over HTTP/1.1 or HTTP/2) for any public-facing API. It is universally understood, easy to debug in a browser, and deeply integrated into standard load balancers, caching proxies, and WAFs. 
However, for internal service-to-service communication behind the firewall, REST is incredibly inefficient due to JSON serialization overhead and the connection management limits of HTTP/1.1. In this case, I would strictly use gRPC. gRPC uses Protobuf (a highly compressed binary format) over HTTP/2, which allows multiplexing dozens of concurrent requests over a single, persistent TCP connection. It drastically lowers CPU serialization costs and network egress costs (crucial in AWS/Azure). Its strict schema also prevents teams from accidentally breaking internal API contracts.

**Q2: How do you handle pagination in a RESTful API returning millions of transaction records?**
*Answer*: 
1. **Offset Pagination** (`?limit=100&offset=500`): The simplest to implement, but terrible for performance at scale. `OFFSET 1000000` requires the database to scan and discard 1 million rows before returning the 100 requested. It also suffers from data drift (if a new record is inserted while the user pages, they might see duplicates or miss items).
2. **Keyset (Cursor) Pagination**: The enterprise standard. The client passes the ID (or timestamp) of the last row they saw (`?limit=100&after_id=A987`). The database query uses a highly optimized index lookup (`SELECT * FROM txns WHERE id > 'A987' ORDER BY id ASC LIMIT 100`). This is infinitely fast, regardless of the page depth, and prevents data drift. I would implement Keyset pagination and return a `next_cursor` token in the API response payload or Link header.

**Q3: We are building a live dashboard showing the real-time stock price of Apple to a million users. Should we use REST polling, WebSockets, or Server-Sent Events (SSE)?**
*Answer*: REST polling (the client asking every 1 second) is an anti-pattern here; it will crush the backend with millions of empty requests and tear down/rebuild millions of TCP handshakes. WebSockets provide full-duplex communication, but they are overkill because the user is only *receiving* data, not actively sending high-frequency messages back. Furthermore, load balancers struggle with holding a million persistent WebSocket connections open.
The architectural optimum here is **Server-Sent Events (SSE)**. It uses a standard HTTP connection but keeps it open, allowing the server to push a unidirectional stream of string data down to the browser. It natively supports automatic reconnection (unlike WebSockets) and is far easier to load balance and proxy through standard enterprise firewalls.

**Q4: Explain the API Gateway pattern. What problems does it solve, and what is the biggest risk of introducing one?**
*Answer*: An API Gateway acts as a reverse proxy and the single entry point for all external traffic. It solves cross-cutting concerns: instead of implementing JWT validation, SSL termination, and rate limiting in 50 different microservices, we centralize it in the gateway (e.g., Kong). It also allows routing (masking internal microservice boundaries from the client). 
The biggest risk is that it becomes a Single Point of Failure (SPOF) and a development bottleneck. If the gateway logic becomes too complex (e.g., writing heavy custom Lua or JavaScript transformations inside the gateway), it degrades into a "Distributed Monolith" governed by a dedicated API Gateway team that slows down all other feature teams. Gateways should remain as "dumb" network edge routers.

## Real-World Enterprise Scenarios

**Scenario: API Versioning for an Open Banking Platform**
*   **Context**: Under PSD2 / Open Banking regulations, banks must expose APIs to third-party fintechs (like Plaid or Mint) to access user account data. 
*   **Problem**: You have 5,000 external companies using `GET /accounts`. You need to change the response payload to include a new, mandatory regulatory array, which breaks the existing JSON schema. You cannot force 5,000 external companies to deploy code on the same day.
*   **Strategy**: You must use strict API versioning.
    *   **URI Versioning**: `GET /v1/accounts` remains live. You launch `GET /v2/accounts`. 
    *   **Implementation**: At the API Gateway, `/v1/` routes to the old version of the microservice, and `/v2/` routes to the new. Or, if you want a single codebase, the API Gateway inspects the `Accept: application/vnd.bank.v2+json` header and routes accordingly. You give third parties a 12-month deprecation window, monitor usage of V1, and forcefully turn it off once traffic drops to zero (or the deadline hits).

## Common Pitfalls & Best Practices

**Pitfalls:**
*   **HTTP Status Code Abuse**: Returning a `200 OK` but putting `{ status: "error", code: 500, message: "Database Down" }` in the JSON body. This breaks all standard monitoring tools, load balancers, and circuit breakers which rely on inspecting HTTP status headers to determine system health. Use standard codes (`4xx` for client error, `5xx` for server error).
*   **Ignoring Idempotency on POST**: In a distributed system, network timeouts are standard. If an API is not idempotent, a client retrying a timed-out request will cause disastrous duplicate writes.

**Best Practices:**
*   **Rate Limit by Identity, Not Just IP**: In banking, a malicious script might cycle through thousands of VPN IPs to brute-force a login endpoint. Rate limit at the API Gateway based on the `Authorization` token header or extracted User ID, in addition to IP-based WAF protections.
*   **Strict Security at the Edge**: Never pass a JWT token deep into the internal network if it isn't necessary. The API Gateway should validate the JWT, extract the User ID, and pass a simple internal header (e.g., `X-Internal-User-Id: 123`) to the backend microservice over an mTLS-secured internal channel.

## Comparison Tables

| Feature | REST | GraphQL | gRPC |
| :--- | :--- | :--- | :--- |
| **Payload Format** | JSON / XML (Human Readable) | JSON (Human Readable) | Protobuf (Binary, Unreadable) |
| **Transport Protocol**| HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 | Strictly HTTP/2 (Multiplexed) |
| **Contract / Schema**| Optional (OpenAPI/Swagger) | Mandatory (GraphQL Schema) | Mandatory (.proto files) |
| **Coupling** | Loose | Very Loose (Client drives) | Tight (RPC paradigm) |
| **Caching** | Excellent (Native HTTP caching) | Very Difficult (Always POST) | Poor (Requires custom logic) |

| Communication Type | Protocol Choice | Rationale |
| :--- | :--- | :--- |
| **Public API (B2B)** | REST (JSON) | Ubiquitous, easiest for clients to integrate with. |
| **Mobile App (BFF)** | GraphQL | Mobile bandwidth is precious; eliminate over-fetching. |
| **Internal S2S** | gRPC (Protobuf) | Extreme low-latency, schema enforcement, CPU efficiency. |
| **Live Ticker UI** | Server-Sent Events (SSE)| Unidirectional server push; easy to scale via load balancers. |

## Key Takeaways

*   **Idempotency is king**: Every state-mutating API endpoint (`POST`, `PUT`, `PATCH`) in a financial system must be designed to safely handle network retries using an Idempotency-Key.
*   **Use the right tool**: Don't use REST for everything. Adopt gRPC for internal latency-sensitive microservices, and GraphQL/BFFs to optimize mobile payloads.
*   **Pagination matters**: Never use OFFSET pagination for large datasets; always default to cursor/keyset pagination to preserve database performance.
*   **Protect the Edge**: The API Gateway is the shield of the enterprise. Implement rate limiting, auth validation, and WAF rules there to keep internal services lightweight.

## Further Reading
*   [REST API Design Rulebook (O'Reilly)](https://www.oreilly.com/library/view/rest-api-design/9781449317904/)
*   [Stripe's API Design (The gold standard for Idempotency and Versioning)](https://stripe.com/docs/api)
*   [gRPC vs REST: Understanding gRPC, OpenAPI and REST](https://cloud.google.com/blog/products/api-management/understanding-grpc-openapi-and-rest)
*   [Understanding API Gateways vs Service Mesh](https://konghq.com/learning-center/service-mesh/api-gateway-vs-service-mesh)
