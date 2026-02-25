# Relational Database Design and Storage

## Overview

The database is the beating heart of any enterprise system. While stateless application servers can be spun up or down in seconds to handle load, stateful storage systems are subject to the laws of physics: disk I/O limits, network latency during replication, and the fundamental constraints of the CAP theorem. 

For a Staff or Principal Engineer, the database is often the final frontier of scalability. Interviews heavily index on your ability to design schemas, understand indexing structures (B-Trees vs. Hash Indexes), and articulate strategies for scaling relational databases beyond the limits of a single machine (Replication, Partitioning, and Sharding).

In banking and financial services, Relational Database Management Systems (RDBMS) like PostgreSQL, Oracle, and SQL Server are the undisputed kings. The strict requirement for ACID guarantees—ensuring that a $1,000 transfer deducts exactly $1,000 from Account A and adds exactly $1,000 to Account B without any intermediate state being visible—makes relational databases mandatory for core ledgers. You must understand how they achieve these guarantees and the performance penalties involved.

## Foundational Concepts

### ACID Properties
The cornerstone of relational databases that ensures data validity despite errors, power failures, or concurrent access.
*   **Atomicity**: "All or nothing." A transaction is a single unit of work. If step 3 of a 5-step transaction fails, the entire transaction is rolled back.
*   **Consistency**: A transaction must transition the database from one valid state to another, maintaining all predefined rules, constraints (e.g., foreign keys, unique constraints), and triggers.
*   **Isolation**: Concurrent execution of transactions yields the same state that would be obtained if they were executed sequentially. This is the hardest property to maintain at scale and has different "levels" (see below).
*   **Durability**: Once a transaction is committed, it remains committed even in the event of a system crash. Achieved via the Write-Ahead Log (WAL).

### Isolation Levels
Higher isolation provides more safety but drastically reduces concurrent performance (throughput).
1.  **Read Uncommitted**: The lowest level. Transactions can see uncommitted changes from other transactions (Dirty Reads). Rarely used in banking; a user might see a balance that hasn't actually settled.
2.  **Read Committed**: (Default in PostgreSQL/Oracle). Guarantees that any data read is committed at the moment it is read. Prevents Dirty Reads, but allows Non-Repeatable Reads (reading the same row twice might yield different results if another transaction committed an update in between).
3.  **Repeatable Read**: (Default in MySQL/InnoDB). Guarantees that if you read a row twice within the same transaction, you get the exact same data. Prevents Non-Repeatable Reads, but might allow Phantom Reads (new rows added by other transactions might appear in subsequent range queries).
4.  **Serializable**: The strictest level. Emulates sequential execution. Transactions are completely isolated. Prevents all anomalies but causes massive lock contention and system slowdowns.

## Technical Deep Dive

### Indexing: B-Trees and B+Trees
An index is a data structure that improves the speed of data retrieval operations at the cost of additional storage space and slower writes (because the index must be updated on every `INSERT`/`UPDATE`/`DELETE`).

*   **B-Tree (Balanced Tree)**: A self-balancing tree that maintains sorted data. It allows searches, sequential access, insertions, and deletions in logarithmic time *O(log n)*.
*   **B+Tree**: The standard for most relational databases (e.g., InnoDB). Unlike a B-Tree, all data pointers are stored only in the leaf nodes. 
    *   *Why this matters*: The internal nodes only contain routing keys, meaning more keys fit into a single disk page (usually 4KB or 8KB). This significantly reduces the depth of the tree (fanout), meaning fewer disk I/O operations are required to find a record. The leaf nodes are also linked sequentially, making range queries (`SELECT * WHERE amount BETWEEN 100 AND 500`) blisteringly fast.

**Types of Indexes:**
*   **Clustered Index**: Determines the physical order of data on disk. A table can only have one clustered index (usually the Primary Key). Querying by a clustered index is the fastest possible retrieval.
*   **Non-Clustered (Secondary) Index**: Contains the indexed column values and a pointer back to the actual row data (often the clustered index key). Finding data via a secondary index requires two steps: finding the key, then traversing the clustered index to fetch the full row.
*   **Covering Index**: If a query is `SELECT name FROM users WHERE age = 30;` and you create a composite index on `(age, name)`, the database can satisfy the query *entirely* from the index without ever reading the underlying table. This is a massive optimization technique.

