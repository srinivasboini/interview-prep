# API Security: Authentication & Authorisation

## Overview

API security is the single most critical topic for a Staff/Principal Engineer interview at a bank. Financial institutions face sophisticated adversaries, regulatory mandates (PCI-DSS, GDPR, SOX), and catastrophic liability exposures from security breaches. Getting auth wrong is a career-ending event in banking.

This chapter covers the full spectrum: from Basic Authentication (never use in prod) to OAuth 2.0 grant types, JWT internals and vulnerabilities, mutual TLS, RBAC/ABAC authorisation models, the OWASP API Security Top 10, and banking-specific compliance requirements.

---

## Authentication Mechanisms

### Basic Authentication

```http
GET /api/v1/accounts HTTP/1.1
Authorization: Basic dXNlcjpwYXNzd29yZA==
```

`dXNlcjpwYXNzd29yZA==` = Base64("user:password")

**⚠️ WARNING**: Base64 is encoding, NOT encryption. Fully reversible.

**When acceptable**:
- Development/testing environments only
- Machine-to-machine with internal private network + HTTPS
- Legacy system integration with no better option

**Never in production banking APIs**. Always HTTPS if used.

### API Key Authentication

```http
GET /api/v1/accounts HTTP/1.1
X-API-Key: sk_live_abc123xyz789...

# Or in query parameter (AVOID — appears in logs/browser history):
GET /api/v1/accounts?api_key=sk_live_abc123xyz789
```

**Use cases**:
- Rate limiting per API key (billing, partner tiers)
- Server-to-server integration with third-party partners (Open Banking)
- Service-level access control (not user-level)

**Key management requirements**:
- Store hashed in database (SHA-256), never plaintext
- Prefix with environment indicator: `sk_live_`, `sk_test_`
- Support key rotation without downtime (multiple active keys)
- Transmit only via headers, never query parameters

---

## OAuth 2.0 — Deep Dive

OAuth 2.0 (RFC 6749) is the authorisation framework used by virtually all modern banking APIs. It delegates authorisation to an authorisation server, separating identity from access.

### OAuth 2.0 Roles

| Role | Description | Banking Example |
|---|---|---|
| **Resource Owner** | End user who owns the data | Customer |
| **Client** | Application requesting access | Mobile banking app |
| **Resource Server** | API server hosting protected resources | Account Service API |
| **Authorisation Server** | Issues tokens (after user consent) | Bank's IAM (Okta/Azure AD B2C) |

### Grant Types

#### 1. Authorization Code + PKCE (Most Secure)

Use for: interactive user login (web/mobile apps). PKCE (Proof Key for Code Exchange, RFC 7636) prevents authorization code interception.

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client (App)
    participant AS as Auth Server
    participant RS as Account API

    C->>C: Generate code_verifier (random) + code_challenge = SHA256(code_verifier)
    U->>C: Clicks "Login"
    C->>AS: GET /authorize?response_type=code&client_id=app123&redirect_uri=...&code_challenge=xyz&scope=accounts:read
    AS->>U: Show login/consent screen
    U->>AS: Authenticate + consent
    AS->>C: Redirect with ?code=AUTH_CODE
    C->>AS: POST /token {code=AUTH_CODE, code_verifier=random_string, grant_type=authorization_code}
    AS->>AS: Verify SHA256(code_verifier) == code_challenge
    AS-->>C: {access_token, refresh_token, expires_in}
    C->>RS: GET /api/v1/accounts<br/>Authorization: Bearer {access_token}
    RS-->>C: Account data
```

#### 2. Client Credentials (Service-to-Service)

Use for: machine-to-machine, no user involved. Microservice calling another microservice.

```mermaid
sequenceDiagram
    participant S as Payment Service
    participant AS as Auth Server
    participant RS as Fraud API

    S->>AS: POST /token {grant_type=client_credentials, client_id=payment-svc, client_secret=SECRET}
    AS-->>S: {access_token, expires_in: 3600}
    S->>RS: GET /api/v1/fraud-check?txnId=123<br/>Authorization: Bearer {access_token}
    RS-->>S: {riskScore: 0.12, recommendation: APPROVE}
