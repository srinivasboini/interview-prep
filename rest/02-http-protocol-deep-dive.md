# HTTP Protocol Deep Dive

## Overview

HTTP (Hypertext Transfer Protocol) is the application-layer protocol underpinning all REST APIs. Despite being "just a transport layer" to many developers, HTTP has rich semantics — methods, status codes, headers, caching directives — that are the primary tools of API design.

Deep HTTP knowledge is a critical differentiator at Staff/Principal Engineer interviews. Every API design decision maps back to HTTP semantics: idempotency to PUT/DELETE, resource creation to POST+Location, caching to ETag+Cache-Control. Banking APIs with 99.99% SLA targets depend entirely on correct HTTP semantics in proxies, CDNs, and load balancers.

---

## HTTP Methods (Verbs)

### Properties Matrix

| Method | Safe | Idempotent | Cacheable | Request Body | Response Body |
|---|---|---|---|---|---|
| GET | ✅ | ✅ | ✅ | ❌ (undefined) | ✅ |
| HEAD | ✅ | ✅ | ✅ | ❌ | ❌ (headers only) |
| OPTIONS | ✅ | ✅ | ❌ | ❌ | ✅ |
| POST | ❌ | ❌ | Conditional | ✅ | ✅ |
| PUT | ❌ | ✅ | ❌ | ✅ | ✅ |
| PATCH | ❌ | ❌ | ❌ | ✅ | ✅ |
| DELETE | ❌ | ✅ | ❌ | ❌ | Optional |
| TRACE | ✅ | ✅ | ❌ | ❌ | ✅ |

**Safe**: Does not modify server state (read-only). Clients may call safe methods freely.
**Idempotent**: Multiple identical calls produce the same result. Critical for retry logic.
**Cacheable**: Responses eligible for caching by proxies and browsers.

### Method Details

#### GET
- **Purpose**: Retrieve a resource representation
- **Idempotent**: Yes — calling 100 times doesn't change anything
- **Safe**: Yes — read-only operation
- **Response**: 200 OK with body; 404 Not Found; 304 Not Modified (conditional)
- **Anti-pattern**: `GET /accounts?action=delete` — use DELETE instead

```http
GET /api/v1/accounts/ACC-001?fields=id,balance,currency HTTP/1.1
Authorization: Bearer eyJ...
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private, max-age=0
ETag: "v1-50000-gbp"
{"id":"ACC-001","balance":50000.00,"currency":"GBP"}
```

#### POST
- **Purpose**: Create a new resource (or trigger an action)
- **Idempotent**: No — multiple calls create multiple resources
- **Safe**: No — modifies server state
- **Response**: 201 Created + Location header (resource creation); 200 OK (action); 202 Accepted (async)
- **Note**: Use `Idempotency-Key` header for payment POST requests to prevent duplicate processing

```http
POST /api/v1/accounts HTTP/1.1
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{"customerId":"CUST-123","accountType":"SAVINGS","currency":"GBP"}

HTTP/1.1 201 Created
Location: /api/v1/accounts/ACC-002
Content-Type: application/json

{"id":"ACC-002","customerId":"CUST-123","status":"PENDING_VERIFICATION"}
```

#### PUT
- **Purpose**: Full resource replacement (create-or-update)
- **Idempotent**: Yes — PUT same body twice = same result
- **Safe**: No
- **Response**: 200 OK (updated with body); 204 No Content (updated, no body); 201 Created (if resource was created)
- **Tricky interview question**: "Is PUT always idempotent?" — Not if the server generates values (like `updatedAt` timestamp) but semantically it should be.

```http
PUT /api/v1/accounts/ACC-001 HTTP/1.1
Content-Type: application/json
If-Match: "v1-50000-gbp"  ← Optimistic locking: only update if ETag matches

{"accountType":"PREMIUM_SAVINGS","currency":"GBP","overdraftLimit":5000}

HTTP/1.1 200 OK
ETag: "v2-50000-gbp-premium"
{"id":"ACC-001","accountType":"PREMIUM_SAVINGS","overdraftLimit":5000}
```

