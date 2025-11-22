# DP Patterns: Knapsack - Complete Master Guide

## Overview
The Knapsack problem is one of the most fundamental optimization problems in computer science. It's the parent pattern for hundreds of DP problems involving resource allocation, subset selection, and optimization under constraints.

**Key Insight**: Knapsack = Choose items to maximize value subject to capacity constraint.

For Senior/Staff Engineers, mastering knapsack means:
- Understanding 0/1 vs unbounded variants
- Recognizing knapsack patterns in disguise
- Space optimization from 2D to 1D
- Discussing production applications (resource allocation, portfolio optimization)

---

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [0/1 Knapsack](#01-knapsack)
3. [Unbounded Knapsack](#unbounded-knapsack)
4. [Knapsack Variants](#knapsack-variants)
5. [15+ Solved Problems](#solved-problems)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Fundamentals

### Problem Statement

**Given**:
- `n` items, each with weight `w[i]` and value `v[i]`
- Knapsack with capacity `W`

**Goal**: Maximize total value without exceeding capacity.

### Variants

**0/1 Knapsack**: Each item can be taken at most once  
**Unbounded Knapsack**: Each item can be taken unlimited times  
**Bounded Knapsack**: Each item can be taken at most `k[i]` times  
**Fractional Knapsack**: Can take fractions of items (greedy, not DP)  

---

## 0/1 Knapsack

### Recurrence

```
dp[i][w] = maximum value using first i items with capacity w

dp[i][w] = max(
    dp[i-1][w],              // Don't take item i
    v[i] + dp[i-1][w-w[i]]   // Take item i
)
```

### 2D Solution

```java
/**
 * 0/1 Knapsack - 2D DP.
 * Time: O(n × W), Space: O(n × W)
 */
public int knapsack01(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[][] dp = new int[n + 1][W + 1];
    
    for (int i = 1; i <= n; i++) {
        for (int w = 1; w <= W; w++) {
            // Don't take item i-1
            dp[i][w] = dp[i-1][w];
            
            // Take item i-1 if possible
            if (weights[i-1] <= w) {
                dp[i][w] = Math.max(dp[i][w], 
                                    values[i-1] + dp[i-1][w - weights[i-1]]);
            }
        }
    }
    
    return dp[n][W];
}
```

### 1D Space-Optimized Solution

```java
/**
 * 0/1 Knapsack - 1D DP (space optimized).
 * Time: O(n × W), Space: O(W)
 */
public int knapsack01Optimized(int[] weights, int[] values, int W) {
    int[] dp = new int[W + 1];
    
    for (int i = 0; i < weights.length; i++) {
        // CRITICAL: Iterate backwards to avoid using same item twice
        for (int w = W; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
        }
    }
    
    return dp[W];
}
```

**Why backwards?** To ensure we don't use the same item multiple times in one iteration.

---

## Unbounded Knapsack

### Recurrence

```
dp[w] = maximum value with capacity w

dp[w] = max(dp[w], v[i] + dp[w - w[i]]) for all items i
```

### Solution

```java
/**
 * Unbounded Knapsack.
 * Time: O(n × W), Space: O(W)
 */
public int knapsackUnbounded(int[] weights, int[] values, int W) {
    int[] dp = new int[W + 1];
    
    for (int w = 1; w <= W; w++) {
        for (int i = 0; i < weights.length; i++) {
            if (weights[i] <= w) {
                dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
            }
        }
    }
    
    return dp[W];
}

// Alternative: Iterate items in outer loop, capacity forwards
public int knapsackUnboundedAlt(int[] weights, int[] values, int W) {
    int[] dp = new int[W + 1];
    
    for (int i = 0; i < weights.length; i++) {
        // CRITICAL: Iterate forwards to allow multiple uses
        for (int w = weights[i]; w <= W; w++) {
            dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
        }
    }
    
    return dp[W];
}
```

**Why forwards?** To allow using the same item multiple times.

---

## Knapsack Variants

### Subset Sum

**Problem**: Can we select subset with sum = target?

```java
/**
 * Subset sum - special case of 0/1 knapsack.
 * Time: O(n × sum), Space: O(sum)
 */
public boolean canPartition(int[] nums, int target) {
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    
    for (int num : nums) {
        for (int j = target; j >= num; j--) {
            dp[j] = dp[j] || dp[j - num];
        }
    }
    
    return dp[target];
}
```

### Count Ways

**Problem**: Count number of ways to achieve target sum.

```java
/**
 * Count ways to make sum.
 * Time: O(n × sum), Space: O(sum)
 */
public int countWays(int[] nums, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    
    for (int num : nums) {
        for (int j = target; j >= num; j--) {
            dp[j] += dp[j - num];
        }
    }
    
    return dp[target];
}
```

---

## Solved Problems

### Problem 1: Partition Equal Subset Sum (Medium)

```java
/**
 * Can partition into two equal sum subsets?
 * Time: O(n × sum), Space: O(sum)
 */
public boolean canPartition(int[] nums) {
    int sum = 0;
    for (int num : nums) sum += num;
    if (sum % 2 != 0) return false;
    
    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    
    for (int num : nums) {
        for (int j = target; j >= num; j--) {
            dp[j] = dp[j] || dp[j - num];
        }
    }
    
    return dp[target];
}
```

### Problem 2: Target Sum (Medium)

```java
/**
 * Count ways to assign +/- to reach target.
 * Time: O(n × sum), Space: O(sum)
 */
public int findTargetSumWays(int[] nums, int target) {
    int sum = 0;
    for (int num : nums) sum += num;
    
    if (Math.abs(target) > sum || (sum + target) % 2 != 0) {
        return 0;
    }
    
    int subsetSum = (sum + target) / 2;
    int[] dp = new int[subsetSum + 1];
    dp[0] = 1;
    
    for (int num : nums) {
        for (int j = subsetSum; j >= num; j--) {
            dp[j] += dp[j - num];
        }
    }
    
    return dp[subsetSum];
}
```

### Problem 3: Coin Change (Medium)

```java
/**
 * Minimum coins to make amount.
 * Time: O(amount × coins), Space: O(amount)
 */
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
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

### Problem 4: Coin Change II (Medium)

```java
/**
 * Number of ways to make amount.
 * Time: O(amount × coins), Space: O(amount)
 */
public int change(int amount, int[] coins) {
    int[] dp = new int[amount + 1];
    dp[0] = 1;
    
    for (int coin : coins) {
        for (int i = coin; i <= amount; i++) {
            dp[i] += dp[i - coin];
        }
    }
    
    return dp[amount];
}
```

### Problem 5: Perfect Squares (Medium)

```java
/**
 * Minimum perfect squares that sum to n.
 * Time: O(n × √n), Space: O(n)
 */
public int numSquares(int n) {
    int[] dp = new int[n + 1];
    Arrays.fill(dp, Integer.MAX_VALUE);
    dp[0] = 0;
    
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j * j <= i; j++) {
            dp[i] = Math.min(dp[i], dp[i - j * j] + 1);
        }
    }
    
    return dp[n];
}
```

### Problem 6: Last Stone Weight II (Medium)

```java
/**
 * Minimum possible weight of last stone.
 * Time: O(n × sum), Space: O(sum)
 */
public int lastStoneWeightII(int[] stones) {
    int sum = 0;
    for (int stone : stones) sum += stone;
    
    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    
    for (int stone : stones) {
        for (int j = target; j >= stone; j--) {
            dp[j] = dp[j] || dp[j - stone];
        }
    }
    
    for (int i = target; i >= 0; i--) {
        if (dp[i]) {
            return sum - 2 * i;
        }
    }
    
    return 0;
}
```

### Problem 7: Ones and Zeroes (Medium)

```java
/**
 * Maximum subset size with at most m 0s and n 1s.
 * Time: O(l × m × n), Space: O(m × n)
 */
public int findMaxForm(String[] strs, int m, int n) {
    int[][] dp = new int[m + 1][n + 1];
    
    for (String str : strs) {
        int zeros = 0, ones = 0;
        for (char c : str.toCharArray()) {
            if (c == '0') zeros++;
            else ones++;
        }
        
        for (int i = m; i >= zeros; i--) {
            for (int j = n; j >= ones; j--) {
                dp[i][j] = Math.max(dp[i][j], 1 + dp[i - zeros][j - ones]);
            }
        }
    }
    
    return dp[m][n];
}
```

### Problem 8: Combination Sum IV (Medium)

```java
/**
 * Number of combinations that sum to target (order matters).
 * Time: O(target × n), Space: O(target)
 */
public int combinationSum4(int[] nums, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    
    for (int i = 1; i <= target; i++) {
        for (int num : nums) {
            if (num <= i) {
                dp[i] += dp[i - num];
            }
        }
    }
    
    return dp[target];
}
```

---

## Interview Questions & Answers

### Q1: "Explain the difference between 0/1 and unbounded knapsack."

**Model Answer:**
"The key difference is item reusability:

**0/1 Knapsack**:
- Each item can be used **at most once**
- **Iteration**: Backwards through capacity
- **Recurrence**: `dp[i][w] = max(dp[i-1][w], v[i] + dp[i-1][w-w[i]])`
- **Example**: Selecting stocks to buy (can't buy same stock twice)

```java
for (int i = 0; i < n; i++) {
    for (int w = W; w >= weights[i]; w--) {  // Backwards
        dp[w] = max(dp[w], values[i] + dp[w - weights[i]]);
    }
}
```

**Unbounded Knapsack**:
- Each item can be used **unlimited times**
- **Iteration**: Forwards through capacity
- **Recurrence**: `dp[w] = max(dp[w], v[i] + dp[w-w[i]])`
- **Example**: Making change with coins (unlimited coins of each denomination)

```java
for (int i = 0; i < n; i++) {
    for (int w = weights[i]; w <= W; w++) {  // Forwards
        dp[w] = max(dp[w], values[i] + dp[w - weights[i]]);
    }
}
```

**Why direction matters**:
- **Backwards**: Uses old `dp[w-weight]` (from previous iteration) → item used once
- **Forwards**: Uses new `dp[w-weight]` (from current iteration) → item can be reused

**Production example**:
In portfolio allocation:
- 0/1: Selecting unique assets (can't buy same asset twice)
- Unbounded: Allocating budget across asset types (can buy multiple units)"

### Q2: "How do you optimize knapsack from 2D to 1D?"

**Model Answer:**
"Space optimization exploits the fact that `dp[i][w]` only depends on `dp[i-1][*]`:

**2D approach** (O(n × W) space):
```java
int[][] dp = new int[n+1][W+1];
for (int i = 1; i <= n; i++) {
    for (int w = 1; w <= W; w++) {
        dp[i][w] = dp[i-1][w];  // From previous row
        if (weights[i-1] <= w) {
            dp[i][w] = max(dp[i][w], 
                          values[i-1] + dp[i-1][w-weights[i-1]]);
        }
    }
}
```

**1D approach** (O(W) space):
```java
int[] dp = new int[W+1];
for (int i = 0; i < n; i++) {
    for (int w = W; w >= weights[i]; w--) {  // Backwards!
        dp[w] = max(dp[w], values[i] + dp[w-weights[i]]);
    }
}
```

**Key insights**:
1. **Overwrite in place**: `dp[w]` represents both `dp[i-1][w]` and `dp[i][w]`
2. **Backwards iteration**: Ensures we use old values (from previous iteration)
3. **Start from W**: Only update reachable capacities

**Why backwards is critical**:
```
If forwards:
dp[5] = max(dp[5], v[0] + dp[2])  // Uses NEW dp[2]
dp[2] = max(dp[2], v[0] + dp[-1]) // Already updated!
→ Item used twice!

If backwards:
dp[5] = max(dp[5], v[0] + dp[2])  // Uses OLD dp[2]
dp[2] = max(dp[2], v[0] + dp[-1]) // Updates after
→ Item used once ✓
```

**Production impact**:
For n=1000, W=100,000:
- 2D: 400MB
- 1D: 400KB
This difference is critical for embedded systems or high-frequency trading."

### Q3: "How do you identify a knapsack problem in disguise?"

**Model Answer:**
"I look for these patterns:

**1. Optimization with constraint**:
- Maximize/minimize something
- Subject to capacity/budget/limit
- Example: 'Maximize profit with budget $1M'

**2. Subset selection**:
- Choose items from a set
- Each item has cost and value
- Example: 'Select stocks to maximize return'

**3. Resource allocation**:
- Limited resource to distribute
- Each allocation has benefit
- Example: 'Allocate memory to processes'

**Common disguises**:

**Partition Equal Subset Sum**:
- Looks like: 'Can you split array into two equal sums?'
- Actually: 'Can you select subset with sum = total/2?' → 0/1 knapsack

**Target Sum**:
- Looks like: 'Assign +/- to reach target'
- Actually: 'Select subset with specific sum' → 0/1 knapsack

**Coin Change**:
- Looks like: 'Minimum coins to make amount'
- Actually: 'Fill capacity with minimum items' → Unbounded knapsack

**Perfect Squares**:
- Looks like: 'Minimum squares summing to n'
- Actually: 'Fill capacity with perfect squares' → Unbounded knapsack

**Recognition checklist**:
- ✓ Items with weight/cost?
- ✓ Capacity/budget constraint?
- ✓ Maximize/minimize objective?
- ✓ Each item usable once (0/1) or multiple times (unbounded)?

**Production example**:
In banking, many problems are knapsack:
- Portfolio optimization (select assets, maximize return, budget constraint)
- Trade execution (select orders, maximize volume, capital constraint)
- Resource allocation (select projects, maximize ROI, budget constraint)"

---

## 🏦 Banking & Production Context

### Portfolio Optimization

**Scenario**: Select investments to maximize return within budget.

```java
/**
 * Portfolio optimization using 0/1 knapsack.
 */
class PortfolioOptimizer {
    public List<Investment> selectInvestments(
            List<Investment> investments, 
            double budget) {
        
        int n = investments.size();
        int W = (int) (budget * 100);  // Convert to cents
        
        int[] costs = new int[n];
        int[] returns = new int[n];
        
        for (int i = 0; i < n; i++) {
            costs[i] = (int) (investments.get(i).cost * 100);
            returns[i] = (int) (investments.get(i).expectedReturn * 100);
        }
        
        // DP to find maximum return
        int[][] dp = new int[n + 1][W + 1];
        
        for (int i = 1; i <= n; i++) {
            for (int w = 1; w <= W; w++) {
                dp[i][w] = dp[i-1][w];
                
                if (costs[i-1] <= w) {
                    dp[i][w] = Math.max(dp[i][w], 
                                        returns[i-1] + dp[i-1][w - costs[i-1]]);
                }
            }
        }
        
        // Backtrack to find selected investments
        List<Investment> selected = new ArrayList<>();
        int w = W;
        
        for (int i = n; i > 0 && w > 0; i--) {
            if (dp[i][w] != dp[i-1][w]) {
                selected.add(investments.get(i-1));
                w -= costs[i-1];
            }
        }
        
        return selected;
    }
}

class Investment {
    String name;
    double cost;
    double expectedReturn;
    
    Investment(String name, double cost, double expectedReturn) {
        this.name = name;
        this.cost = cost;
        this.expectedReturn = expectedReturn;
    }
}
```

### Trade Execution Optimization

**Scenario**: Select orders to maximize volume within capital constraint.

```java
/**
 * Trade execution using unbounded knapsack.
 */
class TradeExecutor {
    public int maxVolume(int[] orderSizes, int[] prices, int capital) {
        int[] dp = new int[capital + 1];
        
        for (int i = 0; i < orderSizes.length; i++) {
            for (int c = prices[i]; c <= capital; c++) {
                dp[c] = Math.max(dp[c], orderSizes[i] + dp[c - prices[i]]);
            }
        }
        
        return dp[capital];
    }
}
```

---

## Key Takeaways

1. **0/1 Knapsack**: Each item once, iterate backwards
2. **Unbounded**: Each item unlimited, iterate forwards
3. **Space optimization**: 2D → 1D by iterating backwards
4. **Subset sum**: Special case with boolean DP
5. **Count ways**: Use addition instead of max
6. **Recognition**: Optimization + constraint + subset selection
7. **Production**: Portfolio optimization, resource allocation, trade execution

---

**Next**: [DP Patterns: Strings](16-dp-patterns-strings.md)
