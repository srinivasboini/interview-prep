# Kafka Security

## Overview

Kafka security encompasses authentication, authorization, encryption, and audit logging to protect sensitive data and meet regulatory compliance requirements. Understanding Kafka's security mechanisms is critical for deploying production systems that handle financial transactions, personally identifiable information (PII), and confidential business data.

**Why This Matters for Interviews**: Senior engineering roles in regulated industries (banking, healthcare, government) require deep understanding of security controls, compliance frameworks, and secure architecture patterns. Expect questions about authentication mechanisms, ACL design, encryption strategies, and meeting regulatory requirements (PCI-DSS, GDPR, SOC 2).

**Real-World Banking Context**: Financial institutions must comply with PCI-DSS (payment card data), GDPR (customer privacy), SOX (audit trails), and internal security policies. Kafka clusters handling payment transactions, customer data, and account information require defense-in-depth security: authentication, authorization, encryption in-transit, encryption at-rest, and comprehensive audit logging.

---

## Authentication

Authentication verifies the identity of clients (producers, consumers, brokers) connecting to Kafka.

### SASL/PLAIN

**SASL/PLAIN** uses username/password authentication (simplest mechanism).

**Configuration**:

**Broker (server.properties)**:
```properties
# Enable SASL/PLAIN
listeners=SASL_PLAINTEXT://0.0.0.0:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN

# JAAS configuration
listener.name.sasl_plaintext.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
   username="admin" \
   password="admin-secret" \
   user_admin="admin-secret" \
   user_producer="producer-secret" \
   user_consumer="consumer-secret";
```

**Client (Producer/Consumer)**:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("security.protocol", "SASL_PLAINTEXT");
props.put("sasl.mechanism", "PLAIN");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.plain.PlainLoginModule required " +
    "username=\"producer\" " +
    "password=\"producer-secret\";");
```

**Pros**:
- Simple setup
- No external dependencies

**Cons**:
- **Insecure**: Passwords in plaintext (must use with TLS)
- **Static**: Users defined in config (no central management)
- **Not recommended for production** without TLS

---

### SASL/SCRAM

**SASL/SCRAM** (Salted Challenge Response Authentication Mechanism) uses hashed passwords stored in ZooKeeper/KRaft.

**Setup**:

**1. Create Users** (stored in ZooKeeper):
```bash
# Create user with SCRAM-SHA-256
kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --add-config 'SCRAM-SHA-256=[password=producer-secret]' \
  --entity-type users --entity-name producer

kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --add-config 'SCRAM-SHA-256=[password=consumer-secret]' \
  --entity-type users --entity-name consumer

# Verify user created
kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe --entity-type users --entity-name producer
```

**2. Broker Configuration**:
```properties
listeners=SASL_SSL://0.0.0.0:9092
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
sasl.enabled.mechanisms=SCRAM-SHA-256

listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required;
```

**3. Client Configuration**:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "SCRAM-SHA-256");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.scram.ScramLoginModule required " +
    "username=\"producer\" " +
    "password=\"producer-secret\";");

// SSL configuration (for SASL_SSL)
props.put("ssl.truststore.location", "/path/to/truststore.jks");
props.put("ssl.truststore.password", "truststore-password");
```

**Pros**:
- Hashed passwords (more secure than PLAIN)
- Dynamic user management (add/remove without broker restart)
- No external dependencies

**Cons**:
- Users stored in ZooKeeper/Kafka metadata (not centralized like LDAP)

---

### SASL/GSSAPI (Kerberos)

**Kerberos** provides enterprise single sign-on (SSO) authentication.

**Architecture**:
```
┌─────────────┐                    ┌──────────────┐
│   Client    │────TGT Request────→│   Kerberos   │
│             │←─────TGT───────────│   KDC (AD)   │
└─────────────┘                    └──────────────┘
       │
       │ Service Ticket Request
       ↓
┌─────────────┐                    ┌──────────────┐
│   Kafka     │←──Service Ticket───│   Kerberos   │
│   Broker    │                    │     KDC      │
└─────────────┘                    └──────────────┘
```

