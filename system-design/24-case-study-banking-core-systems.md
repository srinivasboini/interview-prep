# Case Study: Banking Core Systems (Ledger & Transactions)

## Overview

The "Core Banking System" is the absolute heart of any financial institution. It is the definitive system of record holding user accounts, balances, loan originations, and the ledger of every transaction that has ever occurred. 

Historically, this was a massive IBM mainframe running COBOL (often called the "Monolith"). Today, neo-banks (like Monzo or Revolut) and modernizing traditional banks are rebuilding these cores as distributed microservices. 

For a Staff/Principal Engineer, the interview focuses on extreme consistency, handling "End of Day" (EOD) batch processing at scale, and architecting an **Immutable Double-Entry Ledger** that mathematically proves no money was created or destroyed out of thin air.

---

## Part 1: Requirements Clarification

### Functional
*   Create customer accounts (Checking, Savings, Loan).
*   Process real-time transactions (Deposits, Withdrawals, Transfers).
*   Calculate and apply daily/monthly interest.
*   Generate account statements.
*   Maintain a pristine, auditable ledger.

### Non-Functional
*   **Absolute Consistency (ACID)**: No eventual consistency for the source of truth. If Bob transfers \$100 to Alice, Bob's account must debit exactly when Alice's account credits.
*   **Immutability**: You cannot execute a SQL `UPDATE` to change a historical transaction. You cannot `DELETE` a mistake.
*   **High Throughput vs. Latency**: While low latency is nice, throughput and correctness are paramount (e.g., processing millions of interest calculations at midnight).
*   **Idempotency**: Retrying a timed-out network request must never result in a double charge.

---

## Part 2: The Core Concept: Double-Entry Accounting

In naive web development, a transfer looks like this:
```sql
-- ANTI-PATTERN
UPDATE accounts SET balance = balance - 100 WHERE id = 'Bob';
UPDATE accounts SET balance = balance + 100 WHERE id = 'Alice';
```
*Why is this disastrous?* 
1. If the database crashes between the two statements, \$100 vanishes from reality. 
2. There is no historical record of *why* Bob's balance changed. 
3. If an admin manually updates Bob's balance to fix a bug, there is no corresponding debit, breaking the fundamental law of accounting: **Assets = Liabilities + Equity**.

### The Double-Entry Ledger Data Model

Every transaction creates exactly two equal and opposite entries (a Debit and a Credit) that sum to zero. The database is **Append-Only**.

**Table: `ledger_entries`**

| Entry ID | Transaction ID | Account ID | Amount | Type   | Timestamp           |
| :---     | :---           | :---       | :---   | :---   | :---                |
| 1001     | TXN-ABC        | Bob-123    | -100   | DEBIT  | 2024-01-01 10:00:00 |
| 1002     | TXN-ABC        | Alice-456  | +100   | CREDIT | 2024-01-01 10:00:00 |

To find Bob's current balance, you do not read a simple `balance` column. You compute the sum of all his ledger entries:
`SELECT SUM(amount) FROM ledger_entries WHERE account_id = 'Bob-123';`

---

## Part 3: High-Level Architecture (CQRS Pattern)

Calculating the sum of 10,000 ledger entries every time Bob opens his mobile app is too slow. We must separate the Write Path (the immutable ledger) from the Read Path (the cached balance). This is **Command Query Responsibility Segregation (CQRS)**.

```mermaid
flowchart TD
    Client[Mobile App]
    
    API[API Gateway]
    
    subgraph Write Path (Command)
        Core[Core Ledger Service]
        LedgerDB[(PostgreSQL \n Ledger_Entries)]
        CDC[Debezium CDC]
    end
    
    subgraph Read Path (Query)
        Kafka[Kafka \n Event Bus]
        BalanceSvc[Balance Projection Service]
        BalanceDB[(Redis / Fast SQL \n Account_Balances)]
    end

    Client -->|1. POST /transfer| API
    API -->|2. Execute ACID Tx| Core
    Core -->|3. Append Debit & Credit| LedgerDB
    
    LedgerDB -->|4. Log Capture| CDC
    CDC -->|5. Publish Event| Kafka
    
    Kafka -->|6. Consume Event| BalanceSvc
    BalanceSvc -->|7. UPDATE balance = balance - 100| BalanceDB
    
    Client -->|8. GET /balance| API
    API --> BalanceSvc
    BalanceSvc -->|9. Fast Read O(1)| BalanceDB
```

