# Bit Manipulation - Complete Master Guide

## Overview
Bit manipulation involves operating on integers at the binary level using bitwise operators. It's essential for low-level optimizations, space-efficient algorithms, and certain problem categories that are impossible to solve efficiently otherwise.

**Key Insight**: Bits are the most fundamental unit of computation—mastering bit manipulation unlocks powerful optimizations.

For Senior/Staff Engineers, mastering bit manipulation means:
- Understanding all bitwise operators and their properties
- Recognizing bit manipulation patterns
- Using bits for space optimization (bitmasks)
- Discussing production applications (permissions, flags, compression)

---

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Common Tricks](#common-tricks)
3. [Common Patterns](#common-patterns)
4. [15+ Solved Problems](#solved-problems)
5. [Advanced Topics](#advanced-topics)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Fundamentals

### Bitwise Operators

**AND (`&`)**: Both bits must be 1
```
  1010
& 1100
------
  1000
```

**OR (`|`)**: At least one bit must be 1
```
  1010
| 1100
------
  1110
```

**XOR (`^`)**: Bits must be different
```
  1010
^ 1100
------
  0110
```

**NOT (`~`)**: Invert all bits
```
~ 1010
------
  0101 (in 4-bit representation)
```

**Left Shift (`<<`)**: Multiply by 2^n
```
1010 << 2 = 101000 (multiply by 4)
```

**Right Shift (`>>`)**: Divide by 2^n (arithmetic, sign-preserving)
```
1010 >> 2 = 0010 (divide by 4)
```

**Unsigned Right Shift (`>>>`)**: Zero-fill right shift
```
-1 >> 1  = -1 (sign extended)
-1 >>> 1 = 2147483647 (zero filled)
```

### Binary Representation

```java
int x = 5;           // 0000 0101
int y = 3;           // 0000 0011

x & y = 1;           // 0000 0001
x | y = 7;           // 0000 0111
x ^ y = 6;           // 0000 0110
~x = -6;             // 1111 1010 (two's complement)
x << 1 = 10;         // 0000 1010
x >> 1 = 2;          // 0000 0010
```

---

## Common Tricks

### XOR Properties

```java
x ^ x = 0           // Self-cancellation
x ^ 0 = x           // Identity
x ^ y = y ^ x       // Commutative
(x ^ y) ^ z = x ^ (y ^ z)  // Associative
```

**Use case**: Find single number in array where all others appear twice.

### Set/Clear/Toggle Bits

```java
// Set bit at position i
x | (1 << i)

// Clear bit at position i
x & ~(1 << i)

// Toggle bit at position i
x ^ (1 << i)

// Check if bit at position i is set
(x & (1 << i)) != 0
```

### Clear Lowest Set Bit

```java
x & (x - 1)
```

**Example**: `1010 & 1001 = 1000`

**Use case**: Count number of set bits.

### Get Lowest Set Bit

```java
x & (-x)
```

**Example**: `1010 & 0110 = 0010`

**Use case**: Isolate rightmost 1-bit.

### Check if Power of 2

```java
x > 0 && (x & (x - 1)) == 0
```

**Explanation**: Power of 2 has exactly one bit set.

### Swap Two Numbers

```java
a ^= b;
b ^= a;
a ^= b;
```

**Explanation**: XOR swap without temporary variable.

---

## Common Patterns

### Pattern 1: XOR for Finding Unique

```java
/**
 * Find single number (all others appear twice).
 * Time: O(n), Space: O(1)
 */
public int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num;
    }
    return result;
}
```

### Pattern 2: Bit Counting

```java
/**
 * Count number of 1 bits (Hamming weight).
 * Time: O(1), Space: O(1)
 */
public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1);  // Clear lowest set bit
        count++;
    }
    return count;
}

// Alternative: Brian Kernighan's algorithm
public int hammingWeightAlt(int n) {
    int count = 0;
    while (n != 0) {
        count += n & 1;
        n >>>= 1;
    }
    return count;
}
```

### Pattern 3: Bitmask for Subsets

```java
/**
 * Generate all subsets using bitmask.
 * Time: O(n × 2^n), Space: O(1)
 */
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    int n = nums.length;
    
    for (int mask = 0; mask < (1 << n); mask++) {
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) {
                subset.add(nums[i]);
            }
        }
        result.add(subset);
    }
    
    return result;
}
```

---

## Solved Problems

### Problem 1: Single Number (Easy)

```java
/**
 * Find single number where all others appear twice.
 * Time: O(n), Space: O(1)
 */
public int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num;
    }
    return result;
}
```

### Problem 2: Number of 1 Bits (Easy)

```java
/**
 * Count set bits.
 * Time: O(1), Space: O(1)
 */
public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1);
        count++;
    }
    return count;
}
```

### Problem 3: Counting Bits (Easy)

```java
/**
 * Count bits for 0 to n.
 * Time: O(n), Space: O(1)
 */
public int[] countBits(int n) {
    int[] result = new int[n + 1];
    for (int i = 1; i <= n; i++) {
        result[i] = result[i >> 1] + (i & 1);
    }
    return result;
}
```

### Problem 4: Reverse Bits (Easy)

```java
/**
 * Reverse bits of 32-bit integer.
 * Time: O(1), Space: O(1)
 */
public int reverseBits(int n) {
    int result = 0;
    for (int i = 0; i < 32; i++) {
        result <<= 1;
        result |= (n & 1);
        n >>= 1;
    }
    return result;
}
```

### Problem 5: Missing Number (Easy)

```java
/**
 * Find missing number in 0 to n.
 * Time: O(n), Space: O(1)
 */
public int missingNumber(int[] nums) {
    int result = nums.length;
    for (int i = 0; i < nums.length; i++) {
        result ^= i ^ nums[i];
    }
    return result;
}
```

### Problem 6: Power of Two (Easy)

```java
/**
 * Check if number is power of 2.
 * Time: O(1), Space: O(1)
 */
public boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}
```

### Problem 7: Single Number II (Medium)

```java
/**
 * Find single number where all others appear 3 times.
 * Time: O(n), Space: O(1)
 */
public int singleNumber(int[] nums) {
    int ones = 0, twos = 0;
    
    for (int num : nums) {
        twos |= ones & num;
        ones ^= num;
        int threes = ones & twos;
        ones &= ~threes;
        twos &= ~threes;
    }
    
    return ones;
}
```

### Problem 8: Sum of Two Integers (Medium)

```java
/**
 * Add two integers without + or - operators.
 * Time: O(1), Space: O(1)
 */
public int getSum(int a, int b) {
    while (b != 0) {
        int carry = a & b;
        a = a ^ b;
        b = carry << 1;
    }
    return a;
}
```

### Problem 9: Bitwise AND of Numbers Range (Medium)

```java
/**
 * Bitwise AND of all numbers in range [left, right].
 * Time: O(log n), Space: O(1)
 */
public int rangeBitwiseAnd(int left, int right) {
    int shift = 0;
    while (left < right) {
        left >>= 1;
        right >>= 1;
        shift++;
    }
    return left << shift;
}
```

### Problem 10: Maximum XOR of Two Numbers (Medium)

```java
/**
 * Find maximum XOR of two numbers in array.
 * Time: O(n), Space: O(1)
 */
public int findMaximumXOR(int[] nums) {
    int max = 0;
    int mask = 0;
    
    for (int i = 31; i >= 0; i--) {
        mask |= (1 << i);
        Set<Integer> prefixes = new HashSet<>();
        
        for (int num : nums) {
            prefixes.add(num & mask);
        }
        
        int candidate = max | (1 << i);
        for (int prefix : prefixes) {
            if (prefixes.contains(candidate ^ prefix)) {
                max = candidate;
                break;
            }
        }
    }
    
    return max;
}
```

### Problem 11: Subsets (Medium)

```java
/**
 * Generate all subsets using bit manipulation.
 * Time: O(n × 2^n), Space: O(1)
 */
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    int n = nums.length;
    
    for (int mask = 0; mask < (1 << n); mask++) {
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) {
                subset.add(nums[i]);
            }
        }
        result.add(subset);
    }
    
    return result;
}
```

---

## Advanced Topics

### Gray Code

**Definition**: Sequence where consecutive values differ by exactly one bit.

```java
/**
 * Generate n-bit Gray code.
 * Time: O(2^n), Space: O(1)
 */
public List<Integer> grayCode(int n) {
    List<Integer> result = new ArrayList<>();
    for (int i = 0; i < (1 << n); i++) {
        result.add(i ^ (i >> 1));
    }
    return result;
}
```

### Bit Manipulation in DP

**Example**: Traveling Salesman Problem (TSP)

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

---

## Interview Questions & Answers

### Q1: "Explain why XOR is useful for finding unique elements."

**Model Answer:**
"XOR has unique properties that make it perfect for finding unique elements:

**Key properties**:
1. `x ^ x = 0` (self-cancellation)
2. `x ^ 0 = x` (identity)
3. Commutative and associative

**Example - Single Number**:
```
[4, 1, 2, 1, 2]
4 ^ 1 ^ 2 ^ 1 ^ 2
= 4 ^ (1 ^ 1) ^ (2 ^ 2)
= 4 ^ 0 ^ 0
= 4
```

All duplicates cancel out, leaving only the unique number.

**Why not hash table?**
- XOR: O(1) space, O(n) time
- Hash table: O(n) space, O(n) time

**Extensions**:
- All appear twice, one unique: Simple XOR
- All appear 3 times, one unique: Use two variables (ones, twos)
- Two unique numbers: XOR all, then partition by any set bit

**Production example**:
In data integrity checks, XOR is used for parity bits and checksums—errors (duplicates) cancel out, leaving only corruption (unique changes)."

### Q2: "How do you count set bits efficiently?"

**Model Answer:**
"There are several approaches with different trade-offs:

**Approach 1: Brian Kernighan's Algorithm**
```java
int count = 0;
while (n != 0) {
    n &= (n - 1);  // Clear lowest set bit
    count++;
}
```
- **Time**: O(k) where k is number of set bits
- **Best for**: Sparse bit patterns

**Approach 2: Shift and Check**
```java
int count = 0;
while (n != 0) {
    count += n & 1;
    n >>>= 1;
}
```
- **Time**: O(32) for 32-bit integer
- **Best for**: Dense bit patterns

**Approach 3: Lookup Table**
```java
private static final int[] BIT_COUNT = new int[256];
static {
    for (int i = 0; i < 256; i++) {
        BIT_COUNT[i] = (i & 1) + BIT_COUNT[i / 2];
    }
}

int count = BIT_COUNT[n & 0xff] + 
            BIT_COUNT[(n >> 8) & 0xff] + 
            BIT_COUNT[(n >> 16) & 0xff] + 
            BIT_COUNT[(n >> 24) & 0xff];
```
- **Time**: O(1)
- **Space**: O(256)
- **Best for**: Many repeated calls

**Approach 4: Built-in**
```java
Integer.bitCount(n)
```
- **Time**: O(1) (hardware instruction)
- **Best for**: Production code

**Production choice**: Use `Integer.bitCount()` unless you have specific constraints."

### Q3: "Explain bitmask DP and when to use it."

**Model Answer:**
"Bitmask DP uses integers to represent sets, enabling efficient state representation:

**Key idea**: Use bits to represent subset membership
- Bit i set → element i is in subset
- 2^n possible subsets for n elements

**When to use**:
1. **Small n** (typically n ≤ 20): 2^20 = 1M states is manageable
2. **Subset-based states**: Need to track which elements are used/visited
3. **Permutation problems**: Track which positions are filled

**Classic problems**:
- Traveling Salesman Problem (TSP)
- Assignment problems
- Subset sum with constraints
- Hamiltonian path

**Example - TSP**:
```java
dp[mask][i] = minimum cost to visit cities in 'mask', ending at city i
mask = 0b1011 means cities 0, 1, 3 are visited
```

**Advantages**:
- Compact state representation
- Fast state transitions (bitwise operations)
- Easy to iterate all states

**Limitations**:
- Exponential space: O(2^n)
- Only works for small n

**Production example**:
In workforce scheduling with n=15 employees, use bitmask DP to assign shifts:
- State: which employees are assigned
- Transition: assign next employee to available shift
- 2^15 = 32K states is very manageable"

---

## 🏦 Banking & Production Context

### Permission System

**Scenario**: Implement role-based access control using bitmasks.

```java
/**
 * Permission system using bit flags.
 */
class PermissionSystem {
    // Permission flags
    public static final int READ = 1 << 0;      // 0001
    public static final int WRITE = 1 << 1;     // 0010
    public static final int EXECUTE = 1 << 2;   // 0100
    public static final int DELETE = 1 << 3;    // 1000
    
    private Map<String, Integer> userPermissions = new HashMap<>();
    
    public void grantPermission(String user, int permission) {
        userPermissions.merge(user, permission, (old, perm) -> old | perm);
    }
    
    public void revokePermission(String user, int permission) {
        userPermissions.computeIfPresent(user, (u, perms) -> perms & ~permission);
    }
    
    public boolean hasPermission(String user, int permission) {
        return (userPermissions.getOrDefault(user, 0) & permission) == permission;
    }
    
    public boolean hasAnyPermission(String user, int permissions) {
        return (userPermissions.getOrDefault(user, 0) & permissions) != 0;
    }
    
    public List<String> getPermissionNames(int permissions) {
        List<String> names = new ArrayList<>();
        if ((permissions & READ) != 0) names.add("READ");
        if ((permissions & WRITE) != 0) names.add("WRITE");
        if ((permissions & EXECUTE) != 0) names.add("EXECUTE");
        if ((permissions & DELETE) != 0) names.add("DELETE");
        return names;
    }
}

// Usage
PermissionSystem system = new PermissionSystem();
system.grantPermission("alice", READ | WRITE);
system.grantPermission("bob", READ | EXECUTE);

boolean canWrite = system.hasPermission("alice", WRITE);  // true
boolean canDelete = system.hasPermission("alice", DELETE);  // false
```

### Feature Flags

**Scenario**: Enable/disable features using bit flags.

```java
/**
 * Feature flag system.
 */
class FeatureFlags {
    public static final long FEATURE_A = 1L << 0;
    public static final long FEATURE_B = 1L << 1;
    public static final long FEATURE_C = 1L << 2;
    // ... up to 64 features
    
    private long enabledFeatures = 0;
    
    public void enableFeature(long feature) {
        enabledFeatures |= feature;
    }
    
    public void disableFeature(long feature) {
        enabledFeatures &= ~feature;
    }
    
    public boolean isEnabled(long feature) {
        return (enabledFeatures & feature) != 0;
    }
    
    public void toggleFeature(long feature) {
        enabledFeatures ^= feature;
    }
}
```

---

## Key Takeaways

1. **XOR properties**: Self-cancellation, identity, commutative
2. **Set bit**: `x | (1 << i)`
3. **Clear bit**: `x & ~(1 << i)`
4. **Toggle bit**: `x ^ (1 << i)`
5. **Clear lowest set bit**: `x & (x - 1)`
6. **Power of 2**: `x > 0 && (x & (x - 1)) == 0`
7. **Bitmask DP**: Use for small n (≤ 20), subset-based states
8. **Production**: Permissions, feature flags, compression

---

**Next**: [Math and Number Theory](23-math-and-number-theory.md)
