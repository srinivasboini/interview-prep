# Dynamic Programming Problems Index

This directory contains standalone, in-depth breakdowns of essential dynamic programming problems designed for Senior/Staff Engineer interview preparation. 

## Required Problem Structure

To maximize understanding and interview readiness, **every problem in this directory must strictly follow this visual and structural format**:

1. **Problem Description**: Clear problem statement and constraints (like LeetCode).
2. **Recursive Solution**: The intuitive top-down approach before memoization.
3. **Recursion Tree Visualization**: A Mermaid diagram showing the overlapping subproblems.
4. **Bottom-Up DP Solution**: The optimal iterative tabulation approach.
5. **DP Matrix/Array Visualization**: A table or array diagram showing how the state is built up.
6. **Complexity Analysis**: Time and Space complexities for both approaches.

---

## Problems List & Variations

To truly master Dynamic Programming, you must recognize underlying patterns and adapt to their variations. Below is an expanded list of 20 classic problems (covering LeetCode, HackerRank, etc.), categorized by DP pattern with a focus on their unique variations. 

### 1. 1D Dynamic Programming
Problems where the state depends on a few previous states in a linear sequence.
- 1. [Climbing Stairs](01-climbing-stairs.md) — **Variation:** The fundamental recurrence relation (Fibonacci-style logic).
- 2. [House Robber](02-house-robber.md) — **Variation:** Take vs. Skip (inclusion/exclusion with adjacent elements constraint).
- 8. [Maximum Product Subarray](08-maximum-product-subarray.md) — **Variation:** Tracking two states simultaneously (current min and current max) to handle negative multipliers.
- 10. [Decode Ways](10-decode-ways.md) — **Variation:** String parsing with variable length look-backs (1 or 2 characters valid mapping).

### 2. Knapsack Patterns (0/1 and Unbounded)
Selecting a subset of items to maximize/minimize a target, bounded by a capacity.
- 6. [0/1 Knapsack](06-01-knapsack.md) — **Variation:** The classic bounded optimization (pick once or not at all).
- 3. [Coin Change](03-coin-change.md) — **Variation:** Unbounded knapsack (infinite supply of each item) targeting the minimum number of elements.
- 9. [Word Break](09-word-break.md) — **Variation:** Substring splitting modeled as an unbounded knapsack problem.
- 11. [Partition Equal Subset Sum](11-partition-equal-subset-sum.md) — **Variation:** 0/1 Knapsack where the "capacity" is explicitly derived as `Total Sum / 2`.
- 12. [Target Sum](12-target-sum.md) — **Variation:** State space manipulation; assigning `+` or `-` is mathematically equivalent to a subset sum problem.

### 3. String / Subsequence DP
Problems comparing two sequences or analyzing palindromes within a string.
- 5. [Longest Common Subsequence](05-longest-common-subsequence.md) — **Variation:** Standard 2D DP for comparing two separate sequences.
- 7. [Edit Distance](07-edit-distance.md) — **Variation:** LCS extended with three distinct transformation operations (Insert, Delete, Replace).
- 15. [Palindromic Substrings](15-palindromic-substrings.md) — **Variation:** Interval DP where the DP state expands from the center or relies strictly on shrinking boundaries `(i, j) -> (i+1, j-1)`.
- 16. [Longest Palindromic Subsequence](16-longest-palindromic-subsequence.md) — **Variation:** Finding palindromic patterns by applying LCS logic between a string and its reversed self.

### 4. 2D Dynamic Programming / Grids
Problems involving 2D matrices where pathing and moves are restricted (e.g., only moving right/down).
- 13. [Minimum Path Sum](13-minimum-path-sum.md) — **Variation:** Cost accumulation along a 2D traversal.
- 14. [Unique Paths](14-unique-paths.md) — **Variation:** Combinatorics/counting valid paths on a grid, dealing with geometry and obstacles.

### 5. Longest Increasing Subsequence (LIS)
Problems mapping out lengths of strictly increasing/decreasing elements.
- 4. [Longest Increasing Subsequence](04-longest-increasing-subsequence.md) — **Variation:** The core $O(N^2)$ DP array building (which paves the way for the advanced $O(N \log N)$ binary search method).
- 20. [Russian Doll Envelopes](20-russian-doll-envelopes.md) — **Variation:** 2D LIS. Requires multidimensional custom sorting before applying the standard 1D LIS approach.

### 6. State Machine DP
Problems where each step conceptually transitions through multiple mutually exclusive "states".
- 17. [Best Time to Buy and Sell Stock with Cooldown](17-stock-cooldown.md) — **Variation:** Requires an explicit state machine with custom transition rules (States: Held, Sold, Rest) mapping distinct flow networks.

### 7. Interval / Matrix Chain Multiplication (MCM) DP
Problems solving a complete interval by splitting it dynamically into smaller sub-intervals and optimally combining them.
- 18. [Burst Balloons](18-burst-balloons.md) — **Variation:** Interval DP working "backwards" by pinpointing the *last* balloon to burst to isolate subproblems.
- 19. [Matrix Chain Multiplication](19-matrix-chain-multiplication.md) — **Variation:** Classic $O(N^3)$ divide and conquer with DP, deciding optimal slicing points in a contiguous block.
