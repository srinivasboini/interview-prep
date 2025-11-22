# DP Patterns: Strings - Complete Master Guide

## Overview
String Dynamic Programming problems involve comparing strings, finding patterns, or transforming one string to another. These are among the most common DP problems in interviews and have critical applications in text processing, bioinformatics, and data matching.

**Key Insight**: String DP typically uses 2D state `dp[i][j]` representing result for prefixes `s1[0...i]` and `s2[0...j]`.

For Senior/Staff Engineers, mastering string DP means:
- Understanding LCS, LIS, and edit distance patterns
- Recognizing palindrome patterns
- Space optimization techniques
- Discussing production applications (fuzzy matching, diff algorithms, DNA sequencing)

---

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Common Patterns](#common-patterns)
3. [15+ Solved Problems](#solved-problems)
4. [Advanced Topics](#advanced-topics)
5. [Interview Questions & Answers](#interview-questions--answers)
6. [Banking & Production Context](#banking--production-context)

---

## Fundamentals

### State Definition

**2D DP**: `dp[i][j]` = result for `s1[0...i-1]` and `s2[0...j-1]`

**Common patterns**:
- LCS: Length of longest common subsequence
- Edit Distance: Minimum operations to transform
- Palindrome: Is substring `s[i...j]` a palindrome?

### Template

```java
public int stringDP(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    // Base cases
    for (int i = 0; i <= m; i++) dp[i][0] = baseCase(i);
    for (int j = 0; j <= n; j++) dp[0][j] = baseCase(j);
    
    // Fill DP table
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i-1) == s2.charAt(j-1)) {
                dp[i][j] = matchCase(dp, i, j);
            } else {
                dp[i][j] = mismatchCase(dp, i, j);
            }
        }
    }
    
    return dp[m][n];
}
```

---

## Common Patterns

### Pattern 1: Longest Common Subsequence (LCS)

**Recurrence**:
```
if s1[i] == s2[j]:
    dp[i][j] = 1 + dp[i-1][j-1]
else:
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

```java
/**
 * Longest Common Subsequence.
 * Time: O(m × n), Space: O(m × n)
 */
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i-1) == text2.charAt(j-1)) {
                dp[i][j] = 1 + dp[i-1][j-1];
            } else {
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
            }
        }
    }
    
    return dp[m][n];
}
```

### Pattern 2: Edit Distance (Levenshtein)

**Recurrence**:
```
if s1[i] == s2[j]:
    dp[i][j] = dp[i-1][j-1]
else:
    dp[i][j] = 1 + min(
        dp[i-1][j],      // Delete
        dp[i][j-1],      // Insert
        dp[i-1][j-1]     // Replace
    )
```

```java
/**
 * Edit Distance (Levenshtein Distance).
 * Time: O(m × n), Space: O(m × n)
 */
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    // Base cases
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i-1) == word2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = 1 + Math.min(
                    Math.min(dp[i-1][j], dp[i][j-1]),
                    dp[i-1][j-1]
                );
            }
        }
    }
    
    return dp[m][n];
}
```

### Pattern 3: Palindrome DP

**Recurrence**:
```
dp[i][j] = (s[i] == s[j]) && dp[i+1][j-1]
```

```java
/**
 * Longest Palindromic Substring.
 * Time: O(n²), Space: O(n²)
 */
