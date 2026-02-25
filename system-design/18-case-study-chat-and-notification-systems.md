# Case Study: Chat Systems (WhatsApp/Discord) & Notifications

## Overview

Designing a global chat application like WhatsApp, Discord, or Facebook Messenger is one of the most complex challenges in system design. It touches upon every advanced distributed systems concept: maintaining millions of long-lived persistent connections, guaranteeing message ordering, handling high-throughput write-heavy workloads, and synchronizing state across multiple devices.

For a Staff/Principal Engineer, an interviewer is looking for deep knowledge of connection management (WebSockets vs. Long-Polling), how state is maintained when connections drop, and how to query message history efficiently from a database like Cassandra or HBase without crushing the cluster.

In the banking context, this translates directly to building real-time secure customer support chat, high-frequency internal trader chat rooms, or broadcasting critical fraud alerts (Push Notifications) to millions of devices simultaneously.

---

## Part 1: Design a Chat System (WhatsApp/Discord)

### 1. Requirements Clarification
*   **Functional:**
    *   1-on-1 chatting with low latency delivery.
    *   Group chatting (up to 500 members).
    *   Online presence indicator (Online/Offline/Typing).
    *   Multi-device support (Desktop & Mobile synchronized).
    *   Message history storage.
*   **Non-Functional:**
    *   **Low Latency**: Messages must arrive in < 200ms.
    *   **High Connections**: 500 Million Daily Active Users (DAU), 50 Million concurrent connections.
    *   **Consistency**: Messages must appear exactly in the order they were sent.

### 2. Back-of-the-Envelope Estimation (Capacity Planning)
*   **Concurrent Connections**: 50 Million.
*   **Write QPS**: If an average user sends 40 messages/day.
    *   500M * 40 = 20 Billion messages/day.
    *   20 Billion / 86400 = **~230,000 QPS**. (Extremely write-heavy).
*   **Storage**: 20 Billion * 100 bytes/message = 2 TB/day -> **730 TB/year**.
*   **Memory for Connections**: Each WebSocket connection requires ~50KB of memory on the server.
    *   50 Million * 50 KB = **~2.5 Terabytes** of RAM distributed across our Chat Servers.

### 3. High-Level Architecture

```mermaid
flowchart TD
    Alice[Client Alice]
    Bob[Client Bob]
    
    API[API Gateway \n (Stateless HTTP)]
    
    subgraph Stateful WebSocket Servers
        Chat1[Chat Server 1 \n (Holds Alice's TCP Conn)]
        Chat2[Chat Server 2 \n (Holds Bob's TCP Conn)]
        Sessions[(Session Service \n Redis: User -> Server Mapping)]
    end
    
    subgraph Message Bus
        Kafka[Kafka / Event Bus]
    end
    
    subgraph Storage
        DB[(Cassandra / HBase)]
    end

    Alice <-->|1. Persistent WebSocket| Chat1
    Chat1 -->|2. Register Mapping| Sessions
    
    Alice -->|3. Send Message to Bob| Chat1
    Chat1 -->|4. Push Event| Kafka
    Kafka -->|5. Save to DB| DB
    
    Kafka -->|6. Lookup Bob's Server| Sessions
    Sessions -.->|Bob is on Chat2| Chat1
    Chat1 -->|7. Route Message via RPC| Chat2
    Chat2 -->|8. Push to Bob| Bob
```

### 4. Deep Dive: Connection Management & State

A standard stateless HTTP backend (Spring Boot / Express) will collapse if 1 million clients hold open HTTP connections simultaneously (thread exhaustion).