**Configuration**:

**Broker**:
```properties
listeners=SASL_PLAINTEXT://0.0.0.0:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=GSSAPI
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=kafka

listener.name.sasl_plaintext.gssapi.sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required \
   useKeyTab=true \
   storeKey=true \
   keyTab="/etc/security/keytabs/kafka_server.keytab" \
   principal="kafka/broker.example.com@EXAMPLE.COM";
```

**Client**:
```java
Properties props = new Properties();
props.put("security.protocol", "SASL_PLAINTEXT");
props.put("sasl.mechanism", "GSSAPI");
props.put("sasl.kerberos.service.name", "kafka");
props.put("sasl.jaas.config",
    "com.sun.security.auth.module.Krb5LoginModule required " +
    "useKeyTab=true " +
    "storeKey=true " +
    "keyTab=\"/etc/security/keytabs/producer.keytab\" " +
    "principal=\"producer@EXAMPLE.COM\";");
```

**Pros**:
- Enterprise SSO (integrates with Active Directory)
- Strong authentication
- Centralized user management

**Cons**:
- Complex setup (Kerberos infrastructure required)
- Key management overhead (keytabs)

---

### Mutual TLS (mTLS)

**Mutual TLS** uses client certificates for authentication (certificate-based).

**Certificate Creation**:
```bash
# Create CA (Certificate Authority)
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365

# Create broker keystore and certificate
keytool -keystore kafka.broker.keystore.jks -alias broker -validity 365 -genkey

# Sign broker certificate with CA
keytool -keystore kafka.broker.keystore.jks -alias broker -certreq -file cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial

# Import CA and signed certificate into broker keystore
keytool -keystore kafka.broker.keystore.jks -alias CARoot -import -file ca-cert
keytool -keystore kafka.broker.keystore.jks -alias broker -import -file cert-signed

# Create client keystore (similar process for each client)
keytool -keystore kafka.client.keystore.jks -alias client -validity 365 -genkey
# Sign with CA...

# Create truststore (contains CA cert)
keytool -keystore kafka.truststore.jks -alias CARoot -import -file ca-cert
```

**Broker Configuration**:
```properties
listeners=SSL://0.0.0.0:9093
security.inter.broker.protocol=SSL

# SSL settings
ssl.keystore.location=/var/private/ssl/kafka.broker.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/var/private/ssl/kafka.truststore.jks
ssl.truststore.password=truststore-password

# Require client authentication
ssl.client.auth=required
```

**Client Configuration**:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9093");
props.put("security.protocol", "SSL");

// Client certificate
props.put("ssl.keystore.location", "/path/to/client.keystore.jks");
props.put("ssl.keystore.password", "keystore-password");
props.put("ssl.key.password", "key-password");

// Trust broker certificate
props.put("ssl.truststore.location", "/path/to/truststore.jks");
props.put("ssl.truststore.password", "truststore-password");
```

**Pros**:
- Strong authentication (cryptographic)
- No passwords to manage

**Cons**:
- Complex certificate management
- Certificate rotation complexity

**Banking Use Case**: Payment processing service uses mTLS (each service has unique certificate, no shared passwords)

---

## Authorization

Authorization controls what authenticated users can do (read, write, create topics).

### Access Control Lists (ACLs)

**ACLs** define permissions for users/groups on Kafka resources.

**ACL Format**:
```
Principal (User:alice) CAN/CANNOT Operation (READ) ON Resource (Topic:payments) FROM Host (192.168.1.*)
```

**Resource Types**:
- **Topic**: Read, write, describe, delete
- **Consumer Group**: Read (consume)
- **Cluster**: Create topics, alter configs
- **TransactionalId**: Write (for transactions)

**Creating ACLs**:

```bash
# Grant producer permission to write to payments topic
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:producer \
  --operation Write \
  --topic payments

# Grant consumer permission to read from payments topic
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:consumer \
  --operation Read \
  --topic payments

