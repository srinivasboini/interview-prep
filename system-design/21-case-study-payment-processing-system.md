# Case Study: Payment Processing System (Stripe / PayPal)

## Overview

Designing a Payment Processing System is the ultimate test of your understanding of extreme data consistency, fault tolerance, and idempotency. In a social network, dropping a "Like" is a minor inconvenience. In a payment system, dropping a transaction or double-charging a user breaks the law, violates PCI-DSS compliance, and destroys trust instantly.

For a Staff/Principal Engineer in banking, this domain is your bread and butter. You must architect a system that can withstand severe network partitions, handle unresponsive third-party gateways (Visa/Mastercard) without holding open database locks, and mathematically guarantee that a \$100 debit on Account A results in exactly a \$100 credit on Account B—even if half the servers physically burn down during the transfer.

---

## Part 1: Requirements Clarification

### Functional
*   Users can initiate a payment to another user or merchant.
*   System must integrate with external Payment Service Providers (PSPs) like Stripe or Braintree.
*   Support for webhooks to notify merchants of payment success/failure.
*   A robust reconciliation system to ensure internal ledgers match external banks.

### Non-Functional
*   **Absolute Consistency (ACID)**: Eventual consistency is often unacceptable for the core money-movement ledger. (Consistency > Availability).
*   **Double-Charge Prevention (Idempotency)**: The system must perfectly handle retries caused by network timeouts.
*   **Security & Compliance**: PCI-DSS requirements. Raw PANs (Primary Account Numbers) must strictly be vaulted or tokenized and never logged in plain text.
*   **High Throughput**: 1,000 TPS (Transactions Per Second) average, 10,000 TPS on Black Friday.

### Back-of-the-Envelope
*   **Throughput**: 1,000 TPS.
*   **Storage**: 1,000 TPS * 86,400s = ~86 Million transactions/day. Each record is ~1KB. ~86GB/day -> ~31TB/year. Relational DBs can handle this with proper partitioning (e.g., partitioned by month).

---

## Part 2: High-Level Architecture

The core philosophy of a Payment System is to treat every step as a state machine transition stored permanently in a database *before* making any external network calls.

```mermaid
flowchart TD
    Client[Web / Mobile Client]
    
    API[API Gateway \n (Rate Limiter, WAF)]
    
    subgraph Payment Core
        Orch[Payment Orchestrator]
        LedgerSvc[Internal Ledger Service]
        WalletSvc[Wallet / User Balance Service]
    end
    
    subgraph Storage
        MainDB[(PostgreSQL \n Payment State)]
        LedgerDB[(PostgreSQL \n Immutable Ledger)]
    end
    
    subgraph External
        PSP[External Payment Gateway \n (Stripe / Visa)]
        Webhook[Merchant Webhook URL]
    end

    Client -->|1. POST /payments| API
    API -->|2. Route| Orch
    
    Orch -->|3. Save State: PENDING| MainDB
    Orch <-->|4. Reserve Funds| WalletSvc
    Orch <-->|5. HTTP POST \n (Idempotent)| PSP
    
    PSP -.->|6. Async Webhook / Polling| Orch
    
    Orch -->|7. Save State: SUCCESS| MainDB
    Orch -->|8. Append Entry| LedgerSvc
    LedgerSvc --> LedgerDB
```

---

## Part 3: Deep Dive - Idempotency is Everything

A user clicks "Pay $100." The network request reaches the `Payment Orchestrator`. The Orchestrator calls the `External PSP`. The PSP processes the payment, but the network drops the HTTP 200 OK response.
From the user's perspective, the app is loading infinitely. They refresh the page and click "Pay $100" again.

If the system is not idempotent, the user is charged $200.

**The Solution: The Idempotency Key**

1.  **Client Generation**: The mobile app generates a mathematically unique `UUID v4` (e.g., `8f14e...`) when the checkout page loads.
2.  **The Request**: The user clicks Pay. The app sends `POST /payments` with HTTP Header: `Idempotency-Key: 8f14e...`.
3.  **The Database Lock (First Pass)**: The API catches the request. It attempts to `INSERT INTO idempotency_keys (key, response_body) VALUES ('8f14e...', NULL)`.
    *   Since it's the first time, the row inserts successfully.
