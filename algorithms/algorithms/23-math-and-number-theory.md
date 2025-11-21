# Math and Number Theory

## Overview
Math problems often test your ability to find a mathematical formula to solve a problem in O(1) or O(log n) instead of O(n).

## Fundamentals

### Prime Numbers
*   **Sieve of Eratosthenes**: Find all primes up to N in O(N log log N).

### GCD / LCM
*   **Euclidean Algorithm**: `gcd(a, b) = gcd(b, a % b)`.

## Interview Problems

### Problem 1: Count Primes (Medium)
**Pattern**: Sieve of Eratosthenes

```java
/**
 * Count primes less than n.
 * Time: O(n log log n)
 */
public int countPrimes(int n) {
    boolean[] notPrime = new boolean[n];
    int count = 0;
    for (int i = 2; i < n; i++) {
        if (!notPrime[i]) {
            count++;
            for (int j = 2; i * j < n; j++) {
                notPrime[i * j] = true;
            }
        }
    }
    return count;
}
```

## 🏦 Banking Context: Cryptography
*   **Scenario**: RSA Encryption.
*   **Math**: Relies heavily on **Prime Factorization** and **Modular Arithmetic** of very large numbers (BigInteger).

---
**Next**: [Sorting Algorithms](24-sorting-algorithms.md)