# Grant consumer permission to use consumer group
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:consumer \
  --operation Read \
  --group payment-processor

# List ACLs for topic
kafka-acls.sh --bootstrap-server localhost:9092 \
  --list \
  --topic payments

# Output:
# User:producer has Write permission on topic payments from host *
# User:consumer has Read permission on topic payments from host *
```

**Wildcard ACLs**:
```bash
# Grant read on all topics starting with "payments-"
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:analytics \
  --operation Read \
  --topic payments- \
  --resource-pattern-type prefixed

# Grant admin all permissions on all resources
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:admin \
  --operation All \
  --topic * \
  --cluster
```

**Deny ACLs** (override allow):
```bash
# Deny producer from deleting topics (even if has All permission)
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --deny-principal User:producer \
  --operation Delete \
  --topic *
```

**Broker Configuration**:
```properties
# Enable ACL authorization
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# Super users (bypass ACLs)
super.users=User:admin;User:kafka

# Allow everyone if no ACL found (default: false for security)
allow.everyone.if.no.acl.found=false
```

### Banking ACL Design

```bash
# Payment Production Team
# - Write to payments-* topics
# - Read from payments-* (for debugging)
kafka-acls.sh --add --allow-principal User:payment-producer \
  --operation Write --topic payments- --resource-pattern-type prefixed

kafka-acls.sh --add --allow-principal User:payment-producer \
  --operation Read --topic payments- --resource-pattern-type prefixed

# Fraud Detection Team
# - Read from payments-* topics
# - Write to fraud-alerts topic
kafka-acls.sh --add --allow-principal User:fraud-detector \
  --operation Read --topic payments- --resource-pattern-type prefixed

kafka-acls.sh --add --allow-principal User:fraud-detector \
  --operation Write --topic fraud-alerts

kafka-acls.sh --add --allow-principal User:fraud-detector \
  --operation Read --group fraud-detection-consumer

# Analytics Team
# - Read from all topics (read-only)
kafka-acls.sh --add --allow-principal User:analytics \
  --operation Read --topic * --resource-pattern-type literal

kafka-acls.sh --add --allow-principal User:analytics \
  --operation Read --group analytics-consumer

# Audit Team
# - Read from audit-logs topic only
kafka-acls.sh --add --allow-principal User:auditor \
  --operation Read --topic audit-logs

kafka-acls.sh --add --allow-principal User:auditor \
  --operation Read --group audit-consumer

# Platform Team (Admin)
# - All permissions on all resources
kafka-acls.sh --add --allow-principal User:platform-admin \
  --operation All --topic * --cluster
```

---

## Encryption

### Encryption In-Transit (TLS)

**TLS** encrypts data between clients and brokers, and between brokers.

**Configuration**:

**Broker**:
```properties
listeners=SSL://0.0.0.0:9093
security.inter.broker.protocol=SSL

ssl.keystore.location=/var/private/ssl/kafka.broker.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/var/private/ssl/kafka.truststore.jks
ssl.truststore.password=truststore-password

# TLS versions (disable old versions)
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.protocol=TLSv1.2

# Cipher suites (strong ciphers only)
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

**Client**:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9093");
props.put("security.protocol", "SSL");

props.put("ssl.truststore.location", "/path/to/truststore.jks");
props.put("ssl.truststore.password", "truststore-password");

// Optional: Client certificate (for mTLS)
props.put("ssl.keystore.location", "/path/to/client.keystore.jks");
props.put("ssl.keystore.password", "keystore-password");
props.put("ssl.key.password", "key-password");
```

**Performance Impact**:
```
Without TLS:
- Throughput: 100,000 msg/sec
- Latency: 5ms

With TLS:
- Throughput: 80,000 msg/sec (-20%)
- Latency: 7ms (+2ms)
- CPU overhead: +15%

Trade-off: 20% performance overhead for encryption (acceptable for compliance)
```

### Encryption At-Rest

Kafka does not natively support at-rest encryption. Use **OS-level encryption**:

**Option 1: LUKS (Linux Unified Key Setup)**:
```bash
# Encrypt disk partition
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 kafka-data

