# DP Patterns: 1D

## Overview
1D DP problems involve a linear sequence (array or string) where the state depends on previous indices.

## Common Patterns

### 1. Fibonacci Style
*   `dp[i] = dp[i-1] + dp[i-2]`
*   Examples: Climbing Stairs, House Robber.

### 2. Longest Increasing Subsequence (LIS)
*   `dp[i] = max(dp[j] + 1)` for all `j < i` where `nums[j] < nums[i]`.
*   Complexity: O(N^2) (can be optimized to O(N log N) with Binary Search).

## Interview Problems

### Problem 1: House Robber (Medium)
**Pattern**: 1D DP (Choice: Rob or Skip)

```java
/**
 * Max money without alerting police (cannot rob adjacent).
 * dp[i] = max(nums[i] + dp[i-2], dp[i-1])
 * Time: O(n)
 * Space: O(1)
 */
public int rob(int[] nums) {
    if (nums.length == 0) return 0;
    
    int prev1 = 0;
    int prev2 = 0;
    
    for (int num : nums) {
        int temp = prev1;
        prev1 = Math.max(prev2 + num, prev1);
        prev2 = temp;
    }
    
    return prev1;
}
```

### Problem 2: Longest Increasing Subsequence (Medium)
**Pattern**: LIS

```java
/**
 * Find length of LIS.
 * Time: O(n^2)
 * Space: O(n)
 */
public int lengthOfLIS(int[] nums) {
    if (nums.length == 0) return 0;
    
    int[] dp = new int[nums.length];
    Arrays.fill(dp, 1);
    int maxLen = 1;
    
    for (int i = 1; i < nums.length; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[i] > nums[j]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    
    return maxLen;
}
```

## 🏦 Banking Context: Portfolio Optimization
*   **Scenario**: Selecting a sequence of investments over time to maximize return while adhering to constraints (e.g., "cannot trade same stock 2 days in a row").
*   **Mapping**: This is exactly the **House Robber** pattern (constraint on adjacent choices).

---
**Next**: [DP Patterns: Knapsack](15-dp-patterns-knapsack.md)
