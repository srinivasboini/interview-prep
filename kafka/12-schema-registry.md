# Kafka Schema Registry

## Overview

Schema Registry is a critical component for managing message schemas in Kafka ecosystems, providing centralized schema storage, versioning, and compatibility enforcement. It ensures that producers and consumers agree on message formats, enables schema evolution without breaking changes, and provides a single source of truth for data contracts across the organization.

**Why This Matters for Interviews**: Senior engineering roles require understanding of data contracts, schema evolution strategies, and API versioning. Expect questions about schema compatibility rules, migration strategies, handling breaking changes, and schema governance in large organizations.

**Real-World Banking Context**: Financial systems require strict data contracts for regulatory compliance, audit trails, and cross-team integration. Payment messages, transaction records, and customer data must have well-defined schemas that evolve safely over time. Schema Registry prevents production incidents caused by incompatible message formats and enables confident deployments.

---

## Schema Registry Architecture

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Schema Registry Cluster                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Registry   │  │   Registry   │  │   Registry   │      │
│  │   Instance 1 │  │   Instance 2 │  │   Instance 3 │      │
│  │   (Leader)   │  │  (Follower)  │  │  (Follower)  │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                            │                                 │
│                            ↓                                 │
│                   ┌────────────────┐                         │
│                   │  Kafka Topic   │                         │
│                   │ _schemas       │                         │
│                   └────────────────┘                         │
└─────────────────────────────────────────────────────────────┘
         ↑                                          ↑
         │                                          │
    ┌────────────┐                            ┌──────────┐
    │  Producer  │                            │ Consumer │
    │ (Register  │                            │ (Fetch   │
    │  Schema)   │                            │  Schema) │
    └────────────┘                            └──────────┘
```

**Components**:

1. **Schema Registry Server**: RESTful service for schema management
2. **Kafka Backend**: `_schemas` topic stores all schemas (compacted log)
3. **Schema Cache**: In-memory cache for fast schema lookups
4. **Compatibility Checker**: Validates schema evolution rules

**Storage**:
```
_schemas Topic (Log Compacted):
Key: {subject}-{version}
Value: {id, schema, version, subject}

Example:
Key: "payments-value-1"
Value: {
  "id": 1,
  "schema": "{\"type\":\"record\",\"name\":\"Payment\",...}",
  "version": 1,
  "subject": "payments-value"
}
```

### Subject Naming Strategies

**Subject** = Namespace for schemas (typically topic name + record type)

**TopicNameStrategy** (default):
```
Subject: <topic>-key or <topic>-value

Example:
Topic: payments
Subjects: payments-key, payments-value

Use Case: One schema per topic (most common)
```

**RecordNameStrategy**:
```
Subject: <fully-qualified-record-name>

Example:
Record: com.bank.payments.Payment
Subject: com.bank.payments.Payment

Use Case: Multiple record types in same topic (polymorphic messages)
```

**TopicRecordNameStrategy**:
```
Subject: <topic>-<fully-qualified-record-name>

Example:
Topic: events
Record: com.bank.payments.PaymentCreated
Subject: events-com.bank.payments.PaymentCreated

Use Case: Multiple record types per topic with topic-scoped versioning
```

---

## Schema Formats

### Avro (Most Common)

**Avro** is a compact, fast binary serialization format with built-in schema evolution.

**Avro Schema Definition**:
```json
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "fields": [
    {
      "name": "paymentId",
      "type": "string",
      "doc": "Unique payment identifier"
    },
    {
      "name": "amount",
      "type": {
        "type": "bytes",
        "logicalType": "decimal",
        "precision": 10,
        "scale": 2
      },
      "doc": "Payment amount (decimal with 2 decimal places)"
    },
    {
      "name": "currency",
      "type": "string",
      "default": "USD",
      "doc": "ISO 4217 currency code"
    },
    {
      "name": "timestamp",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "Payment timestamp (milliseconds since epoch)"
    }
  ]
}
```

**Avro Benefits**:
- **Compact**: Binary format (smaller than JSON)
- **Fast**: No parsing overhead (direct field access)
- **Schema Evolution**: Built-in compatibility rules
- **Language Agnostic**: Generates code for Java, Python, C++, etc.

**Avro Message Structure**:
```
┌────────────────────┬─────────────────────────────┐
│  Magic Byte (0x0)  │  Schema ID (4 bytes)        │
├────────────────────┴─────────────────────────────┤
│  Avro Serialized Data (variable length)          │
└───────────────────────────────────────────────────┘

