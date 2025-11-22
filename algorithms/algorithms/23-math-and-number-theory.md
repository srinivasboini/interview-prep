# Math and Number Theory - Complete Master Guide

## Overview
Mathematical problems test your ability to find elegant formulas and algorithms that avoid brute force. These problems often have O(1) or O(log n) solutions instead of O(n) or O(n²).

**Key Insight**: Look for mathematical patterns, properties, and formulas before implementing brute force.

For Senior/Staff Engineers, mastering math means:
- Understanding prime numbers and factorization
- Implementing GCD, LCM, and modular arithmetic
- Recognizing mathematical patterns
- Discussing production applications (cryptography, hashing, random number generation)

---

## Table of Contents
1. [Prime Numbers](#prime-numbers)
2. [GCD and LCM](#gcd-and-lcm)
3. [Modular Arithmetic](#modular-arithmetic)
4. [Combinatorics](#combinatorics)
5. [15+ Solved Problems](#solved-problems)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Prime Numbers

### Sieve of Eratosthenes

**Problem**: Find all primes up to n

**Algorithm**: Mark multiples of each prime as composite

**Time**: O(n log log n), **Space**: O(n)

```java
/**
 * Sieve of Eratosthenes.
 * Time: O(n log log n), Space: O(n)
 */
public List<Integer> sieveOfEratosthenes(int n) {
    boolean[] isPrime = new boolean[n + 1];
    Arrays.fill(isPrime, true);
    isPrime[0] = isPrime[1] = false;
    
    for (int i = 2; i * i <= n; i++) {
        if (isPrime[i]) {
            for (int j = i * i; j <= n; j += i) {
                isPrime[j] = false;
            }
        }
    }
    
    List<Integer> primes = new ArrayList<>();
    for (int i = 2; i <= n; i++) {
        if (isPrime[i]) {
            primes.add(i);
        }
    }
    
    return primes;
}
```

### Primality Testing

```java
/**
 * Check if number is prime.
 * Time: O(√n), Space: O(1)
 */
public boolean isPrime(int n) {
    if (n <= 1) return false;
    if (n <= 3) return true;
    if (n % 2 == 0 || n % 3 == 0) return false;
    
    for (int i = 5; i * i <= n; i += 6) {
        if (n % i == 0 || n % (i + 2) == 0) {
            return false;
        }
    }
    
    return true;
}
```

### Prime Factorization

```java
/**
 * Prime factorization.
 * Time: O(√n), Space: O(log n)
 */
public List<Integer> primeFactors(int n) {
    List<Integer> factors = new ArrayList<>();
    
    while (n % 2 == 0) {
        factors.add(2);
        n /= 2;
    }
    
    for (int i = 3; i * i <= n; i += 2) {
        while (n % i == 0) {
            factors.add(i);
            n /= i;
        }
    }
    
    if (n > 2) {
        factors.add(n);
    }
    
    return factors;
}
```

---

## GCD and LCM

### Euclidean Algorithm

```java
/**
 * Greatest Common Divisor using Euclidean algorithm.
 * Time: O(log min(a, b)), Space: O(1)
 */
public int gcd(int a, int b) {
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

// Recursive version
public int gcdRecursive(int a, int b) {
    return b == 0 ? a : gcdRecursive(b, a % b);
}
```

### Least Common Multiple

```java
/**
 * Least Common Multiple.
 * Time: O(log min(a, b)), Space: O(1)
 */
public int lcm(int a, int b) {
    return (a / gcd(a, b)) * b;  // Avoid overflow
}
```

---

## Modular Arithmetic

### Modular Exponentiation

```java
/**
 * Compute (base^exp) % mod efficiently.
 * Time: O(log exp), Space: O(1)
 */
public long modPow(long base, long exp, long mod) {
    long result = 1;
    base %= mod;
    
    while (exp > 0) {
        if (exp % 2 == 1) {
            result = (result * base) % mod;
        }
        base = (base * base) % mod;
        exp /= 2;
    }
    
    return result;
}
```

### Modular Inverse

```java
/**
 * Modular multiplicative inverse using Fermat's Little Theorem.
 * Works when mod is prime.
 * Time: O(log mod), Space: O(1)
 */
public long modInverse(long a, long mod) {
    return modPow(a, mod - 2, mod);
}
```

---

## Combinatorics

### Factorial with Mod

```java
/**
 * Factorial modulo m.
 * Time: O(n), Space: O(1)
 */
public long factorialMod(int n, long mod) {
    long result = 1;
    for (int i = 2; i <= n; i++) {
        result = (result * i) % mod;
    }
    return result;
}
```

### Combinations (nCr)

```java
/**
 * Compute nCr % mod.
 * Time: O(n), Space: O(n)
 */
public long nCr(int n, int r, long mod) {
    if (r > n) return 0;
    if (r == 0 || r == n) return 1;
    
    // Precompute factorials
    long[] fact = new long[n + 1];
    fact[0] = 1;
    for (int i = 1; i <= n; i++) {
        fact[i] = (fact[i-1] * i) % mod;
    }
    
    // nCr = n! / (r! * (n-r)!)
    long numerator = fact[n];
    long denominator = (fact[r] * fact[n-r]) % mod;
    
    return (numerator * modInverse(denominator, mod)) % mod;
}
```

---

## Solved Problems

### Problem 1: Count Primes (Medium)

```java
/**
 * Count primes less than n.
 * Time: O(n log log n), Space: O(n)
 */
public int countPrimes(int n) {
    if (n <= 2) return 0;
    
    boolean[] isPrime = new boolean[n];
    Arrays.fill(isPrime, true);
    isPrime[0] = isPrime[1] = false;
    
    for (int i = 2; i * i < n; i++) {
        if (isPrime[i]) {
            for (int j = i * i; j < n; j += i) {
                isPrime[j] = false;
            }
        }
    }
    
    int count = 0;
    for (boolean prime : isPrime) {
        if (prime) count++;
    }
    
    return count;
}
```

### Problem 2: Power of Three (Easy)

```java
/**
 * Check if n is power of 3.
 * Time: O(log n), Space: O(1)
 */
public boolean isPowerOfThree(int n) {
    if (n <= 0) return false;
    
    while (n % 3 == 0) {
        n /= 3;
    }
    
    return n == 1;
}

// O(1) solution using math
public boolean isPowerOfThreeOptimized(int n) {
    // 3^19 = 1162261467 is largest power of 3 in int range
    return n > 0 && 1162261467 % n == 0;
}
```

### Problem 3: Ugly Number II (Medium)

```java
/**
 * Find nth ugly number (factors only 2, 3, 5).
 * Time: O(n), Space: O(n)
 */
public int nthUglyNumber(int n) {
    int[] ugly = new int[n];
    ugly[0] = 1;
    
    int i2 = 0, i3 = 0, i5 = 0;
    
    for (int i = 1; i < n; i++) {
        int next2 = ugly[i2] * 2;
        int next3 = ugly[i3] * 3;
        int next5 = ugly[i5] * 5;
        
        ugly[i] = Math.min(next2, Math.min(next3, next5));
        
        if (ugly[i] == next2) i2++;
        if (ugly[i] == next3) i3++;
        if (ugly[i] == next5) i5++;
    }
    
    return ugly[n-1];
}
```

### Problem 4: Happy Number (Easy)

```java
/**
 * Check if number is happy.
 * Time: O(log n), Space: O(log n)
 */
public boolean isHappy(int n) {
    Set<Integer> seen = new HashSet<>();
    
    while (n != 1 && !seen.contains(n)) {
        seen.add(n);
        n = sumOfSquares(n);
    }
    
    return n == 1;
}

private int sumOfSquares(int n) {
    int sum = 0;
    while (n > 0) {
        int digit = n % 10;
        sum += digit * digit;
        n /= 10;
    }
    return sum;
}
```

### Problem 5: Factorial Trailing Zeroes (Medium)

```java
/**
 * Count trailing zeroes in n!
 * Time: O(log n), Space: O(1)
 */
public int trailingZeroes(int n) {
    int count = 0;
    
    while (n > 0) {
        n /= 5;
        count += n;
    }
    
    return count;
}
```

### Problem 6: Excel Sheet Column Number (Easy)

```java
/**
 * Convert Excel column to number.
 * Time: O(n), Space: O(1)
 */
public int titleToNumber(String columnTitle) {
    int result = 0;
    
    for (char c : columnTitle.toCharArray()) {
        result = result * 26 + (c - 'A' + 1);
    }
    
    return result;
}
```

### Problem 7: Pow(x, n) (Medium)

```java
/**
 * Implement pow(x, n).
 * Time: O(log n), Space: O(log n)
 */
public double myPow(double x, int n) {
    long N = n;
    if (N < 0) {
        x = 1 / x;
        N = -N;
    }
    
    return fastPow(x, N);
}

private double fastPow(double x, long n) {
    if (n == 0) return 1.0;
    
    double half = fastPow(x, n / 2);
    
    if (n % 2 == 0) {
        return half * half;
    } else {
        return half * half * x;
    }
}
```

### Problem 8: Sqrt(x) (Easy)

```java
/**
 * Compute square root (integer part).
 * Time: O(log n), Space: O(1)
 */
public int mySqrt(int x) {
    if (x < 2) return x;
    
    long left = 1, right = x / 2;
    
    while (left <= right) {
        long mid = left + (right - left) / 2;
        long square = mid * mid;
        
        if (square == x) {
            return (int) mid;
        } else if (square < x) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return (int) right;
}
```

---

## Interview Questions & Answers

### Q1: "Explain the Sieve of Eratosthenes and its complexity."

**Model Answer:**
"Sieve of Eratosthenes finds all primes up to n efficiently:

**Algorithm**:
1. Create boolean array of size n+1, all true
2. Mark 0 and 1 as not prime
3. For each prime p from 2 to √n:
   - Mark all multiples of p (starting from p²) as not prime

**Why start from p²?**
All smaller multiples already marked by smaller primes.

**Complexity**:
- **Time**: O(n log log n)
- **Space**: O(n)

**Why O(n log log n)?**
Number of operations:
```
n/2 + n/3 + n/5 + n/7 + ... (for each prime p)
= n × (1/2 + 1/3 + 1/5 + 1/7 + ...)
= n × log log n
```

**Optimization**: Segmented sieve for very large n
- Process in chunks
- Reduces space to O(√n)

**Production use**:
In cryptography (RSA), generate large primes:
- Use probabilistic tests (Miller-Rabin) for very large numbers
- Sieve for smaller primes (< 10^6)"

### Q2: "How does modular exponentiation work?"

**Model Answer:**
"Modular exponentiation computes (base^exp) % mod efficiently:

**Naive approach**: O(exp) - too slow for large exp
```java
long result = 1;
for (int i = 0; i < exp; i++) {
    result = (result * base) % mod;
}
```

**Optimized approach**: O(log exp) using binary exponentiation
```java
long result = 1;
while (exp > 0) {
    if (exp % 2 == 1) {
        result = (result * base) % mod;
    }
    base = (base * base) % mod;
    exp /= 2;
}
```

**Key insight**: 
```
x^10 = (x^5)^2
x^5 = x × (x^2)^2
x^2 = (x^1)^2
```

**Example**: Compute 3^13 % 7
```
13 in binary: 1101
3^13 = 3^8 × 3^4 × 3^1

Step 1: 3^1 % 7 = 3
Step 2: 3^2 % 7 = 2
Step 3: 3^4 % 7 = 4
Step 4: 3^8 % 7 = 2
Result: (2 × 4 × 3) % 7 = 3
```

**Production use**:
1. **Cryptography**: RSA encryption/decryption
2. **Hashing**: Polynomial rolling hash
3. **Random number generation**: Linear congruential generators"

### Q3: "Explain GCD and its applications."

**Model Answer:**
"GCD (Greatest Common Divisor) is the largest number that divides both a and b:

**Euclidean Algorithm**:
```java
int gcd(int a, int b) {
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}
```

**Why it works**:
```
gcd(a, b) = gcd(b, a % b)

Example: gcd(48, 18)
= gcd(18, 48 % 18)
= gcd(18, 12)
= gcd(12, 6)
= gcd(6, 0)
= 6
```

**Complexity**: O(log min(a, b))

**Applications**:

**1. Simplifying fractions**:
```java
int g = gcd(numerator, denominator);
numerator /= g;
denominator /= g;
```

**2. LCM calculation**:
```java
lcm(a, b) = (a × b) / gcd(a, b)
```

**3. Modular arithmetic**:
Check if modular inverse exists: gcd(a, m) = 1

**4. Cryptography**:
RSA key generation requires gcd(e, φ(n)) = 1

**Production example**:
In financial calculations:
- Simplify interest rate fractions
- Find common time periods for recurring payments
- Synchronize periodic tasks"

---

## 🏦 Banking & Production Context

### RSA Encryption

**Scenario**: Secure message encryption using RSA.

```java
/**
 * Simplified RSA encryption/decryption.
 */
class RSA {
    private long n, e, d;
    
    public void generateKeys(int p, int q) {
        n = p * q;
        long phi = (p - 1) * (long)(q - 1);
        
        // Choose e (commonly 65537)
        e = 65537;
        
        // Compute d = e^(-1) mod phi
        d = modInverse(e, phi);
    }
    
    public long encrypt(long message) {
        return modPow(message, e, n);
    }
    
    public long decrypt(long ciphertext) {
        return modPow(ciphertext, d, n);
    }
    
    private long modPow(long base, long exp, long mod) {
        long result = 1;
        base %= mod;
        
        while (exp > 0) {
            if (exp % 2 == 1) {
                result = (result * base) % mod;
            }
            base = (base * base) % mod;
            exp /= 2;
        }
        
        return result;
    }
    
    private long modInverse(long a, long m) {
        return modPow(a, m - 2, m);  // Fermat's Little Theorem
    }
}
```

### Checksum Validation

**Scenario**: Validate account numbers using Luhn algorithm.

```java
/**
 * Luhn algorithm for checksum validation.
 */
class LuhnValidator {
    public boolean validate(String number) {
        int sum = 0;
        boolean alternate = false;
        
        for (int i = number.length() - 1; i >= 0; i--) {
            int digit = number.charAt(i) - '0';
            
            if (alternate) {
                digit *= 2;
                if (digit > 9) {
                    digit -= 9;
                }
            }
            
            sum += digit;
            alternate = !alternate;
        }
        
        return sum % 10 == 0;
    }
    
    public char generateCheckDigit(String number) {
        int sum = 0;
        boolean alternate = true;
        
        for (int i = number.length() - 1; i >= 0; i--) {
            int digit = number.charAt(i) - '0';
            
            if (alternate) {
                digit *= 2;
                if (digit > 9) {
                    digit -= 9;
                }
            }
            
            sum += digit;
            alternate = !alternate;
        }
        
        int checkDigit = (10 - (sum % 10)) % 10;
        return (char) ('0' + checkDigit);
    }
}
```

---

## Key Takeaways

1. **Sieve of Eratosthenes**: O(n log log n) to find all primes up to n
2. **GCD**: Euclidean algorithm, O(log min(a, b))
3. **Modular exponentiation**: O(log exp) using binary exponentiation
4. **Prime factorization**: O(√n)
5. **Combinatorics**: Use modular arithmetic to avoid overflow
6. **Production**: Cryptography (RSA), checksums (Luhn), hashing
7. **Optimization**: Look for mathematical patterns before brute force

---

**Next**: [Sorting Algorithms](24-sorting-algorithms.md)