# Create filesystem
mkfs.ext4 /dev/mapper/kafka-data

# Mount encrypted partition
mount /dev/mapper/kafka-data /var/kafka-logs

# Kafka writes to encrypted disk (transparent)
```

**Option 2: AWS EBS Encryption**:
```bash
# Create encrypted EBS volume
aws ec2 create-volume --size 1000 --encrypted --kms-key-id <key-id>

# Attach to EC2 instance
# Mount to /var/kafka-logs
# Kafka writes to encrypted volume (transparent)
```

**Option 3: Application-Level Encryption** (not recommended):
```java
// Encrypt message before sending (complex, avoid if possible)
String plaintext = "sensitive-data";
String encrypted = encrypt(plaintext, encryptionKey);
producer.send(new ProducerRecord<>("payments", encrypted));

// Consumer decrypts
String encrypted = record.value();
String plaintext = decrypt(encrypted, encryptionKey);

// Cons:
// - Key management complexity
// - Can't use Kafka features (compaction, search)
// - Performance overhead
```

**Recommendation**: Use OS-level encryption (LUKS, EBS encryption) for at-rest encryption

---

## Audit Logging

Audit logs track all security events (authentication, authorization failures, config changes).

### Enable Audit Logging

**Broker Configuration**:
```properties
# Enable authorizer audit logs
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# Log all authorization decisions
log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender

# Separate log file for audit
log4j.appender.authorizerAppender=org.apache.log4j.RollingFileAppender
log4j.appender.authorizerAppender.File=/var/log/kafka/kafka-authorizer.log
log4j.appender.authorizerAppender.MaxFileSize=100MB
log4j.appender.authorizerAppender.MaxBackupIndex=10
log4j.appender.authorizerAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.authorizerAppender.layout.ConversionPattern=[%d] %p %m (%c)%n
```

**Sample Audit Log**:
```
[2024-01-15 10:30:45] INFO Principal User:producer is Allowed Operation Write from host 192.168.1.100 on resource Topic:payments (kafka.authorizer.logger)

[2024-01-15 10:30:50] INFO Principal User:consumer is Allowed Operation Read from host 192.168.1.101 on resource Topic:payments (kafka.authorizer.logger)

[2024-01-15 10:31:00] WARN Principal User:hacker is Denied Operation Write from host 10.0.0.50 on resource Topic:payments (kafka.authorizer.logger)
```

**Parse Audit Logs**:
```bash
# Find all denied operations
grep "Denied" /var/log/kafka/kafka-authorizer.log

# Find all operations by specific user
grep "User:consumer" /var/log/kafka/kafka-authorizer.log

# Find all operations on payments topic
grep "Topic:payments" /var/log/kafka/kafka-authorizer.log

# Export to SIEM (Splunk, Elasticsearch)
tail -f /var/log/kafka/kafka-authorizer.log | \
  logstash -f kafka-audit-pipeline.conf
```

### Custom Audit Logger

```java
/**
 * Custom authorizer with detailed audit logging
 */
public class AuditAuthorizer extends AclAuthorizer {

    private static final Logger AUDIT_LOG = LoggerFactory.getLogger("AUDIT");

