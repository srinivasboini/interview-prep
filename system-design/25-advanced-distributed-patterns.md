# Advanced Distributed Patterns

## Overview

As a Staff or Principal Engineer, you move beyond basic microservices and CRUD applications. You must design systems that handle massive concurrency, guarantee exactly-once processing (or synthesize it), and coordinate complex, multi-step workflows across independent networked systems without a centralized database lock.

The patterns listed here are the hallmarks of senior-level system design. They address the fundamental realities of distributed computing: the network is unreliable, nodes fail constantly, and distributed state is inherently messy.

---

## 1. The Saga Pattern (Distributed Transactions)

### The Problem
In a monolith, if a user books a flight, hotel, and rental car, you open a single database transaction. If the car rental fails (e.g., out of inventory), you simply call `ROLLBACK`, and the flight and hotel DB inserts are magically undone.
In microservices, the Flight Service (Postgres), Hotel Service (MongoDB), and Car Service (Cassandra) do not share a database connection. You cannot use a standard ACID transaction.

### The Anti-Pattern: Two-Phase Commit (2PC)
A centralized coordinator asks all three databases to "Prepare" (lock their rows). If everyone says YES, it then broadcasts "Commit". 
*Why it fails at scale*: It relies on heavy, synchronous locking across the network. If the Car Service DB goes offline during the "Prepare" phase, the Flight and Hotel rows remain locked indefinitely, bringing the entire travel platform to a halt.

### The Modern Solution: Sagas
A Saga is a sequence of *local* transactions. Each service updates its own database and immediately publishes an event to trigger the next step. If a step fails, the Saga executes **Compensating Transactions** to undo the previous steps.

#### Orchestration vs. Choreography

1.  **Choreography (Event-Driven):**
    *   *How it works*: No central controller. Flight Service books the flight and emits `FlightBooked`. Hotel Service hears it, books the room, and emits `HotelBooked`. Car Service hears it, fails, and emits `CarBookingFailed`. The Flight and Hotel services hear the failure and execute their own local undo scripts.
    *   *Pros*: Decoupled, no single point of failure.
    *   *Cons*: Extremely hard to debug. You cannot easily ask "What is the status of Order #123?" because the state is scattered across 3 event logs. "Event Spaghettis" emerge.

2.  **Orchestration (The Controller):**
    *   *How it works*: A central "Saga Orchestrator" microservice manages the flow. It explicitly commands the Flight Service: `BookFlight()`. If it returns success, it commands the Hotel: `BookHotel()`. If the Car fails, the Orchestrator explicitly commands the Flight and Hotel to `CancelBooking()`.
    *   *Pros*: Explicit workflow definition. Single place to check the overall state. Easier to maintain complex, multi-step rollbacks.
    *   *Cons*: Risk of the Orchestrator becoming a monolithic bottleneck.

---

## 2. CQRS (Command Query Responsibility Segregation)

### The Problem
In a high-traffic system (like a banking ledger or Twitter feed), optimizing the database for Writes (heavy `INSERT` operations with complex foreign keys and constraints) fundamentally conflicts with optimizing for Reads (massive `SELECT` queries requiring complex JOINs, aggregations, and fast text search).
A traditional CRUD architecture forces a compromise, resulting in a database that does neither exceptionally well.

### The Solution: CQRS
Separate the Write Path (Command) from the Read Path (Query) completely, often using entirely different databases.

*   **Commands (Writes)**: Represent intent to change state (`TransferFunds`, `CreatePost`). Handled by the Command Service. Written to an optimized database (e.g., append-only event store or relational DB configured for high insert throughput).
*   **Queries (Reads)**: Represent requests for data (`GetBalance`, `GetNewsfeed`). Handled by the Query Service. Read from a highly denormalized database optimized for instant retrieval (e.g., Redis, Elasticsearch, or a materialized view in Postgres).