```

#### 3. Deprecated Grant Types

| Grant Type | Status | Reason |
|---|---|---|
| Implicit | ❌ Deprecated (RFC 9700) | Tokens exposed in URL fragment, no refresh tokens |
| Resource Owner Password Credentials | ❌ Avoid | Client handles user credentials — kills SSO |
| Device Authorization | ✅ Active | Smart TVs, IoT, CLI tools with no browser |

### Token Types

| Token | Format | Validation | Revocation | Performance |
|---|---|---|---|---|
| **Opaque** | Random string | Call introspection endpoint | Immediate | Slower (network call) |
| **JWT** | Self-contained | Verify signature locally | Complex (blocklist needed) | Fast (local verification) |

**Banking trade-off**: JWTs are preferred for performance at scale. Short expiry (5-15 min) + refresh tokens compensates for revocation complexity.

---

## JWT (JSON Web Tokens) — RFC 7519

### Structure

A JWT consists of three Base64URL-encoded parts separated by dots:
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJDVVNULTEyMyIsImlzcyI6Imh0dHBzOi8vYXV0aC5iYW5rLmNvbSIsImF1ZCI6WyJhY2NvdW50cy1hcGkiXSwiZXhwIjoxNzM1NjUwMDAwLCJpYXQiOjE3MzU2NDY0MDAsImp0aSI6InVuaXF1ZS10b2tlbi1pZCIsInNjb3BlIjoiYWNjb3VudHM6cmVhZCBwYXltZW50czp3cml0ZSIsImN1c3RvbWVySWQiOiJDVVNULTEyMyIsInRpZXIiOiJQUkVNSVVNIn0.SIGNATURE
HEADER                                . PAYLOAD                                                                                                                                                                                                    . SIGNATURE
```

**Header**:
```json
{
  "alg": "RS256",    ← Signing algorithm
  "typ": "JWT",      ← Token type
  "kid": "key-001"   ← Key ID (for key rotation)
}
```

**Payload (Claims)**:
```json
{
  "iss": "https://auth.bank.com",     ← Issuer
  "sub": "CUST-123",                  ← Subject (user/system)
  "aud": ["accounts-api"],            ← Audience (intended recipient)
  "exp": 1735650000,                  ← Expiry (Unix timestamp)
  "nbf": 1735646400,                  ← Not Before
  "iat": 1735646400,                  ← Issued At
  "jti": "unique-token-id",           ← JWT ID (for revocation/replay prevention)
  "scope": "accounts:read payments:write",
  "customerId": "CUST-123",           ← Custom claims
  "tier": "PREMIUM"
}
```

**Signature**:
```
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  private_key
)
```

### JWT Signing Algorithms

| Algorithm | Type | Key Type | Use Case |
|---|---|---|---|
| HS256 | Symmetric | Shared secret | Internal microservices only |
| RS256 | Asymmetric | RSA private/public | Most common for OAuth 2.0 |
| ES256 | Asymmetric | ECDSA | Compact signatures, mobile |
| PS256 | Asymmetric | RSA-PSS | PSD2 banking compliance |

**Banking recommendation**: RS256 or ES256. Never expose RSA private keys. Store in HSM (Hardware Security Module) or Azure Key Vault.

### JWT Validation Process

```mermaid
flowchart TD
    A[Receive JWT in Authorization header] --> B{Parse three parts?}
    B --> |No| FAIL1[400 Bad Request]
    B --> |Yes| C{Verify signature with public key}
    C --> |Invalid| FAIL2[401 Unauthorized]
    C --> |Valid| D{Check exp - current time}
    D --> |Expired| FAIL3[401 Unauthorized / token refresh needed]
    D --> |Valid| E{Check iss - matches expected issuer}
    E --> |Mismatch| FAIL4[401 Unauthorized]
    E --> |Match| F{Check aud - includes this API}
    F --> |Missing/wrong| FAIL5[401 Unauthorized]
    F --> |Match| G{Check nbf - not before time}
    G --> |Too early| FAIL6[401 Unauthorized]
    G --> |Valid| H{Check jti - not in revocation list}
    H --> |Revoked| FAIL7[401 Unauthorized]
    H --> |Valid| SUCCESS[✅ Token valid - proceed]
```

### JWT Security Best Practices

✅ **DO**:
- Set short expiry: 5-15 minutes for access tokens
- Always validate **all claims**: `iss`, `aud`, `exp`, `nbf`
- Use RS256 or ES256 (asymmetric — public key can be shared)
- Include `jti` for revocation capability
- Rotate signing keys with `kid` header (zero-downtime rotation)
- Store tokens in `httpOnly` cookies or secure storage (not localStorage)
- Validate audience (`aud`) to prevent token reuse across APIs

