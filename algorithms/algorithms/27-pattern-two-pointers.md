# Pattern: Two Pointers - Complete Guide

## Overview
Two Pointers technique uses two pointers to iterate through data structure, often to find pairs or optimize space.

**Time**: O(n) instead of O(n²)  
**Space**: O(1)

---

## Templates

### Opposite Ends
```java
public boolean twoSum(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    
    while (left < right) {
        int sum = arr[left] + arr[right];
        if (sum == target) return true;
        else if (sum < target) left++;
        else right--;
    }
    return false;
}
```

### Same Direction
```java
public int removeDuplicates(int[] arr) {
    int slow = 0;
    for (int fast = 1; fast < arr.length; fast++) {
        if (arr[fast] != arr[slow]) {
            arr[++slow] = arr[fast];
        }
    }
    return slow + 1;
}
```

---

## Problems

### 1. Two Sum II (Easy)
```java
public int[] twoSum(int[] numbers, int target) {
    int left = 0, right = numbers.length - 1;
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        if (sum == target) return new int[]{left + 1, right + 1};
        else if (sum < target) left++;
        else right--;
    }
    return new int[]{-1, -1};
}
```

### 2. Container With Most Water (Medium)
```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxArea = 0;
    
    while (left < right) {
        int area = Math.min(height[left], height[right]) * (right - left);
        maxArea = Math.max(maxArea, area);
        
        if (height[left] < height[right]) left++;
        else right--;
    }
    
    return maxArea;
}
```

### 3. 3Sum (Medium)
```java
public List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;
        
        int left = i + 1, right = nums.length - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                while (left < right && nums[left] == nums[left+1]) left++;
                while (left < right && nums[right] == nums[right-1]) right--;
                left++;
                right--;
            } else if (sum < 0) left++;
            else right--;
        }
    }
    
    return result;
}
```

---

## 🏦 Banking Context
**Scenario**: Find pairs of transactions that sum to suspicious amount.  
**Solution**: Two pointers on sorted transaction amounts.

---

**Next**: [Pattern: Fast & Slow Pointers](28-pattern-fast-slow-pointers.md)