### Scaling the RDBMS

#### 1. Vertical Scaling (Scale Up)
Adding more RAM, CPU, or migrating to faster NVMe SSDs. This is always the first logical step before sharding, as it requires zero application changes. However, it has hardware limits and remains a Single Point of Failure.

#### 2. Replication (Scale Reads)
*   **Master-Slave (Primary-Replica)**: All writes go to the Primary. The Primary asynchronously (or synchronously) streams changes to one or more Replicas. All reads (or read-heavy reporting queries) are routed to the Replicas.
*   **The Problem**: Replication Lag. If a user updates their profile on the Primary and immediately refreshes the page, the read might hit a Replica that hasn't received the update yet (Stale Read). In banking, this breaks the "Read-Your-Writes" consistency requirement.

#### 3. Partitioning (Scale Table Size)
Dividing a massive, single logical table into smaller physical pieces to improve query performance and manageability.
*   **Range Partitioning**: Dividing data based on a range of values (e.g., partitioning `transactions` by date: one partition per month). Queries filtering by date can skip irrelevant partitions entirely (*Partition Pruning*).
*   **Hash Partitioning**: Distributing data evenly across partitions using a hash of the partition key (e.g., `hash(user_id) % 4`). Good for balanced access patterns.

#### 4. Sharding (Horizontal Scaling / Scale Writes)
The ultimate, most complex scaling technique. Sharding splits the data across entirely independent database instances (nodes). 
*   **Example**: Node A holds users A-M; Node B holds users N-Z.
*   **The Hard Part**: Choosing the **Shard Key**. If you shard by `user_id`, all of a single user's data is co-located on one machine. This makes `SELECT * FROM accounts WHERE user_id = 123` lightning fast. However, aggregating total balances across *all* users now requires querying every shard and merging the results in the application memory (Scatter-Gather). Furthermore, ACID transactions spanning multiple shards require distributed 2PC or Sagas.

## Visual Representations

### B+Tree Internal Structure for Range Queries

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#E3F2FD', 'edgeLabelBackground':'#FFF'}}}%%
graph TD
    classDef root fill:#BBDEFB,stroke:#1976D2,stroke-width:2px;
    classDef internal fill:#E1BEE7,stroke:#8E24AA,stroke-width:2px;
    classDef leaf fill:#C8E6C9,stroke:#388E3C,stroke-width:2px;

    R[Root Node: [50, 100]]:::root
    
    I1[Internal: [25]]:::internal
    I2[Internal: [75]]:::internal
    I3[Internal: [125]]:::internal

    R --> I1
    R --> I2
    R --> I3

    L1[Leaf: 10, 20]:::leaf
    L2[Leaf: 30, 40]:::leaf
    L3[Leaf: 60, 70]:::leaf
    L4[Leaf: 80, 90]:::leaf
    L5[Leaf: 110, 120]:::leaf
    L6[Leaf: 130, 140]:::leaf

    I1 --> L1
    I1 --> L2
    I2 --> L3
    I2 --> L4
    I3 --> L5
    I3 --> L6

    %% Sequential linking for blazing fast Range Queries
    L1 -.-> L2 -.-> L3 -.-> L4 -.-> L5 -.-> L6
```

### Primary-Replica Architecture with Read/Write Splitting

```mermaid
flowchart TD
    App[Application (Load Balanced)]
    
    subgraph Database Cluster
        Primary[(Primary DB \n Write-Heavy)]
        Replica1[(Replica 1 \n Read-Heavy)]
        Replica2[(Replica 2 \n Analytics/Batch)]
    end
    
    App -->|1. Write (INSERT/UPDATE/DELETE)| Primary
    Primary -.->|2. Async Replication (WAL)| Replica1
    Primary -.->|2. Async Replication (WAL)| Replica2
    
    App -->|3. Read (SELECT)| Replica1
    
    BI_Tool[Tableau/BI Tool] -->|4. Heavy Aggregations| Replica2
