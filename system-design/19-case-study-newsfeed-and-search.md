# Case Study: Newsfeed (Instagram/Twitter) and Distributed Search

## Overview

Designing a Newsfeed or Timeline (like Twitter, Instagram, or Facebook) is a masterclass in read/write optimizations and caching strategies. The core challenge is simple to describe but monstrous to execute at scale: User A follows 500 people. When User A opens the app, the system must instantly aggregate the most recent, relevant posts from all 500 people, sort them (chronologically or algorithmically), and return the first 20 within 200 milliseconds.

For a Staff/Principal Engineer, the interview centers around the **Fan-out** problem. Do you pre-compute the feed when someone posts (Push model), or do you compute it on-the-fly when someone opens the app (Pull model)? 

In a banking context, this is identical to creating a unified "Recent Activity Dashboard" that aggregates transactions, system alerts, loan payment reminds, and secure messages scattered across 15 different backend mainframe systems into a single seamless UI.

---

## Part 1: Design a Newsfeed System

### 1. Requirements Clarification
*   **Functional:**
    *   Users can publish posts (text/images).
    *   Users can follow other users.
    *   Users view a Newsfeed aggregating posts from people they follow.
    *   Chronological sorting (to simplify the scope, algorithmic sorting is an extension).
*   **Non-Functional:**
    *   **High Availability**: The feed must always load.
    *   **Low Latency**: Feed generation `< 200ms`.
    *   **Scale**: 300 Million Daily Active Users (DAU), 10 Million posts per day.
    *   Follow graph can be extreme: A normal user follows 200 people. A celebrity (Justin Bieber) has 100 million followers.

### 2. Back-of-the-Envelope Estimation
*   **Write QPS**: 10M posts / 86400s ≈ **115 QPS**. (Low write volume).
*   **Read QPS**: 300M DAU * 5 feed refreshes/day = 1.5 Billion / 86400s ≈ **17,300 QPS**. (Extreme read volume).
*   **Storage**: 10M posts * 1KB avg text size = 10 GB/day. (Images/Videos are stored externally in S3/CDNs, only metadata URLs are in the DB).

### 3. High-Level Architecture (The Fanout Models)

The absolute core of this interview is the Fanout Strategy.

#### Strategy 1: Fan-out on Read (The Pull Model)
*   *How it works*: When Justin Bieber posts, we just save the post to the DB: `INSERT INTO posts (user_id, content)`. That's 1 write operation.
*   *The Feed Load*: When User A opens the app, the system does:
    1.  `SELECT follows_id FROM user_follows WHERE user_id = A` (Gets 500 IDs).
    2.  `SELECT * FROM posts WHERE user_id IN (ID1...ID500) ORDER BY time DESC LIMIT 20`.
*   *Pros*: Instant writes. Storage is cheap (no duplication).
*   *Cons*: The reads are devastatingly slow. Doing a massive aggregated query across 500 different users' tables on every single feed refresh (17,300 times a second) will instantly crash the database.

#### Strategy 2: Fan-out on Write (The Push Model)
*   *How it works*: User A opens the app. The system simply queries a pre-computed Redis list: `LRANGE feed:UserA 0 20`. It takes 1 millisecond.
*   *The Post Action*: When User B posts, the system:
    1. Gets User B's followers (User A, User C...).
    2. Pushes the Post ID into *every single follower's* individual Redis list: `LPUSH feed:UserA PostID`, `LPUSH feed:UserC PostID`.
*   *Pros*: Blisteringly fast reads for all users.
*   *Cons*: The Write volume is explosive. If Justin Bieber (100M followers) posts a photo, the system must execute 100 Million `LPUSH` operations to Redis instantly. This causes immense lag and queue backups (The Celebrity Problem).

#### Strategy 3: The Hybrid Model (The Enterprise Standard)
We combine both strategies.
*   For **Normal Users** (e.g., < 10,000 followers), we use **Fan-out on Write**. It's fast to push to 500 Redis lists.
*   For **Celebrities** (> 10,000 followers), we use **Fan-out on Read**. We *do not* push their posts to millions of Redis lists. 
*   *The Feed Assembly*: When User A opens the app, the system fetches their fast pre-computed Redis feed (`LRANGE feed:UserA 0 20`), *and* it concurrently queries the database for recent posts from the 3 Celebrities they happen to follow. The API merges the lists in memory, sorts them, and returns them to the user.

