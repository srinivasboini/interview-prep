# DP Patterns: 1D - Complete Master Guide

## Overview
1D Dynamic Programming problems involve a linear sequence (array or string) where the state at position `i` depends on previous positions. These are the most fundamental DP patterns and form the foundation for more complex problems.

**Key Insight**: 1D DP = State depends on previous indices in a single dimension.

For Senior/Staff Engineers, mastering 1D DP means:
- Recognizing Fibonacci-style patterns
- Understanding state transitions
- Optimizing space from O(n) to O(1)
- Discussing production applications (time series, sequential decision-making)

---

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Common Patterns](#common-patterns)
3. [15+ Solved Problems](#solved-problems)
4. [Space Optimization](#space-optimization)
5. [Advanced Topics](#advanced-topics)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Fundamentals

### What is 1D DP?

**Definition**: Problems where optimal solution at position `i` depends on solutions at previous positions.

**General form**:
```
dp[i] = f(dp[i-1], dp[i-2], ..., dp[0])
```

### Template

```java
public int solve1D(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    
    // Base cases
    dp[0] = baseCase0;
    if (n > 1) dp[1] = baseCase1;
    
    // Fill DP table
    for (int i = 2; i < n; i++) {
        dp[i] = transition(dp, i, nums);
    }
    
    return dp[n - 1];
}
```

---

## Common Patterns

### Pattern 1: Fibonacci Style

**Recurrence**: `dp[i] = dp[i-1] + dp[i-2]`

**Examples**: Climbing Stairs, Fibonacci, Tribonacci

```java
/**
 * Climbing Stairs - Fibonacci pattern.
 * Time: O(n), Space: O(1)
 */
public int climbStairs(int n) {
    if (n <= 2) return n;
    
    int prev2 = 1, prev1 = 2;
    
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Pattern 2: House Robber (Choice Pattern)

**Recurrence**: `dp[i] = max(nums[i] + dp[i-2], dp[i-1])`

**Key insight**: At each house, choose to rob or skip.

```java
/**
 * House Robber - Choice pattern.
 * Time: O(n), Space: O(1)
 */
public int rob(int[] nums) {
    if (nums.length == 0) return 0;
    
    int prev2 = 0, prev1 = 0;
    
    for (int num : nums) {
        int curr = Math.max(num + prev2, prev1);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Pattern 3: Longest Increasing Subsequence

**Recurrence**: `dp[i] = max(dp[j] + 1)` for all `j < i` where `nums[j] < nums[i]`

```java
/**
 * LIS - O(n²) solution.
 * Time: O(n²), Space: O(n)
 */
public int lengthOfLIS(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    int maxLen = 1;
    
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[i] > nums[j]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    
    return maxLen;
}

/**
 * LIS - O(n log n) with binary search.
 * Time: O(n log n), Space: O(n)
 */
public int lengthOfLISOptimized(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    
    for (int num : nums) {
        int pos = binarySearch(tails, num);
        if (pos == tails.size()) {
            tails.add(num);
        } else {
            tails.set(pos, num);
        }
    }
    
    return tails.size();
}

private int binarySearch(List<Integer> tails, int target) {
    int left = 0, right = tails.size();
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (tails.get(mid) < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    
    return left;
}
```

---

## Solved Problems

### Problem 1: Climbing Stairs (Easy)

```java
/**
 * Number of ways to climb n stairs (1 or 2 steps at a time).
 * Time: O(n), Space: O(1)
 */
public int climbStairs(int n) {
    if (n <= 2) return n;
    
    int prev2 = 1, prev1 = 2;
    
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Problem 2: Min Cost Climbing Stairs (Easy)

```java
/**
 * Minimum cost to reach top of stairs.
 * Time: O(n), Space: O(1)
 */
public int minCostClimbingStairs(int[] cost) {
    int n = cost.length;
    int prev2 = 0, prev1 = 0;
    
    for (int i = 2; i <= n; i++) {
        int curr = Math.min(prev1 + cost[i-1], prev2 + cost[i-2]);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Problem 3: House Robber (Medium)

```java
/**
 * Maximum money without robbing adjacent houses.
 * Time: O(n), Space: O(1)
 */
public int rob(int[] nums) {
    if (nums.length == 0) return 0;
    
    int prev2 = 0, prev1 = 0;
    
    for (int num : nums) {
        int curr = Math.max(num + prev2, prev1);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Problem 4: House Robber II (Medium)

```java
/**
 * Houses arranged in circle (first and last are adjacent).
 * Time: O(n), Space: O(1)
 */
public int robCircular(int[] nums) {
    if (nums.length == 1) return nums[0];
    
    return Math.max(
        robRange(nums, 0, nums.length - 2),
        robRange(nums, 1, nums.length - 1)
    );
}

private int robRange(int[] nums, int start, int end) {
    int prev2 = 0, prev1 = 0;
    
    for (int i = start; i <= end; i++) {
        int curr = Math.max(nums[i] + prev2, prev1);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Problem 5: Delete and Earn (Medium)

```java
/**
 * Delete number and earn points, but lose adjacent numbers.
 * Time: O(n + m), Space: O(m) where m is max value
 */
public int deleteAndEarn(int[] nums) {
    int max = 0;
    for (int num : nums) {
        max = Math.max(max, num);
    }
    
    int[] points = new int[max + 1];
    for (int num : nums) {
        points[num] += num;
    }
    
    // House Robber pattern
    int prev2 = 0, prev1 = 0;
    
    for (int point : points) {
        int curr = Math.max(point + prev2, prev1);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Problem 6: Decode Ways (Medium)

```java
/**
 * Number of ways to decode a string.
 * Time: O(n), Space: O(1)
 */
public int numDecodings(String s) {
    if (s.charAt(0) == '0') return 0;
    
    int n = s.length();
    int prev2 = 1, prev1 = 1;
    
    for (int i = 1; i < n; i++) {
        int curr = 0;
        
        // Single digit
        if (s.charAt(i) != '0') {
            curr += prev1;
        }
        
        // Two digits
        int twoDigit = Integer.parseInt(s.substring(i-1, i+1));
        if (twoDigit >= 10 && twoDigit <= 26) {
            curr += prev2;
        }
        
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### Problem 7: Maximum Product Subarray (Medium)

```java
/**
 * Maximum product of contiguous subarray.
 * Time: O(n), Space: O(1)
 */
public int maxProduct(int[] nums) {
    int maxProd = nums[0];
    int currMax = nums[0];
    int currMin = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        if (nums[i] < 0) {
            int temp = currMax;
            currMax = currMin;
            currMin = temp;
        }
        
        currMax = Math.max(nums[i], currMax * nums[i]);
        currMin = Math.min(nums[i], currMin * nums[i]);
        
        maxProd = Math.max(maxProd, currMax);
    }
    
    return maxProd;
}
```

### Problem 8: Jump Game (Medium)

```java
/**
 * Can reach last index?
 * Time: O(n), Space: O(1)
 */
public boolean canJump(int[] nums) {
    int maxReach = 0;
    
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false;
        maxReach = Math.max(maxReach, i + nums[i]);
    }
    
    return true;
}
```

### Problem 9: Jump Game II (Medium)

```java
/**
 * Minimum jumps to reach last index.
 * Time: O(n), Space: O(1)
 */
public int jump(int[] nums) {
    int jumps = 0;
    int currEnd = 0;
    int maxReach = 0;
    
    for (int i = 0; i < nums.length - 1; i++) {
        maxReach = Math.max(maxReach, i + nums[i]);
        
        if (i == currEnd) {
            jumps++;
            currEnd = maxReach;
        }
    }
    
    return jumps;
}
```

### Problem 10: Longest Palindromic Substring (Medium)

```java
/**
 * Find longest palindromic substring.
 * Time: O(n²), Space: O(1)
 */
public String longestPalindrome(String s) {
    if (s.length() < 2) return s;
    
    int start = 0, maxLen = 0;
    
    for (int i = 0; i < s.length(); i++) {
        int len1 = expandAroundCenter(s, i, i);
        int len2 = expandAroundCenter(s, i, i + 1);
        int len = Math.max(len1, len2);
        
        if (len > maxLen) {
            maxLen = len;
            start = i - (len - 1) / 2;
        }
    }
    
    return s.substring(start, start + maxLen);
}

private int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1;
}
```

---

## Space Optimization

### From O(n) to O(1)

**Pattern**: If `dp[i]` only depends on previous few values, use variables instead of array.

**Example - Fibonacci**:
```java
// O(n) space
int[] dp = new int[n];
dp[i] = dp[i-1] + dp[i-2];

// O(1) space
int prev2 = 1, prev1 = 1;
int curr = prev1 + prev2;
prev2 = prev1;
prev1 = curr;
```

---

## Interview Questions & Answers

### Q1: "How do you identify a 1D DP problem?"

**Model Answer:**
"I look for these indicators:

**1. Optimal substructure**: Solution at position `i` depends on previous positions.

**2. Overlapping subproblems**: Same subproblems solved multiple times.

**3. Sequential decision-making**: Choices at each step affect future choices.

**4. Common keywords**:
- 'Maximum/minimum'
- 'Number of ways'
- 'Longest/shortest'
- 'Can you reach'

**Example - House Robber**:
- Optimal substructure: Max money at house `i` = max(rob `i` + max at `i-2`, max at `i-1`)
- Overlapping: Max at `i-1` used by both `i` and `i+1`
- Sequential: Robbing house `i` affects whether you can rob `i+1`

**vs Greedy**: Greedy makes locally optimal choice without considering future. DP considers all possibilities.

**Production example**:
In trading, deciding whether to buy/sell stock each day based on previous days' decisions—classic 1D DP (Best Time to Buy/Sell Stock series)."

### Q2: "Explain space optimization in DP."

**Model Answer:**
"Space optimization reduces memory from O(n) to O(1) when only recent states are needed:

**General principle**:
If `dp[i]` only depends on `dp[i-1]`, `dp[i-2]`, ..., `dp[i-k]`, use `k` variables instead of array.

**Example - Fibonacci**:
```java
// O(n) space
int[] dp = new int[n];
for (int i = 2; i < n; i++) {
    dp[i] = dp[i-1] + dp[i-2];
}

// O(1) space
int prev2 = 0, prev1 = 1;
for (int i = 2; i < n; i++) {
    int curr = prev1 + prev2;
    prev2 = prev1;
    prev1 = curr;
}
```

**When possible**:
- Only recent values needed
- No need to reconstruct solution path
- Forward iteration only

**When NOT possible**:
- Need entire DP table (e.g., LCS reconstruction)
- Random access to DP values
- 2D DP with dependencies

**Production impact**:
For n=1,000,000:
- O(n) space: 4MB (int array)
- O(1) space: 12 bytes (3 ints)

In high-frequency trading with millions of data points, this difference is critical for cache performance."

### Q3: "Compare O(n²) vs O(n log n) LIS solutions."

**Model Answer:**
"LIS has two approaches with different trade-offs:

**O(n²) DP solution**:
```java
for (int i = 0; i < n; i++) {
    for (int j = 0; j < i; j++) {
        if (nums[i] > nums[j]) {
            dp[i] = max(dp[i], dp[j] + 1);
        }
    }
}
```
- **Pros**: Simple, easy to understand, easy to reconstruct sequence
- **Cons**: Slow for large n

**O(n log n) Binary Search solution**:
- Maintain array of smallest tail elements for each length
- Binary search to find position for each element
- **Pros**: Much faster for large n
- **Cons**: Harder to understand, harder to reconstruct sequence

**When to use each**:
- n ≤ 1,000: O(n²) is fine, simpler code
- n > 10,000: O(n log n) necessary
- Need sequence reconstruction: O(n²) easier

**Performance comparison**:
For n=100,000:
- O(n²): 10 billion operations (~10 seconds)
- O(n log n): 1.7 million operations (~0.02 seconds)

**Production example**:
In financial analysis, finding longest increasing trend in stock prices:
- Historical analysis (small n): Use O(n²)
- Real-time monitoring (large n): Use O(n log n)"

---

## 🏦 Banking & Production Context

### Portfolio Optimization

**Scenario**: Select investments over time with constraints.

```java
/**
 * Maximum profit with cooldown period.
 */
class StockTrading {
    public int maxProfit(int[] prices) {
        if (prices.length <= 1) return 0;
        
        int sold = 0;
        int held = -prices[0];
        int cooldown = 0;
        
        for (int i = 1; i < prices.length; i++) {
            int prevSold = sold;
            int prevHeld = held;
            int prevCooldown = cooldown;
            
            sold = prevHeld + prices[i];
            held = Math.max(prevHeld, prevCooldown - prices[i]);
            cooldown = Math.max(prevCooldown, prevSold);
        }
        
        return Math.max(sold, cooldown);
    }
}
```

### Risk-Adjusted Returns

**Scenario**: Maximize returns while limiting consecutive high-risk investments.

```java
/**
 * Maximum return with risk constraints.
 */
class RiskAdjustedInvestment {
    public double maxReturn(double[] returns, int[] riskLevels, int maxConsecutiveHighRisk) {
        int n = returns.length;
        double[] dp = new double[n];
        int[] consecutiveHighRisk = new int[n];
        
        dp[0] = returns[0];
        consecutiveHighRisk[0] = riskLevels[0] > 5 ? 1 : 0;
        
        for (int i = 1; i < n; i++) {
            dp[i] = returns[i];
            
            for (int j = i - 1; j >= 0; j--) {
                int consecutive = riskLevels[i] > 5 ? consecutiveHighRisk[j] + 1 : 0;
                
                if (consecutive <= maxConsecutiveHighRisk) {
                    dp[i] = Math.max(dp[i], dp[j] + returns[i]);
                    consecutiveHighRisk[i] = consecutive;
                }
            }
        }
        
        double maxReturn = 0;
        for (double ret : dp) {
            maxReturn = Math.max(maxReturn, ret);
        }
        
        return maxReturn;
    }
}
```

---

## Key Takeaways

1. **1D DP**: State depends on previous indices in single dimension
2. **Fibonacci pattern**: `dp[i] = dp[i-1] + dp[i-2]`
3. **Choice pattern**: `dp[i] = max(take, skip)`
4. **LIS**: O(n²) simple, O(n log n) with binary search
5. **Space optimization**: O(n) → O(1) when only recent values needed
6. **Identification**: Optimal substructure + overlapping subproblems
7. **Production**: Sequential decision-making, time series analysis

---

**Next**: [DP Patterns: Knapsack](15-dp-patterns-knapsack.md)
