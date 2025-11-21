# DP Patterns: Knapsack

## Overview
The Knapsack problem is the parent of many optimization problems.
*   **0/1 Knapsack**: Items can be picked once or not at all.
*   **Unbounded Knapsack**: Items can be picked multiple times.

## Fundamentals

### 0/1 Knapsack Template
*   **State**: `dp[i][w]` = Max value using first `i` items with capacity `w`.
*   **Transition**: `dp[i][w] = max(dp[i-1][w], value[i] + dp[i-1][w-weight[i]])`.
*   **Space Optimization**: Reduce to 1D array `dp[w]`. Iterate backwards.

### Unbounded Knapsack Template
*   **Transition**: `dp[w] = max(dp[w], value[i] + dp[w-weight[i]])`.
*   **Space Optimization**: 1D array. Iterate forwards.

## Interview Problems

### Problem 1: Partition Equal Subset Sum (Medium)
**Pattern**: 0/1 Knapsack (Subset Sum)

```java
/**
 * Can array be partitioned into two subsets with equal sum?
 * Equivalent to: Find subset with sum = totalSum / 2.
 * Time: O(n * sum)
 * Space: O(sum)
 */
public boolean canPartition(int[] nums) {
    int sum = 0;
    for (int num : nums) sum += num;
    if (sum % 2 != 0) return false;
    
    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    
    for (int num : nums) {
        // Iterate backwards to avoid using same item twice in same step
        for (int j = target; j >= num; j--) {
            dp[j] = dp[j] || dp[j - num];
        }
    }
    
    return dp[target];
}
```

### Problem 2: Coin Change (Medium)
**Pattern**: Unbounded Knapsack

```java
/**
 * Min coins to make up amount.
 * Time: O(amount * coins)
 * Space: O(amount)
 */
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // Initialize with max
    dp[0] = 0;
    
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    
    return dp[amount] > amount ? -1 : dp[amount];
}
```

## 🏦 Banking Context: Asset Allocation
*   **Scenario**: You have a budget (capacity) and a set of possible investments (items), each with a cost and expected return (value).
*   **Goal**: Maximize return without exceeding budget.
*   **Application**: This is literally the **Knapsack Problem**.

---
**Next**: [DP Patterns: Strings](16-dp-patterns-strings.md)
