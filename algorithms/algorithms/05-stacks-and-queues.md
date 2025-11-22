# Stacks and Queues: Complete Master Guide

## Overview
Stacks and Queues are **fundamental linear data structures** that differ in their access patterns. While conceptually simple, they are the backbone of many complex algorithms (DFS, BFS, expression parsing, monotonic structures) and production systems (message queues, undo/redo, browser history).

For a Senior/Staff Engineer, mastering stacks and queues means:
- Recognizing when LIFO vs FIFO is needed
- Understanding monotonic stack/queue patterns
- Implementing efficient concurrent queues
- Discussing production message queue systems

---

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Implementation Details](#implementation-details)
3. [Core Patterns](#core-patterns)
4. [10+ Solved Problems](#solved-problems)
5. [Advanced Topics](#advanced-topics)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Fundamentals

### Stack (LIFO - Last In, First Out)

**Analogy**: Stack of plates—you can only add/remove from the top.

**Operations:**
- `push(item)`: Add element to top - O(1)
- `pop()`: Remove and return top element - O(1)
- `peek()`/`top()`: View top element without removing - O(1)
- `isEmpty()`: Check if stack is empty - O(1)

**Java Implementation:**
```java
// RECOMMENDED: Use Deque interface
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
int top = stack.pop();
int peek = stack.peek();

// AVOID: Legacy Stack class (synchronized, slow)
Stack<Integer> legacyStack = new Stack<>();  // Don't use this!
```

**Why ArrayDeque over Stack class?**
- `Stack` extends `Vector` (synchronized, thread-safe but slow)
- `ArrayDeque` is faster (no synchronization overhead)
- `ArrayDeque` is more memory-efficient

### Queue (FIFO - First In, First Out)

**Analogy**: Line at a bank—first person in line is served first.

**Operations:**
- `offer(item)`/`add(item)`: Add element to rear - O(1)
- `poll()`/`remove()`: Remove and return front element - O(1)
- `peek()`/`element()`: View front element - O(1)
- `isEmpty()`: Check if queue is empty - O(1)

**Java Implementation:**
```java
// Option 1: ArrayDeque (usually faster)
Queue<Integer> queue = new ArrayDeque<>();

// Option 2: LinkedList (allows null elements)
Queue<Integer> queue = new LinkedList<>();

queue.offer(1);
int front = queue.poll();
int peek = queue.peek();
```

**offer() vs add():**
- `offer()`: Returns false if operation fails
- `add()`: Throws exception if operation fails
- **Prefer `offer()`** for bounded queues

### Deque (Double-Ended Queue)

**Can add/remove from both ends:**
```java
Deque<Integer> deque = new ArrayDeque<>();

// Stack operations
deque.push(1);      // addFirst
deque.pop();        // removeFirst

// Queue operations
deque.offer(1);     // addLast
deque.poll();       // removeFirst

// Deque-specific
deque.offerFirst(1);
deque.offerLast(2);
deque.pollFirst();
deque.pollLast();
```

---

## Implementation Details

### Array-Based Stack

```java
class ArrayStack {
    private int[] arr;
    private int top;
    private int capacity;
    
    public ArrayStack(int size) {
        arr = new int[size];
        capacity = size;
        top = -1;
    }
    
    public void push(int x) {
        if (top == capacity - 1) {
            throw new StackOverflowError("Stack is full");
        }
        arr[++top] = x;
    }
    
    public int pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return arr[top--];
    }
    
    public int peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return arr[top];
    }
    
    public boolean isEmpty() {
        return top == -1;
    }
}
```

### Linked List-Based Stack

```java
class LinkedStack {
    private class Node {
        int data;
        Node next;
        Node(int data) { this.data = data; }
    }
    
    private Node top;
    
    public void push(int x) {
        Node newNode = new Node(x);
        newNode.next = top;
        top = newNode;
    }
    
    public int pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        int data = top.data;
        top = top.next;
        return data;
    }
    
    public boolean isEmpty() {
        return top == null;
    }
}
```

### Circular Queue (Array-Based)

```java
class CircularQueue {
    private int[] arr;
    private int front, rear, size, capacity;
    
    public CircularQueue(int k) {
        arr = new int[k];
        capacity = k;
        front = 0;
        rear = -1;
        size = 0;
    }
    
    public boolean enqueue(int value) {
        if (isFull()) return false;
        rear = (rear + 1) % capacity;
        arr[rear] = value;
        size++;
        return true;
    }
    
    public boolean dequeue() {
        if (isEmpty()) return false;
        front = (front + 1) % capacity;
        size--;
        return true;
    }
    
    public int front() {
        return isEmpty() ? -1 : arr[front];
    }
    
    public boolean isEmpty() {
        return size == 0;
    }
    
    public boolean isFull() {
        return size == capacity;
    }
}
```

---

## Core Patterns

### Pattern 1: Monotonic Stack

**When to use:**
- Next Greater Element
- Next Smaller Element
- Largest Rectangle in Histogram
- Stock Span Problem

**Concept**: Maintain stack in increasing or decreasing order.

**Template (Next Greater Element):**
```java
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>();  // Stores indices
    
    for (int i = 0; i < n; i++) {
        // While current element is greater than stack top
        while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
            int idx = stack.pop();
            result[idx] = nums[i];  // Found next greater
        }
        stack.push(i);
    }
    
    return result;
}
```

**Why O(n)?** Each element is pushed and popped at most once.

### Pattern 2: Parentheses/Bracket Matching

**Template:**
```java
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    
    for (char c : s.toCharArray()) {
        if (c == '(') stack.push(')');
        else if (c == '{') stack.push('}');
        else if (c == '[') stack.push(']');
        else if (stack.isEmpty() || stack.pop() != c) {
            return false;
        }
    }
    
    return stack.isEmpty();
}
```

### Pattern 3: BFS with Queue

**Template:**
```java
public void bfs(Node start) {
    Queue<Node> queue = new ArrayDeque<>();
    Set<Node> visited = new HashSet<>();
    
    queue.offer(start);
    visited.add(start);
    
    while (!queue.isEmpty()) {
        Node current = queue.poll();
        // Process current
        
        for (Node neighbor : current.neighbors) {
            if (!visited.contains(neighbor)) {
                queue.offer(neighbor);
                visited.add(neighbor);
            }
        }
    }
}
```

### Pattern 4: Sliding Window Maximum (Monotonic Deque)

**Template:**
```java
public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>();  // Stores indices
    int[] result = new int[nums.length - k + 1];
    
    for (int i = 0; i < nums.length; i++) {
        // Remove elements outside window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        
        // Remove smaller elements (maintain decreasing order)
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        deque.offerLast(i);
        
        // Add to result
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    
    return result;
}
```

---

## Solved Problems

### Problem 1: Valid Parentheses (Easy)

```java
/**
 * Determine if string has valid parentheses.
 * Time: O(n), Space: O(n)
 */
public boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    
    for (char c : s.toCharArray()) {
        if (c == '(') stack.push(')');
        else if (c == '{') stack.push('}');
        else if (c == '[') stack.push(']');
        else if (stack.isEmpty() || stack.pop() != c) {
            return false;
        }
    }
    
    return stack.isEmpty();
}
```

### Problem 2: Min Stack (Easy)

**Design stack that supports push, pop, top, and retrieving minimum in O(1).**

```java
class MinStack {
    private Deque<Integer> stack;
    private Deque<Integer> minStack;
    
    public MinStack() {
        stack = new ArrayDeque<>();
        minStack = new ArrayDeque<>();
    }
    
    public void push(int val) {
        stack.push(val);
        if (minStack.isEmpty() || val <= minStack.peek()) {
            minStack.push(val);
        }
    }
    
    public void pop() {
        int val = stack.pop();
        if (val == minStack.peek()) {
            minStack.pop();
        }
    }
    
    public int top() {
        return stack.peek();
    }
    
    public int getMin() {
        return minStack.peek();
    }
}
```

### Problem 3: Daily Temperatures (Medium)

```java
/**
 * Find days to wait for warmer temperature.
 * Time: O(n), Space: O(n)
 */
public int[] dailyTemperatures(int[] temperatures) {
    int n = temperatures.length;
    int[] result = new int[n];
    Deque<Integer> stack = new ArrayDeque<>();
    
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
            int idx = stack.pop();
            result[idx] = i - idx;
        }
        stack.push(i);
    }
    
    return result;
}
```

**Dry Run:**
```
Input: [73, 74, 75, 71, 69, 72, 76, 73]

i=0, temp=73: stack=[0]
i=1, temp=74 > 73: pop 0, result[0]=1, stack=[1]
i=2, temp=75 > 74: pop 1, result[1]=1, stack=[2]
i=3, temp=71 < 75: stack=[2,3]
i=4, temp=69 < 71: stack=[2,3,4]
i=5, temp=72 > 69,71: pop 4,3, result[4]=1, result[3]=2, stack=[2,5]
i=6, temp=76 > 72,75: pop 5,2, result[5]=1, result[2]=4, stack=[6]
i=7, temp=73 < 76: stack=[6,7]

Result: [1,1,4,2,1,1,0,0]
```

### Problem 4: Largest Rectangle in Histogram (Hard)

```java
/**
 * Find largest rectangle area in histogram.
 * Time: O(n), Space: O(n)
 */
public int largestRectangleArea(int[] heights) {
    Deque<Integer> stack = new ArrayDeque<>();
    int maxArea = 0;
    int n = heights.length;
    
    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];
        
        while (!stack.isEmpty() && h < heights[stack.peek()]) {
            int height = heights[stack.pop()];
            int width = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, height * width);
        }
        
        stack.push(i);
    }
    
    return maxArea;
}
```

### Problem 5: Implement Queue using Stacks (Easy)

```java
class MyQueue {
    private Deque<Integer> input;
    private Deque<Integer> output;
    
    public MyQueue() {
        input = new ArrayDeque<>();
        output = new ArrayDeque<>();
    }
    
    public void push(int x) {
        input.push(x);
    }
    
    public int pop() {
        peek();
        return output.pop();
    }
    
    public int peek() {
        if (output.isEmpty()) {
            while (!input.isEmpty()) {
                output.push(input.pop());
            }
        }
        return output.peek();
    }
    
    public boolean empty() {
        return input.isEmpty() && output.isEmpty();
    }
}
```

**Amortized O(1)**: Each element is moved at most twice (input→output).

### Problem 6: Evaluate Reverse Polish Notation (Medium)

```java
/**
 * Evaluate RPN expression.
 * Time: O(n), Space: O(n)
 */
public int evalRPN(String[] tokens) {
    Deque<Integer> stack = new ArrayDeque<>();
    
    for (String token : tokens) {
        if (token.equals("+")) {
            stack.push(stack.pop() + stack.pop());
        } else if (token.equals("-")) {
            int b = stack.pop();
            int a = stack.pop();
            stack.push(a - b);
        } else if (token.equals("*")) {
            stack.push(stack.pop() * stack.pop());
        } else if (token.equals("/")) {
            int b = stack.pop();
            int a = stack.pop();
            stack.push(a / b);
        } else {
            stack.push(Integer.parseInt(token));
        }
    }
    
    return stack.pop();
}
```

### Problem 7: Decode String (Medium)

```java
/**
 * Decode string like "3[a2[c]]" → "accaccacc".
 * Time: O(n), Space: O(n)
 */
public String decodeString(String s) {
    Deque<Integer> countStack = new ArrayDeque<>();
    Deque<StringBuilder> stringStack = new ArrayDeque<>();
    StringBuilder current = new StringBuilder();
    int k = 0;
    
    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            k = k * 10 + (c - '0');
        } else if (c == '[') {
            countStack.push(k);
            stringStack.push(current);
            current = new StringBuilder();
            k = 0;
        } else if (c == ']') {
            StringBuilder decoded = stringStack.pop();
            int count = countStack.pop();
            for (int i = 0; i < count; i++) {
                decoded.append(current);
            }
            current = decoded;
        } else {
            current.append(c);
        }
    }
    
    return current.toString();
}
```

---

## Advanced Topics

### Priority Queue (Heap-Based)

**Not a true queue (not FIFO), but important:**

```java
// Min heap (default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
```

**Time Complexity:**
- Insert: O(log n)
- Remove min/max: O(log n)
- Peek: O(1)

### Concurrent Queues

**Production systems use thread-safe queues:**

```java
// Blocking queue (thread-safe)
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();

// Producer
queue.put(task);  // Blocks if full

// Consumer
Task task = queue.take();  // Blocks if empty
```

**Types:**
- `ArrayBlockingQueue`: Bounded, array-based
- `LinkedBlockingQueue`: Optionally bounded, linked-list-based
- `PriorityBlockingQueue`: Unbounded priority queue
- `DelayQueue`: Elements available after delay

---

## Interview Questions & Answers

### Q1: "Why use Deque instead of Stack class in Java?"

**Model Answer:**
"Java's `Stack` class is a legacy class that extends `Vector`, which means all its methods are synchronized. This synchronization overhead makes it slower than necessary for single-threaded use cases, which are the vast majority.

`Deque` (Double-Ended Queue) interface, typically implemented by `ArrayDeque`, provides:
1. **Better performance**: No synchronization overhead
2. **More flexibility**: Can be used as stack, queue, or deque
3. **Modern design**: Part of Collections Framework
4. **Memory efficiency**: ArrayDeque uses circular array, more cache-friendly

In production code, I always use `Deque<Integer> stack = new ArrayDeque<>()` for stack operations. The only time I'd consider synchronized collections is in multi-threaded scenarios, but even then, I'd use `ConcurrentLinkedDeque` or explicit synchronization rather than the legacy `Stack` class."

### Q2: "Explain monotonic stack and when to use it."

**Model Answer:**
"A monotonic stack maintains elements in either increasing or decreasing order. When adding a new element, we pop elements that violate the monotonic property.

**Use cases:**
- Next Greater/Smaller Element
- Largest Rectangle in Histogram
- Stock Span Problem
- Trapping Rain Water

**Why it's efficient:** Each element is pushed and popped at most once, giving O(n) time complexity instead of O(n²) brute force.

**Example - Next Greater Element:**
```java
// Maintain decreasing stack
while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
    int idx = stack.pop();
    result[idx] = nums[i];  // Found next greater
}
stack.push(i);
```

In financial systems, we use this pattern for analyzing price movements—finding the next day a stock price exceeds a threshold, which is critical for options pricing and trading strategies."

### Q3: "How would you implement a queue with O(1) max operation?"

**Model Answer:**
"I'd use a monotonic deque (double-ended queue) to track maximums, similar to the Sliding Window Maximum problem.

```java
class MaxQueue {
    private Deque<Integer> queue;  // Actual queue
    private Deque<Integer> maxDeque;  // Monotonic decreasing deque
    
    public void enqueue(int val) {
        queue.offerLast(val);
        // Remove smaller elements from maxDeque
        while (!maxDeque.isEmpty() && maxDeque.peekLast() < val) {
            maxDeque.pollLast();
        }
        maxDeque.offerLast(val);
    }
    
    public int dequeue() {
        int val = queue.pollFirst();
        if (val == maxDeque.peekFirst()) {
            maxDeque.pollFirst();
        }
        return val;
    }
    
    public int max() {
        return maxDeque.peekFirst();  // O(1)
    }
}
```

The key insight is that `maxDeque` maintains potential maximums in decreasing order. When we dequeue, we only remove from `maxDeque` if it's the current maximum.

This pattern is used in real-time analytics systems where we need to track maximum values over sliding windows—like peak transaction volumes or maximum price movements."

---

## 🏦 Banking & Production Context

### Message Queues (Kafka, RabbitMQ, JMS)

**Scenario**: Order processing in trading systems

**Why FIFO matters**: Ensures fairness—orders processed in exact sequence received.

```java
// Kafka consumer (simplified)
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Order> record : records) {
        processOrder(record.value());  // FIFO guarantee
    }
}
```

**Dead Letter Queue (DLQ)**: Failed messages go to special queue for manual inspection.

### Undo/Redo Functionality

**Scenario**: Trading platform UI with drawing tools

```java
class CommandHistory {
    private Deque<Command> undoStack = new ArrayDeque<>();
    private Deque<Command> redoStack = new ArrayDeque<>();
    
    public void execute(Command cmd) {
        cmd.execute();
        undoStack.push(cmd);
        redoStack.clear();  // Clear redo history
    }
    
    public void undo() {
        if (!undoStack.isEmpty()) {
            Command cmd = undoStack.pop();
            cmd.undo();
            redoStack.push(cmd);
        }
    }
    
    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute();
            undoStack.push(cmd);
        }
    }
}
```

### Rate Limiting with Queue

**Scenario**: Limit API requests to prevent abuse

```java
class RateLimiter {
    private Queue<Long> timestamps;
    private int maxRequests;
    private long windowMs;
    
    public boolean allowRequest() {
        long now = System.currentTimeMillis();
        
        // Remove old timestamps
        while (!timestamps.isEmpty() && now - timestamps.peek() > windowMs) {
            timestamps.poll();
        }
        
        if (timestamps.size() < maxRequests) {
            timestamps.offer(now);
            return true;
        }
        
        return false;
    }
}
```

---

## Key Takeaways

1. **Use Deque, not Stack**: ArrayDeque is faster and more flexible
2. **Monotonic stack**: O(n) solution for next greater/smaller problems
3. **BFS uses queue**: Level-order traversal, shortest path
4. **Amortized analysis**: Queue with two stacks is O(1) amortized
5. **Production**: Message queues are backbone of distributed systems
6. **Thread-safety**: Use BlockingQueue for concurrent scenarios
7. **Deque**: Can function as stack, queue, or both

---

**Next**: [Trees: Binary Trees](06-trees-binary-trees.md)
