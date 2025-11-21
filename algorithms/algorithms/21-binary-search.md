# Binary Search

## Overview
Binary Search finds a target in a **sorted** collection in **O(log n)** time. It's one of the most powerful optimization techniques.

## Fundamentals

### The Template
```java
public int binarySearch(int[] nums, int target) {
    int left = 0;
    int right = nums.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2; // Prevent overflow
        
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    
    return -1;
}
```

## Common Patterns

### 1. Search in Rotated Sorted Array
*   One half is always sorted. Check if target is in that half.

### 2. Binary Search on Answer
*   If the answer space is monotonic (e.g., "Can we do it in X hours?"), we can binary search for the minimum X.

## Interview Problems

### Problem 1: Search in Rotated Sorted Array (Medium)
**Pattern**: Modified Binary Search

```java
/**
 * Search in rotated array.
 * Time: O(log n)
 */
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;
        
        // Left half is sorted
        if (nums[left] <= nums[mid]) {
            if (target >= nums[left] && target < nums[mid]) right = mid - 1;
            else left = mid + 1;
        } 
        // Right half is sorted
        else {
            if (target > nums[mid] && target <= nums[right]) left = mid + 1;
            else right = mid - 1;
        }
    }
    return -1;
}
```

### Problem 2: Koko Eating Bananas (Medium)
**Pattern**: Binary Search on Answer

```java
/**
 * Find min speed K to eat all bananas in H hours.
 * Time: O(N log M) where M is max pile size.
 */
public int minEatingSpeed(int[] piles, int h) {
    int left = 1, right = 1_000_000_000;
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canEatAll(piles, h, mid)) {
            right = mid;
        } else {
            left = mid + 1;
        }
    }
    return left;
}

private boolean canEatAll(int[] piles, int h, int k) {
    int hours = 0;
    for (int pile : piles) {
        hours += (pile + k - 1) / k; // Ceiling division
    }
    return hours <= h;
}
```

## 🏦 Banking Context: Historical Data Query
*   **Scenario**: "Find the first transaction after timestamp T".
*   **Data**: Transactions are stored in sorted order by time.
*   **Algorithm**: **Binary Search** (specifically `lower_bound`) to find the entry point in O(log N).

---
**Next**: [Bit Manipulation](22-bit-manipulation.md)