4.  **Processing**: We process the payment with Stripe, get the HTTP 200, and generate the final JSON response describing the success.
5.  **The Database Lock (Release)**: We `UPDATE idempotency_keys SET response_body = '{success...}' WHERE key = '8f14e...'`. We return the JSON to the client.
6.  **The Retry (Second Pass)**: The user's internet dropped. They hit "Pay" again (sending the *exact same* Idempotency Key).
7.  **The Interception**: The API attempts to `INSERT` the key. The database violently rejects it (`UniqueConstraintViolation`). 
8.  **The Safe Return**: The API catches the exception, runs `SELECT response_body FROM idempotency_keys WHERE key='8f14e...'`, and returns the exact same success JSON it generated 5 seconds ago. We bypass Stripe completely.

---

## Part 4: Deep Dive - The State Machine (Sagas)

You cannot execute a 3-second external network call to Stripe while holding an open SQL transaction lock on your `users` table.

```java
// ANTI-PATTERN: DO NOT DO THIS
@Transactional
public void processPayment() {
    wallet.deduct(100); // Locks the user's row in DB
    stripe.charge(100); // Network call... could take 10 seconds or timeout!
    ledger.record();    
} // Lock finally released. Database becomes a massive bottleneck.
```

**The Solution: The State Machine (Orchestrated Saga)**

Instead of long-running ACID transactions, we use state transitions.
1.  **State `CREATED`**: User expresses intent. We write exactly this to the database instantly.
2.  **State `FUNDS_RESERVED`**: We make a local DB call to deduct from the user's logical wallet to prevent them from spending the $100 twice, but the money hasn't moved yet.
3.  **State `PSP_CALL_INITIATED`**: We record that we are officially talking to Stripe. If our server loses power right now, a background cron job knows *exactly* what state we died in.
4.  **State `SUCCESS` or `FAILED`**: Based on Stripe's response (or an async webhook).
5.  **State `COMPENSATING` (Rollback)**: If Stripe fails, the background worker invokes a Saga rollback, changing the Wallet back to its original state by issuing a credit.

---

## Part 5: Deep Dive - The Double-Entry Ledger System

A core banking payment system never maintains just a `balance` column in a `users` table. If a bug alters the value to zero, the money is gone forever without a trace.

We must implement an **Immutable Double-Entry Ledger**.
Every transaction fundamentally consists of two entries: a Credit (movement out) and a Debit (movement in) that must precisely sum to $0$.

When User A pays User B $100:
1.  `INSERT INTO ledger (tx_id, account_id, amount, type) VALUES ('T1', 'UserA', -100.00, 'DEBIT')`
2.  `INSERT INTO ledger (tx_id, account_id, amount, type) VALUES ('T1', 'UserB', +100.00, 'CREDIT')`

*   **Immutability**: The ledger is append-only. No application is ever granted `UPDATE` or `DELETE` permissions on this table.
*   **Balance Calculation**: To find User A's balance, you do `SELECT SUM(amount) FROM ledger WHERE account_id = 'UserA'`. Because this is slow for millions of rows, we utilize a background CQRS (Command Query Responsibility Segregation) job or Materialized Views to snapshot the balance nightly.
*   **Correcting Errors**: If a refund is required, you *do not delete* the original rows. You issue a new transaction (`T2`) that applies a `+100` to User A and `-100` to User B, providing a mathematically perfect audit trail.

---

## Part 6: Deep Dive - Reconciliation

Is our internal MongoDB / Postgres perfectly aligned with Visa's mainframe? If they drop an HTTP connection, our database might say `PENDING` while Visa's says `SETTLED`. If we don't fix this, we lose millions of dollars.

*   Every night, the PSP generates an SFTP batch file (a settlement report) containing millions of transaction IDs and statuses.
*   Our **Reconciliation Service** pulls this file.
*   It compares our Ledger Database rows (`select * from payments where date = 'yesterday'`) against the SFTP file row-by-row.
*   **Discrepancy Resolution**:
    *   Our DB says `PENDING`, SFTP says `SUCCESS`: We missed a webhook. Update our DB to `SUCCESS` and credit the merchant.
    *   Our DB says `SUCCESS`, SFTP says nothing (Missing): Catastrophic bug. The user got the goods, but Visa never charged them. Raise an SEV-1 pager alert immediately for manual engineering review.

---

## Interview Questions & Model Answers

