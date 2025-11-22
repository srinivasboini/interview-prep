# Pattern: Sliding Window - Complete Guide

## Overview
The Sliding Window pattern optimizes problems involving contiguous subarrays/substrings by maintaining a window that slides through the data structure, avoiding redundant computations.

**Key Insight**: Instead of recalculating from scratch for each position, maintain state as window slides.

**Time Complexity**: Typically O(n) instead of O(n²)

---

## When to Use
- Input is linear (array, string, linked list)
- Find longest/shortest substring/subarray
- Keywords: "contiguous", "substring", "subarray", "window"

---

## Templates

### Fixed Size Window
```java
public int fixedWindow(int[] arr, int k) {
    int windowSum = 0;
    
    // Initial window
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    
    int maxSum = windowSum;
    
    // Slide window
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k];
        maxSum = Math.max(maxSum, windowSum);
    }
    
    return maxSum;
}
```

### Variable Size Window
```java
public int variableWindow(String s) {
    int left = 0, maxLen = 0;
    Map<Character, Integer> map = new HashMap<>();
    
    for (int right = 0; right < s.length(); right++) {
        // Expand window
        char c = s.charAt(right);
        map.put(c, map.getOrDefault(c, 0) + 1);
        
        // Shrink window if invalid
        while (/* condition violated */) {
            char leftChar = s.charAt(left);
            map.put(leftChar, map.get(leftChar) - 1);
            if (map.get(leftChar) == 0) map.remove(leftChar);
            left++;
        }
        
        // Update result
        maxLen = Math.max(maxLen, right - left + 1);
    }
    
    return maxLen;
}
```

---

## Problems

### 1. Maximum Sum Subarray of Size K (Easy)
```java
public int maxSum(int[] arr, int k) {
    int sum = 0;
    for (int i = 0; i < k; i++) sum += arr[i];
    
    int maxSum = sum;
    for (int i = k; i < arr.length; i++) {
        sum += arr[i] - arr[i - k];
        maxSum = Math.max(maxSum, sum);
    }
    return maxSum;
}
```

### 2. Longest Substring Without Repeating Characters (Medium)
```java
public int lengthOfLongestSubstring(String s) {
    int left = 0, maxLen = 0;
    Set<Character> set = new HashSet<>();
    
    for (int right = 0; right < s.length(); right++) {
        while (set.contains(s.charAt(right))) {
            set.remove(s.charAt(left++));
        }
        set.add(s.charAt(right));
        maxLen = Math.max(maxLen, right - left + 1);
    }
    
    return maxLen;
}
```

### 3. Minimum Window Substring (Hard)
```java
public String minWindow(String s, String t) {
    int[] map = new int[128];
    for (char c : t.toCharArray()) map[c]++;
    
    int left = 0, minLen = Integer.MAX_VALUE, startIndex = 0;
    int count = t.length();
    
    for (int right = 0; right < s.length(); right++) {
        if (map[s.charAt(right)]-- > 0) count--;
        
        while (count == 0) {
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                startIndex = left;
            }
            
            if (map[s.charAt(left++)]++ == 0) count++;
        }
    }
    
    return minLen == Integer.MAX_VALUE ? "" : s.substring(startIndex, startIndex + minLen);
}
```

---

## 🏦 Banking Context
**Scenario**: Calculate max drawdown over any 30-day window in 10 years of stock data.  
**Solution**: Fixed-size sliding window to track min/max prices efficiently.

---

**Next**: [Pattern: Two Pointers](27-pattern-two-pointers.md)