### 4. The Data Model (Graph + Wide Column)

*   **User Graph**: Who follows whom. Best modeled in a Graph Database (Neo4j) or a strongly optimized Relational structure:
    ```sql
    CREATE TABLE follows (
        follower_id bigint,     -- Who is looking (User A)
        followee_id bigint,     -- Who they watch (User B)
        created_at timestamp,
        PRIMARY KEY (follower_id, followee_id)
    );
    ```
*   **Post Storage**: Massive scale, requires Cassandra or DynamoDB.
    ```sql
    CREATE TABLE user_posts (
        user_id bigint,         -- Partition Key
        post_id timeuuid,       -- Clustering Key (Chronological sort)
        content text,
        PRIMARY KEY (user_id, post_id)
    );
    ```

### 5. Newsfeed Architecture Diagram

```mermaid
flowchart TD
    UserA[User A \n (Reads Feed)]
    UserB[User B \n (Writes Post)]
    
    API[API Gateway]
    
    subgraph Write Path
        PostSvc[Post Service]
        FanoutWorker[Kafka + Fanout Workers]
    end
    
    subgraph Storage & Cache
        PostDB[(Cassandra: Posts)]
        GraphDB[(Follower Graph DB)]
        
        Cache[(Redis Cluster \n Pre-computed Feeds)]
    end
    
    subgraph Read Path
        FeedSvc[Feed Aggregator Service]
    end

    UserB -->|1. POST /posts| API
    API --> PostSvc
    PostSvc -->|2. Save Post| PostDB
    PostSvc -->|3. Publish Event| FanoutWorker
    
    FanoutWorker -->|4. Get 500 Followers| GraphDB
    FanoutWorker -->|5. LPUSH PostID| Cache
    
    UserA -->|6. GET /feed| API
    API --> FeedSvc
    FeedSvc -->|7. Fetch Feed List| Cache
    FeedSvc --->|8. O(1) Fetch Post Details| PostDB
    FeedSvc -.->|9. Merge Celeb Posts \n (Pull Model)| PostDB
```

---

## Part 2: Design a Distributed Search System (Elasticsearch)

If a user wants to search for "Tech Conference 2025" across all 10 billion historical posts, standard SQL `LIKE '%Tech%'` on a massive Postgres table will result in a full table scan, taking several minutes and locking the database.

To implement lightning-fast text search, we require an **Inverted Index**, typically powered by Elasticsearch/Lucene.

### 1. The Inverted Index concept
Instead of storing documents mapped to words:
`Post 1 -> "The quick brown fox"`
`Post 2 -> "A fast brown dog"`

An inverted index maps *words* to the Documents they appear in:
`brown -> [Post 1, Post 2]`
`fox -> [Post 1]`
`fast -> [Post 2]`

When a user searches for "brown fox", Elasticsearch finds the intersection of lists instantly: `[Post 1]`.

### 2. Distributed Architecture (Elasticsearch)
*   **Sharding**: An index containing 10 billion posts cannot fit on one node. We shard the index based on Document ID (e.g., `hash(post_id) % 5 nodes`).
*   **The Scatter-Gather Search**:
    1.  The user queries the API Gateway for "brown fox".
    2.  The API sends the query to a Coordinating Node in Elasticsearch.
    3.  **Scatter**: The Coordinating Node fires the query to *all 5 shards* simultaneously.
    4.  Each shard searches its local inverted index and returns its Top 10 results (along with the TF-IDF relevance score).
    5.  **Gather**: The Coordinating Node receives 50 results, merges them, sorts them by the final relevance score, and returns the absolute Top 10 to the user.

### 3. Real-Time Indexing Pipeline
If the Newsfeed is built on Cassandra, how does the data get into Elasticsearch?
*   **Anti-Pattern (Dual Write)**: The Post Service saves to Cassandra, then makes a synchronous HTTP call to save to Elasticsearch. If the ES network fails, data diverges.
*   **Enterprise Pattern (Change Data Capture)**: 
    1.  The Post Service only writes to Cassandra (the Source of Truth).
    2.  We deploy Debezium (or utilize DynamoDB Streams) to tail the transaction log.
    3.  Every insert is streamed as an event into Kafka.
    4.  A Logstash (or Kafka Connect) worker consumes the event and asynchronously bulk-indexes the document into Elasticsearch. 
    5.  *Result*: Perfect decoupling and eventual consistency (search latency is typically < 2 seconds).

