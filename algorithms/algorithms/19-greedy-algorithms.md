# Greedy Algorithms: Complete Master Guide

## Overview
Greedy algorithms make **locally optimal choices** at each step, hoping to find a global optimum. They're simple, fast, and elegant—but don't always work. The challenge is proving when greedy is correct.

**Key Insight**: Greedy works when the problem has the **greedy choice property** (local optimum leads to global optimum) and **optimal substructure**.

For Senior/Staff Engineers, mastering greedy algorithms means:
- Recognizing when greedy works (and when it doesn't)
- Proving correctness (exchange argument, stays-ahead)
- Comparing greedy vs DP
- Understanding classic greedy problems (intervals, Huffman, Dijkstra)

---

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [When Greedy Works](#when-greedy-works)
3. [Common Patterns](#common-patterns)
4. [15+ Solved Problems](#solved-problems)
5. [Greedy vs DP](#greedy-vs-dp)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Fundamentals

### What is a Greedy Algorithm?

**Definition**: Make the locally optimal choice at each step without reconsidering previous choices.

**Characteristics**:
- **Simple**: Easy to implement
- **Fast**: Usually O(n log n) or O(n)
- **No backtracking**: Once a choice is made, it's final
- **Not always correct**: Requires proof

**Example - Coin Change**:
```
Coins: [25, 10, 5, 1]
Amount: 41

Greedy: 25 + 10 + 5 + 1 = 4 coins ✓ (works for US coins)

Coins: [25, 10, 1]
Amount: 30

Greedy: 25 + 1 + 1 + 1 + 1 + 1 = 6 coins ✗
Optimal: 10 + 10 + 10 = 3 coins

Greedy fails! Need DP.
```

---

## When Greedy Works

### Two Required Properties

**1. Greedy Choice Property**
- Locally optimal choice leads to globally optimal solution
- Can make choice without considering future consequences

**2. Optimal Substructure**
- Optimal solution contains optimal solutions to subproblems
- (Same as DP!)

### Proving Greedy Correctness

**Method 1: Exchange Argument**
- Assume optimal solution differs from greedy
- Show we can exchange to match greedy without worsening
- Contradiction → greedy is optimal

**Method 2: Stays-Ahead Argument**
- Show greedy solution is always "ahead" of any other solution
- At each step, greedy is at least as good

**Example - Activity Selection**:
```
Greedy: Always pick activity that ends earliest
Proof: If optimal picks different activity, we can exchange it
       with greedy choice without reducing number of activities.
```

---

## Common Patterns

### Pattern 1: Interval Scheduling

**Problem**: Select maximum non-overlapping intervals.

**Greedy strategy**: Sort by end time, pick earliest ending.

```java
/**
 * Maximum non-overlapping intervals.
 * Time: O(n log n), Space: O(1)
 */
public int maxNonOverlapping(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[1] - b[1]);  // Sort by end time
    
    int count = 0;
    int lastEnd = Integer.MIN_VALUE;
    
    for (int[] interval : intervals) {
        if (interval[0] >= lastEnd) {  // No overlap
            count++;
            lastEnd = interval[1];
        }
    }
    
    return count;
}
```

### Pattern 2: Fractional Knapsack

**Problem**: Maximize value with fractional items allowed.

**Greedy strategy**: Sort by value/weight ratio, take highest first.

```java
/**
 * Fractional knapsack.
 * Time: O(n log n), Space: O(n)
 */
public double fractionalKnapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    Item[] items = new Item[n];
    
    for (int i = 0; i < n; i++) {
        items[i] = new Item(weights[i], values[i]);
    }
    
    Arrays.sort(items, (a, b) -> 
        Double.compare(b.ratio(), a.ratio()));  // Descending ratio
    
    double totalValue = 0;
    int remainingCapacity = capacity;
    
    for (Item item : items) {
        if (remainingCapacity >= item.weight) {
            totalValue += item.value;
            remainingCapacity -= item.weight;
        } else {
            totalValue += item.value * ((double) remainingCapacity / item.weight);
            break;
        }
    }
    
    return totalValue;
}

class Item {
    int weight, value;
    Item(int w, int v) { weight = w; value = v; }
    double ratio() { return (double) value / weight; }
}
```

### Pattern 3: Huffman Coding

**Problem**: Optimal prefix-free binary encoding.

**Greedy strategy**: Build tree bottom-up, merging two smallest frequencies.

```java
/**
 * Huffman coding.
 * Time: O(n log n), Space: O(n)
 */
class HuffmanNode {
    char ch;
    int freq;
    HuffmanNode left, right;
}

public Map<Character, String> huffmanCoding(Map<Character, Integer> freqMap) {
    PriorityQueue<HuffmanNode> pq = new PriorityQueue<>(
        (a, b) -> a.freq - b.freq
    );
    
    // Create leaf nodes
    for (Map.Entry<Character, Integer> entry : freqMap.entrySet()) {
        HuffmanNode node = new HuffmanNode();
        node.ch = entry.getKey();
        node.freq = entry.getValue();
        pq.offer(node);
    }
    
    // Build tree
    while (pq.size() > 1) {
        HuffmanNode left = pq.poll();
        HuffmanNode right = pq.poll();
        
        HuffmanNode parent = new HuffmanNode();
        parent.freq = left.freq + right.freq;
        parent.left = left;
        parent.right = right;
        
        pq.offer(parent);
    }
    
    // Generate codes
    Map<Character, String> codes = new HashMap<>();
    generateCodes(pq.peek(), "", codes);
    return codes;
}

private void generateCodes(HuffmanNode node, String code, Map<Character, String> codes) {
    if (node == null) return;
    
    if (node.left == null && node.right == null) {
        codes.put(node.ch, code);
        return;
    }
    
    generateCodes(node.left, code + "0", codes);
    generateCodes(node.right, code + "1", codes);
}
```

---

## Solved Problems

### Problem 1: Jump Game (Medium)

```java
/**
 * Can reach last index?
 * Time: O(n), Space: O(1)
 */
public boolean canJump(int[] nums) {
    int maxReach = 0;
    
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false;  // Can't reach current position
        maxReach = Math.max(maxReach, i + nums[i]);
    }
    
    return true;
}
```

### Problem 2: Jump Game II (Medium)

```java
/**
 * Minimum jumps to reach end.
 * Time: O(n), Space: O(1)
 */
public int jump(int[] nums) {
    int jumps = 0;
    int currentEnd = 0;
    int farthest = 0;
    
    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);
        
        if (i == currentEnd) {
            jumps++;
            currentEnd = farthest;
        }
    }
    
    return jumps;
}
```

### Problem 3: Gas Station (Medium)

```java
/**
 * Find starting gas station to complete circuit.
 * Time: O(n), Space: O(1)
 */
public int canCompleteCircuit(int[] gas, int[] cost) {
    int totalGas = 0;
    int totalCost = 0;
    int tank = 0;
    int start = 0;
    
    for (int i = 0; i < gas.length; i++) {
        totalGas += gas[i];
        totalCost += cost[i];
        tank += gas[i] - cost[i];
        
        if (tank < 0) {
            start = i + 1;
            tank = 0;
        }
    }
    
    return totalGas >= totalCost ? start : -1;
}
```

### Problem 4: Partition Labels (Medium)

```java
/**
 * Partition string into max parts where each letter appears in at most one part.
 * Time: O(n), Space: O(1)
 */
public List<Integer> partitionLabels(String s) {
    int[] last = new int[26];
    
    // Record last occurrence of each character
    for (int i = 0; i < s.length(); i++) {
        last[s.charAt(i) - 'a'] = i;
    }
    
    List<Integer> result = new ArrayList<>();
    int start = 0;
    int end = 0;
    
    for (int i = 0; i < s.length(); i++) {
        end = Math.max(end, last[s.charAt(i) - 'a']);
        
        if (i == end) {
            result.add(end - start + 1);
            start = i + 1;
        }
    }
    
    return result;
}
```

### Problem 5: Merge Intervals (Medium)

```java
/**
 * Merge overlapping intervals.
 * Time: O(n log n), Space: O(n)
 */
public int[][] merge(int[][] intervals) {
    if (intervals.length <= 1) return intervals;
    
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    
    List<int[]> result = new ArrayList<>();
    int[] current = intervals[0];
    result.add(current);
    
    for (int[] interval : intervals) {
        if (interval[0] <= current[1]) {
            current[1] = Math.max(current[1], interval[1]);  // Merge
        } else {
            current = interval;
            result.add(current);
        }
    }
    
    return result.toArray(new int[result.size()][]);
}
```

### Problem 6: Meeting Rooms II (Medium)

```java
/**
 * Minimum meeting rooms needed.
 * Time: O(n log n), Space: O(n)
 */
public int minMeetingRooms(int[][] intervals) {
    int[] starts = new int[intervals.length];
    int[] ends = new int[intervals.length];
    
    for (int i = 0; i < intervals.length; i++) {
        starts[i] = intervals[i][0];
        ends[i] = intervals[i][1];
    }
    
    Arrays.sort(starts);
    Arrays.sort(ends);
    
    int rooms = 0;
    int endIdx = 0;
    
    for (int start : starts) {
        if (start < ends[endIdx]) {
            rooms++;  // Need new room
        } else {
            endIdx++;  // Reuse room
        }
    }
    
    return rooms;
}
```

### Problem 7: Task Scheduler (Medium)

```java
/**
 * Minimum time to complete tasks with cooldown.
 * Time: O(n), Space: O(1)
 */
public int leastInterval(char[] tasks, int n) {
    int[] freq = new int[26];
    int maxFreq = 0;
    
    for (char task : tasks) {
        freq[task - 'A']++;
        maxFreq = Math.max(maxFreq, freq[task - 'A']);
    }
    
    int maxCount = 0;
    for (int f : freq) {
        if (f == maxFreq) maxCount++;
    }
    
    int partCount = maxFreq - 1;
    int partLength = n - (maxCount - 1);
    int emptySlots = partCount * partLength;
    int availableTasks = tasks.length - maxFreq * maxCount;
    int idles = Math.max(0, emptySlots - availableTasks);
    
    return tasks.length + idles;
}
```

### Problem 8: Remove K Digits (Medium)

```java
/**
 * Remove k digits to get smallest number.
 * Time: O(n), Space: O(n)
 */
public String removeKdigits(String num, int k) {
    Deque<Character> stack = new ArrayDeque<>();
    
    for (char digit : num.toCharArray()) {
        while (!stack.isEmpty() && k > 0 && stack.peek() > digit) {
            stack.pop();
            k--;
        }
        stack.push(digit);
    }
    
    // Remove remaining k digits
    while (k > 0) {
        stack.pop();
        k--;
    }
    
    // Build result
    StringBuilder sb = new StringBuilder();
    while (!stack.isEmpty()) {
        sb.append(stack.pollLast());
    }
    
    // Remove leading zeros
    while (sb.length() > 1 && sb.charAt(0) == '0') {
        sb.deleteCharAt(0);
    }
    
    return sb.length() == 0 ? "0" : sb.toString();
}
```

---

## Greedy vs DP

| Aspect | Greedy | Dynamic Programming |
|--------|--------|---------------------|
| **Decision** | Irrevocable | Considers all options |
| **Subproblems** | Independent | Overlapping |
| **Correctness** | Needs proof | Always correct if formulated right |
| **Time** | Usually O(n log n) | Usually O(n²) or O(n×m) |
| **Examples** | Activity Selection, Huffman | 0/1 Knapsack, LCS |

**When to use Greedy**:
- Greedy choice property holds
- Simpler and faster than DP

**When to use DP**:
- Greedy fails (need to try all options)
- Overlapping subproblems exist

**Example - Coin Change**:
- **US coins [25,10,5,1]**: Greedy works (canonical system)
- **Arbitrary coins [25,10,1]**: Greedy fails, need DP

---

## Interview Questions & Answers

### Q1: "How do you prove a greedy algorithm is correct?"

**Model Answer:**
"I use two main proof techniques:

**1. Exchange Argument**:
- Assume optimal solution O differs from greedy solution G
- Show we can exchange elements of O to match G without worsening
- Contradiction → G must be optimal

**Example - Activity Selection**:
```
Greedy: Pick earliest ending activity
Proof: If optimal picks different first activity, we can replace it
       with greedy choice (ends earlier), still fits all others.
```

**2. Stays-Ahead Argument**:
- Show greedy is always 'ahead' of any other solution
- At each step, greedy is at least as good

**Example - Fractional Knapsack**:
```
Greedy: Pick highest value/weight ratio first
Proof: At any point, greedy has extracted maximum value per unit weight.
```

In interviews, I always mention that greedy needs proof—unlike DP which is correct by construction. I've seen production bugs where greedy was applied incorrectly (e.g., task scheduling with dependencies)."

### Q2: "When does greedy fail for coin change?"

**Model Answer:**
"Greedy fails when the coin system is **non-canonical**.

**Canonical system** (greedy works):
- US coins: [25, 10, 5, 1]
- For any amount, greedy gives optimal solution
- Example: 41 = 25 + 10 + 5 + 1 (4 coins) ✓

**Non-canonical system** (greedy fails):
- Coins: [25, 10, 1]
- Amount: 30
- Greedy: 25 + 1 + 1 + 1 + 1 + 1 = 6 coins ✗
- Optimal: 10 + 10 + 10 = 3 coins ✓

**Why greedy fails**: Choosing 25 prevents us from using three 10s.

**Solution**: Use Dynamic Programming
```java
dp[i] = min(dp[i - coin] + 1) for all coins
```

In banking systems, this matters for transaction fee optimization. We can't use greedy for arbitrary fee structures—need DP to find minimum cost."

### Q3: "Explain the greedy choice property."

**Model Answer:**
"The greedy choice property means that **making the locally optimal choice at each step leads to a globally optimal solution**.

**Formally**: We can assemble a global optimum by making locally optimal choices without reconsidering.

**Example - Activity Selection**:
- Greedy choice: Pick activity that ends earliest
- Property: This choice doesn't prevent optimal solution
- Why: Earliest ending leaves maximum room for future activities

**Counter-example - 0/1 Knapsack**:
- Greedy choice: Pick highest value/weight ratio
- Fails: High-ratio small items might prevent taking one high-value item
- Need DP to try all combinations

**Testing for greedy choice property**:
1. Make greedy choice
2. Show remaining problem has optimal substructure
3. Prove greedy choice doesn't prevent optimal solution

In production, I always verify greedy choice property before implementing. For example, in trade execution, greedy 'execute largest order first' might violate market impact constraints—need more sophisticated optimization."

---

## 🏦 Banking & Production Context

### ATM Cash Dispensing

**Scenario**: Dispense $180 with minimum bills.

```java
/**
 * ATM cash dispensing (greedy works for standard denominations).
 */
public Map<Integer, Integer> dispenseCash(int amount) {
    int[] denominations = {100, 50, 20, 10, 5, 1};
    Map<Integer, Integer> result = new LinkedHashMap<>();
    
    for (int denom : denominations) {
        if (amount >= denom) {
            int count = amount / denom;
            result.put(denom, count);
            amount %= denom;
        }
    }
    
    return result;
}
```

### Trade Execution (VWAP)

**Scenario**: Execute large order with minimum market impact.

**Greedy strategy**: Split order across time intervals to match volume profile.

```java
/**
 * VWAP execution strategy.
 */
class VWAPExecutor {
    public List<Order> splitOrder(int totalShares, int[] volumeProfile) {
        List<Order> orders = new ArrayList<>();
        int totalVolume = Arrays.stream(volumeProfile).sum();
        
        for (int i = 0; i < volumeProfile.length; i++) {
            int shares = (int) ((double) totalShares * volumeProfile[i] / totalVolume);
            orders.add(new Order(shares, i));
        }
        
        return orders;
    }
}
```

### Loan Repayment Optimization

**Scenario**: Pay off loans to minimize interest.

**Greedy strategy**: Pay highest interest rate loans first (avalanche method).

```java
/**
 * Loan repayment strategy.
 */
public List<Loan> repaymentOrder(List<Loan> loans) {
    loans.sort((a, b) -> Double.compare(b.interestRate, a.interestRate));
    return loans;  // Pay highest rate first
}
```

---

## Key Takeaways

1. **Greedy = local optimum**: Make best choice at each step
2. **Needs proof**: Exchange argument or stays-ahead
3. **Fast**: Usually O(n log n) from sorting
4. **Doesn't always work**: Check greedy choice property
5. **Common patterns**: Intervals, fractional knapsack, Huffman
6. **vs DP**: Greedy is simpler but less general
7. **Production**: ATM dispensing, trade execution, loan repayment

---

**Next**: [Divide and Conquer](20-divide-and-conquer.md)
