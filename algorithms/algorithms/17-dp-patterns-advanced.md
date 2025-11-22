# DP Patterns: Advanced - Complete Master Guide

## Overview
Advanced DP patterns include interval DP, bitmask DP, digit DP, and state machine DP. These patterns appear in harder problems and distinguish senior engineers who can recognize and apply complex DP techniques.

**Key Insight**: Advanced DP extends basic patterns with additional dimensions, constraints, or optimization techniques.

For Senior/Staff Engineers, mastering advanced DP means:
- Understanding interval DP (Matrix Chain Multiplication style)
- Implementing bitmask DP for subset problems
- Recognizing digit DP patterns
- Discussing production applications (trade execution, resource optimization)

---

## Table of Contents
1. [Interval DP](#interval-dp)
2. [Bitmask DP](#bitmask-DP)
3. [Digit DP](#digit-dp)
4. [State Machine DP](#state-machine-dp)
5. [10+ Solved Problems](#solved-problems)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Interval DP

### Fundamentals

**Pattern**: `dp[i][j]` = optimal solution for interval `[i, j]`

**Recurrence**: Try all possible split points `k` in `[i, j]`

```
dp[i][j] = min/max over k in [i, j] of:
    dp[i][k] + dp[k+1][j] + cost(i, j, k)
```

### Template

```java
public int intervalDP(int[] arr) {
    int n = arr.length;
    int[][] dp = new int[n][n];
    
    // Base case: intervals of length 1
    for (int i = 0; i < n; i++) {
        dp[i][i] = baseCase(i);
    }
    
    // Fill by increasing interval length
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            dp[i][j] = Integer.MAX_VALUE;
            
            // Try all split points
            for (int k = i; k < j; k++) {
                dp[i][j] = Math.min(dp[i][j],
                    dp[i][k] + dp[k+1][j] + cost(i, j, k));
            }
        }
    }
    
    return dp[0][n-1];
}
```

---

## Bitmask DP

### Fundamentals

**Pattern**: Use bits to represent state (subset membership)

**Constraint**: Typically n ≤ 20 (2^20 = 1M states)

**State**: `dp[mask]` where bit i indicates if element i is included

### Template

```java
public int bitmaskDP(int[] arr) {
    int n = arr.length;
    int[] dp = new int[1 << n];  // 2^n states
    
    // Base case
    dp[0] = 0;
    
    // Iterate all states
    for (int mask = 0; mask < (1 << n); mask++) {
        // Try adding each element
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) == 0) {  // Element i not in mask
                int newMask = mask | (1 << i);
                dp[newMask] = Math.max(dp[newMask], 
                                       dp[mask] + value(mask, i));
            }
        }
    }
    
    return dp[(1 << n) - 1];  // All elements
}
```

---

## Digit DP

### Fundamentals

**Pattern**: Count numbers with certain properties in range [L, R]

**Approach**: Build numbers digit by digit, tracking constraints

### Template

```java
public int digitDP(int n) {
    String s = String.valueOf(n);
    int len = s.length();
    
    // dp[pos][tight][...other states]
    Integer[][][] dp = new Integer[len][2][...];
    
    return solve(0, true, ...initial states, s, dp);
}

private int solve(int pos, boolean tight, ...states, String s, Integer[][][] dp) {
    if (pos == s.length()) {
        return satisfiesCondition(states) ? 1 : 0;
    }
    
    if (dp[pos][tight ? 1 : 0][...] != null) {
        return dp[pos][tight ? 1 : 0][...];
    }
    
    int limit = tight ? (s.charAt(pos) - '0') : 9;
    int result = 0;
    
    for (int digit = 0; digit <= limit; digit++) {
        result += solve(pos + 1, 
                       tight && (digit == limit),
                       ...updated states, s, dp);
    }
    
    return dp[pos][tight ? 1 : 0][...] = result;
}
```

---

## State Machine DP

### Fundamentals

**Pattern**: Model problem as state transitions

**Example**: Stock trading with cooldown

```java
/**
 * Best time to buy/sell stock with cooldown.
 * States: hold, sold, cooldown
 */
public int maxProfit(int[] prices) {
    int sold = 0;
    int held = Integer.MIN_VALUE;
    int cooldown = 0;
    
    for (int price : prices) {
        int prevSold = sold;
        int prevHeld = held;
        int prevCooldown = cooldown;
        
        sold = prevHeld + price;
        held = Math.max(prevHeld, prevCooldown - price);
        cooldown = Math.max(prevCooldown, prevSold);
    }
    
    return Math.max(sold, cooldown);
}
```

---

## Solved Problems

### Problem 1: Burst Balloons (Hard)

```java
/**
 * Maximum coins from bursting balloons.
 * Time: O(n³), Space: O(n²)
 */
public int maxCoins(int[] nums) {
    int n = nums.length;
    int[] arr = new int[n + 2];
    arr[0] = 1;
    arr[n + 1] = 1;
    System.arraycopy(nums, 0, arr, 1, n);
    
    int[][] dp = new int[n + 2][n + 2];
    
    for (int len = 1; len <= n; len++) {
        for (int left = 1; left <= n - len + 1; left++) {
            int right = left + len - 1;
            
            for (int k = left; k <= right; k++) {
                dp[left][right] = Math.max(dp[left][right],
                    dp[left][k-1] + 
                    arr[left-1] * arr[k] * arr[right+1] + 
                    dp[k+1][right]);
            }
        }
    }
    
    return dp[1][n];
}
```

### Problem 2: Minimum Cost to Merge Stones (Hard)

```java
/**
 * Minimum cost to merge stones into one pile.
 * Time: O(n³), Space: O(n²)
 */
public int mergeStones(int[] stones, int k) {
    int n = stones.length;
    if ((n - 1) % (k - 1) != 0) return -1;
    
    int[] prefix = new int[n + 1];
    for (int i = 0; i < n; i++) {
        prefix[i + 1] = prefix[i] + stones[i];
    }
    
    int[][] dp = new int[n][n];
    
    for (int len = k; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            dp[i][j] = Integer.MAX_VALUE;
            
            for (int mid = i; mid < j; mid += k - 1) {
                dp[i][j] = Math.min(dp[i][j], 
                                    dp[i][mid] + dp[mid + 1][j]);
            }
            
            if ((j - i) % (k - 1) == 0) {
                dp[i][j] += prefix[j + 1] - prefix[i];
            }
        }
    }
    
    return dp[0][n - 1];
}
```

### Problem 3: Palindrome Partitioning II (Hard)

```java
/**
 * Minimum cuts for palindrome partitioning.
 * Time: O(n²), Space: O(n²)
 */
public int minCut(String s) {
    int n = s.length();
    boolean[][] isPalin = new boolean[n][n];
    
    // Build palindrome table
    for (int i = n - 1; i >= 0; i--) {
        for (int j = i; j < n; j++) {
            if (s.charAt(i) == s.charAt(j)) {
                isPalin[i][j] = (j - i <= 2) || isPalin[i + 1][j - 1];
            }
        }
    }
    
    int[] dp = new int[n];
    for (int i = 0; i < n; i++) {
        if (isPalin[0][i]) {
            dp[i] = 0;
        } else {
            dp[i] = i;
            for (int j = 1; j <= i; j++) {
                if (isPalin[j][i]) {
                    dp[i] = Math.min(dp[i], dp[j - 1] + 1);
                }
            }
        }
    }
    
    return dp[n - 1];
}
```

### Problem 4: Traveling Salesman Problem (Hard)

```java
/**
 * TSP using bitmask DP.
 * Time: O(n² × 2^n), Space: O(n × 2^n)
 */
public int tsp(int[][] dist) {
    int n = dist.length;
    int[][] dp = new int[1 << n][n];
    
    for (int[] row : dp) {
        Arrays.fill(row, Integer.MAX_VALUE / 2);
    }
    
    dp[1][0] = 0;
    
    for (int mask = 1; mask < (1 << n); mask++) {
        for (int u = 0; u < n; u++) {
            if ((mask & (1 << u)) == 0) continue;
            
            for (int v = 0; v < n; v++) {
                if ((mask & (1 << v)) != 0) continue;
                
                int newMask = mask | (1 << v);
                dp[newMask][v] = Math.min(dp[newMask][v],
                                          dp[mask][u] + dist[u][v]);
            }
        }
    }
    
    int result = Integer.MAX_VALUE;
    for (int i = 0; i < n; i++) {
        result = Math.min(result, dp[(1 << n) - 1][i] + dist[i][0]);
    }
    
    return result;
}
```

### Problem 5: Count Numbers with Unique Digits (Medium)

```java
/**
 * Count numbers with unique digits using digit DP.
 * Time: O(n), Space: O(1)
 */
public int countNumbersWithUniqueDigits(int n) {
    if (n == 0) return 1;
    
    int result = 10;  // n=1: 0-9
    int uniqueDigits = 9;
    int availableNumbers = 9;
    
    for (int i = 2; i <= n && availableNumbers > 0; i++) {
        uniqueDigits *= availableNumbers;
        result += uniqueDigits;
        availableNumbers--;
    }
    
    return result;
}
```

### Problem 6: Best Time to Buy/Sell Stock with Cooldown (Medium)

```java
/**
 * Stock trading with cooldown - state machine DP.
 * Time: O(n), Space: O(1)
 */
public int maxProfit(int[] prices) {
    int sold = 0;
    int held = Integer.MIN_VALUE;
    int cooldown = 0;
    
    for (int price : prices) {
        int prevSold = sold;
        int prevHeld = held;
        int prevCooldown = cooldown;
        
        sold = prevHeld + price;
        held = Math.max(prevHeld, prevCooldown - price);
        cooldown = Math.max(prevCooldown, prevSold);
    }
    
    return Math.max(sold, cooldown);
}
```

### Problem 7: Stone Game (Medium)

```java
/**
 * Stone game - interval DP.
 * Time: O(n²), Space: O(n²)
 */
public boolean stoneGame(int[] piles) {
    int n = piles.length;
    int[][] dp = new int[n][n];
    
    for (int i = 0; i < n; i++) {
        dp[i][i] = piles[i];
    }
    
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            dp[i][j] = Math.max(piles[i] - dp[i+1][j],
                                piles[j] - dp[i][j-1]);
        }
    }
    
    return dp[0][n-1] > 0;
}
```

---

## Interview Questions & Answers

### Q1: "Explain interval DP and when to use it."

**Model Answer:**
"Interval DP solves problems on contiguous subarrays/substrings:

**Pattern**: `dp[i][j]` = optimal solution for interval `[i, j]`

**Recurrence**: Try all split points:
```java
for (int k = i; k < j; k++) {
    dp[i][j] = optimize(dp[i][k], dp[k+1][j], cost(i, j, k));
}
```

**When to use**:
1. Problem involves contiguous intervals
2. Optimal solution depends on splitting interval
3. Subproblems overlap

**Classic problems**:
- **Matrix Chain Multiplication**: Optimal parenthesization
- **Burst Balloons**: Which balloon to burst last
- **Palindrome Partitioning**: Minimum cuts
- **Merge Stones**: Optimal merge order

**Key insight**: For interval `[i, j]`, try all ways to split it into two parts, solve recursively, and combine.

**Production example**:
In trading, optimal execution of large orders:
- Split order into smaller chunks
- Each split has market impact cost
- Find optimal split sequence to minimize total cost
- This is interval DP (similar to Matrix Chain Multiplication)"

### Q2: "How does bitmask DP work and what are its limitations?"

**Model Answer:**
"Bitmask DP uses integers to represent sets:

**Representation**:
- Bit i = 1: Element i is in set
- Bit i = 0: Element i is not in set
- 2^n possible subsets for n elements

**Operations**:
```java
// Check if element i is in mask
(mask & (1 << i)) != 0

// Add element i to mask
mask | (1 << i)

// Remove element i from mask
mask & ~(1 << i)

// Iterate all subsets
for (int mask = 0; mask < (1 << n); mask++)
```

**When to use**:
- Small n (typically n ≤ 20)
- Need to track subset membership
- Permutation/combination problems

**Limitations**:
1. **Exponential space**: O(2^n)
   - n=20: 1M states ✓
   - n=30: 1B states ✗ (too large)

2. **Only works for small n**

**Classic problems**:
- Traveling Salesman Problem (TSP)
- Assignment problems
- Hamiltonian path
- Subset sum with constraints

**Production example**:
In workforce scheduling (n=15 employees):
- State: which employees are assigned
- Transition: assign next employee
- 2^15 = 32K states is manageable

For larger n, use approximation algorithms or heuristics."

### Q3: "What is digit DP and how is it different from regular DP?"

**Model Answer:**
"Digit DP counts numbers with certain properties in range [L, R]:

**Approach**: Build numbers digit by digit, tracking:
1. **Position**: Current digit position
2. **Tight**: Whether we're still bounded by limit
3. **State**: Problem-specific state (e.g., digit sum, last digit)

**Template**:
```java
int solve(int pos, boolean tight, ...states) {
    if (pos == length) return satisfiesCondition(states) ? 1 : 0;
    
    int limit = tight ? digit[pos] : 9;
    int result = 0;
    
    for (int d = 0; d <= limit; d++) {
        result += solve(pos + 1, 
                       tight && (d == limit),
                       ...updated states);
    }
    
    return result;
}
```

**vs Regular DP**:
- **Regular DP**: State is problem-specific
- **Digit DP**: State includes position and tight constraint

**When to use**:
- Count numbers in range with property
- Property can be checked digit by digit

**Examples**:
- Count numbers with digit sum = k
- Count numbers with no consecutive 1s in binary
- Count palindromic numbers

**Production example**:
In banking, count account numbers in range satisfying checksum:
- Range: [1000000, 9999999]
- Property: Luhn algorithm checksum valid
- Use digit DP to count efficiently"

---

## 🏦 Banking & Production Context

### Optimal Trade Execution

**Scenario**: Split large order to minimize market impact.

```java
/**
 * Optimal order splitting using interval DP.
 */
class TradeExecutionOptimizer {
    public double minImpact(int totalShares, int[] timeSlots, double[][] impactCost) {
        int n = timeSlots.length;
        double[][] dp = new double[n][n];
        
        // Base case: single time slot
        for (int i = 0; i < n; i++) {
            dp[i][i] = impactCost[i][totalShares];
        }
        
        // Fill by increasing interval length
        for (int len = 2; len <= n; len++) {
            for (int i = 0; i <= n - len; i++) {
                int j = i + len - 1;
                dp[i][j] = Double.MAX_VALUE;
                
                // Try all split points
                for (int k = i; k < j; k++) {
                    for (int leftShares = 0; leftShares <= totalShares; leftShares++) {
                        int rightShares = totalShares - leftShares;
                        
                        double cost = dp[i][k] + dp[k+1][j] +
                                     crossImpact(leftShares, rightShares);
                        
                        dp[i][j] = Math.min(dp[i][j], cost);
                    }
                }
            }
        }
        
        return dp[0][n-1];
    }
    
    private double crossImpact(int leftShares, int rightShares) {
        // Model interaction between consecutive trades
        return 0.01 * leftShares * rightShares / 1000000.0;
    }
}
```

---

## Key Takeaways

1. **Interval DP**: `dp[i][j]` for interval, try all split points
2. **Bitmask DP**: Use bits for subsets, n ≤ 20
3. **Digit DP**: Build numbers digit by digit, track tight constraint
4. **State Machine DP**: Model as state transitions
5. **Complexity**: Interval O(n³), Bitmask O(2^n), Digit O(n × states)
6. **Production**: Trade execution, scheduling, counting problems
7. **Limitations**: Exponential space for bitmask, small n only

---

**Next**: [Backtracking and Recursion](18-backtracking-and-recursion.md)