---

## Interview Questions & Model Answers

**Q1: In the Fan-out on Write model, what happens if I am inactive and haven't opened the app in two years? Will the system continually compute and store my feed in Redis?**
*Answer*: No, that would be a catastrophic waste of expensive Redis RAM. We must implement **Active User Optimization**. 
The Fan-out workers querying the Graph DB for followers should check the `last_login_date` of each follower. If a follower hasn't logged in within the last 14 days, the worker skips them entirely. When that inactive user finally logs in months later, they will experience a cache miss in Redis. The system will fall back to the slower "Fan-out on Read" query hitting Cassandra once, populate their Redis list, and resume standard push behavior.

**Q2: We need to implement pagination for the Newsfeed when the user scrolls down. Why can't we just use `OFFSET 20 LIMIT 20`, and what should we use instead?**
*Answer*: `OFFSET` relies on integer positions. If A, B, and C are my top 3 feed posts, and I view page 1 (`LIMIT 3 OFFSET 0`), I see A, B, C. While I am reading, 2 new posts arrive at the top (X, Y). The internal database list is now X, Y, A, B, C, D, E. When I scroll down and request Page 2 (`LIMIT 3 OFFSET 3`), the database skips the first 3 items (X, Y, A) and returns B, C, D. I have just seen posts B and C localized on my screen *twice*. This is a terrible user experience.
Instead, we must use **Cursor-Based Pagination** (Keyset). The client passes the exact `message_id` or `timestamp` of the last post they saw (e.g., `?limit=20&older_than=timestamp_of_C`). The database uses an index jump to immediately fetch records older than C, perfectly avoiding duplication regardless of new insertions at the top of the feed.

**Q3: When integrating Elasticsearch for text search, how do you handle partial word matches, typos, or synonyms (e.g., searching "run" but finding "running")?**
*Answer*: This happens during the *Analysis phase* of building the Inverted Index inside Elasticsearch/Lucene. When a post says "The user is running fast", it passes through an Analyzer pipeline:
1. **Tokenizer**: Splits the string into words `["The", "user", "is", "running", "fast"]`.
2. **Stop-word Filter**: Removes generic, pointless words `["user", "running", "fast"]`.
3. **Stemmer**: Reduces words to their root or base form `["user", "run", "fast"]`.
This exact parsed output is what is saved into the Inverted Index. When the user searches for "run", the search query undergoes the *same* stemming process, perfectly matching the root word stored in the index. Typos are handled via Levenshtein Distance (Fuzzy Queries) during query execution, which finds words that are 1 or 2 character edits away from the search term.

**Q4: In our Hybrid Fanout model, User A opens the app. The API pulls their pre-computed Redis feed and concurrently queries Cassandra for Justin Bieber's recent posts. How do you merge and sort these two streams efficiently?**
*Answer*: Both streams (the Redis list of Post IDs, and the Cassandra result set) must be intrinsically sorted by timestamp (or TimeUUID).
To merge them in the API gateway, we simply use the classic **Merge two sorted arrays** algorithm (using two pointers or a Min-Heap). Because both arrays are already sorted descending by time, the API compares the top item of the Redis list with the top item of the Celebrity list, picks the newest, and advances the pointer. This operation runs in $O(N)$ time locally in the API container memory, taking negligible CPU cycles. We take the top 20 items, fully hydrate the post payloads, and return them.

## Key Takeaways

*   **Fan-out is fundamental**: Understand the trade-offs. Push models optimize for Read performance but suffer write explosions (Celebrity problem). Pull models optimize for Writes but crush the DB on Reads. The Hybrid model is the standard Staff-level answer.
*   **Redis lists over SQL queries**: An aggregated feed must be pre-computed and stored in memory (Redis List or Sorted Set). Relational joins at read-time will not scale past 10,000 users.
*   **Cursor Pagination over Offset**: Never use OFFSET. It degrades performance (scanning skipped rows) and creates duplicated visual UI records due to data drift.
*   **Decouple Search**: Avoid dual writes. Ensure Elasticsearch is synchronized asynchronously using CDC (Change Data Capture) and Kafka from the primary datastore.