#### PATCH
- **Purpose**: Partial resource update (only send changed fields)
- **Idempotent**: Not necessarily (depends on semantics — increment operations are not idempotent)
- **Two patch formats**:
  - **JSON Merge Patch (RFC 7396)**: Send only changed fields; `null` removes a field
  - **JSON Patch (RFC 6902)**: Array of operations (add, remove, replace, copy, move, test)

```http
# JSON Merge Patch (RFC 7396) — simpler, send only changed fields
PATCH /api/v1/accounts/ACC-001 HTTP/1.1
Content-Type: application/merge-patch+json

{"overdraftLimit": 10000}  ← Only changed field; other fields unchanged

# JSON Patch (RFC 6902) — precise operations
PATCH /api/v1/accounts/ACC-001 HTTP/1.1
Content-Type: application/json-patch+json

[
  {"op":"replace","path":"/overdraftLimit","value":10000},
  {"op":"add","path":"/tags/-","value":"premium-customer"},
  {"op":"test","path":"/currency","value":"GBP"}  ← Assert currency is GBP before patching
]
```

#### DELETE
- **Purpose**: Remove a resource
- **Idempotent**: Yes — deleting an already-deleted resource returns 404 (or 204), same as before
- **Response**: 204 No Content (most common); 200 OK with deletion confirmation body
- **Soft delete pattern**: Change status to CLOSED rather than physical deletion (banking audit trail)

#### HEAD
- **Purpose**: Same as GET but returns only headers, no body
- **Use cases**: Check if resource exists (before downloading), get Content-Length, validate ETag
- **Banking**: Check if a large PDF statement is available before downloading

#### OPTIONS
- **Purpose**: Discover allowed methods on a resource; CORS preflight
- **Response**: `Allow: GET, POST, PUT, DELETE` header
- **CORS**: Browser sends OPTIONS preflight before cross-origin requests

### Method Selection Decision Tree

```mermaid
flowchart TD
    A{What are you doing?} --> B{Retrieving data?}
    B --> |Yes| C{Need body?}
    C --> |No| D[HEAD]
    C --> |Yes| E[GET]
    B --> |No| F{Creating new resource?}
    F --> |Yes| G[POST → 201 Created + Location]
    F --> |No| H{Full or partial update?}
    H --> |Full replacement| I[PUT → 200/204]
    H --> |Partial update| J[PATCH]
    H --> |No, deleting| K[DELETE → 204]
```

---

## HTTP Status Codes

### 2xx Success

| Code | Name | When to Use |
|---|---|---|
| 200 | OK | General success: GET, PUT, PATCH |
| 201 | Created | Resource created: POST. Include Location header |
| 202 | Accepted | Async processing: payment initiation, batch jobs |
| 204 | No Content | Success, no response body: DELETE, PUT (no body) |
| 206 | Partial Content | Range requests (file streaming) |

**Common mistake**: Returning 200 OK for resource creation. Always use 201 + Location for POST.

### 3xx Redirection

| Code | Name | Method Preserved? | When to Use |
|---|---|---|---|
| 301 | Moved Permanently | ❌ Often changes to GET | Permanent URI change |
| 302 | Found | ❌ Often changes to GET | Temporary redirect (avoid for APIs) |
| 304 | Not Modified | — | Conditional GET: resource not changed |
| 307 | Temporary Redirect | ✅ | Temporary redirect, preserves POST/PUT |
| 308 | Permanent Redirect | ✅ | Permanent redirect, preserves POST/PUT |

**Interview insight**: 301/302 historically change POST to GET (browser behavior). For APIs, use 307/308 to preserve method semantics.

### 4xx Client Errors

