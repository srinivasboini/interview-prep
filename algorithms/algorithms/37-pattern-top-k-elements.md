# Pattern: Top K Elements - Complete Guide

## Overview
Find top K elements using heap or quickselect.

**Time**: O(n log k) heap, O(n) average quickselect  
**Space**: O(k)

---

## Template
```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> count = new HashMap<>();
    for (int num : nums) {
        count.put(num, count.getOrDefault(num, 0) + 1);
    }
    
    PriorityQueue<Integer> heap = new PriorityQueue<>(
        (a, b) -> count.get(a) - count.get(b)
    );
    
    for (int num : count.keySet()) {
        heap.offer(num);
        if (heap.size() > k) {
            heap.poll();
        }
    }
    
    int[] result = new int[k];
    for (int i = 0; i < k; i++) {
        result[i] = heap.poll();
    }
    
    return result;
}
```

---

## Problems

### 1. Kth Largest Element (Medium)
```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>();
    
    for (int num : nums) {
        heap.offer(num);
        if (heap.size() > k) {
            heap.poll();
        }
    }
    
    return heap.peek();
}
```

### 2. Top K Frequent Elements (Medium)
```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> count = new HashMap<>();
    for (int num : nums) {
        count.put(num, count.getOrDefault(num, 0) + 1);
    }
    
    PriorityQueue<Integer> heap = new PriorityQueue<>(
        (a, b) -> count.get(a) - count.get(b)
    );
    
    for (int num : count.keySet()) {
        heap.offer(num);
        if (heap.size() > k) heap.poll();
    }
    
    int[] result = new int[k];
    for (int i = 0; i < k; i++) {
        result[i] = heap.poll();
    }
    return result;
}
```

---

## 🏦 Banking Context
**Scenario**: Find top K most active trading accounts.  
**Solution**: Min heap of size K for efficient tracking.

---

**Next**: [Pattern: K-way Merge](38-pattern-k-way-merge.md)