❌ **NEVER**:
- Use `alg: none` — allows unsigned tokens → critical vulnerability
- Trust the `alg` header blindly — attacker can switch RS256 to HS256 and sign with public key
- Put sensitive data in payload (it's base64 — not encrypted, just encoded)
- Use symmetric HS256 for external-facing APIs
- Long-lived access tokens (1 day+)

### Critical JWT Vulnerability: Algorithm Confusion

```
Attack:
1. Attacker has public key (public!)
2. Attacker changes header from {"alg":"RS256"} to {"alg":"HS256"}
3. Attacker signs token with PUBLIC key as HMAC secret
4. Server verifies HS256 with "public key" as secret → ACCEPTS malicious token!

Defence:
- Never use the alg header to determine validation algorithm
- Hardcode expected algorithm in validator
- Reject tokens with unexpected or "none" algorithm
```

---

## OpenID Connect (OIDC)

OIDC is an identity layer on top of OAuth 2.0. Where OAuth 2.0 answers "what can this app do?", OIDC answers "who is this user?"

| | OAuth 2.0 | OIDC |
|---|---|---|
| Purpose | Authorisation | Authentication |
| Token | Access token | ID token (+ access token) |
| User info | Not included | In ID token claims |
| Standard scopes | Custom | `openid`, `profile`, `email` |

**ID Token**: A JWT containing user identity claims. Must be validated like any JWT.

**Open Banking usage**: PSD2 requires OIDC for customer authentication. Banks as Identity Providers issue ID tokens for third-party apps to verify customer identity.

---

## Mutual TLS (mTLS)

Standard TLS: Server proves identity to client (server certificate).
mTLS: Both client AND server authenticate with certificates.

```mermaid
sequenceDiagram
    participant PS as Payment Service
    participant FS as Fraud Service

    PS->>FS: TLS Client Hello
    FS->>PS: Server Certificate (FS cert)
    PS->>FS: Client Certificate (PS cert)
    FS->>PS: Certificate Verified ✅
    PS->>FS: POST /api/v1/fraud-check (encrypted)
    FS-->>PS: 200 OK {riskScore: 0.08}
```

**Use cases in banking**:
- Service-to-service in microservices (zero trust)
- PSD2 Open Banking: Third-party providers must present eIDAS certificates
- Payment networks (SWIFT, SEPA)

**Implementation**: Service mesh (Istio/Linkerd) handles mTLS transparently without code changes.

---

## Authorisation Patterns

### RBAC (Role-Based Access Control)

```java
// Spring Security RBAC
@PreAuthorize("hasRole('ACCOUNT_MANAGER') or hasRole('ADMIN')")
@GetMapping("/{accountId}/transactions")
public ResponseEntity<List<Transaction>> getTransactions(@PathVariable String accountId) {
    return ResponseEntity.ok(transactionService.findByAccount(accountId));
}

// JWT roles claim:
{"sub":"USER-123","roles":["ACCOUNT_MANAGER","PREMIUM_CUSTOMER"]}
```

**Banking roles example**: `TELLER`, `ACCOUNT_MANAGER`, `BRANCH_MANAGER`, `COMPLIANCE_OFFICER`, `ADMIN`

### ABAC (Attribute-Based Access Control)

More granular than RBAC — decisions based on:
- Subject attributes (user role, clearance level, department)
- Resource attributes (account owner, risk level, currency)
- Environment attributes (time of day, IP location, device risk)
- Action attributes (read vs write)

```java
// ABAC using Spring Security with custom expression
@PreAuthorize("@accountSecurity.canAccess(authentication, #accountId)")
@GetMapping("/{accountId}")
public ResponseEntity<Account> getAccount(@PathVariable String accountId) { ... }

@Component("accountSecurity")
public class AccountSecurityService {
    public boolean canAccess(Authentication auth, String accountId) {
        CustomerPrincipal user = (CustomerPrincipal) auth.getPrincipal();
        Account account = accountService.findById(accountId);
        return account.getCustomerId().equals(user.getCustomerId()) // Owner
            || user.hasRole("ACCOUNT_MANAGER")                       // Staff
            || user.hasRole("COMPLIANCE_OFFICER");                   // Compliance
    }
}
```

---

## OWASP API Security Top 10 (2023)

| # | Vulnerability | Banking Example | Prevention |
|---|---|---|---|
| API1 | Broken Object Level Authorisation | Customer accesses other customer's account | Verify ownership for every request |
| API2 | Broken Authentication | Accepting expired/invalid JWTs | Full JWT validation chain |
| API3 | Broken Object Property Level Authorisation | Customer changes their credit limit | Allowlist mutable fields per role |
| API4 | Unrestricted Resource Consumption | No rate limiting → DDoS/billing attack | Rate limiting per key/user |
| API5 | Broken Function Level Authorisation | Customer calls admin-only endpoints | Function-level auth checks |
| API6 | Unrestricted Access to Sensitive Business Flows | Automated account opening fraud | Bot detection, CAPTCHA, velocity checks |
| API7 | Server Side Request Forgery | Webhook URL points to internal services | Allowlist webhook domains |
| API8 | Security Misconfiguration | Swagger UI exposed in production | Environment-specific config |
| API9 | Improper Inventory Management | Shadow APIs not in gateway | API registry, gateway enforcement |
| API10 | Unsafe Consumption of APIs | Trusting third-party API data without validation | Validate all external inputs |

---

## Code Examples

### Spring Security — OAuth 2.0 Resource Server with JWT

```java
package com.bank.config;

import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity  // Enables @PreAuthorize on methods
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // No sessions
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/accounts/**").hasAnyAuthority(
                    "SCOPE_accounts:read", "ROLE_ACCOUNT_MANAGER")
                .requestMatchers(HttpMethod.POST, "/api/v1/payments/**").hasAuthority(
                    "SCOPE_payments:write")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.decoder(jwtDecoder()))
            );
        return http.build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        // Verify tokens issued by our auth server
        // Public key fetched from JWKS endpoint: /oauth2/.well-known/jwks.json
        NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(
            "https://auth.bank.com"
        );

        // Custom validators
        OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer(
            "https://auth.bank.com"
        );
        OAuth2TokenValidator<Jwt> withAudience = new JwtClaimValidator<List<String>>(
            JwtClaimNames.AUD, aud -> aud.contains("accounts-api")
        );

        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
            withIssuer, withAudience
        ));
        return decoder;
    }
}
```

### Idempotent Payment Endpoint with Security

```java
package com.bank.payments.controller;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;

@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    @PostMapping
    @PreAuthorize("hasAuthority('SCOPE_payments:write')")
    public ResponseEntity<PaymentResponse> initiatePayment(
            @AuthenticationPrincipal Jwt jwt,           // JWT principal from SecurityContext
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody PaymentRequest request) {

        String customerId = jwt.getSubject();           // Extract from validated JWT
        String requestedScope = jwt.getClaimAsString("scope");

        // Verify customer can only initiate from their own accounts
        accountService.verifyOwnership(customerId, request.getSourceAccountId());

        // Idempotency: check if this key was already processed
        return idempotencyService.getOrCreate(idempotencyKey, () -> {
            PaymentResponse payment = paymentService.initiate(customerId, request);
            return ResponseEntity.accepted().body(payment); // 202 Accepted (async)
        });
    }
}
```

### curl Security Examples

```bash
# OAuth 2.0 Client Credentials flow (service-to-service)
curl -X POST https://auth.bank.com/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=payment-service&client_secret=SECRET&scope=fraud:read"

# Response: {"access_token":"eyJ...","token_type":"Bearer","expires_in":3600}

# Call API with Bearer token
curl -X GET https://api.bank.com/api/v1/accounts/ACC-001 \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9..." \
  -H "X-Request-ID: $(uuidgen)"

# Decode JWT payload (base64 decode middle part)
echo "eyJzdWIiOiJDVVNULTEyMyJ9" | base64 -d
```

---

## Interview Questions & Model Answers

### Q1: Explain OAuth 2.0 grant types and when to use each.
**Answer**:
- **Authorization Code + PKCE**: Interactive user authentication (web/mobile apps). Most secure for user-facing flows. PKCE prevents auth code interception attacks.
- **Client Credentials**: Service-to-service; no user involved. Payment service calling fraud service. Client authenticates with its own credentials.
- **Implicit**: Deprecated (RFC 9700). Tokens exposed in URL. Replaced by Auth Code + PKCE.
- **Resource Owner Password**: Client handles user credentials — bad for SSO, breaks separation. Legacy only.
- **Device Authorization**: IoT devices, CLIs, smart TVs without browser.

For a banking mobile app: Authorization Code + PKCE. For microservice calling microservice: Client Credentials.

### Q2: What is JWT and how do you validate it?
**Answer**: JWT (JSON Web Token, RFC 7519) is a compact, URL-safe token format with three Base64URL-encoded parts: header (algorithm), payload (claims), signature.

Validation steps:
1. Verify signature using public key from JWKS endpoint
2. Check `alg` header matches expected algorithm (never trust `alg` from token)
3. Verify `exp` (not expired)
4. Verify `iss` (correct issuer)
5. Verify `aud` (intended for this API)
6. Check `nbf` (not before — token not used prematurely)
7. Check `jti` against revocation list if using JWT revocation

Critical vulnerability: **Algorithm confusion attack** — don't trust the `alg` header. Hardcode expected algorithm in validator.

### Q3: 401 vs 403 — what's the exact difference?
**Answer**: Per RFC 9110:
- **401**: Request lacks valid authentication credentials. `WWW-Authenticate` header must be included to tell client how to authenticate. The identity is unknown.
- **403**: Request understood and authenticated, but the server refuses to authorize it. Identity is known, permission denied.

**Banking scenario**: Customer A calls `GET /api/v1/accounts/ACC-999` (B's account):
- No JWT: 401 + `WWW-Authenticate: Bearer realm="api.bank.com"`
- Valid JWT but wrong customer: 403 (known identity, no permission)

**Tricky follow-up**: Should 403 reveal that the resource exists? In security-sensitive APIs, you might return 404 to prevent enumeration attacks — exposing 403 confirms the resource exists.

### Q4: What is OWASP API Security Top 10? Name the most critical for banking.
**Answer**: OWASP API Security Top 10 identifies the most critical API vulnerabilities. For banking specifically:

**API1 (Broken Object Level Authorisation)**: Most common. Customer GET/PUT request should only work for their own accounts. Must verify ownership on EVERY request — don't rely on JWT alone, re-check database ownership.

**API2 (Broken Authentication)**: Accepting expired/invalid JWTs, weak token validation, missing signature verification. Validate ALL JWT claims every time.

**API4 (Resource Consumption)**: No rate limiting allows DDoS or account enumeration attacks. Rate limit per customer, per IP, per API key.

**API8 (Security Misconfiguration)**: Swagger UI exposed in production, debug endpoints active, TLS 1.0 allowed. Environment-specific security profiles mandatory.

### Q5: What is mTLS and why is it used in banking?
**Answer**: Mutual TLS (mTLS) requires both client AND server to authenticate with X.509 certificates. Standard TLS authenticates only the server. mTLS adds client certificate verification.

**Banking use cases**:
1. **Microservice communication**: Zero-trust architecture — every service verifies its caller. Service mesh (Istio) handles mTLS transparently.
2. **Open Banking (PSD2)**: Third-party providers must present eIDAS qualified certificates when calling bank APIs. Banks verify certificate against trust store.
3. **Payment networks**: SWIFT, SEPA, TARGET2 use certificate-based authentication.

**Advantage over API keys**: Keys can be stolen and reused. Certificates bind authentication to the actual process/machine through PKI.

---

## Common Pitfalls & Best Practices

### Security Anti-Patterns
1. **JWT `alg: none` not blocked**: Allows unsigned tokens — critical vulnerability
2. **Not validating JWT audience**: Token for API A accepted by API B
3. **Long-lived access tokens**: 24h tokens give attacker long window
4. **Sensitive data in JWT payload**: Payload is base64-encoded, not encrypted — visible to anyone
5. **Logging Authorization header**: Leaks tokens to log aggregation systems
6. **`Access-Control-Allow-Origin: *` with credentials**: Not possible, browser blocks; but `*` on sensitive APIs is bad practice

### Best Practices
1. **RS256 for JWT signing** — asymmetric, public key can be distributed safely
2. **5-15 minute access token expiry** with refresh tokens
3. **JWKS endpoint** for key rotation without service restart
4. **OAuth 2.0 scopes** for fine-grained API access control
5. **mTLS for microservice communication** — zero trust
6. **Never log Authorization headers** — use correlation IDs instead

---

## Key Takeaways

- **OAuth 2.0 grant type choice depends on actor**: Authorization Code + PKCE for users, Client Credentials for services
- **JWT must validate ALL claims**: `alg`, `iss`, `aud`, `exp`, `nbf`, `jti` — partial validation = vulnerable
- **JWT algorithm confusion is a critical vulnerability** — hardcode expected algorithm
- **401 = no/bad credentials, 403 = no permission** — RFC 9110 requires `WWW-Authenticate` with 401
- **OWASP API1 (Broken Object Level Auth) is most common banking vulnerability** — always verify resource ownership
- **mTLS for service-to-service** — service mesh handles transparently
- **Short-lived access tokens + refresh tokens** — balance security and usability

---

## Further Reading
- RFC 6749: OAuth 2.0 — [https://www.rfc-editor.org/rfc/rfc6749](https://www.rfc-editor.org/rfc/rfc6749)
- RFC 7519: JSON Web Tokens (JWT)
- RFC 7636: PKCE for OAuth 2.0
- RFC 9700: OAuth 2.0 Security Best Current Practice
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [JWT.io](https://jwt.io/) — JWT decoder and algorithm reference
- [Spring Security OAuth2 Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html)
- PSD2 API Security Requirements (EBA Guidelines)