---

## Part 4: Deep Dive - Handling End of Day (EOD) Batch Processing

Historically, banks would physically shut down operations from 11:00 PM to 6:00 AM to calculate interest on 50 million accounts. In a modern 24/7 global economy, taking the bank offline is unacceptable.

How do we calculate 5% APY (Annual Percentage Yield) daily interest on millions of accounts while people are actively swiping their debit cards?

### The Map-Reduce Approach (Apache Spark / Hadoop)
1.  **Snapshotting**: At precisely 23:59:59 UTC, we take a point-in-time snapshot of the `ledger_entries` database. (Modern databases like Postgres allow read-replica snapshotting without locking the primary writer).
2.  **Map (Parallel Computation)**: We spin up a massive Apache Spark cluster. Spark partitions the 50 million accounts into chunks (e.g., 500 parallel executor nodes covering 100,000 accounts each).
3.  **Calculate**: Each executor mathematically aggregates the end-of-day balance and calculates the daily fractional interest (e.g., $\$10,000 * 0.05 / 365 = \$1.36$).
4.  **Reduce (Apply)**: Spark generates 50 million highly optimized `INSERT` statements (as new ledger entries: `CREDIT $1.36, Type: INTEREST`).
5.  **Batch Ingest**: Instead of executing 50 million individual `INSERT` HTTP calls to the Core Service (which would DDOS our own bank), we drop the generated entries into a bulk-load Cassandra/Postgres ingestion tool, or publish them as massive batches onto Kafka for the Core Service to consume at highly controlled capacity limit (e.g., 5,000 TPS).

---

## Part 5: Deep Dive - Saga Pattern for Distributed Transfers

If Bob (Chase Bank) transfers $100 to Charlie (Bank of America), we do not control BoA's database. We cannot use a monolithic SQL ACID transaction. We must use a **Distributed Saga**.

1.  **State Initiation**: Chase creates a local transaction: `TXN-999 | State: PENDING`.
2.  **Local Debit**: Chase appends the `-100` DEBIT to Bob's ledger.
3.  **Outbound Call**: Chase places the instruction on the ACH (Automated Clearing House) or FedNow network. (The network call is extremely slow and susceptible to failure).
4.  **Wait State**: Chase's database row remains `PENDING`.
    *   *Scenario A (Success)*: Two days later, ACH returns a clearing file. Chase updates `TXN-999` to `COMPLETED`. The money has settled.
    *   *Scenario B (Failure/Rejection)*: Charlie's account at BoA is closed. ACH rejects the transfer 48 hours later.
5.  **Compensating Transaction**: Chase must execute a Saga Rollback. We *do not delete* Bob's original debit. We append a new `+100` CREDIT to Bob's ledger, with the reason `REVERSAL_TXN_999_REJECTED`. The ledger immutability is preserved, and the balance mathematically corrects itself.

---

## Interview Questions & Model Answers

**Q1: Our read-replica calculating the balance is 2 seconds behind the primary ledger database due to replication lag. Bob deposits \$1,000 at the ATM, opens his mobile app immediately, and sees \$0. He panics and calls support. How do we solve this Read-After-Write consistency issue?**
*Answer*: This is a classic distributed systems problem where we optimized for throughput (CQRS) but sacrificed strict read consistency.
We can implement **Client-Centric Consistency (or "Sticky Session" caching).**
When Bob initiates the deposit, the API Gateway returns the `Transaction ID` and the newly calculated *projected* balance locally. The mobile app temporarily caches this projected balance and displays it (e.g., "$1,000 (Pending)").
Alternatively, the API Gateway maps Bob's specific `User ID` to strictly route all of his `GET` requests directly to the Primary Writer database for the next 10 seconds, bypassing the lagged Read Replica entirely. After the 10-second TTL expires, his reads fall back to the generic Read Replica, which by then has successfully caught up to the primary.

