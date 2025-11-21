# Stacks and Queues

## Overview
Stacks and Queues are linear data structures that differ in how elements are added and removed. They are often used as intermediate structures in complex algorithms (BFS, DFS, Expression Parsing).

## Fundamentals

### Stack (LIFO - Last In First Out)
*   **Analogy**: A stack of plates.
*   **Operations**:
    *   `push(item)`: Add to top.
    *   `pop()`: Remove from top.
    *   `peek()`: Look at top.
*   **Java Implementation**: Use `Deque` interface (`ArrayDeque`) instead of legacy `Stack` class.
    ```java
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(1);
    stack.pop();
    ```

### Queue (FIFO - First In First Out)
*   **Analogy**: A line at a bank.
*   **Operations**:
    *   `offer(item)`: Add to rear.
    *   `poll()`: Remove from front.
    *   `peek()`: Look at front.
*   **Java Implementation**: `Queue` interface (`LinkedList` or `ArrayDeque`).
    ```java
    Queue<Integer> queue = new ArrayDeque<>(); // or new LinkedList<>();
    queue.offer(1);
    queue.poll();
    ```

## Operations and Complexity

| Operation | Stack | Queue |
|-----------|-------|-------|
| Push/Offer| O(1)  | O(1)  |
| Pop/Poll  | O(1)  | O(1)  |
| Peek      | O(1)  | O(1)  |
| Search    | O(n)  | O(n)  |

## Common Patterns

### 1. Monotonic Stack
Used to find "Next Greater Element" or "Next Smaller Element".
*   **Concept**: Keep stack sorted (increasing or decreasing).
*   **Complexity**: O(n) - Each element pushed and popped at most once.

### 2. BFS (Queue)
Used for level-order traversal in trees/graphs.

### 3. Parentheses Matching
Classic stack problem.

## Visual Diagrams

### Monotonic Stack (Next Greater Element)
```mermaid
graph TD
    subgraph "Input: [2, 1, 5]"
    Step1[Push 2] --> Step2[Push 1 < 2]
    Step2 --> Step3[5 > 1? Pop 1. NGE(1)=5]
    Step3 --> Step4[5 > 2? Pop 2. NGE(2)=5]
    Step4 --> Step5[Push 5]
    end
```

## Interview Problems

### Problem 1: Valid Parentheses (Easy)
**Pattern**: Stack

```java
/**
 * Determine if the input string has valid parentheses.
 * Time: O(n)
 * Space: O(n)
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

### Problem 2: Daily Temperatures (Medium)
**Pattern**: Monotonic Stack

```java
/**
 * Find number of days to wait for a warmer temperature.
 * Time: O(n)
 * Space: O(n)
 */
public int[] dailyTemperatures(int[] temperatures) {
    int n = temperatures.length;
    int[] result = new int[n];
    Deque<Integer> stack = new ArrayDeque<>(); // Stores indices
    
    for (int i = 0; i < n; i++) {
        // While current temp is warmer than stack top
        while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
            int idx = stack.pop();
            result[idx] = i - idx;
        }
        stack.push(i);
    }
    
    return result;
}
```

### Problem 3: Implement Queue using Stacks (Easy)
**Pattern**: Two Stacks

```java
class MyQueue {
    Deque<Integer> input = new ArrayDeque<>();
    Deque<Integer> output = new ArrayDeque<>();

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

## 🏦 Banking Context: Order Processing
*   **Message Queues (Kafka/JMS)**: The backbone of banking systems.
    *   **FIFO**: Ensures orders are processed in the exact sequence received (critical for fairness).
    *   **Dead Letter Queue (DLQ)**: A special queue for failed messages (bad orders) to be inspected later.
*   **Undo/Redo**: Stacks are used in trading UI for undoing drawing tools on charts.

## Common Pitfalls
1.  **Empty Stack Exception**: Always check `!stack.isEmpty()` before popping/peeking.
2.  **Stack vs Deque**: Use `Deque` interface in Java, not `Stack` (which is synchronized and slow).
3.  **Queue Implementation**: `LinkedList` is a standard Queue, but `ArrayDeque` is often faster (better cache locality) if no nulls are needed.

---
**Next**: [Trees: Binary Trees](06-trees-binary-trees.md)
