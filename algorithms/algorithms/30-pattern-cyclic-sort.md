# Pattern: Cyclic Sort

## Overview
This pattern describes an interesting approach to deal with problems involving arrays containing numbers in a given range.

## When to use
*   Input array contains numbers in range `1 to n` or `0 to n`.
*   Find missing/duplicate numbers.
*   O(n) time and O(1) space required.

## Template
```java
void cyclicSort(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        int correctIdx = nums[i] - 1; // If 1-based
        if (nums[i] != nums[correctIdx]) {
            swap(nums, i, correctIdx);
        } else {
            i++;
        }
    }
}
```

## Interview Problems

### Problem: First Missing Positive (Hard)
**Pattern**: Cyclic Sort

```java
/**
 * Find smallest missing positive integer.
 * Time: O(n)
 * Space: O(1)
 */
public int firstMissingPositive(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        // Only swap if num is in range [1, n] and not in correct spot
        if (nums[i] > 0 && nums[i] <= nums.length && nums[nums[i] - 1] != nums[i]) {
            int correctIdx = nums[i] - 1;
            int temp = nums[i];
            nums[i] = nums[correctIdx];
            nums[correctIdx] = temp;
        } else {
            i++;
        }
    }
    
    for (i = 0; i < nums.length; i++) {
        if (nums[i] != i + 1) return i + 1;
    }
    
    return nums.length + 1;
}
```

---
**Next**: [Pattern: LinkedList Reversal](31-pattern-linkedlist-reversal.md)