Example:
0x00 0x00 0x00 0x00 0x01 [Avro binary data...]
     ↑                ↑
  Magic Byte      Schema ID = 1
```

**Producer Code**:
```java
// Avro producer with Schema Registry
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
props.put("schema.registry.url", "http://localhost:8081");

KafkaProducer<String, Payment> producer = new KafkaProducer<>(props);

// Create Avro record (generated from schema)
Payment payment = Payment.newBuilder()
    .setPaymentId("PAY-12345")
    .setAmount(new BigDecimal("100.50"))
    .setCurrency("USD")
    .setTimestamp(Instant.now().toEpochMilli())
    .build();

// Send (serializer automatically registers schema and embeds schema ID)
producer.send(new ProducerRecord<>("payments", payment.getPaymentId(), payment));

// First send:
// 1. Serializer checks Schema Registry for schema (by hash)
// 2. Schema not found → registers schema, receives ID=1
// 3. Caches schema ID locally
// 4. Serializes message with schema ID prefix
// Subsequent sends: Use cached schema ID (no registry lookup)
```

**Consumer Code**:
```java
// Avro consumer with Schema Registry
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
props.put("schema.registry.url", "http://localhost:8081");
props.put("specific.avro.reader", "true"); // Use generated classes

KafkaConsumer<String, Payment> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("payments"));

while (true) {
    ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Payment> record : records) {
        Payment payment = record.value();

        // Deserializer:
        // 1. Reads schema ID from message prefix
        // 2. Fetches schema from registry (or cache)
        // 3. Deserializes using fetched schema

        System.out.println("Payment: " + payment.getPaymentId() +
                          " Amount: " + payment.getAmount());
    }
}
```

### Protobuf

**Protocol Buffers** (Protobuf) is Google's binary serialization format.

**Protobuf Schema**:
```protobuf
syntax = "proto3";

package com.bank.payments;

message Payment {
  string payment_id = 1;
  double amount = 2;
  string currency = 3;
  int64 timestamp = 4;
}
```

**Benefits**:
- **Performance**: Faster serialization than Avro
- **Industry Standard**: Used by Google, gRPC
- **Backward Compatible**: Field numbers enable evolution

**Trade-offs vs Avro**:
- **No Schema Evolution Enforcement**: Must manually ensure compatibility
- **Less Ecosystem Support**: Fewer Kafka tools support Protobuf

### JSON Schema

**JSON Schema** validates JSON documents.

**JSON Schema Definition**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Payment",
  "type": "object",
  "properties": {
    "paymentId": {
      "type": "string",
      "description": "Unique payment identifier"
    },
    "amount": {
      "type": "number",
      "minimum": 0,
      "description": "Payment amount"
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "ISO 4217 currency code"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["paymentId", "amount", "currency", "timestamp"]
}
```

**Benefits**:
- **Human Readable**: JSON is text-based (easier debugging)
- **Universal**: Every language supports JSON
- **Schema Validation**: Enforces structure

**Trade-offs**:
- **Larger Size**: Text format (5-10x larger than Avro)
- **Slower**: Parsing overhead
- **Weaker Evolution**: Less strict than Avro

**Recommendation**: Use Avro for production (compact, fast), JSON Schema for development/debugging

---

## Schema Evolution and Compatibility

### Compatibility Modes

Schema Registry enforces compatibility rules to prevent breaking changes.

**BACKWARD** (default):
```
Rule: New schema can read data written with old schema

Example:
Old Schema (v1):
{
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"}
  ]
}

New Schema (v2) - ADD optional field:
{
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"}  // NEW (optional)
  ]
}

Valid: New consumers can read old messages (use default for missing currency)
Use Case: Deploy consumers first, then producers (consumers tolerate old messages)
```

