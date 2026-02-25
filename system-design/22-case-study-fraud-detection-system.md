# Case Study: Fraud Detection System

## Overview

A Fraud Detection System is the silent guardian of any financial institution, e-commerce platform, or social network. Its job is to analyze massive streams of events (logins, payments, password changes) in real-time, cross-reference them against historical profiles, and make a split-second decision: **Allow, Block, or Flag for Manual Review.**

For a Staff/Principal Engineer, this is a masterclass in **Stream Processing**, **Graph Databases**, and **Machine Learning Model Serving**. The complexity lies in the strict latency requirement. You cannot pause a credit card transaction at the grocery store for 5 seconds while a massive Hadoop cluster runs a map-reduce job. Decisions must be made in `< 50ms`, utilizing both real-time aggregations (e.g., "Has this card been used 5 times in the last minute?") and historical graph relationships (e.g., "Is this IP address 2 hops away from a known fraudster's device ID?").

---

## Part 1: Requirements Clarification

### Functional
*   Ingest thousands of events per second (transactions, logins).
*   Execute boolean rules (e.g., `IF amount > 10000 AND user_age < 1 day THEN Block`).
*   Execute complex ML models (e.g., Anomaly Detection, Random Forest).
*   Provide a case management UI for human Fraud Ops agents to review flagged transactions.
*   Update risk profiles in real-time.

### Non-Functional
*   **Ultra-Low Latency**: The critical path (decision engine) must return a result in `< 50ms`.
*   **High Throughput**: 10,000 requests per second.
*   **High Availability**: If the fraud system goes down, do we block all payments (safest but destroys revenue) or allow all payments (dangerous but keeps business running)? Usually, we fail-open for small amounts and fail-closed for large amounts.
*   **Auditability**: Every decision must be mathematically explainable to regulators (e.g., "Why was this loan denied?").

---

## Part 2: High-Level Architecture (Lambda Architecture)

Fraud detection requires analyzing what is happening *right now* (Stream Processing) juxtaposed against what has happened over the *last 5 years* (Batch Processing).

```mermaid
flowchart TD
    Client[Payment Gateway / Login Service]
    
    API[Fraud API Gateway]
    Kafka[Apache Kafka \n (Event Bus)]
    
    subgraph Real-Time Stream (Speed Layer)
        Flink[Apache Flink \n (Stateful Stream Processing)]
        Rules[Rules Engine \n & ML Inference Model]
        FastStorage[(Redis / Aerospike \n In-Memory Profiles)]
    end
    
    subgraph Batch Analytics (Batch Layer)
        Spark[Apache Spark \n (Nightly Jobs)]
        DataLake[(Hadoop / S3 \n Historical Data)]
        GraphDB[(Neo4j \n Identity Graph)]
    end
    
    subgraph Manual Ops
        CaseMgmt[Case Management UI]
    end

    Client -->|1. Sync POST /evaluate| API
    Client -->|2. Async Produce| Kafka
    
    API -->|3. Request Decision| Rules
    Rules -->|4. Pull strict user profile \n (< 5ms)| FastStorage
    Rules -->|5. Return (Allow/Block/Flag)| API
    
    Kafka -->|6. Consume Stream| Flink
    Flink -->|7. Update sliding windows \n e.g. 'tx_count_1h'| FastStorage
    
    Kafka -->|8. Archive data| DataLake
    Spark -->|9. Train Models / Build Graphs| GraphDB
    Spark -->|10. Push updated rules| Rules
    
    Rules -.->|11. Flagged transaction| CaseMgmt
```

---

## Part 3: Deep Dive - The Speed Layer (Real-Time Aggregations)

If a rule states: `Block if User X attempts > 5 transactions in the last 10 minutes from a new IP.`

You cannot run `SELECT COUNT(*) FROM transactions WHERE user_id = X AND time > NOW() - 10m` against a Postgres database on the critical synchronous path. It is too slow.

**The Solution: Stateful Stream Processing (Apache Flink & Redis)**

1.  **Event Ingestion**: Every time the user clicks on the site, logs in, or buys an item, the microservice asynchronously drops a JSON event onto a Kafka topic `user_events`.
2.  **Stream Processing (Flink)**: Apache Flink consumes this Kafka topic. Flink maintains *sliding time windows* in its own distributed memory (or backs them up to RocksDB).
    *   Flink sees Transaction 1, 2, 3, 4, 5 arrive.
    *   It constantly aggregates these metrics and writes the finalized features to an ultra-fast in-memory Key-Value store (Redis or Aerospike).
    *   `Redis SET user:123:tx_count_10m = 5`
3.  **The Synchronous Call**: When the 6th transaction hits the `Fraud API`, the Rules Engine does not query Postgres. It queries Redis: `GET user:123:tx_count_10m`. Redis returns `5` in $< 1ms$. The Rule Engine executes `IF 5 >= 5 THEN Block`.

---

## Part 4: Deep Dive - The Identity Graph (Graph Databases)

Fraud rings do not operate in isolation. A single fraudster might create 100 fake accounts, using 5 shared credit cards, routing through 3 VPN IP addresses, and having 2 overlapping physical shipping addresses.
Detecting these convoluted, multi-hop relationships using standard SQL `JOIN`s requires writing nested recursive queries that will literally freeze a relational database.

**The Solution: Graph Databases (Neo4j / Amazon Neptune)**

