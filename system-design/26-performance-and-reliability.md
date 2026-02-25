# Performance and Reliability at Scale

## Overview

A system that works perfectly for 1,000 users will fundamentally break when subjected to 10 million users. As a Staff/Principal Engineer, you are responsible for identifying where the system will break *before* it happens, and engineering mechanisms to protect the platform during catastrophic load spikes or hardware failures.

Performance (speed) and Reliability (correctness and uptime) are often opposing forces. A system that writes asynchronously to memory is blazing fast (Performance) but will lose data if the server loses power (Reliability). Navigating this trade-off is the essence of senior-level engineering.

In enterprise banking, Reliability is the non-negotiable baseline. A bank cannot afford to be "fast but occasionally wrong."

---

## 1. Defining the Metrics

You cannot improve what you cannot measure.

### Service Level Agreements (SLAs)
A legally binding contract with customers (e.g., "The Payment Gateway will have 99.99% uptime, or we refund 10% of your monthly bill").
*   **99% (Two Nines)**: ~3.6 days of downtime per year. (Acceptable for internal tools).
*   **99.9% (Three Nines)**: ~8.7 hours of downtime per year. (Standard consumer apps).
*   **99.99% (Four Nines)**: ~52.6 minutes of downtime per year. (Enterprise SaaS/Banking).
*   **99.999% (Five Nines)**: ~5.26 minutes of downtime per year. (Telecoms, AWS core infrastructure). Achieving this requires hyper-expensive active-active multi-region architectures.

### Service Level Objectives (SLOs) and Indicators (SLIs)
*   **SLI**: The actual measurement (e.g., "The 99th percentile latency of the `/checkout` endpoint over the last 5 minutes").
*   **SLO**: The internal goal set by the engineering team (e.g., "The SLI must securely remain under 200ms for 99.9% of all requests in a 30-day rolling window"). The SLO must be stricter than the SLA.

---

## 2. Performance Bottlenecks and Optimization

When an application is slow, it is almost always blocked by one of four finite resources: CPU, Memory, Network I/O, or Disk I/O.

### Database Tuning (Disk I/O avoidance)
The database is the most common bottleneck because disk reads (even on SSDs) are orders of magnitude slower than RAM access.
1.  **Indexing**: A missing B-Tree index causes a Full Table Scan (every single row is loaded from disk into memory to find a match). This is an $O(N)$ operation that destroys performance. Adding an index makes it $O(\log N)$.
    *   *Caveat*: Every index slows down `INSERT`/`UPDATE` operations because the index must also be updated. Over-indexing is a common anti-pattern.
2.  **Connection Pooling**: Opening a TCP connection to Postgres takes several milliseconds and heavy CPU handshaking. Applications must use a Connection Pool (e.g., HikariCP). The pool maintains 50 constantly open, idle connections to the database. Threads checking out a connection takes $< 1\mu s$.
3.  **Read Replicas**: Route all complex, heavy `SELECT` queries (e.g., reporting dashboards) to secondary read-only database nodes, freeing up the primary Master node's CPU specifically for low-latency `INSERT`/`UPDATE` transactions.

### Application Tuning (CPU / Memory)
1.  **Thread Pool Exhaustion**: In Java/Tomcat, every incoming HTTP request consumes one OS thread. If the application makes a slow 5-second HTTP call to a downstream legacy system, that thread is blocked (waiting). If 200 users hit this endpoint simultaneously, all 200 threads block. The 201st user receives an instant "Connection Refused" error, even if the CPU is at 2%.
    *   *Solution*: Asynchronous Non-Blocking I/O (e.g., Spring WebFlux, Netty, Node.js, Go Goroutines). A single Event Loop thread can handle 10,000 concurrent network connections if it doesn't block while waiting.