| Code | Name | When to Use |
|---|---|---|
| 400 | Bad Request | Malformed request syntax, invalid JSON |
| 401 | Unauthorized | Authentication missing or invalid |
| 403 | Forbidden | Authenticated but not authorised |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported on this endpoint |
| 406 | Not Acceptable | Cannot produce requested media type |
| 408 | Request Timeout | Client took too long to send request |
| 409 | Conflict | State conflict: duplicate creation, optimistic lock failure |
| 410 | Gone | Resource permanently removed |
| 412 | Precondition Failed | If-Match or If-None-Match header mismatch |
| 415 | Unsupported Media Type | Request Content-Type not supported |
| 422 | Unprocessable Entity | Syntactically valid but semantically invalid |
| 429 | Too Many Requests | Rate limit exceeded |

**Critical distinction**: 401 vs 403
- **401**: "Who are you?" — authentication missing/expired
- **403**: "I know who you are, but you can't do this" — authorisation denied
- Tip: 401 must include `WWW-Authenticate` header per RFC 9110

**422 vs 400**:
- **400**: Malformed JSON, missing required Content-Type header (can't parse)
- **422**: Valid JSON structure but business rule violation (transfer amount exceeds balance)

### 5xx Server Errors

| Code | Name | When to Use |
|---|---|---|
| 500 | Internal Server Error | Unexpected server error — NullPointerException, etc. |
| 502 | Bad Gateway | Upstream service returned invalid response |
| 503 | Service Unavailable | Server overloaded, maintenance, circuit breaker open |
| 504 | Gateway Timeout | Upstream service timeout |

**Banking note**: 503 with `Retry-After` header for planned maintenance windows or circuit breaker trips.

### Status Code Decision Flowchart

```mermaid
flowchart TD
    A{Request received} --> B{Auth valid?}
    B --> |No auth| C[401 Unauthorized]
    B --> |No permission| D[403 Forbidden]
    B --> |Yes| E{Resource found?}
    E --> |No| F[404 Not Found]
    E --> |Gone| G[410 Gone]
    E --> |Yes| H{Method valid?}
    H --> |Method not allowed| I[405 Method Not Allowed]
    H --> |Acceptable| J{Request valid?}
    J --> |Malformed| K[400 Bad Request]
    J --> |Business invalid| L[422 Unprocessable Entity]
    J --> |Conflict| M[409 Conflict]
    J --> |Valid| N{Process success?}
    N --> |Created| O[201 Created]
    N --> |Updated/Retrieved| P[200 OK]
    N --> |Deleted| Q[204 No Content]
    N --> |Async accepted| R[202 Accepted]
    N --> |Server error| S[500 Internal Server Error]
```

---

## HTTP Headers

### Request Headers

```http
GET /api/v1/accounts/ACC-001 HTTP/1.1
Host: api.bank.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...    ← JWT bearer token
Accept: application/json                           ← Desired response format
Accept-Encoding: gzip, br                         ← Compression support
Accept-Language: en-GB                            ← Content language preference
Content-Type: application/json                    ← Request body format (for POST/PUT/PATCH)
If-None-Match: "abc123"                           ← Conditional: only return if changed
If-Modified-Since: Wed, 01 Jan 2025 00:00:00 GMT ← Conditional: only if modified since
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000  ← Distributed tracing
X-Correlation-ID: order-123-payment-456           ← Business correlation
Idempotency-Key: payment-550e8400-e29b-41d4      ← Deduplication key for POST
```

### Response Headers

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 842
Content-Encoding: gzip                            ← Compressed response
Cache-Control: private, max-age=0                 ← Caching directive
ETag: "v2-acc001-50000gbp"                       ← Resource version identifier
Last-Modified: Tue, 24 Feb 2026 10:00:00 GMT     ← Last modification time
Location: /api/v1/accounts/ACC-002               ← Only for 201/3xx responses
X-RateLimit-Limit: 1000                          ← Max requests per window
X-RateLimit-Remaining: 987                       ← Remaining in window
X-RateLimit-Reset: 1740528000                    ← Unix timestamp when resets
Retry-After: 60                                   ← Seconds to wait (429/503)
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000  ← Echo back for tracing
```

### Security Headers

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
  ← HSTS: Force HTTPS for 1 year. Never send over HTTP.

Content-Security-Policy: default-src 'self'; frame-ancestors 'none'
  ← Prevent XSS/clickjacking for any web-rendered API responses

X-Content-Type-Options: nosniff
  ← Prevent MIME type sniffing (e.g., serving JSON rendered as HTML)

X-Frame-Options: DENY
  ← Prevent embedding in iframes (clickjacking)

X-XSS-Protection: 0
  ← Set to 0 (disable legacy XSS filter — it had bypass vulnerabilities; CSP replaces it)

Access-Control-Allow-Origin: https://banking-app.example.com
  ← CORS: only allow specific origins

Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type, X-Request-ID
Access-Control-Max-Age: 600          ← Cache preflight for 10 minutes
```

### ETag and Conditional Requests

ETags (Entity Tags) enable efficient caching and optimistic concurrency control.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: GET /api/v1/accounts/ACC-001
    S-->>C: 200 OK<br/>ETag: "v1-50000"<br/>{balance: 50000}

    Note over C: Client caches response with ETag

    C->>S: GET /api/v1/accounts/ACC-001<br/>If-None-Match: "v1-50000"
    S-->>C: 304 Not Modified (no body!)
    Note over C: Client uses cached response — saves bandwidth

    C->>S: PUT /api/v1/accounts/ACC-001<br/>If-Match: "v1-50000"<br/>{overdraftLimit: 5000}
    S-->>C: 200 OK<br/>ETag: "v2-50000-od5000"
    Note over S: Update only if ETag matches — prevents lost updates
```

**Strong ETag**: `"abc123"` — byte-for-byte identical
**Weak ETag**: `W/"abc123"` — semantically equivalent (minor differences like whitespace)

---

## HTTP/1.1 vs HTTP/2 vs HTTP/3

### Protocol Evolution

```mermaid
timeline
    title HTTP Protocol Evolution
    1991 : HTTP/0.9 (GET only, no headers)
    1996 : HTTP/1.0 (headers, multiple methods, connection per request)
    1997 : HTTP/1.1 (persistent connections, pipelining, chunked transfer)
    2015 : HTTP/2 (multiplexing, header compression, server push, binary)
    2022 : HTTP/3 (QUIC transport, UDP-based, built-in encryption)
```

### Comparison Table

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **UDP (QUIC)** |
| Encryption | Optional (HTTPS) | Practically required | Always (built-in TLS 1.3) |
| Multiplexing | ❌ HOL blocking | ✅ Multiple streams | ✅ No stream HOL blocking |
| Header Compression | ❌ Plaintext | ✅ HPACK | ✅ QPACK |
| Server Push | ❌ | ✅ (waning use) | ✅ (more selective) |
| Protocol Format | Text | Binary | Binary |
| Connection Setup | 3-way handshake | 3-way + TLS | **0-RTT reconnect** |
| RFC | 9112 (2022) | 9113 (2022) | 9114 (2022) |

### HTTP/2 Key Features for API Design

**Multiplexing**: Multiple requests/responses on a single TCP connection simultaneously. Eliminates HTTP/1.1's "one request at a time" limitation. Critical for high-concurrency APIs.

**Header Compression (HPACK)**: HTTP headers often repeat (Authorization, Content-Type) — HPACK compresses repeated headers. Reduces overhead for stateless APIs where every request carries a JWT token.

**Server Push**: Server proactively sends resources the client will need. Waning support in browsers but useful for co-located microservices.

### gRPC and HTTP/2
gRPC runs exclusively on HTTP/2. This provides:
- Binary Protocol Buffers (smaller payloads than JSON)
- Bidirectional streaming (not possible with HTTP/1.1)
- Connection multiplexing (efficient for many concurrent calls)
- Header compression (efficient for repeated metadata)

For internal microservices, gRPC over HTTP/2 is often more efficient than REST over HTTP/1.1.

---

## CORS (Cross-Origin Resource Sharing)

```mermaid
sequenceDiagram
    participant B as Browser
    participant A as api.bank.com
    participant F as app.fintech.com (Different Origin)

    Note over B,A: Simple GET request (same origin) — no preflight
    B->>A: GET /api/v1/accounts
    A-->>B: 200 OK

    Note over B,F: Cross-origin POST — requires preflight
    B->>A: OPTIONS /api/v1/payments<br/>Origin: https://app.fintech.com<br/>Access-Control-Request-Method: POST<br/>Access-Control-Request-Headers: Authorization

    A-->>B: 200 OK<br/>Access-Control-Allow-Origin: https://app.fintech.com<br/>Access-Control-Allow-Methods: POST<br/>Access-Control-Allow-Headers: Authorization<br/>Access-Control-Max-Age: 600

    B->>A: POST /api/v1/payments<br/>Origin: https://app.fintech.com<br/>Authorization: Bearer eyJ...
    A-->>B: 201 Created
```

**Banking CORS configuration**: Use explicit allowlist, not `Access-Control-Allow-Origin: *`. Star origin disables credential sharing and is inappropriate for banking APIs.

---

## Code Examples

### Spring Boot HTTP Headers and Caching

```java
package com.bank.accounts.controller;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.WebRequest;
import java.time.Instant;
import java.time.temporal.ChronoUnit;

@RestController
@RequestMapping("/api/v1/accounts")
public class AccountCachingController {

    @GetMapping("/{accountId}")
    public ResponseEntity<AccountResponse> getAccount(
            @PathVariable String accountId,
            WebRequest webRequest) {

        AccountResponse account = accountService.findById(accountId);

        // Generate deterministic ETag from version + key fields
        String etag = String.format("\"%s-v%d\"",
                account.getId(), account.getVersion());

        // Check If-None-Match: return 304 if ETag matches (no body transfer!)
        if (webRequest.checkNotModified(etag)) {
            return null; // Spring returns 304 with correct headers automatically
        }

        return ResponseEntity.ok()
                .eTag(etag)
                // Private: only browser cache, not shared CDN cache
                // no-store for balance data (always fresh from server for compliance)
                .cacheControl(CacheControl.noStore().cachePrivate())
                .lastModified(account.getUpdatedAt())
                .body(account);
    }

    // Reference data — can be cached publicly
    @GetMapping("/currency-codes")
    public ResponseEntity<List<CurrencyCode>> getCurrencyCodes() {
        List<CurrencyCode> codes = referenceService.getCurrencyCodes();
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(1, ChronoUnit.HOURS)
                        .cachePublic())                  // CDN-cacheable
                .body(codes);
    }
}
```

### Spring Boot Security Headers Configuration

```java
package com.bank.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.header.writers.*;

