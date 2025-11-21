# Bit Manipulation

## Overview
Bit manipulation involves operating on integers at the binary level. It's fast and essential for low-level optimizations.

## Fundamentals

### Operators
*   `&` (AND): Both 1 -> 1
*   `|` (OR): Any 1 -> 1
*   `^` (XOR): Different -> 1 (Same -> 0)
*   `~` (NOT): Invert bits
*   `<<` (Left Shift): Multiply by 2
*   `>>` (Right Shift): Divide by 2 (Sign preserved)
*   `>>>` (Unsigned Right Shift): Zero fill

### Tricks
*   `x ^ x = 0`
*   `x ^ 0 = x`
*   `x & (x - 1)`: Clears the lowest set bit (Used to count bits).

## Interview Problems

### Problem 1: Single Number (Easy)
**Pattern**: XOR

```java
/**
 * Find the single number where every other appears twice.
 * Time: O(n)
 * Space: O(1)
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
**Pattern**: Bit Counting

```java
/**
 * Count set bits (Hamming Weight).
 * Time: O(1) (Max 32 iterations)
 */
public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
        n = n & (n - 1); // Clear lowest set bit
        count++;
    }
    return count;
}
```

## 🏦 Banking Context: Permissions & Flags
*   **Scenario**: User roles and permissions.
*   **Implementation**: Bitmasks.
    *   `READ = 1 (001)`
    *   `WRITE = 2 (010)`
    *   `EXECUTE = 4 (100)`
    *   `UserPerms = READ | WRITE (011)`
    *   Check: `(UserPerms & WRITE) != 0`

---
**Next**: [Math and Number Theory](23-math-and-number-theory.md)