**Q2: A developer accidentally deployed a bug that credited 500,000 users with an extra $10.00 each over the last 4 hours. How do we fix the database?**
*Answer*: We **never** execute an `UPDATE ledger_entries SET amount = amount - 10`. That destroys the immutable audit trail and violates banking regulations.
We must construct an automated **Compensating Script**. 
We query the database for the exact `Transaction IDs` executed by the buggy deployment over that 4-hour window. The script programmatically loops through those IDs and generates 500,000 brand new pairs of Ledger Entries reversing the exact amount (Debiting \$10.00 from the user, and Crediting \$10.00 back to the Bank's internal Operational Account). Those new rows are appended to the database with a clear metadata tag `Reason: BUG-REVERSAL-JIRA-1234`. The balances will automatically recalculate correctly.

**Q3: We have 5,000 users trying to buy Taylor Swift concert tickets exactly at 9:00 AM. They are all transferring money from their personal external accounts into the single monolithic "Ticketing Merchant Account." The database hits 100% CPU lock contention and transactions start timing out. How do we architect around this?**
*Answer*: This is the dreaded **Hot Account** problem. In a strict ACID relational database, executing 5,000 concurrent `UPDATES` or `INSERTS` linking to the exact same Merchant Account ID results in brutal sequential lock-queueing at the database row level.
To mitigate this, we employ **Sharding the Aggregate Root (Account Sharding)**.
Instead of the Merchant having a single Account ID (`MERCH-1`), we provision 50 distinct sub-accounts (`MERCH-1_01`, `MERCH-1_02`, etc.) behind the scenes.
When the 5,000 users initiate their payments, the API randomly routes their Credits across the 50 sub-accounts in a round-robin fashion. This instantly dilutes the lock contention by 50x, allowing parallel database writes to execute at hyper-speed without crashing. At the end of the day, a single batch job aggregates the 50 sub-accounts and sweeps the consolidated funds into the primary `MERCH-1` account.

**Q4: Explain why UUIDv4 is a terrible choice for the Primary Key of our Ledger table, and what we should use instead.**
*Answer*: A UUIDv4 (`4a6b...`) is completely random. When a Relational Database (like PostgreSQL or MySQL) inserts a Row into a table with a B-Tree clustered index, it must physically place the row in order. If the keys are completely random, the database is forced to constantly split disk pages and rewrite data to squeeze the random keys in between historical records. This causes monumental **Index Fragmentation** and destroys `INSERT` performance as the table grows to billions of rows.
Furthermore, UUIDv4 holds no temporal data; you cannot sort transactions by time using the ID alone.
We must use a **Time-Sorted Unique Identifier** like Twitter Snowflake or ULID (Universally Unique Lexicographically Sortable Identifier). These IDs are guaranteed unique across distributed microservices, but their prefix contains the exact millisecond timestamp. Therefore, every single new `INSERT` predictably appends to the very end of the B-Tree on disk, resulting in zero fragmentation and maximizing write throughput.

## Key Takeaways

*   **Immutability is Law**: Core banking systems never update or delete historical financial records. Use append-only Double-Entry ledgers and Compensating Transactions for reversals.
*   **Decouple for Scale (CQRS)**: Separating the strict Write Path (ACID Ledger) from the asynchronous Read Path (Cached Balances via Kafka/CDC) is the only way to achieve both correctness and high-speed UI loading.
*   **The Hot Account Pattern**: Solve massive concurrent writes to a single endpoint by logically sharding the target account into dozens of sub-accounts.
*   **Sagas over 2PC**: Distributed network calls (like moving money to another bank) fail constantly. Use state machines (Sagas) to monitor long-running network responses and execute local compensatory rollbacks when they fail.
