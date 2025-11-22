# Pattern: K-way Merge - Complete Guide

## Overview
Merge K sorted arrays/lists using min heap.

**Time**: O(n log k) where n is total elements  
**Space**: O(k)

---

## Template
```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>(
        (a, b) -> a.val - b.val
    );
    
    for (ListNode list : lists) {
        if (list != null) {
            heap.offer(list);
        }
    }
    
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;
    
    while (!heap.isEmpty()) {
        ListNode node = heap.poll();
        curr.next = node;
        curr = curr.next;
        
        if (node.next != null) {
            heap.offer(node.next);
        }
    }
    
    return dummy.next;
}
```

---

## Problems

### 1. Merge K Sorted Lists (Hard)
```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> a.val - b.val);
    
    for (ListNode list : lists) {
        if (list != null) pq.offer(list);
    }
    
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;
    
    while (!pq.isEmpty()) {
        ListNode node = pq.poll();
        curr.next = node;
        curr = curr.next;
        if (node.next != null) pq.offer(node.next);
    }
    
    return dummy.next;
}
```

### 2. Smallest Range Covering K Lists (Hard)
```java
public int[] smallestRange(List<List<Integer>> nums) {
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    int max = Integer.MIN_VALUE;
    
    for (int i = 0; i < nums.size(); i++) {
        pq.offer(new int[]{nums.get(i).get(0), i, 0});
        max = Math.max(max, nums.get(i).get(0));
    }
    
    int[] range = new int[]{0, Integer.MAX_VALUE};
    
    while (pq.size() == nums.size()) {
        int[] curr = pq.poll();
        int min = curr[0], listIdx = curr[1], elemIdx = curr[2];
        
        if (max - min < range[1] - range[0]) {
            range = new int[]{min, max};
        }
        
        if (elemIdx + 1 < nums.get(listIdx).size()) {
            int next = nums.get(listIdx).get(elemIdx + 1);
            pq.offer(new int[]{next, listIdx, elemIdx + 1});
            max = Math.max(max, next);
        }
    }
    
    return range;
}
```

---

## 🏦 Banking Context
**Scenario**: Merge sorted transaction logs from K branches.  
**Solution**: K-way merge using min heap.

---

**Next**: [Pattern: Topological Sort](39-pattern-topological-sort.md)