public String longestPalindrome(String s) {
    int n = s.length();
    boolean[][] dp = new boolean[n][n];
    int start = 0, maxLen = 1;
    
    // Base case: single characters
    for (int i = 0; i < n; i++) {
        dp[i][i] = true;
    }
    
    // Base case: two characters
    for (int i = 0; i < n - 1; i++) {
        if (s.charAt(i) == s.charAt(i+1)) {
            dp[i][i+1] = true;
            start = i;
            maxLen = 2;
        }
    }
    
    // Length 3+
    for (int len = 3; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            
            if (s.charAt(i) == s.charAt(j) && dp[i+1][j-1]) {
                dp[i][j] = true;
                start = i;
                maxLen = len;
            }
        }
    }
    
    return s.substring(start, start + maxLen);
}
```

---

## Solved Problems

### Problem 1: Longest Common Subsequence (Medium)

```java
/**
 * LCS - classic problem.
 * Time: O(m × n), Space: O(m × n)
 */
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i-1) == text2.charAt(j-1)) {
                dp[i][j] = 1 + dp[i-1][j-1];
            } else {
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
            }
        }
    }
    
    return dp[m][n];
}
```

### Problem 2: Edit Distance (Hard)

```java
/**
 * Minimum operations to convert word1 to word2.
 * Time: O(m × n), Space: O(m × n)
 */
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i-1) == word2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = 1 + Math.min(
                    Math.min(dp[i-1][j], dp[i][j-1]),
                    dp[i-1][j-1]
                );
            }
        }
    }
    
    return dp[m][n];
}
```

### Problem 3: Longest Palindromic Subsequence (Medium)

```java
/**
 * Length of longest palindromic subsequence.
 * Time: O(n²), Space: O(n²)
 */
public int longestPalindromeSubseq(String s) {
    int n = s.length();
    int[][] dp = new int[n][n];
    
    for (int i = n - 1; i >= 0; i--) {
        dp[i][i] = 1;
        for (int j = i + 1; j < n; j++) {
            if (s.charAt(i) == s.charAt(j)) {
                dp[i][j] = 2 + dp[i+1][j-1];
            } else {
                dp[i][j] = Math.max(dp[i+1][j], dp[i][j-1]);
            }
        }
    }
    
    return dp[0][n-1];
}
```

### Problem 4: Distinct Subsequences (Hard)

```java
/**
 * Number of distinct subsequences of s that equal t.
 * Time: O(m × n), Space: O(m × n)
 */
public int numDistinct(String s, String t) {
    int m = s.length(), n = t.length();
    long[][] dp = new long[m + 1][n + 1];
    
    for (int i = 0; i <= m; i++) {
        dp[i][0] = 1;
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            dp[i][j] = dp[i-1][j];
            if (s.charAt(i-1) == t.charAt(j-1)) {
                dp[i][j] += dp[i-1][j-1];
            }
        }
    }
    
    return (int) dp[m][n];
}
```

### Problem 5: Interleaving String (Medium)

```java
/**
 * Is s3 formed by interleaving s1 and s2?
 * Time: O(m × n), Space: O(m × n)
 */
public boolean isInterleave(String s1, String s2, String s3) {
    int m = s1.length(), n = s2.length();
    if (m + n != s3.length()) return false;
    
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;
    
    for (int i = 1; i <= m; i++) {
        dp[i][0] = dp[i-1][0] && s1.charAt(i-1) == s3.charAt(i-1);
    }
    
    for (int j = 1; j <= n; j++) {
        dp[0][j] = dp[0][j-1] && s2.charAt(j-1) == s3.charAt(j-1);
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            dp[i][j] = (dp[i-1][j] && s1.charAt(i-1) == s3.charAt(i+j-1)) ||
                       (dp[i][j-1] && s2.charAt(j-1) == s3.charAt(i+j-1));
        }
    }
    
    return dp[m][n];
}
```

### Problem 6: Regular Expression Matching (Hard)

```java
/**
 * Implement regex with '.' and '*'.
 * Time: O(m × n), Space: O(m × n)
 */
public boolean isMatch(String s, String p) {
    int m = s.length(), n = p.length();
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;
    
    // Handle patterns like a*, a*b*, etc.
    for (int j = 2; j <= n; j++) {
        if (p.charAt(j-1) == '*') {
            dp[0][j] = dp[0][j-2];
        }
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            char sc = s.charAt(i-1);
            char pc = p.charAt(j-1);
            
            if (pc == '*') {
                dp[i][j] = dp[i][j-2];  // Zero occurrences
                
                char prevPC = p.charAt(j-2);
                if (prevPC == '.' || prevPC == sc) {
                    dp[i][j] |= dp[i-1][j];  // One or more occurrences
                }
            } else if (pc == '.' || pc == sc) {
                dp[i][j] = dp[i-1][j-1];
            }
        }
    }
    
    return dp[m][n];
}
```

### Problem 7: Wildcard Matching (Hard)

```java
/**
 * Implement wildcard with '?' and '*'.
 * Time: O(m × n), Space: O(m × n)
 */