2.  **Garbage Collection (JVM)**: Creating millions of temporary Java objects per second fills the Eden space rapidly. The JVM must pause all application threads ("Stop the World") to clean up memory. A 2-second GC pause looks exactly like a 2-second network latency spike to the user.
    *   *Solution*: Tune the JVM (G1GC or Shenandoah), increase Heap size, and rigorously profile the application to eliminate unnecessary object allocations within `while` loops.

---

## 3. Reliability Patterns (Protecting the System)

If a system cannot handle a spike, it must defend itself. A system that crashes completely is worse than a system that serves 50% of users and rejects the rest.

### 1. Rate Limiting and Throttling
We discussed this in the API Gateway section. You must aggressively protect your backend databases from DDOS or accidental internal retry storms by enforcing strict quotas per User ID or IP address at the absolute edge of your network (Token Bucket / Redis).

### 2. Circuit Breakers (The Hystrix/Resilience4j Pattern)
If the Fraud Checking Microservice goes down, the Payment Service will sit there trying to connect, waiting 5 seconds for a TCP timeout, and exhausting its thread pool.
*   **Closed**: Traffic flows normally.
*   **Open**: The Fraud Service throws 50 errors in 10 seconds. The Circuit Breaker trips "Open". The Payment Service instantly stops attempting to call the Fraud Service. Any call immediately returns a predefined static Fallback response (e.g., "Proceed with payment, flag for manual review tomorrow") in 1ms, saving the Payment Service's threads.
*   **Half-Open**: After 30 seconds, it lets 1 request through to test the waters. If it succeeds, the circuit closes.

### 3. Bulkheads (Isolation)
Divide resources so failure in one area doesn't cascade.
If the API has `/payments` (critical) and `/generate-pdf-statement` (heavy CPU, not critical), do not let them share the same generic Tomcat thread pool. Configure 180 threads strictly for Payments, and 20 threads for PDFs. If the PDF generation hangs entirely, the Payment engine remains mathematically untouched and operational.

### 4. Timeouts and Exponential Backoff Retries
*   **Timeouts**: Never execute a network call (HTTP, DB, Redis) without a strict, aggressive timeout (e.g., 500ms). If a downstream system is broken, failing fast is better than hanging indefinitely.
*   **Retries**: If a network packet drops, try again.
    *   *The Danger*: If the downstream database is struggling to breathe at 100% CPU, and 50 microservices instantly retry, they effectively DDOS the database, guaranteeing it crashes.
    *   *The Solution*: **Exponential Backoff with Jitter**. First retry: wait 100ms. If fail, wait 200ms. Then 400ms. Then 800ms. 
    *   **Jitter**: Add randomness (e.g., $100ms \pm 20ms$). If 10,000 clients all fail at exactly 12:00:00, without jitter, they will all perfectly sync up and execute their retry at exactly 12:00:01, repeatedly crushing the database in waves. Jitter desynchronizes the herd.

---

## Interview Questions & Model Answers

**Q1: Our primary PostgreSQL database is hitting 95% CPU utilization during peak trading hours. We cannot vertically scale the AWS RDS instance any further. A full database shard reorganization will take 6 months. What tactical steps can you take this week to drop the CPU?**
*Answer*: Throwing hardware at the problem failed. I would target the workload immediately via a profiling mindset.
1.  **Analyze `pg_stat_statements`**: I would generate a report of the top 10 most expensive average queries. 
2.  **Missing Indices**: Are any of those heavy queries performing Sequential Table Scans (`Seq Scan`)? I would examine the execution plan (`EXPLAIN ANALYZE`) and immediately create targeted composite B-Tree indices to turn those heavy scans into lightweight index lookups.
3.  **Connection Pool Tuning**: If there are 5,000 open connections but only 50 active queries, the kernel is wasting massive CPU on context switching. I'd install `PgBouncer` to pool connections efficiently.
4.  **Offload Reads**: I'd identify read-heavy dashboard queries and forcefully route them to a Read Replica instance.
5.  **Caching**: I'd identify static queries (e.g., fetching a static list of support countries) and wrap them in a Redis or local Guava cache with a 5-minute TTL, completely eliminating those queries from hitting the database CPU altogether.

