# DP Patterns: Strings

## Overview
String DP problems usually involve comparing two strings or finding properties within one string.
*   **State**: `dp[i][j]` usually means result for `s1[0...i]` and `s2[0...j]`.

## Common Patterns

### 1. Longest Common Subsequence (LCS)
*   If `s1[i] == s2[j]`: `dp[i][j] = 1 + dp[i-1][j-1]`
*   Else: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`

### 2. Edit Distance
*   Insert, Delete, Replace operations.

### 3. Palindromes
*   `dp[i][j]` = is substring `s[i...j]` a palindrome?

## Interview Problems

### Problem 1: Longest Common Subsequence (Medium)
**Pattern**: LCS

```java
/**
 * Find length of LCS.
 * Time: O(m * n)
 * Space: O(m * n)
 */
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length();
    int n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = 1 + dp[i - 1][j - 1];
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[m][n];
}
```

### Problem 2: Edit Distance (Hard)
**Pattern**: String Transformation

```java
/**
 * Min operations to convert word1 to word2.
 * Time: O(m * n)
 * Space: O(m * n)
 */
public int minDistance(String word1, String word2) {
    int m = word1.length();
    int n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1], // Replace
                               Math.min(dp[i - 1][j],     // Delete
                                        dp[i][j - 1]));   // Insert
            }
        }
    }
    
    return dp[m][n];
}
```

## 🏦 Banking Context: Fuzzy Matching
*   **Scenario**: Matching names in a transaction against a Sanctions List (OFAC).
*   **Problem**: "Osama Bin Laden" vs "Usama Bin Ladin".
*   **Solution**: **Levenshtein Distance** (Edit Distance) is used to calculate similarity scores. If distance is small, flag for manual review.

---
**Next**: [DP Patterns: Advanced](17-dp-patterns-advanced.md)
