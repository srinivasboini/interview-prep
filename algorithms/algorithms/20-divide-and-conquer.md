# Divide and Conquer

## Overview
Divide and Conquer works by:
1.  **Divide**: Break problem into subproblems.
2.  **Conquer**: Solve subproblems recursively.
3.  **Combine**: Merge solutions.

## Common Patterns

### 1. Merge Sort
*   Divide array in half, sort halves, merge sorted halves.

### 2. Quick Sort
*   Partition array around pivot, sort partitions.

## Interview Problems

### Problem 1: Sort an Array (Medium)
**Pattern**: Merge Sort

```java
/**
 * Implement Merge Sort.
 * Time: O(N log N)
 * Space: O(N)
 */
public int[] sortArray(int[] nums) {
    if (nums.length <= 1) return nums;
    int mid = nums.length / 2;
    int[] left = sortArray(Arrays.copyOfRange(nums, 0, mid));
    int[] right = sortArray(Arrays.copyOfRange(nums, mid, nums.length));
    return merge(left, right);
}

private int[] merge(int[] left, int[] right) {
    int[] res = new int[left.length + right.length];
    int i = 0, j = 0, k = 0;
    while (i < left.length && j < right.length) {
        if (left[i] < right[j]) res[k++] = left[i++];
        else res[k++] = right[j++];
    }
    while (i < left.length) res[k++] = left[i++];
    while (j < right.length) res[k++] = right[j++];
    return res;
}
```

## 🏦 Banking Context: MapReduce
*   **Scenario**: Processing terabytes of transaction logs.
*   **Algorithm**: **MapReduce** is essentially Divide and Conquer distributed across a cluster.
    *   **Map (Divide)**: Split logs, process locally.
    *   **Reduce (Combine)**: Aggregate results.

---
**Next**: [Binary Search](21-binary-search.md)