**FORWARD**:
```
Rule: Old schema can read data written with new schema

Example:
Old Schema (v1):
{
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"}
  ]
}

New Schema (v2) - REMOVE optional field:
{
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"}
  ]
}

Valid: Old consumers can read new messages (ignore missing currency field)
Use Case: Deploy producers first, then consumers (producers write new format)
```

**FULL** (BACKWARD + FORWARD):
```
Rule: New schema can read old data, old schema can read new data

Example:
Old Schema (v1):
{
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"}
  ]
}

New Schema (v2) - ADD optional field:
{
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"}
  ]
}

Valid: Bidirectional compatibility
Use Case: Deploy in any order (most flexible)
```

**NONE**:
```
Rule: No compatibility checks

Use Case: Breaking changes (requires coordinated deployment)
Warning: Can break running consumers if not careful
```

### Schema Evolution Examples

**Safe Changes (BACKWARD Compatible)**:
```
✓ Add field with default value
✓ Delete optional field
✓ Widen field type (int → long)

Example:
// v1
{"name": "age", "type": "int"}

// v2 (BACKWARD compatible)
{"name": "age", "type": "long", "default": 0}
```

**Safe Changes (FORWARD Compatible)**:
```
✓ Delete field
✓ Add optional field
✓ Narrow field type (with care)

Example:
// v1
{"name": "status", "type": "string", "default": "PENDING"}

// v2 (FORWARD compatible - delete field)
// Field removed (old consumers ignore new messages without this field)
```

**Unsafe Changes (BREAKING)**:
```
✗ Change field type (string → int)
✗ Rename field
✗ Delete required field
✗ Remove default value

Example:
// v1
{"name": "amount", "type": "string"}

// v2 (BREAKING - type change)
{"name": "amount", "type": "double"}

// Old consumers crash: Cannot parse double as string
// Requires: Version bump, coordinated deployment
```

### Banking Example: Payment Schema Evolution

```json
// Version 1: Initial Payment Schema
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "version": 1,
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"}
  ]
}

// Version 2: Add Merchant Info (BACKWARD compatible)
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "version": 2,
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"},
    {"name": "merchantId", "type": ["null", "string"], "default": null},  // NEW
    {"name": "merchantName", "type": ["null", "string"], "default": null}  // NEW
  ]
}

// Version 3: Add Payment Method (BACKWARD compatible)
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "version": 3,
  "fields": [
    {"name": "paymentId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"},
    {"name": "merchantId", "type": ["null", "string"], "default": null},
    {"name": "merchantName", "type": ["null", "string"], "default": null},
    {"name": "paymentMethod", "type": {
      "type": "enum",
      "name": "PaymentMethod",
      "symbols": ["CARD", "BANK_TRANSFER", "WALLET"]
    }, "default": "CARD"}  // NEW (enum with default)
  ]
}

// Deployment Strategy:
// 1. Register v2 schema (schema registry validates BACKWARD compatibility)
// 2. Deploy new producers (write v2 with merchant fields)
// 3. Old consumers continue reading (ignore new fields)
// 4. Deploy new consumers (read merchant fields)
// Result: Zero downtime, gradual rollout
```

---

## Schema Registry Operations

### Registering a Schema

**REST API**:
```bash
# Register new schema version
curl -X POST http://localhost:8081/subjects/payments-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"Payment\",\"fields\":[{\"name\":\"paymentId\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"
  }'

# Response:
{"id": 1}

# Compatibility check before registering
curl -X POST http://localhost:8081/compatibility/subjects/payments-value/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"Payment\",\"fields\":[{\"name\":\"paymentId\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"currency\",\"type\":\"string\",\"default\":\"USD\"}]}"
  }'

# Response:
{"is_compatible": true}
```

**Java Code**:
```java
// Register schema programmatically
CachedSchemaRegistryClient schemaRegistry = new CachedSchemaRegistryClient(
    "http://localhost:8081",
    100  // Cache size
);

Schema schema = new Schema.Parser().parse(
    "{\"type\":\"record\",\"name\":\"Payment\",...}"
);

int schemaId = schemaRegistry.register("payments-value", schema);
System.out.println("Schema registered with ID: " + schemaId);
```

