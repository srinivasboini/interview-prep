# Pattern: Modified Binary Search

## Overview
Binary Search on non-standard inputs (rotated arrays, infinite streams, answer space).

## Interview Problems

### Problem: Search in Rotated Sorted Array II (Medium)
**Pattern**: Binary Search with Duplicates

```java
/**
 * Search in rotated sorted array with duplicates.
 * Time: O(log n) avg, O(n) worst
 */
public boolean search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return true;
        
        if (nums[left] == nums[mid] && nums[mid] == nums[right]) {
            left++; right--; // Skip duplicates
        } else if (nums[left] <= nums[mid]) {
            if (target >= nums[left] && target < nums[mid]) right = mid - 1;
            else left = mid + 1;
        } else {
            if (target > nums[mid] && target <= nums[right]) left = mid + 1;
            else right = mid - 1;
        }
    }
    return false;
}
```

---
**Next**: [Pattern: Top K Elements](37-pattern-top-k-elements.md)