@Configuration
public class SecurityHeadersConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                // HSTS — force HTTPS for 1 year, include subdomains
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true)
                    .preload(true))
                // Prevent MIME sniffing
                .contentTypeOptions(withDefaults())
                // Prevent clickjacking
                .frameOptions(frame -> frame.deny())
                // Custom CSP for API responses
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; frame-ancestors 'none'"))
            );
        return http.build();
    }
}
```

### CORS Configuration

```java
package com.bank.config;

import org.springframework.context.annotation.Bean;
import org.springframework.web.cors.*;
import org.springframework.web.filter.CorsFilter;

@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();

        // NEVER use "*" for banking APIs — explicit allowlist only
        config.setAllowedOrigins(List.of(
            "https://banking-app.bank.com",
            "https://partner-fintech.com"
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of(
            "Authorization", "Content-Type", "X-Request-ID",
            "X-Correlation-ID", "Idempotency-Key"
        ));
        config.setExposedHeaders(List.of(
            "X-RateLimit-Limit", "X-RateLimit-Remaining",
            "X-Request-ID", "Location", "ETag"
        ));
        config.setAllowCredentials(true);
        config.setMaxAge(600L); // Cache preflight 10 minutes

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return new CorsFilter(source);
    }
}
```

### curl Examples for HTTP Headers

```bash
# Conditional GET — use ETag (returns 304 if unchanged)
curl -X GET https://api.bank.com/api/v1/accounts/ACC-001 \
  -H "Authorization: Bearer eyJ..." \
  -H "If-None-Match: \"v1-50000-gbp\""
