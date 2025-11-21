# Pattern: Merge Intervals

## Overview
This pattern deals with overlapping intervals. In a lot of problems involving intervals, you either need to find overlapping intervals or merge intervals if they overlap.

## When to use
*   Input is a collection of intervals.
*   "Merge overlapping", "Insert interval", "Meeting rooms".

## Interview Problems

### Problem: Insert Interval (Medium)
**Pattern**: Merge Intervals

```java
/**
 * Insert newInterval into sorted intervals and merge.
 * Time: O(n)
 * Space: O(n)
 */
public int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> result = new ArrayList<>();
    int i = 0;
    
    // Add all intervals ending before newInterval starts
    while (i < intervals.length && intervals[i][1] < newInterval[0]) {
        result.add(intervals[i++]);
    }
    
    // Merge overlapping intervals
    while (i < intervals.length && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.add(newInterval);
    
    // Add remaining
    while (i < intervals.length) {
        result.add(intervals[i++]);
    }
    
    return result.toArray(new int[result.size()][]);
}
```

---
**Next**: [Pattern: Cyclic Sort](30-pattern-cyclic-sort.md)
