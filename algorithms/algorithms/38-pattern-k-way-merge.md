# Pattern: K-Way Merge

## Overview
Merge K sorted arrays/lists.

## Interview Problems

### Problem: Smallest Range Covering Elements from K Lists (Hard)
**Pattern**: Min Heap

```java
/**
 * Find smallest range that includes at least one number from each of the k lists.
 * Time: O(N log K)
 */
public int[] smallestRange(List<List<Integer>> nums) {
    PriorityQueue<int[]> minHeap = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    int max = Integer.MIN_VALUE;
    
    for (int i = 0; i < nums.size(); i++) {
        int val = nums.get(i).get(0);
        minHeap.offer(new int[]{val, i, 0});
        max = Math.max(max, val);
    }
    
    int minRange = Integer.MAX_VALUE;
    int start = -1, end = -1;
    
    while (minHeap.size() == nums.size()) {
        int[] curr = minHeap.poll();
        int min = curr[0];
        int listIdx = curr[1];
        int elemIdx = curr[2];
        
        if (max - min < minRange) {
            minRange = max - min;
            start = min;
            end = max;
        }
        
        if (elemIdx + 1 < nums.get(listIdx).size()) {
            int nextVal = nums.get(listIdx).get(elemIdx + 1);
            minHeap.offer(new int[]{nextVal, listIdx, elemIdx + 1});
            max = Math.max(max, nextVal);
        }
    }
    
    return new int[]{start, end};
}
```

---
**Next**: [Pattern: Topological Sort](39-pattern-topological-sort.md)
