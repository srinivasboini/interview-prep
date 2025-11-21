# Searching Algorithms

## Overview
Searching is the process of retrieving information stored in some data structure.

## Types
1.  **Linear Search**: O(n). Works on unsorted data.
2.  **Binary Search**: O(log n). Works on sorted data.
3.  **BFS/DFS**: For Graphs/Trees.

## 🏦 Banking Context: KYC Search
*   **Scenario**: Searching for a customer by SSN.
*   **Implementation**:
    *   **Database**: B-Tree Index (O(log N)).
    *   **Cache**: Hash Map (O(1)).
    *   **Fuzzy**: Elasticsearch (Inverted Index).

---
**Next**: [Pattern: Sliding Window](26-pattern-sliding-window.md)
