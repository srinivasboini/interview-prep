# Pattern: Sliding Window

## Overview
The Sliding Window pattern is used to perform a required operation on a specific window size of a given array or linked list, such as finding the longest subarray containing all 1s. Sliding Windows start from the 1st element and keep shifting right by one element and adjust the length of the window according to the problem that you are solving.

## When to use
*   Input is a linear data structure (Linked List, Array, String).
*   You're asked to find the longest/shortest substring, subarray, or a desired value.

## Template
```java
public int slidingWindow(String s) {
    int left = 0;
    int result = 0;
    
    for (int right = 0; right < s.length(); right++) {
        // 1. Add element at 'right' to window
        char c = s.charAt(right);
        // update state (e.g., map, count)
        
        // 2. Shrink window from 'left' if condition is violated
        while (/* condition violated */) {
            char leftChar = s.charAt(left);
            // update state (remove leftChar)
            left++;
        }
        
        // 3. Update result
        result = Math.max(result, right - left + 1);
    }
    
    return result;
}
```

## Interview Problems

### Problem: Minimum Window Substring (Hard)
**Pattern**: Variable Size Sliding Window

```java
/**
 * Find smallest substring of s containing all chars of t.
 * Time: O(S + T)
 * Space: O(1) (128 chars)
 */
public String minWindow(String s, String t) {
    if (s == null || t == null || s.length() == 0 || t.length() == 0) return "";
    
    int[] map = new int[128];
    for (char c : t.toCharArray()) map[c]++;
    
    int left = 0, right = 0, minLen = Integer.MAX_VALUE, startIndex = 0;
    int count = t.length();
    
    while (right < s.length()) {
        char c = s.charAt(right++);
        if (map[c]-- > 0) count--; // Found a char in t
        
        while (count == 0) { // Window is valid
            if (right - left < minLen) {
                startIndex = left;
                minLen = right - left;
            }
            
            if (map[s.charAt(left++)]++ == 0) count++; // Lost a char in t
        }
    }
    
    return minLen == Integer.MAX_VALUE ? "" : s.substring(startIndex, startIndex + minLen);
}
```

## 🏦 Banking Context: Rolling Risk
*   **Scenario**: "Calculate the max drawdown over any 30-day window in the last 10 years."
*   **Solution**: Fixed Size Sliding Window.

---
**Next**: [Pattern: Two Pointers](27-pattern-two-pointers.md)