public boolean isMatch(String s, String p) {
    int m = s.length(), n = p.length();
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;
    
    for (int j = 1; j <= n; j++) {
        if (p.charAt(j-1) == '*') {
            dp[0][j] = dp[0][j-1];
        }
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            char sc = s.charAt(i-1);
            char pc = p.charAt(j-1);
            
            if (pc == '*') {
                dp[i][j] = dp[i-1][j] || dp[i][j-1];
            } else if (pc == '?' || pc == sc) {
                dp[i][j] = dp[i-1][j-1];
            }
        }
    }
    
    return dp[m][n];
}
```

### Problem 8: Minimum ASCII Delete Sum (Medium)

```java
/**
 * Minimum ASCII sum to make two strings equal.
 * Time: O(m × n), Space: O(m × n)
 */
public int minimumDeleteSum(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        dp[i][0] = dp[i-1][0] + s1.charAt(i-1);
    }
    
    for (int j = 1; j <= n; j++) {
        dp[0][j] = dp[0][j-1] + s2.charAt(j-1);
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i-1) == s2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = Math.min(
                    dp[i-1][j] + s1.charAt(i-1),
                    dp[i][j-1] + s2.charAt(j-1)
                );
            }
        }
    }
    
    return dp[m][n];
}
```

---

## Advanced Topics

### Space Optimization

**LCS with O(n) space**:
```java
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[] prev = new int[n + 1];
    int[] curr = new int[n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i-1) == text2.charAt(j-1)) {
                curr[j] = 1 + prev[j-1];
            } else {
                curr[j] = Math.max(prev[j], curr[j-1]);
            }
        }
        int[] temp = prev;
        prev = curr;
        curr = temp;
    }
    
    return prev[n];
}
```

---

## Interview Questions & Answers

### Q1: "Explain the difference between LCS and longest common substring."

**Model Answer:**
"LCS and longest common substring are different:

**Longest Common Subsequence (LCS)**:
- Characters don't need to be consecutive
- Can skip characters
- Example: `"abcde"` and `"ace"` → LCS = `"ace"` (length 3)

**Longest Common Substring**:
- Characters must be consecutive
- Cannot skip characters
- Example: `"abcde"` and `"ace"` → Substring = `"a"` or `"ce"` (length 2)

**DP difference**:

**LCS**:
```java
if (s1[i] == s2[j]):
    dp[i][j] = 1 + dp[i-1][j-1]
else:
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])  // Can skip
```

**Substring**:
```java
if (s1[i] == s2[j]):
    dp[i][j] = 1 + dp[i-1][j-1]
else:
    dp[i][j] = 0  // Reset if mismatch
```

**Production example**:
In version control (git diff):
- LCS finds matching lines (can be non-consecutive)
- Substring finds consecutive matching blocks"

### Q2: "How does edit distance work and what are its applications?"

**Model Answer:**
"Edit distance (Levenshtein distance) measures minimum operations to transform one string to another:

**Operations**:
1. **Insert**: Add a character
2. **Delete**: Remove a character
3. **Replace**: Change a character

**DP recurrence**:
```java
if (s1[i] == s2[j]):
    dp[i][j] = dp[i-1][j-1]  // No operation needed
else:
    dp[i][j] = 1 + min(
        dp[i-1][j],      // Delete from s1
        dp[i][j-1],      // Insert into s1
        dp[i-1][j-1]     // Replace in s1
    )
```

**Applications**:

**1. Spell checking**:
- Find closest dictionary word
- Example: "teh" → "the" (distance 1)

**2. DNA sequencing**:
- Measure genetic similarity
- Align DNA sequences

**3. Fuzzy matching** (banking):
- Match names against sanctions lists
- "Osama Bin Laden" vs "Usama Bin Ladin" (distance 2)
- If distance < threshold, flag for review

**4. Plagiarism detection**:
- Measure text similarity

**5. Auto-correct**:
- Suggest corrections based on edit distance

**Production optimization**:
For large-scale matching (millions of names), use:
- BK-trees for fast nearest neighbor search
- Locality-sensitive hashing
- Phonetic algorithms (Soundex, Metaphone) as pre-filter"

### Q3: "How do you optimize string DP from O(m×n) space to O(n)?"

**Model Answer:**
"Space optimization exploits the fact that `dp[i][j]` only depends on current and previous rows:

**Original O(m×n)**:
```java
int[][] dp = new int[m+1][n+1];
for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        dp[i][j] = f(dp[i-1][j-1], dp[i-1][j], dp[i][j-1]);
    }
}
```

**Optimized O(n)** (two arrays):
```java
int[] prev = new int[n+1];
int[] curr = new int[n+1];

