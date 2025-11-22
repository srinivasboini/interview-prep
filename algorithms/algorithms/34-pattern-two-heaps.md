# Pattern: Two Heaps - Complete Guide

## Overview
Two Heaps pattern maintains median or balances elements using max heap and min heap.

**Time**: O(log n) insert, O(1) median  
**Space**: O(n)

---

## Template
```java
class MedianFinder {
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    
    public void addNum(int num) {
        maxHeap.offer(num);
        minHeap.offer(maxHeap.poll());
        
        if (maxHeap.size() < minHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }
    
    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) {
            return maxHeap.peek();
        }
        return (maxHeap.peek() + minHeap.peek()) / 2.0;
    }
}
```

---

## Problems

### 1. Find Median from Data Stream (Hard)
```java
class MedianFinder {
    PriorityQueue<Integer> small = new PriorityQueue<>(Collections.reverseOrder());
    PriorityQueue<Integer> large = new PriorityQueue<>();
    
    public void addNum(int num) {
        small.offer(num);
        large.offer(small.poll());
        
        if (small.size() < large.size()) {
            small.offer(large.poll());
        }
    }
    
    public double findMedian() {
        return small.size() > large.size() ? 
               small.peek() : 
               (small.peek() + large.peek()) / 2.0;
    }
}
```

### 2. Sliding Window Median (Hard)
```java
public double[] medianSlidingWindow(int[] nums, int k) {
    TreeMap<Integer, Integer> left = new TreeMap<>(Collections.reverseOrder());
    TreeMap<Integer, Integer> right = new TreeMap<>();
    double[] result = new double[nums.length - k + 1];
    
    for (int i = 0; i < nums.length; i++) {
        add(left, right, nums[i], k);
        
        if (i >= k - 1) {
            result[i - k + 1] = getMedian(left, right, k);
            remove(left, right, nums[i - k + 1], k);
        }
    }
    
    return result;
}
```

---

## 🏦 Banking Context
**Scenario**: Track median transaction amount in real-time.  
**Solution**: Two heaps for O(1) median queries.

---

**Next**: [Pattern: Subsets](35-pattern-subsets.md)
