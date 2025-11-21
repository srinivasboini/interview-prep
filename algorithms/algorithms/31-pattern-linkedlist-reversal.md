# Pattern: LinkedList Reversal

## Overview
In-place reversal of linked lists.

## Interview Problems

### Problem: Reverse Nodes in k-Group (Hard)
**Pattern**: In-place Reversal

```java
/**
 * Reverse nodes in k-group.
 * Time: O(n)
 * Space: O(1)
 */
public ListNode reverseKGroup(ListNode head, int k) {
    if (head == null || k == 1) return head;
    
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode curr = head, prev = dummy;
    int count = 0;
    
    while (curr != null) {
        count++;
        if (count % k == 0) {
            prev = reverse(prev, curr.next);
            curr = prev.next;
        } else {
            curr = curr.next;
        }
    }
    return dummy.next;
}

private ListNode reverse(ListNode begin, ListNode end) {
    ListNode prev = begin;
    ListNode curr = begin.next;
    ListNode first = curr;
    ListNode next;
    
    while (curr != end) {
        next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    
    begin.next = prev;
    first.next = curr;
    return first;
}
```

---
**Next**: [Pattern: Tree BFS](32-pattern-tree-bfs.md)