**Q2: We launched an email marketing campaign. Users click the link, and immediately make an API request to "Claim Reward." The API checks their eligibility against an extremely slow older legacy CRM system that can only handle 50 requests per second. The marketing campaign results in 500 requests per second. Our API threads exhaust and crash. How do we insulate our API?**
*Answer*: This is a classic **Load Leveling / Buffering** scenario. The API must not wait synchronously for the CRM.
1. We implement **Queue-Based Load Leveling**. The user clicks the link. The API validates the JWT token, does *not* call the CRM, and immediately places the `ClaimRewardEvent(UserId)` onto an AWS SQS queue or a Kafka topic.
2. The API returns an immediate HTTP `202 Accepted` to the mobile app (taking $< 2ms$). The UI displays "Your reward is being processed."
3. We deploy a separate pool of background Worker Pods. We strictly limit their concurrency (e.g., 50 threads total). These workers pull messages from the Queue and call the slow legacy CRM at a highly controlled, steady pace of 50 per second. The CRM survives, and the Queue simply absorbs the 500 req/sec spike gracefully over a few minutes.

**Q3: Explain the "Thundering Herd" problem in the context of expiring tokens or cache invalidation, and how to prevent it.**
*Answer*: 
Imagine an expensive database query whose result (e.g., a massive JSON blob of the day's active traders) is cached in Redis with a TTL of 1 hour, explicitly expiring at 1:00 PM.
At exactly 1:00:01 PM, 10,000 users request the page. All 10,000 threads hit Redis and receive a Cache Miss simultaneously. All 10,000 threads instantly fire the identical expensive query against the database, crashing it.
**Solutions**:
*   **Distributed Locking (Mutex)**: When the 10,000 threads miss the cache, they all attempt to acquire a Redis Lock `SETNX build_cache_lock`. Only *one* thread wins. That single thread executes the heavy DB query and repopulates Redis. The remaining 9,999 threads explicitly wait (sleep for 50ms loops) and then magically read the fresh data from Redis once the winner releases the lock.
*   **Probabilistic Early Recomputation**: If the cache expires at 1:00 PM, a script allows a random 1% of users hitting the cache at 12:59 PM to artificially perceive a miss, preemptively computing the cache *before* it actually disappears globally.

**Q4: A third-party API returns HTTP 500 errors sporadically due to their own internal deploy process. How do we build resilience without causing an outage?**
*Answer*:
1. I would implement the **Circuit Breaker Pattern** using a library like Resilience4j. If the API fails 10% of the time, the circuit stays closed, and we rely on localized **Retries with Exponential Backoff and Jitter**.
2. If the third-party deploy goes catastrophically wrong, and they fail 50% of the time over a 30-second window, the circuit trips Open. We stop actively calling them.
3. We must implement a **Fallback Mechanism**. If the call was to fetch live stock prices, the Fallback might return the last known cached price from Redis with a "Delayed Data" warning flag, allowing our UI to continue functioning safely rather than displaying a blank error screen.

## Key Takeaways

*   **Protect the DB**: The database is the most precious resource. Use connection pooling, Read Replicas, indexing, and CQRS to shield it from heavy IO constraints.
*   **Buffer Spikes with Queues**: Use Kafka/SQS to absorb sudden massive traffic spikes (Load Leveling) to protect slow, fragile downstream dependencies.
*   **Circuit Breakers**: Anticipate that downstream APIs *will* fail. Detect the failure quickly, stop sending traffic (to save your own threads), and return graceful fallbacks.
*   **Retries require Jitter**: Never write a plain `while(failed) { sleep(1000); retry; }` loop. Without chaotic jitter, 5,000 pods will synchronize their retry bursts and DDoS your own infrastructure.