### The Synchronization (Eventual Consistency)
How does the Read DB stay updated?
*   When the Command Service saves a state change, it asynchronously publishes an Event (e.g., via Kafka or Change Data Capture/Debezium).
*   The Query Service consumes this event and updates its denormalized View.
*   *Trade-off*: Eventual Consistency. A user might update their profile picture, refresh the page, and see the old picture for 500ms while the Kafka event propagates to the Read DB.

---

## 3. Event Sourcing

### The Problem
Traditional databases store the *current state*. If a user's balance is \$50, the database stores `50`. You do not inherently know *how* they got to \$50 unless you build complex audit logging around the update logic, which is prone to bugs.

### The Solution: Event Sourcing
Instead of storing the current state, store the exact sequence of immutable events that led to the state. The Database becomes an append-only log.

*   **The Ledger**:
    1. `AccountCreated (Balance: 0)`
    2. `Deposited (Amount: 100)`
    3. `Withdrawn (Amount: 50)`
*   **Deriving State**: To calculate the current balance, you "replay" the events in order. ($0 + 100 - 50 = \$50$).
*   **Pros**:
    *   *Perfect Auditability*: You can literally rewind the system mathematically to see exactly what the balance was at 11:42 AM on Tuesday.
    *   *Time Travel Debugging*: If a bug is introduced on Wednesday, you can fix the bug, delete the corrupted projection, and completely replay the raw, uncorrupted event log from scratch to perfectly reconstruct the correct state.
    *   *Extreme Write Performance*: Appending to an immutable log (like Kafka or EventStoreDB) is the fastest database operation possible. No locking, no B-Tree rebalancing.
*   **Cons**:
    *   Replaying 10 million events to find a balance is slow. You must implement "Snapshots" (saving the calculated state every 1,000 events) and CQRS (to maintain a fast read-only View).

---

## 4. The Transactional Outbox Pattern

### The Problem
A microservice must save data to its database *and* publish a Kafka message. 
    `db.save(user); kafka.send(userCreatedEvent);`
If the database commits, but the network connection to Kafka drops, the system is permanently inconsistent (the user exists, but the Welcome Email service never receives the event).
You cannot use a distributed 2PC transaction encompassing Postgres and Kafka.

### The Solution: Outbox Table + Polling/CDC
Save the data and the event into the *same database* in a single ACID transaction.

1.  **The Atomic Write**:
    ```sql
    BEGIN TRANSACTION;
    INSERT INTO users (id, name) VALUES (1, 'Alice');
    INSERT INTO outbox_events (aggregate_id, event_type, payload) VALUES (1, 'UserCreated', '{"id": 1, "name": "Alice"}');
    COMMIT;
    ```
2.  **The Relay (Asynchronous)**:
    *   A completely separate background process continuously polls the `outbox_events` table (or uses Debezium CDC to tail the transaction log).
    *   It reads the pending events, publishes them to Kafka, and marks the outbox row as "Sent" (or deletes it).
3.  **Guarantee**: At-Least-Once delivery. The event is mathematically guaranteed to reach Kafka eventually, even if the primary application server crashes immediately after the database commit.

---

## 5. Bulkheads (Failure Isolation)

### The Problem
Ship hulls are divided into watertight compartments (bulkheads). If the hull is breached, only one compartment floods, preventing the entire ship from sinking.
In microservices, if Service A connects to Database X and API Y, using a single shared thread pool, a slowdown in API Y will exhaust all available threads. Service A will stop responding entirely, failing to process totally unrelated requests for Database X.

### The Solution
Isolate resources.
*   **Thread Pool Bulkheads**: Dedicate 10 threads specifically for communicating with API Y, and 40 threads for Database X. If API Y hangs, only those 10 threads lock up. The other 40 threads continue serving database requests flawlessly.
*   **Infrastructure Bulkheads**: Do not share a single massive Kafka cluster or Redis instance across every microservice. Provision separate clusters for distinct business domains to isolate the blast radius of a catastrophic crash.

