# Case Study: Trading Platform & Order Matching Engine

## Overview

Designing a cryptocurrency exchange (like Coinbase/Binance) or a traditional stock trading platform (like Robinhood) is the apex of low-latency, high-throughput system design. Unlike a standard e-commerce site where adding an item to a cart takes 1 second, a trading platform's **Order Matching Engine** must match buyers and sellers in microseconds. 

If your matching engine is slow, High-Frequency Trading (HFT) firms will exploit arbitrage opportunities, users will experience "slippage" (getting a worse price than they clicked), and the platform will lose millions. 

For a Staff/Principal Engineer, this interview focuses entirely on the data structures required for an Order Book (L2 Cache), in-memory state machines (LMAX Disruptor), and event sourcing for auditability.

---

## Part 1: Requirements Clarification

### Functional
*   Users can place Market Orders (execute immediately at current price) and Limit Orders (execute only at a specific price or better).
*   The system must maintain an Order Book (visible list of active buy/sell orders).
*   The system must match orders perfectly (e.g., Alice's buy meets Bob's sell).
*   Users can view a real-time price ticker and candlestick charts.

### Non-Functional
*   **Ultra-Low Latency**: The Matching Engine must process an order in $< 1ms$ (microseconds if possible).
*   **High Availability**: The platform cannot go down during market volatility (flash crashes).
*   **Strict Consistency**: You absolutely cannot lose a trade, double-spend a user's fiat balance, or execute an order out of sequence. (Consistency > Availability).
*   **Sequential Ordering**: Orders *must* be processed in the exact nanosecond order they were received (FIFO).

---

## Part 2: High-Level Architecture

The architecture is divided into the slow edge (API/Risk) and the hyper-fast core (Matching Engine).

```mermaid
flowchart TD
    Client[Trader URL / Mobile App]
    
    API[API Gateway]
    
    subgraph Risk & Balance Validation (Slow: ms)
        Risk[Risk Engine]
        Ledger[(Relational Database \n User Balances)]
    end
    
    subgraph The Core (Fast: µs)
        Sequencer[Order Sequencer \n (Kafka / Aeron)]
        ME[Order Matching Engine \n (In-Memory)]
    end
    
    subgraph Data Distribution
        Ticker[Ticker Push \n (WebSockets)]
        Archiver[(Cassandra \n Historical Trades)]
    end

    Client -->|1. Place Order| API
    API -->|2. Check Balance| Risk
    Risk <--> Ledger
    
    Risk -->|3. Valid Order| Sequencer
    Sequencer -->|4. Totally Ordered Stream| ME
    
    ME -->|5. Trade Executed Event| Sequencer
    
    Sequencer -->|6. Async Update| Ledger
    Sequencer -->|7. Update Clients| Ticker
    Sequencer -->|8. Save for Audit| Archiver
```

---

## Part 3: Deep Dive - The Order Matching Engine

If Alice wants to buy 1 BTC at \$50,000, and Bob wants to sell 1 BTC at \$50,000, the engine must match them. You cannot do this with SQL (`SELECT * FROM orders WHERE price = 50000`). It is orders of magnitude too slow.

The Matching Engine is a **single-threaded, completely in-memory state machine**. 
Why single-threaded? Because locking (Mutexes) in a multi-threaded environment causes context switching and destroys microsecond performance. A single optimized Java/C++ thread can process millions of operations per second if it never has to wait for disk I/O or network locks.

### The Data Structure: The Limit Order Book (LOB)

The Order Book maintains exactly two sides: Bids (Buyers) and Asks (Sellers).
It must support finding the best price instantly $O(1)$, inserting a new order fast $O(\log N)$, and deleting a canceled order fast $O(1)$.

*   **Bids (Buyers)**: We want the highest price at the top.
*   **Asks (Sellers)**: We want the lowest price at the top.

**The Hybrid Structure:**
1.  **Price Levels (Red-Black Tree or Skip List)**: A sorted map where the key is the Price ($50,000.00), and the value is a pointer to a Queue of orders. This keeps the prices perfectly ordered $O(\log N)$.
2.  **Order Queues (Doubly Linked List)**: At the \$50,000 price level, there might be 50 people waiting. They are arranged in a FIFO linked list. Finding the oldest order is $O(1)$.
3.  **Fast Lookup (HashMap)**: To cancel Order #1234 instantly, we maintain a global HashMap tracking `OrderID -> Pointer to node in Linked List`. Cancellation is $O(1)$.

Whenever a Market Order arrives, the engine simply pops the head off the best price queue and executes the trade.

---

## Part 4: Deep Dive - Event Sourcing & The LMAX Architecture

If the Matching Engine holds all active orders entirely in RAM, what happens if the server loses power? All unmatched orders vanish, and millions of dollars are lost.

We must implement **Event Sourcing** (often utilizing the LMAX Disruptor pattern or a high-performance sequencer like Apache Kafka/Aeron).

1.  **The Sequencer**: Before an order ever touches the in-memory engine, it is durably written to a highly available, strictly ordered commit log (Kafka).
2.  **The State Machine**: The Matching Engine consumes this log sequentially.
3.  **Crash Recovery**: If the engine crashes, the orchestrator spins up a new instance. The new instance starts at Offset 0 of the Kafka log, perfectly replaying every single historical `PLACE_ORDER`, `CANCEL_ORDER`, and `TRADE_EXECUTED` event. Within seconds, it mathematically reconstructs the exact state of the in-memory Order Book and resumes processing new orders where the dead node left off.