*   **Vertices (Nodes)**: `User`, `DeviceID`, `IPAddress`, `CreditCard`, `ShippingAddress`.
*   **Edges (Relationships)**: `USED_DEVICE`, `LOGGED_IN_FROM`, `SHIPPED_TO`.

*   **The Query**: When "User 99" tries to make a payment, we asynchronously query the Graph Database (often as part of the Batch or near-real-time layer):
    *   "Find all Users who have shared a DeviceID or IPAddress with User 99. If any of those Users have a status of `FRAUDULENT`, instantly flag User 99's account."
    *   In Neo4j (Cypher query language), this graph traversal takes milliseconds, even if it has to hop through millions of nodes.

---

## Part 5: Deep Dive - Machine Learning Model Serving

Rules are brittle. (`IF amount > 500 AND country="RU"`). Hackers simply change their VPNs and make transactions for $499.
Financial institutions rely on ML models (XGBoost, Neural Networks) to detect subtle, complex anomalies.

*   **The Challenge**: A Data Scientist trains a Python model on a Jupyter Notebook taking 4 hours. How do you deploy this into a Java Spring Boot API that must return a response in 50ms?
*   **Model Format (PMML / ONNX)**: The Data Scientist exports the trained model weights into a serialized, portable format like PMML (Predictive Model Markup Language) or ONNX.
*   **Inference Engine**: The Fraud API Gateway loads this PMML file directly into its JVM memory.
*   **Execution**:
    1.  The API receives the raw transaction JSON.
    2.  It retrieves the User's historical feature vector from Redis (e.g., `[avg_tx_amount: 45, account_age_days: 120, failed_logins: 0]`).
    3.  It concatenates the raw JSON and the Redis features into a single matrix.
    4.  It passes the matrix into the in-memory ML model. The model simply performs heavy matrix multiplication locally on the CPU and outputs a `Risk Score (0.0 to 1.0)`.
    5.  `IF Score > 0.85 THEN Route to Manual Review.`

---

## Interview Questions & Model Answers

**Q1: Our primary Redis cluster holding the real-time aggregations (average spend, login counts) crashes. Does our payment system completely shut down, or do we allow all transactions through?**
*Answer*: In a banking environment, a total payment blackout is economically disastrous and creates monumental reputational damage. We must design for **Graceful Degradation** (Fail-Open with Fallbacks).
If Redis times out or crashes:
1.  The Fraud API Gateway catches the Exception.
2.  It instantly falls back to a purely stateless, hardcoded baseline ruleset that requires zero external network calls (e.g., `IF amount > $50,000 THEN Block. ELSE IF IP is blacklisted in local JVM cache THEN Block. ELSE Allow.`).
3.  We might temporarily restrict the maximum transaction size across the entire platform until Redis recovers.
4.  All allowed transactions during the outage are dumped into a Kafka Priority Queue. When the Fraud systems recover, they retroactively score all bypassed transactions. If an incredibly severe anomaly is detected, we automatically trigger an extreme backend process (e.g., voiding the credit card transfer before the nightly batch settlement window closes).

**Q2: We are experiencing massive false positives. The ML model is blocking legitimate users who are traveling abroad. How do we architect the system to quickly retrain and evaluate new models without destroying production traffic?**
*Answer*: We must implement a **Shadow Deployment (or Dark Launch) Architecture**.
When the Data Science team builds "Model V2" that allegedly fixes the travel bug:
1.  We deploy Model V2 into production alongside Model V1.
2.  When a transaction arrives, the API runs *both* models in parallel.
3.  We use Model V1's decision (Block/Allow) to actually control the user experience.
4.  We take Model V2's decision and silently log it to a Kafka topic ("Shadow Results") without affecting the user.
5.  After 7 days, we query the data lake: "How many times did V1 block, but V2 would have allowed?" We cross-reference this against customer service logs. If V2 significantly reduced the false positive rate while catching the same actual fraud, we promote V2 to the primary decision path.

**Q3: A fraudster writes a script to test 10,000 stole credit card numbers per minute by making $1.00 purchases. How do you detect and stop this velocity attack?**
*Answer*: This is a classic **Carding Attack**. We cannot rely on the slow nightly Batch Layer; they will steal $10,000 before the Hadoop job finishes.
We rely on the **Speed Layer (Apache Flink)**.
1. Flink consumes the Kafka stream of all payment attempts globally.
2. We configure Flink to maintain a Sliding Window partitioned by `IP_Address` or `Device_Fingerprint`.
3. If Flink detects `> 15 unique PANs (Primary Account Numbers) attempted from the same IP within 60 seconds`, it instantly emits a "High Velocity Attack" event.
4. The Fraud API subscribes to this event and writes the IP to an immediate "Hot Blocklist" in Redis.
5. All subsequent requests from that IP are instantly dropped at the WAF (Web Application Firewall) layer with an HTTP 403, completely shielding our core database and external PSP from the brute-force traffic.

## Key Takeaways

*   **Divide and Conquer**: Fraud detection fundamentally requires splitting processing into a sub-50ms Speed Layer (Redis/Flink) and a massive historical Batch Layer (HDFS/Spark).
*   **Graph DBs are essential**: Relational joins cannot navigate complex, multi-hop identity rings efficiently. Use Neo4j to find linked fraudsters.
*   **Fail-Open vs. Fail-Closed**: In finance, understand when a system must cautiously allow degraded service versus completely halting operations.
*   **Shadow Testing**: Never deploy an ML model directly into the critical path without running it in "Dark Mode" on live traffic first to verify its precision and recall.
