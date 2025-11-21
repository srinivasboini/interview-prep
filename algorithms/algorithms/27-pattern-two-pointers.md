# Pattern: Two Pointers

## Overview
Two Pointers is a pattern where two pointers iterate through the data structure in tandem until one or both of the pointers hit a certain condition.

## When to use
*   Sorted arrays (or Linked Lists).
*   Need to find a set of elements that fulfill certain constraints (e.g., sum to a value).
*   Need to reverse/swap elements.

## Template
```java
public void twoPointers(int[] nums) {
    int left = 0;
    int right = nums.length - 1;
    
    while (left < right) {
        if (condition(left, right)) {
            return; // Found
        } else if (tooSmall) {
            left++;
        } else {
            right--;
        }
    }
}
```

## Interview Problems

### Problem: 3Sum (Medium)
**Pattern**: Two Pointers (with Sorting)

```java
/**
 * Find all unique triplets that sum to zero.
 * Time: O(n^2)
 * Space: O(1) (excluding output)
 */
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    
    for (int i = 0; i < nums.length - 2; i++) {
        if (i == 0 || (i > 0 && nums[i] != nums[i-1])) { // Skip duplicates
            int lo = i + 1, hi = nums.length - 1, sum = 0 - nums[i];
            
            while (lo < hi) {
                if (nums[lo] + nums[hi] == sum) {
                    res.add(Arrays.asList(nums[i], nums[lo], nums[hi]));
                    while (lo < hi && nums[lo] == nums[lo+1]) lo++; // Skip duplicates
                    while (lo < hi && nums[hi] == nums[hi-1]) hi--; // Skip duplicates
                    lo++; hi--;
                } else if (nums[lo] + nums[hi] < sum) lo++;
                else hi--;
            }
        }
    }
    return res;
}
```

---
**Next**: [Pattern: Fast & Slow Pointers](28-pattern-fast-slow-pointers.md)