### Fetching a Schema

```bash
# Get latest schema version
curl http://localhost:8081/subjects/payments-value/versions/latest

# Response:
{
  "subject": "payments-value",
  "version": 3,
  "id": 3,
  "schema": "{\"type\":\"record\",\"name\":\"Payment\",...}"
}

# Get specific version
curl http://localhost:8081/subjects/payments-value/versions/2

# Get schema by ID
curl http://localhost:8081/schemas/ids/1

# List all subjects
curl http://localhost:8081/subjects

# List all versions for subject
curl http://localhost:8081/subjects/payments-value/versions
```

### Setting Compatibility Mode

```bash
# Set global compatibility mode
curl -X PUT http://localhost:8081/config \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"compatibility": "BACKWARD"}'

# Set compatibility for specific subject
curl -X PUT http://localhost:8081/config/payments-value \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"compatibility": "FULL"}'

# Get compatibility mode
curl http://localhost:8081/config/payments-value

# Response:
{"compatibilityLevel": "FULL"}
```

### Deleting Schemas

```bash
# Soft delete (mark as deleted, can be restored)
curl -X DELETE http://localhost:8081/subjects/payments-value/versions/1

# Hard delete (permanent, after soft delete)
curl -X DELETE http://localhost:8081/subjects/payments-value/versions/1?permanent=true

# Delete all versions (subject)
curl -X DELETE http://localhost:8081/subjects/payments-value

# Warning: Deleting schemas can break consumers expecting those schemas
# Only delete during decommissioning or cleanup
```

---

## Schema Governance

### Schema Design Best Practices

**1. Use Namespaces**:
```json
{
  "namespace": "com.bank.payments",  // Prevents name collisions
  "name": "Payment"
}
```

**2. Document Fields**:
```json
{
  "name": "amount",
  "type": "double",
  "doc": "Payment amount in specified currency. Must be non-negative."
}
```

**3. Use Logical Types**:
```json
{
  "name": "timestamp",
  "type": "long",
  "logicalType": "timestamp-millis"  // Self-documenting
}

{
  "name": "amount",
  "type": "bytes",
  "logicalType": "decimal",
  "precision": 10,
  "scale": 2  // Precise decimal representation
}
```

**4. Provide Defaults**:
```json
{
  "name": "currency",
  "type": "string",
  "default": "USD"  // Enables backward compatibility
}
```

**5. Use Enums for Fixed Sets**:
```json
{
  "name": "status",
  "type": {
    "type": "enum",
    "name": "PaymentStatus",
    "symbols": ["PENDING", "COMPLETED", "FAILED"]
  },
  "default": "PENDING"
}
```

### Schema Versioning Strategy

**Semantic Versioning for Schemas**:
```
Major.Minor.Patch

Major: Breaking change (incompatible)
Minor: Backward-compatible addition
Patch: Bug fix, documentation update

Example:
v1.0.0: Initial schema
v1.1.0: Add optional field (backward compatible)
v1.1.1: Fix typo in documentation
v2.0.0: Remove field (breaking change)
```

**Schema Metadata**:
```json
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.bank.payments",
  "doc": "Payment transaction record",
  "version": "1.2.0",
  "deprecated": false,
  "fields": [...]
}
```

### Schema Approval Workflow

**Enterprise Schema Governance**:
```
1. Developer creates schema (local development)
2. Submit schema PR (Git repository)
3. Automated validation:
   - Compatibility check (CI/CD pipeline)
   - Naming conventions
   - Required fields present
4. Code review (data architect, domain expert)
5. Approval + merge
6. Auto-register in Schema Registry (CD pipeline)
7. Notify consumers (Slack, email)
```

