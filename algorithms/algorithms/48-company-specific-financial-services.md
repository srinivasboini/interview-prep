# Company Specific: Financial Services (JPMC, Goldman, UBS)

## Overview
Banks focus on **Concurrency**, **Hash Maps**, **Priority Queues**, and **Practical Algorithms**. They care about **correctness** and **thread safety**.

## Top Patterns for Banks
1.  **Hash Maps**: Fast lookups (Caching).
2.  **Priority Queues**: Order matching, scheduling.
3.  **Sliding Window**: Time-series analysis.
4.  **Concurrency**: Producer-Consumer, Thread Pool implementation.

## Top Questions
1.  **Best Time to Buy and Sell Stock** (I, II, III, IV)
2.  **Merge Intervals** (Time blocks)
3.  **Two Sum**
4.  **Design HashMap**
5.  **LRU Cache**
6.  **Min Stack**
7.  **Trapping Rain Water**
8.  **Number of Islands** (Risk clusters)
9.  **Median of Two Sorted Arrays**
10. **Design Underground System** (Tracking stats)

## Java Specifics (CRITICAL)
*   **Collections**: Know `ConcurrentHashMap`, `BlockingQueue`, `CopyOnWriteArrayList`.
*   **Streams**: Be comfortable with Java 8 Streams API.
*   **Memory**: Understand Heap vs Stack, Garbage Collection.

## Interview Structure
*   **CoderPad**: Collaborative coding in a browser.
*   **Super Day**: Back-to-back rounds (Technical + Behavioral).

## Strategy
*   **Ask about Concurrency**: "Is this code running in a multi-threaded environment?"
*   **Variable Types**: Use `BigDecimal` for money, never `double`.
*   **Edge Cases**: Handle nulls, empty inputs, and overflows gracefully.

---
**Next**: [Interview Strategies](49-interview-strategies.md)