---

## Part 5: Deep Dive - Market Data (Candlesticks & WebSockets)

The platform must display the real-time price and historical candlestick charts (Open, High, Low, Close - OHLC) to millions of concurrent users.

1.  **Raw Trades**: The Matching Engine spits out a massive stream of `Trade(Price: 50000, Amount: 1.5)`.
2.  **Stream Aggregation (Flink/Spark Streaming)**: We use Flink to consume the raw trade stream and tumble it into time windows (e.g., 1-minute, 5-minute, 1-hour). 
    *   Flink continuously calculates the OHLC values for that minute in memory.
3.  **Distribution (Redis & WebSockets)**: 
    *   Flink writes the final 1-minute candlestick to a PostgreSQL/TimescaleDB historical table.
    *   More importantly, Flink publishes the tick update to a Redis Pub/Sub channel.
    *   A fleet of lightweight WebSocket servers (Node.js/Go) subscribe to Redis and spray the visual updates to the millions of connected browser clients.

---

## Interview Questions & Model Answers

**Q1: A user submits a Market Buy order for 10 BTC. The best Ask (Seller) is offering 2 BTC at \$50,000. The next seller is offering 8 BTC at \$51,000. How does the Matching Engine process this?**
*Answer*: This illustrates the concept of "walking the book" and "slippage."
The Matching Engine processes this sequentially within its single thread:
1. It matches the first 2 BTC at the \$50,000 level. It creates a `TradeExecuted` event for 2 BTC and removes that Ask order from the internal linked list.
2. The user's order still has 8 BTC remaining. The engine moves up the Red-Black tree to the next price level (\$51,000).
3. It matches the remaining 8 BTC at \$51,000. It creates a second `TradeExecuted` event.
4. The user received an average fill price of \$50,800. This is slippage. The Matching Engine guarantees deterministic, sequential execution against the best available liquidity.

**Q2: We need to scale the Matching Engine because the single C++ thread is maxing out at 100,000 TPS. Can we deploy 5 copies of the Matching Engine behind a Load Balancer?**
*Answer*: Absolutely not. If you put 5 engines behind a Load Balancer, a Buy order might go to Engine A, and the matching Sell order might go to Engine B. They will never meet, and the Order Book loses completely consistent state.
A Matching Engine for a specific trading pair (e.g., `BTC/USD`) must strictly process on exactly *one* thread for correctness.
To scale, we must **Partition by Trading Pair**. We deploy Engine 1 to handle only `BTC/USD`. We deploy Engine 2 to handle `ETH/USD`. Engine 3 handles `TSLA/USD`. Because trades on Bitcoin have zero mathematical overlap with trades on Ethereum, they can be processed in completely isolated parallel universes (Sharding).

**Q3: The system receives a massive influx of orders during a market crash. The Risk Engine (which checks if the user has enough cash in Postgres) slows down to 100ms per check, creating a monstrous backlog before the Sequencer. Users are complaining they can't cancel their trades fast enough. How do you fix the Risk bottleneck?**
*Answer*: We must decouple Risk from the slow standard Relational Database.
We cannot execute a TCP network call to Postgres (`SELECT balance FROM wallets FOR UPDATE`) for every single order. The database connection pool will exhaust.
Instead, we move the User Balances entirely into **In-Memory Grids (Redis / Hazelcast)** cache, treating the cache as the primary synchronous source of truth for available trading margin.
When Alice deposits \$10,000 in fiat via wire transfer, the slow banking API credits Postgres, and updates her Redis active trading balance.
When Alice trades, the Risk Engine *only* decrements her balance in Redis (taking $< 1ms$), allows the order to proceed to the Matching Engine, and an asynchronous worker eventually synchronizes the final settled trade back into the permanent Postgres ledger for compliance.

**Q4: Explain the difference between high availability in a standard Web App vs. an Order Matching Engine.**
*Answer*: In a web app, if a stateless Tomcat server crashes, the AWS Application Load Balancer immediately routes the next HTTP request to a healthy node. The user might have to re-login, but the system is highly available.
In a Matching Engine, the Node holds the entire global state of the market in its RAM. You cannot just route traffic to an empty backup node; it has no orders to match.
High Availability in trading uses **Active-Passive (or Active-Active Replicated State Machines)**.
1. The Sequencer (Kafka/Aeron) multicast streams the same exact order events to *both* the Primary Engine and a Secondary Standby Engine.
2. Both engines process the exact same sequence of orders in memory perfectly.
3. Only the Primary Engine's output (Trades/Ticker data) is published to the downstream databases.
4. If the Primary's physical server burns down, the Secondary is already fully hydrated in memory and exactly synchronized to the nanosecond. The network routing (e.g., floating IP) instantly switches to the Secondary, resulting in a failover that takes $< 1$ second with zero data loss.

## Key Takeaways

*   **Single-Threaded Core**: Avoid distributed locks and DB transactions. Treat the core matching logic as a single-threaded deterministic process wrapped around an in-memory data structure.
*   **Event Sourcing**: All state (the Order Book) is simply a projection of a totally ordered, durable event log (Kafka).
*   **The Data Structure**: Master the combination of Sorted Maps (Price levels) and Doubly Linked Lists (FIFO wait queues) to achieve $O(1)$ and $O(\log N)$ performance.
*   **Partitioning**: You scale matching engines by splitting the asset classes (sharding), never by round-robin load balancing.
