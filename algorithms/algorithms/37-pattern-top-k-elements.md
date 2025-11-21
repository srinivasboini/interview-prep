# Pattern: Top K Elements

## Overview
Find K largest/smallest/frequent elements. Use Heap or QuickSelect.

## Interview Problems

### Problem: Top K Frequent Elements (Medium)
**Pattern**: Min Heap or Bucket Sort

```java
/**
 * Find k most frequent elements.
 * Time: O(N log k)
 */
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> count = new HashMap<>();
    for (int n : nums) count.put(n, count.getOrDefault(n, 0) + 1);
    
    PriorityQueue<Integer> heap = new PriorityQueue<>((a, b) -> count.get(a) - count.get(b));
    
    for (int n : count.keySet()) {
        heap.offer(n);
        if (heap.size() > k) heap.poll();
    }
    
    int[] res = new int[k];
    for (int i = 0; i < k; i++) res[i] = heap.poll();
    return res;
}
```

---
**Next**: [Pattern: K-Way Merge](38-pattern-k-way-merge.md)
