# Pattern: Fast & Slow Pointers - Complete Guide

## Overview
Fast & Slow Pointers (Floyd's Cycle Detection) uses two pointers moving at different speeds to detect cycles or find middle elements.

**Use Cases**: Cycle detection, finding middle, finding kth from end

---

## Template
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        
        if (slow == fast) return true;
    }
    
    return false;
}
```

---

## Problems

### 1. Linked List Cycle (Easy)
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### 2. Find Cycle Start (Medium)
```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) break;
    }
    
    if (fast == null || fast.next == null) return null;
    
    slow = head;
    while (slow != fast) {
        slow = slow.next;
        fast = fast.next;
    }
    
    return slow;
}
```

### 3. Happy Number (Easy)
```java
public boolean isHappy(int n) {
    int slow = n, fast = n;
    
    do {
        slow = sumOfSquares(slow);
        fast = sumOfSquares(sumOfSquares(fast));
    } while (slow != fast);
    
    return slow == 1;
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

---

## 🏦 Banking Context
**Scenario**: Detect circular payment chains (money laundering).  
**Solution**: Fast & slow pointers to detect cycles in transaction graph.

---

**Next**: [Pattern: Merge Intervals](29-pattern-merge-intervals.md)