```

## Code/Configuration Examples

### Query Optimization: Execution Plans
An interviewer might ask you to explain why a query is slow. The first step is always to look at the execution plan using `EXPLAIN ANALYZE` (in PostgreSQL).

```sql
-- SLOW: This requires a Sequential Scan (reads every row on disk)
EXPLAIN ANALYZE SELECT * FROM transactions WHERE account_id = 'A123' AND status = 'COMPLETED';

-- Output might show:
-- Seq Scan on transactions (cost=0.00..15243.00 rows=105 width=128) (actual time=0.012..145.231 ms)
```

**Optimization via Covering Index:**
If we frequently just need the transaction reference ID for completed transactions, we can dramatically speed this up.

```sql
-- Create a composite index. Order matters! Lead with the highest cardinality column used in equality checks.
CREATE INDEX idx_acct_status_ref ON transactions (account_id, status) INCLUDE (reference_id);

-- FAST: Now the database traverses the B+Tree (Index Only Scan) and returns the reference_id without touching the main table heap.
EXPLAIN ANALYZE SELECT reference_id FROM transactions WHERE account_id = 'A123' AND status = 'COMPLETED';

-- Output might show:
-- Index Only Scan using idx_acct_status_ref on transactions (cost=0.42..4.55 rows=105 width=32) (actual time=0.030..0.045 ms)
-- Impact: Reduced query time from 145ms to 0.045ms.
```

## Interview Questions & Model Answers

**Q1: We are designing an Account Management service. Should we use an RDBMS (like PostgreSQL) or a NoSQL database (like MongoDB)?**
*Answer*: For core account management and ledger systems involving financial transactions, an RDBMS is practically mandatory. We require absolute, strong ACID guarantees. If a transfer fails halfway, atomicity ensures rollback. Isolation ensures concurrent transfers don't corrupt balances. While NoSQL databases offer flexibility and horizontal scale, they generally default to eventual consistency or prioritize availability over consistency during partitions (AP systems). For a ledger, we must prioritize Consistency over Availability (CP system). An RDBMS enforces data integrity strictly at the schema layer, which is essential for auditability and regulatory compliance.

**Q2: Our user profile database is heavily read-bound (95% Reads, 5% Writes). We use PostgreSQL. Adding indexes hasn't helped enough. How do we scale it?**
*Answer*: I would use a Primary-Replica replication pattern. We configure one Primary node to handle all `INSERT`, `UPDATE`, and `DELETE` operations. This Primary asynchronously replicates its Write-Ahead Log (WAL) to multiple Read Replicas. We update the application's connection pool to route all `SELECT` queries to the Replicas, effectively distributing the read load across `N` machines. 
We must handle replication lag, though. If a user updates their password and immediately logs in, they might hit a replica that hasn't received the new password hash yet. To fix this, we can implement "Read-Your-Writes" consistency by routing reads back to the Primary for a short window (e.g., 5 seconds) after a user performs a write.

**Q3: Describe the impact of a poorly chosen Shard Key when horizontally scaling a database.**
*Answer*: A poorly chosen shard key leads to two massive problems: Hotspots and Cross-Shard Joins.
Suppose we are building Venmo and shard transactions by `timestamp`. One shard will contain all of today's transactions and will be overwhelmed by write traffic, while shards holding 2018 data sit idle (a write hotspot). 
Suppose instead we shard by `transaction_id`. To find all transactions for User A, the application must query *every single shard* across the network, pull the results into memory, and aggregate them (a Scatter-Gather, which is incredibly slow and kills the network).
A sound approach is sharding by `user_id`. All data for User A lives on one shard, making localized queries fast, though it makes global aggregations (like total system-wide daily volume) difficult.

**Q4: Explain the difference between B-Tree and B+Tree indexes. Why do modern databases prefer B+Trees?**
*Answer*: In a B-Tree, every node (internal and leaf) contains a key and the actual data (or a pointer to the data row). In a B+Tree, the internal nodes *only* contain routing keys; the actual data pointers solely reside in the leaf nodes. Databases prefer B+Trees because disk I/O operates in blocks (pages). Because internal nodes in a B+Tree don't hold bulky data, many more routing keys fit into a single block. This gives the tree a massive fanout (branching factor), meaning the tree is much shallower. Finding a row among a billion records might only require traversing 3 or 4 levels down the tree, requiring only 3 or 4 disk seeks. Additionally, the leaf nodes in a B+Tree are sequentially linked on disk, making range queries (e.g., `BETWEEN date1 AND date2`) extremely fast compared to a standard B-Tree.

## Real-World Enterprise Scenarios

**Scenario: Designing a Multi-Tenant SaaS Database Architecture**
*   **Context**: Building a B2B payment processor platform acting as an API provider to 5,000 different small businesses.
*   **Architecture Choice**:
    *   **Anti-Pattern**: Putting all tenants in one giant table with a `tenant_id` column. If Tenant A suffers a DDoS attack or runs massive analytics, the shared CPU/Disk throttles Tenant B ("Noisy Neighbor" problem).
    *   **Schema-per-Tenant**: Better. One database instance, but 5,000 separate schemas. Provides logical isolation and prevents noisy neighbors up to the instance limit.
    *   **Database-per-Tenant (Silo)**: Gold standard for large enterprise clients. Each high-value client gets their own dedicated primary/replica cluster.
*   **Trade-off**: Managing schema migrations across 5,000 databases is an operational nightmare requiring robust CI/CD automation and tooling.

## Common Pitfalls & Best Practices

**Pitfalls:**
*   **Over-Indexing**: Indexes speed up reads but slow down writes. Adding 15 indexes to an `orders` table means every new order requires 16 simultaneous disk writes (1 table + 15 indexes). Only index columns used in `WHERE`, `JOIN`, or `ORDER BY` clauses.
*   **N+1 Query Problem in ORMs**: Using Hibernate/JPA blindly. Fetching a list of 100 users, and then iterating through the loop to fetch each user's address, results in 101 database queries. Use `JOIN FETCH` to grab the data in a single query.
*   **Relying purely on the DB for full-text search**: SQL `LIKE '%keyword%'` queries cannot efficiently use B-Tree indexes and cause sequential disk scans. Use Elasticsearch or native Postgres `tsvector`/GIN indexes for complex text searching.

**Best Practices:**
*   **Connection Pooling**: Opening a TCP connection to a database is extremely expensive (handshake, authentication). Always use a connection pooler (like HikariCP in Java or PgBouncer in front of Postgres) to reuse connections.
*   **Execution Plans are Gospel**: Never guess why a database is slow. Run `EXPLAIN ANALYZE` to see exactly how many disk pages were hit, what index was chosen, and how much CPU time was spent on joins.

## Comparison Tables

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Concurrency Performance |
| :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | Possible | Possible | Possible | Very High |
| **Read Committed** | Prevented | Possible | Possible | High |
| **Repeatable Read** | Prevented | Prevented | Possible | Medium |
| **Serializable** | Prevented | Prevented | Prevented | Low (High Contention) |

| Scaling Strategy | Modifies App Code? | Primary Benefit | Limitation |
| :--- | :--- | :--- | :--- |
| **Vertical Scale (Up)** | No | Fixes immediate CPU/RAM bottlenecks | Hardware limits, Costs, SPOF |
| **Read Replicas** | Minimal | Scales Read-heavy workloads | Replication Lag, Cannot scale Writes |
| **Table Partitioning** | No | Speeds up localized queries / maintenance | Does not increase absolute compute power|
| **Sharding (Horizontal)**| Yes (Significant) | Scales Write-heavy workloads infinitely | Complex cross-shard joins, Rebalancing |

## Key Takeaways

*   **ACID is a spectrum**: You can tune isolation levels down to improve throughput, but you must understand the anomalies (Dirty Reads, Phantoms) you inherit.
*   **B+Trees define performance**: Because internal nodes only hold keys, B+Trees are shallow, making `SELECT` operations lightning fast. Because leaves are linked, range queries are highly optimized.
*   **Sharding is a last resort**: It dramatically increases the complexity of your application tier. Always exhaust connection pooling, read replicas, and vertical scaling before sharding an RDBMS.
*   **Always justify your Shard Key**: It dictates the hotspot distribution and query efficiency of your entire system.

## Further Reading
*   *Database Internals by Alex Petrov* (Crucial for Staff/Principal level understanding of B-Trees and WAL).
*   [Use The Index, Luke! (A Guide to Database Performance for Developers)](https://use-the-index-luke.com/)
*   [Scaling Relational Databases (DigitalOcean Guide)](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