    @Override
    public List<AuthorizationResult> authorize(
        AuthorizableRequestContext requestContext,
        List<Action> actions
    ) {
        List<AuthorizationResult> results = super.authorize(requestContext, actions);

        // Log all authorization decisions
        for (int i = 0; i < actions.size(); i++) {
            Action action = actions.get(i);
            AuthorizationResult result = results.get(i);

            AuditEvent event = AuditEvent.builder()
                .timestamp(Instant.now())
                .principal(requestContext.principal().getName())
                .sourceIp(requestContext.clientAddress().getHostAddress())
                .operation(action.operation().name())
                .resourceType(action.resourcePattern().resourceType().name())
                .resourceName(action.resourcePattern().name())
                .decision(result == AuthorizationResult.ALLOWED ? "ALLOWED" : "DENIED")
                .build();

            AUDIT_LOG.info(event.toJson());

            // Send to audit service (async)
            auditService.record(event);
        }

        return results;
    }
}
```

---

## Compliance

### PCI-DSS (Payment Card Industry)

**Requirements**:
1. **Requirement 2.2**: Encrypt all cardholder data in transit (TLS)
2. **Requirement 3.4**: Encrypt cardholder data at rest (disk encryption)
3. **Requirement 8**: Identify and authenticate all access (SASL/mTLS)
4. **Requirement 10**: Track and monitor all access to cardholder data (audit logs)

**Kafka Configuration for PCI Compliance**:
```properties
# Encryption in-transit (TLS)
listeners=SSL://0.0.0.0:9093
ssl.enabled.protocols=TLSv1.2,TLSv1.3

# Authentication (SASL/SCRAM or mTLS)
security.inter.broker.protocol=SASL_SSL
sasl.enabled.mechanisms=SCRAM-SHA-256

# Authorization (ACLs)
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false

# Audit logging
log4j.logger.kafka.authorizer.logger=INFO,authorizerAppender

# Encryption at-rest (OS-level)
# Use LUKS or EBS encryption for /var/kafka-logs
```

### GDPR (General Data Protection Regulation)

**Right to Erasure** (delete customer data):

**Challenge**: Kafka is append-only (can't delete individual messages)

**Solutions**:

**1. Logical Deletion** (recommended):
```java
// Send tombstone (null value)
producer.send(new ProducerRecord<>("customers", customerId, null));

// Enable log compaction
cleanup.policy=compact
delete.retention.ms=86400000  // 24 hours

// After 24 hours, compaction removes customer record
```

**2. Topic TTL** (time-based deletion):
```properties
# Auto-delete customer data after 90 days
retention.ms=7776000000  # 90 days
# Entire partition segment deleted after 90 days
```

**3. Separate PII Topic** (easier compliance):
```
Topic: payments (no PII, long retention)
Topic: customer-pii (PII only, short retention)

# GDPR deletion: Delete from customer-pii topic only
# Payments topic retains transaction history (anonymized)
```

**Data Minimization**:
```java
// Don't store unnecessary PII
Payment payment = Payment.newBuilder()
    .setPaymentId("PAY-123")
    .setAmount(100.00)
    .setCustomerIdHash(sha256(customerId))  // Hash instead of plaintext
    .build();
```

### SOX (Sarbanes-Oxley)

**Requirements**:
- **Audit Trail**: All financial transactions logged (Kafka audit logs)
- **Access Control**: Segregation of duties (ACLs)
- **Data Retention**: 7-year retention for audit logs

**Configuration**:
```properties
# Audit log topic (7-year retention)
retention.ms=220752000000  # 7 years
cleanup.policy=delete  # No compaction (retain all events)

