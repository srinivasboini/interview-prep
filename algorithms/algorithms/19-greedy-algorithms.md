# Greedy Algorithms

## Overview
Greedy algorithms make the **locally optimal choice** at each step with the hope of finding a global optimum.
*   **Pros**: Simple, Fast.
*   **Cons**: Doesn't always work (need to prove correctness).

## Common Patterns

### 1. Interval Scheduling
*   Sort by **end time**. Pick non-overlapping intervals.

### 2. Huffman Coding
*   Lossless data compression.

## Interview Problems

### Problem 1: Merge Intervals (Medium)
**Pattern**: Sorting + Greedy

```java
/**
 * Merge overlapping intervals.
 * Time: O(N log N)
 * Space: O(N)
 */
public int[][] merge(int[][] intervals) {
    if (intervals.length <= 1) return intervals;
    
    // Sort by start time
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    
    List<int[]> result = new ArrayList<>();
    int[] currentInterval = intervals[0];
    result.add(currentInterval);
    
    for (int[] interval : intervals) {
        int currentEnd = currentInterval[1];
        int nextBegin = interval[0];
        int nextEnd = interval[1];
        
        if (currentEnd >= nextBegin) {
            // Overlap, merge
            currentInterval[1] = Math.max(currentEnd, nextEnd);
        } else {
            // No overlap, add new
            currentInterval = interval;
            result.add(currentInterval);
        }
    }
    
    return result.toArray(new int[result.size()][]);
}
```

### Problem 2: Jump Game (Medium)
**Pattern**: Greedy (Farthest Reach)

```java
/**
 * Can you reach the last index?
 * Time: O(n)
 * Space: O(1)
 */
public boolean canJump(int[] nums) {
    int maxReach = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false; // Cannot reach current index
        maxReach = Math.max(maxReach, i + nums[i]);
    }
    return true;
}
```

## 🏦 Banking Context: ATM Cash Dispensing
*   **Scenario**: User withdraws $180.
*   **Algorithm**: Greedy approach (Give largest bills first: $100, then $50, then $20, then $10).
*   **Note**: This works for standard currency denominations (Canonical Coin Systems). It fails for arbitrary coin systems (requires DP).

---
**Next**: [Divide and Conquer](20-divide-and-conquer.md)
