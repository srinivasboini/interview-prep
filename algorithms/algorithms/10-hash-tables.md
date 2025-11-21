# Hash Tables

## Overview
Hash Tables provide **O(1)** average time complexity for insertion, deletion, and lookup. They are the most commonly used data structure in interviews for optimization.

## Fundamentals

### How it Works
*   **Hash Function**: Maps a key to an index in an array (bucket).
*   **Collision Handling**:
    *   **Chaining**: Linked List (or Tree in Java 8+) at each bucket.
    *   **Open Addressing**: Probing next slots.

### Java Implementation
*   `HashMap<K, V>`: Key-Value pairs. Allows one null key.
*   `HashSet<E>`: Unique elements. Backed by HashMap.
*   `LinkedHashMap`: Maintains insertion order.

## Operations and Complexity

| Operation | Average Case | Worst Case (Collisions) |
|-----------|--------------|-------------------------|
| Insert    | O(1)         | O(n)                    |
| Delete    | O(1)         | O(n)                    |
| Search    | O(1)         | O(n)                    |

*Note: Java 8 improves worst case to O(log n) by converting long chains to Red-Black Trees.*

## Common Patterns

### 1. Frequency Map
Count occurrences of elements.
*   `map.put(key, map.getOrDefault(key, 0) + 1);`

### 2. Two Sum Pattern
Check if `target - current` exists in map.

### 3. Deduplication
Use `HashSet` to filter duplicates.

## Visual Diagrams

### Hash Map Collision (Chaining)
```mermaid
graph LR
    Key1[Key: "Apple"] --> Hash1[Hash: 123] --> Bucket3
    Key2[Key: "Banana"] --> Hash2[Hash: 456] --> Bucket5
    Key3[Key: "Grape"] --> Hash3[Hash: 123] --> Bucket3
    
    subgraph "Buckets"
    Bucket3[Index 3] --> Node1["Apple"] --> Node2["Grape"]
    Bucket5[Index 5] --> Node3["Banana"]
    end
```

## Interview Problems

### Problem 1: Two Sum (Easy)
**Pattern**: Hash Map for Complement

```java
/**
 * Find indices of two numbers that add up to target.
 * Time: O(n)
 * Space: O(n)
 */
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[] { map.get(complement), i };
        }
        map.put(nums[i], i);
    }
    
    throw new IllegalArgumentException("No solution");
}
```

### Problem 2: Longest Consecutive Sequence (Medium)
**Pattern**: HashSet for O(1) Lookup

```java
/**
 * Find length of longest consecutive elements sequence.
 * Time: O(n)
 * Space: O(n)
 */
public int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for (int num : nums) set.add(num);
    
    int longest = 0;
    
    for (int num : set) {
        // Only start counting if 'num' is the start of a sequence
        if (!set.contains(num - 1)) {
            int currentNum = num;
            int currentStreak = 1;
            
            while (set.contains(currentNum + 1)) {
                currentNum++;
                currentStreak++;
            }
            
            longest = Math.max(longest, currentStreak);
        }
    }
    
    return longest;
}
```

## 🏦 Banking Context: Caching & Idempotency
*   **Idempotency**: Ensuring processing the same payment request twice doesn't result in double charge.
*   **Implementation**: Use a distributed cache (Redis/Hazelcast - essentially a giant Hash Map) to store `RequestId`. If `RequestId` exists, return previous response.
*   **Distributed Hashing**: Consistent Hashing is used to distribute keys across multiple servers.

## Common Pitfalls
1.  **Mutable Keys**: Never use a mutable object as a key. If the object changes, its hashCode changes, and you lose the value.
2.  **hashCode() vs equals()**: If you override one, you MUST override the other.
3.  **Iteration Order**: `HashMap` does not guarantee order. Use `LinkedHashMap` or `TreeMap` if order matters.

---
**Next**: [Graphs: Fundamentals](11-graphs-fundamentals.md)
