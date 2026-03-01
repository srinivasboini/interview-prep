# 01. Climbing Stairs

## Problem Description

You are climbing a staircase. It takes `n` steps to reach the top.
Each time you can either climb `1` or `2` steps. In how many distinct ways can you climb to the top?

**Example 1:**
- **Input:** `n = 2`
- **Output:** `2`
- **Explanation:** There are two ways to climb to the top.
  1. 1 step + 1 step
  2. 2 steps

**Example 2:**
- **Input:** `n = 3`
- **Output:** `3`
- **Explanation:** There are three ways to climb to the top.
  1. 1 step + 1 step + 1 step
  2. 1 step + 2 steps
  3. 2 steps + 1 step

**Constraints:**
- `1 <= n <= 45`

---

## 1. Recursive Solution (Intuitive Approach)

The problem asks for the number of ways to reach step `n`. 
To reach step `n`, you could have only come from two places:
1. One step below (step `n-1`)
2. Two steps below (step `n-2`)

Therefore, the total ways to reach step `n` is the sum of the ways to reach step `n-1` and the ways to reach step `n-2`. This forms a Fibonacci-like recurrence relation.

### Java Implementation (Naive Recursion)

```java
class Solution {
    public int climbStairs(int n) {
        // Base cases: 
        // 1 way to reach step 1 (1)
        // 2 ways to reach step 2 (1+1, 2)
        if (n == 1) return 1;
        if (n == 2) return 2;
        
        // Recursive step
        return climbStairs(n - 1) + climbStairs(n - 2);
    }
}
```

---

## 2. Recursion Tree Visualization

Let's visualize the recursive calls for `n = 5`. Notice the **overlapping subproblems**.

```mermaid
graph TD
    CS5["climbStairs(5)"] --> CS4["climbStairs(4)"]
    CS5 --> CS3_1["climbStairs(3)"]
    
    CS4 --> CS3_2["climbStairs(3)"]
    CS4 --> CS2_1["climbStairs(2)<br>= 2"]
    
    CS3_1 --> CS2_2["climbStairs(2)<br>= 2"]
    CS3_1 --> CS1_1["climbStairs(1)<br>= 1"]
    
    CS3_2 --> CS2_3["climbStairs(2)<br>= 2"]
    CS3_2 --> CS1_2["climbStairs(1)<br>= 1"]
    
    style CS3_1 fill:#ffcccb,stroke:#333,stroke-width:2px
    style CS3_2 fill:#ffcccb,stroke:#333,stroke-width:2px
    style CS2_1 fill:#add8e6,stroke:#333,stroke-width:2px
    style CS2_2 fill:#add8e6,stroke:#333,stroke-width:2px
    style CS2_3 fill:#add8e6,stroke:#333,stroke-width:2px
```

*Notice how `climbStairs(3)` is calculated twice (red nodes) and `climbStairs(2)` is calculated three times (blue nodes). As `n` grows, this redundancy causes exponential time complexity.*

---

## 3. Bottom-Up DP Solution (Tabulation)

To eliminate redundant calculations, we can build the solution from the base cases up to `n` using an array. Wait, do we even need an array? We only ever need the last two values (step `n-1` and step `n-2`) to compute the current step `n`. This allows us to optimize space from $O(n)$ to $O(1)$.

### Java Implementation (Iterative DP with Space Optimization)

```java
class Solution {
    public int climbStairs(int n) {
        // Base cases
        if (n == 1) return 1;
        if (n == 2) return 2;
        
        // DP variables initialized for n=1 and n=2
        int prev2 = 1; // ways to reach step 1
        int prev1 = 2; // ways to reach step 2
        
        // Iterate from step 3 up to n
        for (int i = 3; i <= n; i++) {
            int current = prev1 + prev2;
            
            // Shift perspective forward
            prev2 = prev1;
            prev1 = current;
        }
        
        return prev1; 
    }
}
```

---

## 4. DP Array Visualization

Let's visualize how the variables update iteratively for `n = 5`.

| Step (`i`) | Computation (`current = prev1 + prev2`) | Result for Step `i` | `prev1` (updated) | `prev2` (updated) |
| :--- | :--- | :--- | :--- | :--- |
| **Start** | `N/A` (Base cases initialized) | `N/A` | `2` (step 2) | `1` (step 1) |
| **3** | `2 + 1` | `3` | `3` (step 3) | `2` (step 2) |
| **4** | `3 + 2` | `5` | `5` (step 4) | `3` (step 3) |
| **5** | `5 + 3` | `8` | `8` (step 5) | `5` (step 4) |

**Final Answer:** 8 total ways to reach step 5.

---

## 5. Complexity Analysis

### Naive Recursive Solution
- **Time Complexity:** $O(2^n)$. Each node branches into two recursive calls, creating a tree of depth $n$.
- **Space Complexity:** $O(n)$. The maximum depth of the recurring call stack tree is $n$.

### Bottom-Up DP Solution 
- **Time Complexity:** $O(n)$. We iterate through a loop exactly $n - 2$ times.
- **Space Complexity:** $O(1)$. We only maintain a few integer variables (`prev1`, `prev2`, `current`), regardless of the size of $n$. *(If we used a standard DP array, space would be $O(n)$).*