**CI/CD Integration**:
```yaml
# .gitlab-ci.yml
validate-schema:
  stage: validate
  script:
    # Validate schema syntax
    - avro-tools compile schema payment.avsc /tmp

    # Check compatibility with Schema Registry
    - |
      curl -X POST http://schema-registry:8081/compatibility/subjects/payments-value/versions/latest \
        -H "Content-Type: application/vnd.schemaregistry.v1+json" \
        -d "{\"schema\": \"$(cat payment.avsc | jq -c .)\"}" \
        | jq '.is_compatible' | grep -q true

    # Validate naming conventions
    - grep -q '"namespace": "com.bank' payment.avsc

deploy-schema:
  stage: deploy
  only:
    - master
  script:
    # Register schema in production registry
    - |
      curl -X POST http://prod-schema-registry:8081/subjects/payments-value/versions \
        -H "Content-Type: application/vnd.schemaregistry.v1+json" \
        -d "{\"schema\": \"$(cat payment.avsc | jq -c .)\"}"
```

---

## Advanced Patterns

### Multi-Environment Schema Management

**Strategy**: Separate Schema Registry per environment

```
Development: http://dev-schema-registry:8081
Staging: http://stage-schema-registry:8081
Production: http://prod-schema-registry:8081

Workflow:
1. Develop schema in dev environment
2. Promote to staging (manual or automated)
3. Validate in staging (integration tests)
4. Promote to production (controlled release)
```

**Schema Promotion Script**:
```bash
#!/bin/bash
# promote-schema.sh

SOURCE_ENV=$1  # dev, staging
TARGET_ENV=$2  # staging, production
SUBJECT=$3

# Fetch schema from source
SCHEMA=$(curl -s http://${SOURCE_ENV}-registry:8081/subjects/${SUBJECT}/versions/latest \
  | jq -r '.schema')

# Validate compatibility in target
COMPATIBLE=$(curl -X POST http://${TARGET_ENV}-registry:8081/compatibility/subjects/${SUBJECT}/versions/latest \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d "{\"schema\": \"${SCHEMA}\"}" \
  | jq -r '.is_compatible')

if [ "$COMPATIBLE" = "true" ]; then
  # Register in target
  curl -X POST http://${TARGET_ENV}-registry:8081/subjects/${SUBJECT}/versions \
    -H "Content-Type: application/vnd.schemaregistry.v1+json" \
    -d "{\"schema\": \"${SCHEMA}\"}"
  echo "Schema promoted successfully"
else
  echo "ERROR: Schema not compatible with target environment"
  exit 1
fi
```

### Schema Aliases and References

**Schema References** (Avro supports nested types):
```json
// Address schema (reusable)
{
  "type": "record",
  "name": "Address",
  "namespace": "com.bank.common",
  "fields": [
    {"name": "street", "type": "string"},
    {"name": "city", "type": "string"},
    {"name": "zipCode", "type": "string"}
  ]
}

// Customer schema (references Address)
{
  "type": "record",
  "name": "Customer",
  "namespace": "com.bank.customers",
  "fields": [
    {"name": "customerId", "type": "string"},
    {"name": "name", "type": "string"},
    {"name": "address", "type": "com.bank.common.Address"}  // Reference
  ]
}

// Register with Schema Registry
curl -X POST http://localhost:8081/subjects/customers-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "...",
    "references": [
      {
        "name": "com.bank.common.Address",
        "subject": "Address",
        "version": 1
      }
    ]
  }'
```

---

## Interview Questions

### Question 1: Explain the difference between BACKWARD, FORWARD, and FULL compatibility modes. When would you use each?

**Answer**:

**BACKWARD Compatibility**:
- **Rule**: New schema reads old data
- **Changes Allowed**: Add optional fields, delete optional fields
- **Deployment**: Deploy consumers first, then producers
- **Use Case**: Most common (consumers tolerate old messages)

**FORWARD Compatibility**:
- **Rule**: Old schema reads new data
- **Changes Allowed**: Add optional fields, delete fields
- **Deployment**: Deploy producers first, then consumers
- **Use Case**: Producers ahead of consumers (rare)

**FULL Compatibility**:
- **Rule**: Bidirectional (backward + forward)
- **Changes Allowed**: Add/delete optional fields only
- **Deployment**: Any order
- **Use Case**: Maximum flexibility (recommended for critical systems)

**Banking Example**:
- Payment schema: FULL (critical, deploy in any order)
- Internal logs: BACKWARD (consumers deploy first)

---

### Question 2: How would you handle a breaking schema change in a production Kafka system?