1.  **WebSockets:** The protocol of choice. It provides a full-duplex, persistent TCP connection. The server can push data to Alice exactly when it arrives, without Alice polling.
2.  **Stateful Chat Servers:** Unlike stateless REST APIs, a Chat Server is intrinsically stateful. When Alice connects, the Load Balancer routes her to `Chat Server 12`. That specific server holds her specific TCP socket in memory.
3.  **The Session Mapping (Redis):** If Alice sends a message to Bob, how does `Chat Server 12` know which server holds Bob's connection?
    *   We maintain a global, low-latency Redis cache: Map `<UserID>` -> `<Chat Server IP>`.
    *   `Chat 12` queries Redis: "Where is Bob?". Redis replies: "Chat Server 45".
    *   `Chat 12` makes an internal high-speed gRPC call to `Chat 45` delivering the message payload. `Chat 45` pushes it down Bob's WebSocket.

### 5. Deep Dive: Message Storage (Cassandra)

We have 230,000 writes per second. A relational database (PostgreSQL) cannot handle this without complex sharding logic. We must use a Wide-Column store like **Apache Cassandra** or **HBase** (what Facebook originally used).

*   **The Schema:**
    To query a user's chat history instantly, we partition the data by `channel_id` (a hash of Alice and Bob's IDs) and cluster (sort) the data on disk by the message ID.
    ```sql
    CREATE TABLE messages (
        channel_id text,       -- Partition Key: All messages for this chat on 1 node
        message_id timeuuid,   -- Clustering Key: Sorted chronologically on disk
        sender_id text,
        content text,
        created_at timestamp,
        PRIMARY KEY (channel_id, message_id)
    ) WITH CLUSTERING ORDER BY (message_id DESC);
    ```

