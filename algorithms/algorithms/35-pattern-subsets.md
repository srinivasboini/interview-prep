# Pattern: Subsets

## Overview
Generate all subsets (Power Set) using BFS or Backtracking.

## Interview Problems

### Problem: Subsets (Medium)
**Pattern**: Cascading (BFS)

```java
/**
 * Generate all subsets.
 * Time: O(N * 2^N)
 * Space: O(N * 2^N)
 */
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    result.add(new ArrayList<>());
    
    for (int num : nums) {
        int size = result.size();
        for (int i = 0; i < size; i++) {
            List<Integer> subset = new ArrayList<>(result.get(i));
            subset.add(num);
            result.add(subset);
        }
    }
    return result;
}
```

---
**Next**: [Pattern: Modified Binary Search](36-pattern-modified-binary-search.md)