# → 304 Not Modified (no body transfer)

# Optimistic locking with If-Match
curl -X PUT https://api.bank.com/api/v1/accounts/ACC-001 \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -H "If-Match: \"v1-50000-gbp\"" \
  -d '{"overdraftLimit": 5000}'
# → 412 Precondition Failed if ETag mismatch (another update occurred)

# CORS preflight (browser does this automatically)
curl -X OPTIONS https://api.bank.com/api/v1/payments \
  -H "Origin: https://app.fintech.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Authorization" \
  -v
```

---

## Interview Questions & Model Answers

### Q1: What's the difference between 401 and 403?
**Answer**:
- **401 Unauthorized**: Authentication required or failed. "Who are you?" The request lacks valid credentials. Per RFC 9110, 401 responses **must** include a `WWW-Authenticate` header indicating how to authenticate.
- **403 Forbidden**: Authenticated but not authorised. "I know who you are, you can't do this." The server understood the request but refuses to authorise it.

**Banking example**: Customer calling another customer's account API:
- If no JWT token: 401 (authenticate first)
- If valid JWT but wrong customer: 403 (authenticated but not permitted)

### Q2: When would you return 202 vs 201?
**Answer**:
- **201 Created**: Resource was created synchronously and the response contains the created resource. Use for: account creation where the account is immediately available.
- **202 Accepted**: Request accepted for asynchronous processing. The resource may not exist yet. Use for: payment initiation (goes through compliance screening), bulk account updates, KYC document processing.

**202 pattern**:
```
POST /api/v1/bulk-payments → 202 Accepted
Response: {"jobId": "job-123", "statusUrl": "/api/v1/jobs/job-123"}
GET /api/v1/jobs/job-123 → 200 OK {"status": "IN_PROGRESS", "progress": 45}
```

### Q3: Explain ETags and when to use them.
**Answer**: An ETag (Entity Tag) is a cache validator — an opaque identifier for a specific version of a resource. Two main uses:
1. **Caching efficiency**: Client sends `If-None-Match: "etag"` on subsequent requests. If unchanged, server returns 304 Not Modified without body — saves bandwidth.
2. **Optimistic concurrency**: Client includes `If-Match: "etag"` on PUT/PATCH. Server returns 412 Precondition Failed if another update occurred between read and write, preventing lost updates.

Banking application: Account balance updates — use ETags to detect concurrent modifications between two branches updating the same account simultaneously.

### Q4: What is idempotency and which methods are idempotent?
**Answer**: Idempotency means calling an operation N times produces the same result as calling it once. Safe means no state modification.
- **Safe + Idempotent**: GET, HEAD, OPTIONS
- **Not Safe + Idempotent**: PUT, DELETE
- **Not Safe + Not Idempotent**: POST (creates new resource each time), PATCH (semantically may not be idempotent — increment operations)
- **TRACE**: Safe + Idempotent (but disable in production — security risk)

**Banking implication**: Payment POST must add `Idempotency-Key` header. Mobile apps on flaky networks retry — same Idempotency-Key means "process only once."

### Q5: What are the differences between HTTP/1.1, HTTP/2, and HTTP/3?
**Answer**: 
- **HTTP/1.1**: Text-based, one request per connection (Head-of-Line blocking), persistent connections with keep-alive
- **HTTP/2**: Binary framing, **multiplexing** (multiple streams per connection, eliminates HOL blocking), HPACK header compression, Server Push. Still uses TCP.
- **HTTP/3**: Uses **QUIC over UDP** instead of TCP. Eliminates TCP-level HOL blocking, 0-RTT reconnection for returning clients, built-in TLS 1.3.

**Impact on REST APIs**: HTTP/2 multiplexing enables higher concurrency on same connection. JWT tokens benefit from HPACK header compression (repeated headers compressed). HTTP/3's low-latency 0-RTT matters for mobile banking apps on cellular networks.

### Q6: What is CORS and why does it exist?
**Answer**: CORS (Cross-Origin Resource Sharing) is a browser security mechanism implementing the Same-Origin Policy. Browsers block scripts from making requests to different origins (protocol + host + port) than the page was loaded from. CORS allows servers to specify which origins are permitted.

**Why it matters for APIs**: A fintech app at `app.fintech.com` cannot call `api.bank.com` in the browser unless the bank's API returns `Access-Control-Allow-Origin: https://app.fintech.com`.

