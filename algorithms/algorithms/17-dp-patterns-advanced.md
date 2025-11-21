# DP Patterns: Advanced

## Overview
These patterns appear less frequently but distinguish top candidates.
*   **Matrix Chain Multiplication (MCM)**: Interval DP.
*   **Bitmask DP**: Small constraints (N <= 20).
*   **Digit DP**: Counting numbers with properties.

## Interview Problems

### Problem 1: Burst Balloons (Hard)
**Pattern**: Interval DP (MCM Style)

```java
/**
 * Max coins by bursting balloons.
 * Time: O(n^3)
 * Space: O(n^2)
 */
public int maxCoins(int[] nums) {
    int n = nums.length;
    int[] arr = new int[n + 2];
    arr[0] = 1; arr[n + 1] = 1;
    System.arraycopy(nums, 0, arr, 1, n);
    
    int[][] dp = new int[n + 2][n + 2];
    
    for (int len = 1; len <= n; len++) {
        for (int left = 1; left <= n - len + 1; left++) {
            int right = left + len - 1;
            for (int k = left; k <= right; k++) {
                dp[left][right] = Math.max(dp[left][right],
                    dp[left][k - 1] + 
                    arr[left - 1] * arr[k] * arr[right + 1] + 
                    dp[k + 1][right]);
            }
        }
    }
    
    return dp[1][n];
}
```

## 🏦 Banking Context: Optimal Execution
*   **Scenario**: Breaking down a large block trade to minimize market impact.
*   **Model**: Similar to **Matrix Chain Multiplication**, finding the optimal sequence of splits to minimize cost function.

---
**Next**: [Backtracking and Recursion](18-backtracking-and-recursion.md)