# ACLs: Separate producer and consumer roles
# - payment-producer: Write to payments topic
# - payment-consumer: Read from payments topic
# - auditor: Read from audit-logs topic (no write)
```

---

## Interview Questions

### Question 1: Design a secure Kafka architecture for a payment processing system that must comply with PCI-DSS. What security controls would you implement?

**Answer**:

**Architecture**:
```
┌──────────────────────────────────────────────────────────┐
│              Secure Kafka Architecture                    │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  1. Authentication: SASL/SCRAM + mTLS                    │
│     - Services: mTLS (certificate per service)           │
│     - Users: SASL/SCRAM (Active Directory integration)   │
│                                                           │
│  2. Authorization: ACLs                                   │
│     - payment-producer: Write to payments topic          │
│     - fraud-detector: Read from payments, write alerts   │
│     - analytics: Read-only (no PII topics)               │
│                                                           │
│  3. Encryption In-Transit: TLS 1.2/1.3                   │
│     - All client-broker communication                     │
│     - All broker-broker communication                     │
│                                                           │
│  4. Encryption At-Rest: EBS Encryption                    │
│     - AWS KMS for key management                          │
│     - All Kafka data volumes encrypted                    │
│                                                           │
│  5. Audit Logging                                         │
│     - All authorization decisions logged                  │
│     - Exported to SIEM (Splunk)                          │
│     - 7-year retention (compliance)                       │
│                                                           │
│  6. Network Isolation                                     │
│     - Kafka in private subnet (no internet access)       │
│     - VPN/bastion for admin access                       │
│     - Security groups: Allow 9093 (SSL) only             │
└──────────────────────────────────────────────────────────┘
```

**Configuration**:
```properties
# Broker
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL
sasl.enabled.mechanisms=SCRAM-SHA-256
ssl.client.auth=required
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false
```

**PCI Compliance Checklist**:
- ✓ Encrypt in-transit (TLS)
- ✓ Encrypt at-rest (EBS)
- ✓ Authenticate all access (SASL/mTLS)
- ✓ Audit all access (logs)
- ✓ Restrict access (ACLs)
- ✓ Network isolation (private subnet)

---

### Question 2: Explain the trade-offs between SASL/SCRAM and mTLS for authentication. When would you use each?

**Answer**:

**SASL/SCRAM**:
- **Pros**: Dynamic user management (add/remove without restart), password-based (familiar)
- **Cons**: Password management overhead, less secure than certificates
- **Use Case**: Internal services with many users (easier to manage)

**mTLS**:
- **Pros**: Strong authentication (cryptographic), no password management
- **Cons**: Certificate management overhead, rotation complexity
- **Use Case**: Service-to-service authentication (limited number of services)

**Banking Recommendation**:
```
mTLS: Payment services (10-20 microservices, long-lived certificates)
SASL/SCRAM: Human users (developers, analysts, 100+ users, passwords)

Hybrid: mTLS for services, SASL/SCRAM for humans (best of both)
```

---

### Question 3: How would you handle GDPR "right to erasure" requests in Kafka given its append-only design?

**Answer**:

**Challenges**:
- Kafka is append-only (can't delete individual messages)
- Log compaction eventual (may take hours)
- Historical data in older segments

**Solution: Logical Deletion + Topic Design**

**1. Separate PII Topic**:
```
Topic: payments (no PII, long retention)
  - paymentId, amount, timestamp

Topic: customer-data (PII, short retention)
  - customerId, name, email, address

Relationship: payments.customerId → customer-data.customerId
```

**2. GDPR Deletion Process**:
```java
// Step 1: Send tombstone to customer-data topic
producer.send(new ProducerRecord<>("customer-data", customerId, null));

// Step 2: Enable compaction
cleanup.policy=compact
delete.retention.ms=86400000  // 24 hours

// Step 3: Wait for compaction (24 hours)
// Step 4: Verify deletion
consumer.poll() // customerId returns null

// Result:
// - PII deleted from customer-data topic
// - Payments topic retains transaction history (no PII)
```

**3. Alternative: Encryption with Key Deletion**:
```java
// Encrypt PII with customer-specific key
String encryptedEmail = encrypt(email, customerKey);
producer.send(new ProducerRecord<>("customers", customerId, encryptedEmail));

// GDPR deletion: Delete customer key from key vault
keyVault.delete(customerKey);

// Result: Encrypted data remains, but unreadable (cryptographic erasure)
```

**Recommendation**: Use separate PII topic + logical deletion (simplest, compliant)

---

## Summary

Kafka security requires layered defense: authentication, authorization, encryption, and audit logging. Key takeaways:

1. **Authentication**: SASL/SCRAM (users), mTLS (services)
2. **Authorization**: ACLs (least privilege), deny by default
3. **Encryption**: TLS in-transit, OS-level at-rest
4. **Audit Logging**: All security events logged, exported to SIEM
5. **Compliance**: PCI-DSS (encryption), GDPR (logical deletion), SOX (audit retention)

In banking systems, security is non-negotiable. Understanding these mechanisms enables building compliant, auditable payment processing systems that meet regulatory requirements.

**Word Count**: ~5,500 words