**Q1: Our PSP integration requires us to store the user's Credit Card (PAN) on our servers so we can charge them monthly for a subscription. How do we architect this to remain PCI-DSS compliant without drastically slowing down our system?**
*Answer*: We aggressively avoid storing raw PANs (Primary Account Numbers) on our core database.
Instead, we use a **Tokenization Vault Service**. This is a highly isolated, heavily encrypted microservice (often provided by a third party like VGS, or built using AWS KMS Envelope Encryption) that runs on an entirely isolated subnet.
When a user inputs their card on the React frontend, the payload is sent directly to the Vault. The Vault securely stores the PAN and returns a non-sensitive Token (e.g., `tok_1a2b3c`). 
The frontend passes `tok_1a2b3c` to our core Payment Service. Our `subscriptions` table only stores this meaningless token. When the monthly billing cron job fires, the Payment Service sends `tok_1a2b3c` back to the Vault, which securely unwraps the real PAN and forwards the charge directly to Stripe on our behalf. Our core system and developer laptops never see the real credit card mathematically.

**Q2: We are experiencing heavy lock contention. 50,000 users are simultaneously trying to transfer money *into* the exact same popular celebrity's bank account during a charity stream. How do we prevent our Postgres database from deadlocking or bringing down the entire cluster?**
*Answer*: This is the classic "Hot Account" (or Hot Partition) problem. Standard row-level locking (`SELECT ... FOR UPDATE`) causes all 50,000 threads to queue sequentially, waiting for the lock on the Celebrity's single row in the balances table.
To solve this, we can use a **Write Buffering / Aggregate Root pattern**. Instead of updating the celebrity's balance instantly, the API writes the 50,000 transfers into an append-only Kafka topic or an intermediate `transfer_intents` table. We are not locking the target account yet.
A background listener consumes the topic and batches the updates. Every 1 second, it executes a *single* SQL update: `UPDATE accounts SET balance = balance + 500000 WHERE account_id = 'Celeb123'`. This turns 50,000 brutal concurrent locks into one highly efficient, low-contention lock per second.

**Q3: A user triggers an internal fund transfer. The orchestrator deducts \$100 from Account A. Right before it credits Account B, the JVM crashes with an OutOfMemoryError. How does the system automatically recover this lost \$100?**
*Answer*: This highlights why the Payment Orchestrator must be a strictly persisted state machine.
When the orchestrator started the flow, it inserted a row: `Transaction 123 | State: IN_PROGRESS | Step: DEBIT_A_COMPLETED`.
Because the JVM crashed, this state remains stagnant in the database.
We deploy a secondary microservice: The **Sweeper (or Recovery) Daemon**. It runs every 5 minutes and queries for "stuck" transactions (`SELECT * FROM transactions WHERE state = 'IN_PROGRESS' AND last_updated < NOW() - 5 minutes`).
The Sweeper picks up Transaction 123. It reads the state, sees that the debit completed but the credit didn't. It places a message onto a Kafka DLQ or resumes the State Machine itself to explicitly execute the remaining step: `CREDIT_B`. Alternatively, if the rules dictate, it executes a Compensating Transaction to reverse `A` (Saga rollback).

**Q4: Explain the Transactional Outbox Pattern in the context of sending a confirmation email after a DB update.**
*Answer*: In a naive implementation, a developer might write `db.commit(); emailService.send();`. If the database commits but the network call to the Email Service fails, the system is permanently inconsistent (user paid but no receipt). If we swap the order, `emailService.send(); db.commit();`, and the DB fails due to a unique constraint, the user gets a receipt for a failed payment. We cannot use a distributed 2PC lock across a database and an SMTP server.
The **Transactional Outbox** solves this. In the exact same atomic local database transaction where we update the Payment status to `SUCCESS`, we also `INSERT INTO outbox_events (type, payload) VALUES ('EmailSent', '{user_id: 123}')`. Because it's the same SQL transaction, it guarantees atomicity: both succeed or both fail.
A highly reliable background process (like Debezium CDC) continuously tails the `outbox_events` table and pushes those events into Kafka or directly to the Email Service, guaranteeing at-least-once delivery of the email without blocking the critical payment path.

## Key Takeaways

*   **Idempotency is mathematically required**: You must implement `Idempotency-Key` headers and database unique constraints to safely handle the inevitability of HTTP retries.
*   **State Machines over Long Transactions**: Do not hold database locks while making unpredictable network calls (TCP over WAN) to downstream banking mainframes. Transition through intermediate database states instead.
*   **Double-Entry Immutable Ledgers**: Financial data is never `UPDATE`d or `DELETE`d. It is strictly appended as a balanced pair of mathematical deviations (Debits + Credits = 0).
*   **Reconciliation is your safety net**: You must build daily asynchronous differential matching systems between internal DB maps and external SFTP flat files to catch edge-case bugs that invariably slip through real-time REST systems.