for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        curr[j] = f(prev[j-1], prev[j], curr[j-1]);
    }
    swap(prev, curr);
}
```

**Further optimized O(n)** (one array):
```java
int[] dp = new int[n+1];

for (int i = 1; i <= m; i++) {
    int prev = 0;  // dp[i-1][j-1]
    for (int j = 1; j <= n; j++) {
        int temp = dp[j];
        dp[j] = f(prev, dp[j], dp[j-1]);
        prev = temp;
    }
}
```

**Key insight**: Process left-to-right, saving `dp[i-1][j-1]` before overwriting.

**Trade-off**:
- **Pro**: 1000x less memory for large strings
- **Con**: Cannot reconstruct solution path easily

**When to use**:
- Only need final result (not path)
- Memory constrained
- Large strings (m, n > 10,000)

**Production example**:
In banking, comparing large transaction logs:
- O(m×n): 10,000 × 10,000 × 4 bytes = 400MB
- O(n): 10,000 × 4 bytes = 40KB
Critical for embedded systems or high-throughput processing."

---

## 🏦 Banking & Production Context

### Fuzzy Name Matching

**Scenario**: Match transaction names against sanctions lists (OFAC).

```java
/**
 * Fuzzy name matching using edit distance.
 */
class SanctionsScreening {
    private static final int THRESHOLD = 3;
    
    public List<Match> screenName(String name, List<String> sanctionsList) {
        List<Match> matches = new ArrayList<>();
        
        for (String sanctioned : sanctionsList) {
            int distance = editDistance(name.toLowerCase(), 
                                       sanctioned.toLowerCase());
            
            if (distance <= THRESHOLD) {
                double similarity = 1.0 - (double) distance / 
                                   Math.max(name.length(), sanctioned.length());
                matches.add(new Match(sanctioned, distance, similarity));
            }
        }
        
        matches.sort((a, b) -> Integer.compare(a.distance, b.distance));
        return matches;
    }
    
    private int editDistance(String s1, String s2) {
        int m = s1.length(), n = s2.length();
        int[] prev = new int[n + 1];
        int[] curr = new int[n + 1];
        
        for (int j = 0; j <= n; j++) prev[j] = j;
        
        for (int i = 1; i <= m; i++) {
            curr[0] = i;
            for (int j = 1; j <= n; j++) {
                if (s1.charAt(i-1) == s2.charAt(j-1)) {
                    curr[j] = prev[j-1];
                } else {
                    curr[j] = 1 + Math.min(
                        Math.min(prev[j], curr[j-1]),
                        prev[j-1]
                    );
                }
            }
            int[] temp = prev;
            prev = curr;
            curr = temp;
        }
        
        return prev[n];
    }
}

class Match {
    String name;
    int distance;
    double similarity;
    
    Match(String name, int distance, double similarity) {
        this.name = name;
        this.distance = distance;
        this.similarity = similarity;
    }
}
```

---

## Key Takeaways

1. **LCS**: Longest common subsequence, `dp[i][j] = 1 + dp[i-1][j-1]` if match
2. **Edit Distance**: Minimum operations, three choices (insert/delete/replace)
3. **Palindrome**: Check `s[i] == s[j]` and `dp[i+1][j-1]`
4. **Space optimization**: O(m×n) → O(n) using rolling arrays
5. **Applications**: Fuzzy matching, spell check, DNA sequencing, diff algorithms
6. **Production**: Sanctions screening, plagiarism detection, auto-correct

---

**Next**: [DP Patterns: Advanced](17-dp-patterns-advanced.md)