**Banking config**: Never use `*` for banking APIs. Use explicit origin allowlists. The `Access-Control-Max-Age` header reduces preflight overhead.

### Q7: Explain JSON Patch vs JSON Merge Patch for PATCH requests.
**Answer**:
- **JSON Merge Patch (RFC 7396)**: Send only changed fields. Null means delete. Simple and widely used. Content-Type: `application/merge-patch+json`
- **JSON Patch (RFC 6902)**: Array of operations (add, remove, replace, move, copy, test). More precise, supports array manipulation, has a `test` operation for conditional patching. Content-Type: `application/json-patch+json`

**When to use JSON Patch**: Complex array manipulations, conditional updates (test + replace), or when you need to atomically modify multiple disparate fields with guaranteed ordering.

**Banking example**: Updating beneficiary list on an account:
- Merge Patch: Replace entire beneficiaries array
- JSON Patch: `add` a single beneficiary to existing array without client needing the full current array

### Q8: What is the difference between 404 and 410?
**Answer**:
- **404 Not Found**: Resource doesn't exist at this time (may exist in future, may have existed before)
- **410 Gone**: Resource permanently removed and will never return

**Banking use**: When a closed account's API endpoint is retired, 410 tells crawlers/clients to stop caching and remove from link databases. 404 means "try again later."