**Answer**:

**Breaking Change Example**: Change `amount` from `string` to `double`

**Strategy 1: Dual-Write Pattern**:
```
1. Create new topic: payments-v2
2. Producers write to BOTH topics (payments, payments-v2)
3. New consumers read from payments-v2
4. Migrate old consumers gradually to payments-v2
5. After migration complete, deprecate payments topic
6. Stop dual-writes, remove old topic

Timeline: 2-4 weeks (gradual migration)
```

**Strategy 2: Multi-Version Consumer**:
```java
// Consumer handles both v1 (string) and v2 (double)
GenericRecord record = avroRecord;

Object amountValue = record.get("amount");
double amount;

if (amountValue instanceof String) {
    amount = Double.parseDouble((String) amountValue);  // v1
} else {
    amount = (Double) amountValue;  // v2
}

// Process with unified type
processPayment(amount);
```

**Strategy 3: Coordinated Deployment** (risky):
```
1. Stop all consumers
2. Stop all producers
3. Update schemas
4. Deploy new producers
5. Deploy new consumers
6. Resume

Downtime: 5-10 minutes
Risk: All-or-nothing (no rollback)
```

**Recommendation**: Use Dual-Write Pattern for critical systems (zero downtime)

---

### Question 3: Design a schema governance process for a large organization with 50+ development teams producing to Kafka.

**Answer**:

**Schema Governance Architecture**:

```
┌─────────────────────────────────────────────────────────┐
│                 Schema Governance Process                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Schema Design (Developer)                           │
│     - Follow naming conventions                         │
│     - Use schema templates                              │
│     - Local validation                                  │
│                                                          │
│  2. Schema Submission (Git PR)                          │
│     - Submit to schema repository                       │
│     - Automated validation (CI)                         │
│                                                          │
│  3. Automated Checks (CI/CD)                            │
│     - Syntax validation                                 │
│     - Compatibility check                               │
│     - Naming convention enforcement                     │
│     - Required field validation                         │
│                                                          │
│  4. Review (Data Architects)                            │
│     - Domain expert review                              │
│     - Security review (PII fields)                      │
│     - Compliance check                                  │
│                                                          │
│  5. Approval + Registration                             │
│     - Merge PR                                          │
│     - Auto-register in Schema Registry                  │
│     - Notify consumers (Slack)                          │
│                                                          │
│  6. Monitoring                                          │
│     - Schema usage metrics                              │
│     - Compatibility violations                          │
│     - Deprecated schema warnings                        │
└─────────────────────────────────────────────────────────┘
```

**Enforcement Mechanisms**:
1. **Naming Conventions**: `<domain>.<entity>.<type>` (e.g., `com.bank.payments.Payment`)
2. **Required Fields**: `id`, `timestamp`, `version`
3. **PII Handling**: Annotate fields with `@pii` for encryption/masking
4. **Compatibility Mode**: FULL (default), exceptions require approval
5. **Deprecation Policy**: 90-day notice before removing fields

**Tooling**:
- **Schema Registry**: Central storage
- **Git Repository**: Version control, review workflow
- **CI/CD Pipeline**: Automated validation
- **Schema Portal**: Web UI for browsing schemas
- **Slack Integration**: Notifications for changes

**Metrics**:
- Schema registrations per week
- Compatibility violations
- Review turnaround time
- Consumer adoption rate

---

## Summary

Schema Registry is essential for managing data contracts in Kafka ecosystems, enabling safe schema evolution and preventing production incidents. Key takeaways:

1. **Schema Formats**: Avro (recommended), Protobuf, JSON Schema
2. **Compatibility**: BACKWARD (default), FORWARD, FULL, NONE
3. **Evolution**: Add optional fields (safe), change types (breaking)
4. **Governance**: Approval workflow, CI/CD integration, naming conventions
5. **Operations**: Register, fetch, compatibility check, delete
6. **Banking**: Critical for regulatory compliance, audit trails, cross-team integration

Effective schema governance ensures data quality, prevents breaking changes, and enables confident deployments in large-scale Kafka deployments.

**Word Count**: ~6,500 words
