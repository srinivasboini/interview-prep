# Pattern: Fast & Slow Pointers

## Overview
Also known as the Hare & Tortoise algorithm. Uses two pointers moving at different speeds.

## When to use
*   Linked List cycle detection.
*   Finding middle of Linked List.
*   Cycle detection in array (Circular Array Loop).

## Interview Problems

### Problem: Happy Number (Easy)
**Pattern**: Fast & Slow Pointers (Implicit Cycle)

```java
/**
 * Determine if n is a happy number.
 * Time: O(log n)
 * Space: O(1)
 */
public boolean isHappy(int n) {
    int slow = n, fast = n;
    do {
        slow = sumSquare(slow);
        fast = sumSquare(sumSquare(fast));
    } while (slow != fast);
    
    return slow == 1;
}

private int sumSquare(int n) {
    int sum = 0;
    while (n > 0) {
        int digit = n % 10;
        sum += digit * digit;
        n /= 10;
    }
    return sum;
}
```

---
**Next**: [Pattern: Merge Intervals](29-pattern-merge-intervals.md)