---

## Common Pitfalls & Best Practices

### Anti-Patterns
1. **Returning 200 for all errors**: Breaks client error handling, monitoring can't detect failures
2. **Using GET with side effects**: `GET /accounts/ACC-001?action=close` — never use GET for mutations
3. **401 without WWW-Authenticate header**: RFC 9110 violation
4. **CORS with `Access-Control-Allow-Origin: *` for authenticated endpoints**: Disables cookies/credentials
5. **Not setting Cache-Control on account balance responses**: CDN might cache stale balances — catastrophic in banking

### Best Practices
1. Always return **201 Created + Location** header for POST resource creation
2. Use **If-Match with ETag** for all PUT/PATCH on shared resources
3. Add **Idempotency-Key** support for POST payment/transaction endpoints
4. Set explicit **Cache-Control** on every response (even `no-store` to be explicit)
5. Configure **HSTS** with preload for all production API endpoints
6. Return **RFC 7807 Problem Details** for all 4xx/5xx errors

---

## Key Takeaways

- **Safe = read-only; Idempotent = same result if called N times** — crucial for retry logic
- **PUT is idempotent, POST is not** — the key use-case difference
- **401 lacks credentials, 403 lacks permission** — always include `WWW-Authenticate` with 401
- **ETag enables both caching efficiency AND optimistic concurrency** — one header, two use cases
- **HTTP/2 multiplexing eliminates connection bottleneck** — critical for high-concurrency banking APIs
- **CORS is browser-only** — server-to-server calls don't need CORS
- **Security headers are non-negotiable** in banking — HSTS, CSP, X-Content-Type-Options minimum

---

## Further Reading
- RFC 9110: HTTP Semantics (2022) — [https://www.rfc-editor.org/rfc/rfc9110](https://www.rfc-editor.org/rfc/rfc9110)
- RFC 9112: HTTP/1.1 (2022)
- RFC 9113: HTTP/2 (2022)
- RFC 9114: HTTP/3 (2022)
- RFC 6902: JSON Patch — [https://www.rfc-editor.org/rfc/rfc6902](https://www.rfc-editor.org/rfc/rfc6902)
- RFC 7396: JSON Merge Patch
- RFC 7616: HTTP Digest Access Authentication
- [MDN HTTP Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [HTTP/2 Frequently Asked Questions](https://http2.github.io/faq/)
