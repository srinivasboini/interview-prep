# REST API Design Cheat Sheet

## 1. URI Naming Conventions
- **Nouns, not verbs**: Use `/accounts` ✅, NEVER `/getAccounts` ❌.
- **Plurals exclusively**: Use `/accounts` (collection) and `/accounts/123` (item).
- **Hierarchy mapping**: `/accounts/123/transactions/456` denotes relational structure. Max depth: 3 levels.
- **Lowercase & Hyphens**: Kebab-case rules (`/api/v1/payment-templates`). No underscores.
- **No File Extensions**: Never append `.json` or `.xml`. Utilize Content Negotiation headers.

## 2. HTTP Methods & Characteristics
- **`GET`**: Retrieve resource. **Safe** & **Idempotent**. (Body typically ignored).
- **`POST`**: Create new resource or generic action. **Not Safe** & **Not Idempotent**.
- **`PUT`**: Fully overwrite/replace a specific resource. **Not Safe** & **Idempotent**.
- **`PATCH`**: Partially modify a resource. **Not Safe** & **Not strictly Idempotent**.
- **`DELETE`**: Remove a resource. **Not Safe** & **Idempotent**.

## 3. Essential Status Codes
**2xx (Success)**
- `200 OK`: Standard success (GET, PUT, PATCH).
- `201 Created`: Resource generated (POST). MUST include a `Location` header.
- `202 Accepted`: Async processing initiated (Long-running jobs).
- `204 No Content`: Successful deletion or update demanding no response payload.

**4xx (Client Error)**
- `400 Bad Request`: Malformed JSON syntax or schema failure.
- `401 Unauthorized`: Authentication missing or Token fully invalid / expired.
- `403 Forbidden`: Authenticated successfully, but lacks specific system Permissions (RBAC).
- `404 Not Found`: URI / ID fundamentally does not exist.
- `409 Conflict`: Resource state conflict (e.g., trying to recreate an active email).
- `412 Precondition Failed`: `If-Match` ETag failure during Optimistic Locking.
- `422 Unprocessable Entity`: Semantic logic failure (e.g., Insufficient Funds available).
- `429 Too Many Requests`: Imposed Rate Limitation. Analyze `Retry-After` header.

**5xx (Server Error)**
- `500 Internal Server Error`: Unhandled application exception crash.
- `502 Bad Gateway`: Proxy/Gateway received an invalid response originating from an upstream.
- `503 Service Unavailable`: System overloaded, connection refused, or Circuit Breaker opened.

## 4. Uncompromising Security Basics
- **Encryption**: TLS 1.2+ mandatory. Absolutely no HTTP.
- **JWT Integrity**: Validate completely: `alg` configuration, `iss` (Issuer), `aud` (Audience), `exp` (Expiry), and signature match.
- **Microservice Trust**: Utilize strict **Mutual TLS (mTLS)** for inter-cluster "Zero-Trust" comms.
- **Idempotency Protection**: Implement `Idempotency-Key` headers safeguarding mutative APIs preventing retried duplicated payments.
- **Header Hardening**: `Strict-Transport-Security`, `Content-Security-Policy`.

## 5. Performance & Scalability Design
- **Pagination**: Implement **Cursor-based** pagination (`?next=cursor123`) for high-scale fluidity. Abandon Offset/Limit architectures.
- **Cache-Control**: 
  - `no-store`: Enforce universally for PII / Financial data.
  - `public, max-age=X`: Utilize aggressively for static reference/config lookup data.
- **Conditional GETs**: Provide `ETag` hashes promoting `304 Not Modified` network optimizations.
- **Compression**: Negotiate `Accept-Encoding: gzip, br` diminishing JSON bloat drastically.

## 6. Standardized Error Handling
Implement **RFC 7807 (Problem Details for HTTP APIs)** exclusively.
```json
{
  "type": "https://developer.bank.com/errors/insufficient-funds",
  "title": "Insufficient Funds Available",
  "status": 422,
  "detail": "Account balance of $50.00 cannot facilitate transfer of $100.00.",
  "instance": "/api/v1/payments/req-898"
}
```

## 7. Operational Observability
- **Trace Propagation**: Retain and pass `X-Correlation-ID` across every internal HTTP call.
- **Timeout Discipline**: Establish rigorous thresholds on every RestTemplate/WebClient execution.
- **Log Masking**: Implement regex blocking recording PCI-DSS restricted data natively in Logback. 
- **Health Realism**: Ascertain K8s `/health/readiness` probes accurately check backing Database connectivity, circumventing false-positives.
