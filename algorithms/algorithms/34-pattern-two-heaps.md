# Pattern: Two Heaps

## Overview
Use two heaps (Min Heap and Max Heap) to divide data into two parts. Useful for Median finding.

## Interview Problems

### Problem: IPO (Hard)
**Pattern**: Two Heaps

```java
/**
 * Maximize capital.
 * Time: O(N log N)
 */
public int findMaximizedCapital(int k, int w, int[] profits, int[] capital) {
    PriorityQueue<int[]> minCapital = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    PriorityQueue<Integer> maxProfit = new PriorityQueue<>((a, b) -> b - a);
    
    for (int i = 0; i < profits.length; i++) {
        minCapital.offer(new int[]{capital[i], profits[i]});
    }
    
    for (int i = 0; i < k; i++) {
        while (!minCapital.isEmpty() && minCapital.peek()[0] <= w) {
            maxProfit.offer(minCapital.poll()[1]);
        }
        if (maxProfit.isEmpty()) break;
        w += maxProfit.poll();
    }
    
    return w;
}
```

---
**Next**: [Pattern: Subsets](35-pattern-subsets.md)
