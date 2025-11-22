# Pattern: Linked List Reversal - Complete Guide

## Overview
Reverse linked list in-place using iterative or recursive approach.

**Time**: O(n), **Space**: O(1) iterative, O(n) recursive

---

## Template
```java
public ListNode reverse(ListNode head) {
    ListNode prev = null, curr = head;
    
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    
    return prev;
}
```

---

## Problems

### 1. Reverse Linked List (Easy)
```java
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    while (head != null) {
        ListNode next = head.next;
        head.next = prev;
        prev = head;
        head = next;
    }
    return prev;
}
```

### 2. Reverse Between (Medium)
```java
public ListNode reverseBetween(ListNode head, int left, int right) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode prev = dummy;
    
    for (int i = 0; i < left - 1; i++) {
        prev = prev.next;
    }
    
    ListNode curr = prev.next;
    for (int i = 0; i < right - left; i++) {
        ListNode next = curr.next;
        curr.next = next.next;
        next.next = prev.next;
        prev.next = next;
    }
    
    return dummy.next;
}
```

### 3. Reverse K Group (Hard)
```java
public ListNode reverseKGroup(ListNode head, int k) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode prev = dummy;
    
    while (true) {
        ListNode kth = getKth(prev, k);
        if (kth == null) break;
        
        ListNode groupNext = kth.next;
        ListNode curr = prev.next, next;
        ListNode prevNode = kth.next;
        
        while (curr != groupNext) {
            next = curr.next;
            curr.next = prevNode;
            prevNode = curr;
            curr = next;
        }
        
        ListNode temp = prev.next;
        prev.next = kth;
        prev = temp;
    }
    
    return dummy.next;
}

private ListNode getKth(ListNode curr, int k) {
    while (curr != null && k > 0) {
        curr = curr.next;
        k--;
    }
    return curr;
}
```

---

## 🏦 Banking Context
**Scenario**: Reverse transaction history for audit trail.  
**Solution**: In-place linked list reversal.

---

**Next**: [Pattern: Tree BFS](32-pattern-tree-bfs.md)
