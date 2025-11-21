# Sorting Algorithms

## Overview
While you rarely implement sort from scratch in production, knowing the internals is crucial for understanding complexity and stability.

## Comparison

| Algorithm | Time (Avg) | Time (Worst) | Space | Stable? |
|-----------|------------|--------------|-------|---------|
| Bubble    | O(n^2)     | O(n^2)       | O(1)  | Yes     |
| Merge     | O(n log n) | O(n log n)   | O(n)  | Yes     |
| Quick     | O(n log n) | O(n^2)       | O(log n)| No    |
| Heap      | O(n log n) | O(n log n)   | O(1)  | No      |

## Java Internals
*   `Arrays.sort(int[])`: **Dual-Pivot Quicksort** (Unstable, O(n log n)).
*   `Arrays.sort(Object[])`: **TimSort** (Stable, Merge Sort + Insertion Sort hybrid).

## Interview Problems

### Problem 1: K Closest Points to Origin (Medium)
**Pattern**: Sorting (or Heap)

```java
/**
 * Find K closest points.
 * Time: O(N log N) with Sort
 */
public int[][] kClosest(int[][] points, int k) {
    Arrays.sort(points, (a, b) -> (a[0]*a[0] + a[1]*a[1]) - (b[0]*b[0] + b[1]*b[1]));
    return Arrays.copyOfRange(points, 0, k);
}
```

## 🏦 Banking Context: Transaction Ordering
*   **Scenario**: Displaying transactions in a statement.
*   **Requirement**: **Stability** is crucial. If two transactions happen at the exact same millisecond, their relative order from the database insertion (ID) should be preserved.

---
**Next**: [Searching Algorithms](25-searching-algorithms.md)
