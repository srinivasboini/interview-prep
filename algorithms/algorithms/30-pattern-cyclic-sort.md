# Pattern: Cyclic Sort - Complete Guide

## Overview
Cyclic Sort places elements at their correct index in O(n) time when dealing with numbers in range [1, n].

**Time**: O(n), **Space**: O(1)

---

## Template
```java
public void cyclicSort(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        int correctIndex = nums[i] - 1;
        if (nums[i] != nums[correctIndex]) {
            swap(nums, i, correctIndex);
        } else {
            i++;
        }
    }
}
```

---

## Problems

### 1. Find Missing Number (Easy)
```java
public int missingNumber(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        if (nums[i] < nums.length && nums[i] != nums[nums[i]]) {
            swap(nums, i, nums[i]);
        } else {
            i++;
        }
    }
    
    for (i = 0; i < nums.length; i++) {
        if (nums[i] != i) return i;
    }
    return nums.length;
}
```

### 2. Find All Duplicates (Medium)
```java
public List<Integer> findDuplicates(int[] nums) {
    List<Integer> result = new ArrayList<>();
    
    for (int i = 0; i < nums.length; i++) {
        int index = Math.abs(nums[i]) - 1;
        if (nums[index] < 0) {
            result.add(index + 1);
        } else {
            nums[index] = -nums[index];
        }
    }
    
    return result;
}
```

---

## 🏦 Banking Context
**Scenario**: Validate sequential transaction IDs.  
**Solution**: Cyclic sort to find missing or duplicate IDs.

---

**Next**: [Pattern: Linked List Reversal](31-pattern-linkedlist-reversal.md)