---

## Interview Questions & Model Answers

**Q1: In an Event Sourced system, what happens when the business logic changes? For example, previously, a withdrawal fee was 1%. Now it is 2%. If you replay the historical events, won't you incorrectly charge 2% on old transactions?**
*Answer*: This is why Event Sourcing requires events to be *facts that happened in the past*, not *commands to execute logic*.
An event should never be `WithdrawalRequested`. It should be `WithdrawalPerformed { amount: 100, fee_applied: 1.00 }`. 
Furthermore, the event handler logic that projects the state must be versioned. The logic that replays events from 2021 must use the 2021 ruleset. Alternatively, the projection logic simply applies the pre-calculated `fee_applied` directly from the immutable payload, rendering the business rule change irrelevant for historical replay. Historical facts never change.

**Q2: How does the Saga Orchestrator handle an infrastructure failure? What if the Orchestrator pod crashes immediately after the Hotel books the room, but before it tells the Car service?**
*Answer*: The Orchestrator's state machine must be durably persisted.
When the Orchestrator receives the success response from the Hotel Service, it first writes to its own local database: `UPDATE saga_instances SET current_step = 'HOTEL_BOOKED' WHERE saga_id = 123`.
If the pod crashes, Kubernetes spins up a new Orchestrator pod. A background recovery daemon queries the database: `SELECT * FROM saga_instances WHERE status = 'IN_PROGRESS'`. The new pod reads Saga #123, sees that the last successful step was `HOTEL_BOOKED`, and seamlessly resumes the workflow by sending the `BookCar` command to the Car Service. This requires all downstream services to be strictly idempotent, as the Car Service might receive the specific command twice during a retry.

**Q3: Describe how CQRS simplifies complex authorization filtering on Read Queries.**
*Answer*: In a highly normalized relational database, checking if a User has permission to see 50 specific rows might require traversing 4 different table JOINs (`User -> Roles -> Permissions -> Document`). Executing this dynamic JOIN on every single read query is devastating to performance.
In CQRS, we offload this heavy lifting to the Write Path. When permissions or documents change, an asynchronous worker pre-calculates the explicit list of Document IDs the user is allowed to see. It writes this denormalized, flattened structure directly into the Read Database (e.g., Elasticsearch or Redis). The Read Service simply executes an $O(1)$ query: `GET user:123:documents`. The Read Path executes zero JOINs and zero complex business logic, resulting in immense read scalability.

**Q4: We implemented the Transactional Outbox pattern. Our relay process reads the database, publishes to Kafka, and then updates the database row to `status = 'SENT'`. However, sometimes the relay publishes to Kafka, but crashes before updating the database. When it restarts, it reads the row again and publishes a duplicate message to Kafka. Is our Outbox broken?**
*Answer*: No, the Outbox is functioning perfectly within its mathematical limits. The Outbox pattern fundamentally guarantees **At-Least-Once Delivery**. It guarantees the message will *never* be lost, but it explicitly accepts that duplicates may occur during network partitions or crashes.
The solution is not to fix the Outbox; the solution is to fix the *consumer* of the Kafka message. Every downstream microservice consuming from Kafka must be strictly **Idempotent**. It must uniquely identify the incoming event (e.g., using a UUID generated by the source database and included in the payload) and ignore it if it has already been processed successfully.

## Key Takeaways
*   **Sagas over 2PC**: Embrace eventual consistency for cross-service workflows. Build robust compensating transactions (rollbacks) instead of relying on distributed locking.
*   **Event Sourcing = Perfect History**: Understand the power of an append-only log of immutable facts. You can derive any state or projection from it dynamically.
*   **CQRS separates scaling vectors**: Reads and Writes have massively different performance requirements. Split them.
*   **Outbox guarantees delivery**: Never use dual-writes (DB + Kafka) in application code. Use an Outbox table to link the database transaction atomically to the event generation.