*   **Message ID Generation (Ordering):** 
    We cannot use a random UUID; it destroys the chronological sorting on disk. We cannot use a centralized SQL auto-increment ID (it will bottleneck).
    We use **Twitter Snowflake** or a local DB sequence (like Cassandra's TimeUUID), generating a 64-bit ID where the first 41 bits represent the exact millisecond timestamp. This guarantees chronological ordering across the distributed system.

### 6. Deep Dive: Group Chats

1-on-1 chat involves looking up 1 recipient. A group chat with 500 people means 1 message generates 500 distinct delivery events (Message Fanout).

*   **Write Fanout (Push):** Alice sends a message to the group. The server looks up all 500 members in the `group_members` table, finds their 500 respective Chat Servers in Redis, and pushes 500 RPC calls simultaneously.
    *   *Pros*: Instant delivery.
    *   *Cons*: If a celebrity posts in a group of 100,000 users, pushing 100,000 WebSockets instantly will crash the servers.
*   **Read Fanout (Pull):** Alice posts the message to the DB. The server pushes a tiny notification to the other users indicating a new message exists, and their clients explicitly `HTTP GET` the new messages from the DB. Used for massive channels (Discord/Slack).

---

## Part 2: Design a Push Notification System

Sending millions of push notifications (Apple APNS, Google FCM) is notoriously slow and unreliable.

### 1. The Architecture

*   **The Problem**: Sending an HTTP request to Apple's APNS takes up to 50ms. If we try to send 1 million alerts sequentially in a Java `for` loop, it will take 14 hours. We must decouple and parallelize.

```mermaid
flowchart LR
    App[Internal Service \n e.g., Fraud Alert]
    
    API[Notification Gateway]
    Kafka[Kafka Topic \n (High Volume Buffer)]
    Workers[Fleet of 500 Worker Pods]
    
    APNS[Apple APNS API]
    FCM[Google FCM API]

    App -->|1. Trigger Alert Event| API
    API -->|2. Write Event| Kafka
    Kafka -->|3. Consume Batch| Workers
    
    Workers -->|4. Async HTTP POST| APNS
    Workers -->|4. Async HTTP POST| FCM
```

### 2. Deep Dive: Avoiding Duplicates & Retries

*   **Idempotency / Deduplication**: A banking system generates the `Fraud_Alert:123` event twice due to a network retry. We cannot text the user twice. Before writing to Kafka, the Notification Gateway checks a fast Redis cache (`SETNX alert:123 true EX 3600`). If it already exists, drop the duplicate.
*   **Failures at the Edge (APNS)**: If Apple's servers return a `429 Too Many Requests` or a `500 Server Error`, the Worker *must not* drop the message. It places the failure payload onto a **Kafka Retry Topic**. A separate slow-moving worker consumes the Retry Topic with Exponential Backoff (10s, 30s, 2m) until it succeeds.

---

## Interview Questions & Model Answers

**Q1: In the real-time chat architecture, how do you handle a user's multi-device sync history (e.g., Alice has WhatsApp open on her phone and her laptop)?**
*Answer*: We must synchronize state using a "Sync Token" or "High Watermark."
When Alice's phone connects, she queries her local SQLite database for the highest `message_id` she has seen (e.g., ID: 500). The phone establishes the WebSocket and sends an initialization command: `SYNC_FROM: 500`. The Chat Server queries Cassandra (`SELECT * FROM messages WHERE channel_id = X AND message_id > 500`) and pushes all missing messages down the socket.
When Alice's laptop turns on a week later, it might send `SYNC_FROM: 100`. The server pulls the massive backlog from Cassandra and synchronizes the laptop. Every device independently tracks its own localized High Watermark.

**Q2: We need to design the "Online/Offline/Typing" presence indicator. Sending a ping every 1 second from 500 million devices will crash our Redis session database. How do we optimize this?**
*Answer*: We cannot treat Presence as a synchronous, highly consistent database operation. 
1. **Heartbeats**: The client WebSockets send a tiny heartbeat ping every 10 seconds. The Chat Server aggregates these locally in memory.
2. **Batching**: Every 10 seconds, the Chat Server takes its local memory map of 10,000 connected users and sends *one single batch update* to Redis updating their `last_active_at` timestamps.
3. **Lazy Loading**: If Alice has 500 contacts, we do not continuously query Redis for all 500. We only query the presence of the 10 contacts currently visible on her screen.
4. **Typing Indicators**: We treat "Typing..." strictly as a transient Pub/Sub event via Kafka, bypassing the database entirely. If the event is dropped over the network, it doesn't matter; it's ephemeral data.

**Q3: Our CEO wants to send an emergency "System Downtime" push notification to all 50 million bank customers *instantly*. If our APNS workers can handle 50,000 req/sec, this will take 16 minutes, which is too slow. How do we accelerate it?**
*Answer*: This is a classic map-reduce scatter problem.
1. The CEO hits the API API.
2. We place 1 single message onto a Kafka "Broadcaster" topic.
3. A Broadcaster worker reads it, queries the User Database to get the total count (50 Million). It breaks the user base into smaller shards (e.g., User IDs 1-1,000,000; 1M-2M, etc.) and pushes 50 smaller messages onto a "Fanout" topic.
4. Fanout workers read those 50 messages. Each worker queries the database for the exact APNS device tokens for their specific 1 million users, constructing massive batch JSON arrays.
5. They push these batches directly into the final Delivery Queue.
Furthermore, we scale our Worker pods from 100 to 2000 dynamically (using K8s HPA based on Kafka lag) to completely exhaust the external APNS rate limits.

## Key Takeaways

*   **WebSockets vs HTTP**: Stateful protocols are mandatory for real-time systems. Understand the thread-exhaustion limits of HTTP Long-Polling.
*   **Centralized State (Redis)**: You must maintain a highly available mapping of User ID to specific generic WebSocket Server IP coordinates.
*   **Disk Storage for Append-Only workloads**: Relational DBs cannot handle 230,000 QPS of inserts without heavy sharding. Cassandra's SSTable architecture is built precisely for massive write ingestion and chronological row sorting.
*   **Asynchronous Fanout**: Never execute heavy I/O loops synchronously. Use Kafka to break large "Send to 1 million users" commands into thousands of parallelizable tiny tasks.
