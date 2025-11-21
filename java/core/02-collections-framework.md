# Java Collections Framework - Interview Guide

## Overview

The Java Collections Framework is a unified architecture for representing and manipulating collections. It's critical for interviews because:
- **Daily usage**: Every Java developer uses collections constantly
- **Implementation knowledge**: Interviewers test understanding of trade-offs
- **Performance**: Choosing wrong collections can tank production systems
- **Thread safety**: Understanding concurrent behavior separates junior from senior developers

Focus on **why** you'd choose one collection over another, not memorizing APIs.

---

## 1. Collections Hierarchy

```
                    ┌─────────────────────────────────────┐
                    │         Iterable<E>                 │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────▼────────────────────┐
                    │       Collection<E>                 │
                    └──┬──────────────────┬───────────────┬┘
                       │                  │               │
              ┌────────▼────────┐  ┌──────▼────────┐  ┌──▼──────────────┐
              │    List<E>      │  │    Set<E>     │  │   Queue<E>      │
              │ (ordered,       │  │ (unique,      │  │ (ordered for    │
              │  duplicate OK)  │  │  unordered)   │  │  processing)    │
              └────┬────────────┘  └──┬────────────┘  └──┬─────────────┘
                   │                  │                  │
        ┌──────────┼──────────────┐   │         ┌────────┼─────────────┐
        │          │              │   │         │        │             │
   ┌────▼───┐ ┌────▼────┐ ┌──────▼─┐ │    ┌────▼──┐ ┌──▼──────┐ ┌────▼────┐
   │ArrayList│ │LinkedList│ │Vector  │ │    │HashSet│ │TreeSet  │ │LinkedHash│
   │         │ │         │ │(legacy)│ │    │       │ │         │ │Set      │
   └─────────┘ └─────────┘ └────────┘ │    └───────┘ └─────────┘ └─────────┘
                                       │
              ┌────────────────────────┼───────────────────┐
              │                        │                   │
         ┌────▼────────┐          ┌────▼─────┐      ┌─────▼──────┐
         │ Map Interface           │ Queue    │      │ Deque      │
         │(key-value pairs)        │          │      │(both ends) │
         └────┬────────┘           └──────────┘      └────────────┘
              │
    ┌─────────┼─────────┬──────────────┐
    │         │         │              │
┌───▼──┐ ┌───▼────┐ ┌──▼───────┐ ┌────▼─────────┐
│HashMap│ │TreeMap │ │LinkedHash│ │ConcurrentHash│
│       │ │        │ │Map       │ │Map           │
└───────┘ └────────┘ └──────────┘ └──────────────┘
```

**Key Insight**: All collections inherit from `Collection` except `Map`, which is a separate hierarchy.

---

## 2. List Implementations

Lists maintain insertion order and allow duplicates.

### ArrayList vs LinkedList Comparison

| Aspect | ArrayList | LinkedList |
|--------|-----------|------------|
| **Internal Structure** | Dynamic resizable array | Doubly-linked list |
| **Random Access** | O(1) - direct index | O(n) - traversal needed |
| **Insert/Delete at end** | O(1) amortized | O(1) |
| **Insert/Delete middle** | O(n) - shift elements | O(n) - but no shift overhead |
| **Memory Overhead** | Minimal | Extra pointers per node |
| **Cache Friendly** | Yes - contiguous memory | No - scattered in memory |
| **Best For** | Read-heavy workloads | Queue/Deque operations |
| **Iteration** | Fast | Moderate (pointer chasing) |

### When to Use

**ArrayList**:
- Default choice for 95% of cases
- Frequent random access (get by index)
- Reading more than writing
- Memory efficiency matters
- Storing primitives (with autoboxing)

**LinkedList**:
- Frequent insertions/deletions at beginning/middle
- Implementing Stack or Queue behavior
- When you never access by index
- Working with Deque interface

### Interview Answer Template

> "ArrayList is backed by a dynamic array with O(1) random access but O(n) insertions in the middle. LinkedList uses a doubly-linked list, making insertions O(1) at known positions but O(n) for accessing elements by index because you must traverse. Use ArrayList by default since most real-world usage is read-heavy. Only use LinkedList if you're implementing a queue-like structure or doing frequent insertions at the beginning."

---

## 3. Set Implementations

Sets contain unique elements; no duplicates allowed.

### HashSet vs TreeSet vs LinkedHashSet

| Aspect | HashSet | TreeSet | LinkedHashSet |
|--------|---------|---------|---------------|
| **Implementation** | Hash table | Red-Black Tree | Hash table + doubly-linked list |
| **Ordering** | No order | Sorted (natural or custom) | Insertion order |
| **Duplicate Handling** | hashCode/equals | Comparator | hashCode/equals |
| **Add/Remove/Contains** | O(1) average | O(log n) | O(1) average |
| **Iteration** | Random order | Sorted order | Insertion order |
| **Memory** | Minimal | Higher (tree overhead) | Moderate (extra pointers) |
| **Thread-Safe** | No | No | No |
| **Use Case** | Fast lookups | Sorted sets, ranges | Maintaining insertion order |

### Why equals() AND hashCode() Matter in HashSet

```java
// If you override equals(), you MUST override hashCode()
// Breaking this contract causes:
// 1. Duplicate objects can exist in the set
// 2. Can't find objects you know are there

class Person {
    String name;

    @Override
    public boolean equals(Object o) {
        return ((Person)o).name.equals(this.name);
    }

    // WRONG: Missing hashCode() override
    // HashSet uses hashCode() to find the bucket, then equals() to find the object
    // Without hashCode(), two equal persons might be in different buckets
}
```

### When to Use Each

**HashSet**:
- You just need "is this element present?" checks
- Performance is critical
- Order doesn't matter

**TreeSet**:
- You need sorted iteration
- Range queries (headSet, tailSet, subSet)
- Custom ordering with Comparator
- Working with NavigableSet interface

**LinkedHashSet**:
- You need insertion order preservation
- Iterating in predictable order
- LRU cache implementation patterns

### Interview Focus

The key differentiator is **ordering requirements**. HashSet is fastest when you don't need order. If you need sorted data, TreeSet's O(log n) is acceptable. LinkedHashSet bridges both with O(1) lookups while preserving insertion order.

---

## 4. Map Implementations

Maps store key-value pairs where each key maps to exactly one value.

### HashMap

**Internal Structure**:
- **Hash Table**: Array of buckets
- **Load Factor**: Default 0.75 (resize when 75% full)
- **Collision Handling**: Chaining with linked lists (or Red-Black trees in Java 8+ when bucket size > 8)
- **Rehashing**: When load factor exceeded, capacity doubles, all entries rehashed

**Time Complexity**:
- Get/Put/Remove: O(1) average case, O(n) worst case (all collisions)
- Iteration: O(n + capacity) - includes empty buckets

**Key Points**:
- Null keys and values allowed (one null key max)
- Not thread-safe - use `Collections.synchronizedMap()` or `ConcurrentHashMap`
- Iteration order undefined
- Resizing is expensive but O(1) amortized per operation

```java
// HashMap internals simplified
class HashMap<K,V> {
    static final int DEFAULT_CAPACITY = 16;
    static final float LOAD_FACTOR = 0.75f;

    Entry<K,V>[] table;  // Array of buckets
    int size;

    // When size > capacity * LOAD_FACTOR, rehash()
}
```

### TreeMap

**Internal Structure**:
- Backed by Red-Black tree (self-balancing BST)
- Maintains sorted order by keys
- Requires Comparable keys or Comparator

**Time Complexity**:
- Get/Put/Remove: O(log n)
- Iteration: O(n) in sorted order
- Range operations: O(log n + result size)

**When to Use**:
```java
// Range queries - only possible with TreeMap
TreeMap<String, Integer> map = new TreeMap<>();
map.put("apple", 1);
map.put("banana", 2);
map.put("cherry", 3);

// Get all entries between "apple" and "cherry"
SortedMap<String, Integer> range = map.subMap("apple", "cherry");

// headMap(), tailMap() for one-sided ranges
SortedMap<String, Integer> before = map.headMap("banana");
```

### ConcurrentHashMap

**Purpose**: Thread-safe HashMap without full synchronization lock

**Key Differences**:
- Uses **segmentation**: Divides hash table into segments (default 16)
- Only locks the segment being modified, not entire map
- Multiple threads can write simultaneously if they use different segments
- Better concurrent write performance than `synchronized HashMap`
- No null keys/values allowed
- Iteration weakly consistent (some updates might not be visible)

**When to Use**:
```java
// High-concurrency scenarios
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Multiple threads writing simultaneously is safe
map.putIfAbsent(key, value);  // Atomic operation
map.computeIfAbsent(key, k -> expensiveComputation(k));
```

### Map Comparison Table

| Aspect | HashMap | TreeMap | ConcurrentHashMap |
|--------|---------|---------|-------------------|
| **Ordering** | None | Sorted keys | None |
| **Put/Get** | O(1) avg | O(log n) | O(1) avg, segment-locked |
| **Range Ops** | Not possible | O(log n) | Not possible |
| **Null Keys** | Yes (one) | No | No |
| **Thread-Safe** | No | No | Yes (partial) |
| **Use When** | Default choice | Need sorted/range | High concurrency writes |

---

## 5. Queue and Deque Basics

### Queue (FIFO)

```
Add → back [ elem1 | elem2 | elem3 ] front → Remove
                   add()              poll()
```

**Common Implementations**:
- **LinkedList**: Generic queue, O(1) add/remove at ends
- **ArrayDeque**: Better than LinkedList for queue/deque (less GC, cache-friendly)
- **PriorityQueue**: Min-heap, elements ordered by priority, O(log n) add/remove

**Usage**:
```java
Queue<String> queue = new LinkedList<>();
queue.add("first");      // Throws if full
queue.offer("second");   // Returns false if full
queue.poll();            // Null if empty
queue.remove();          // Throws if empty
```

### Deque (Double-Ended Queue)

Can add/remove from both ends. Implements both Queue and Stack.

```
Add/Remove from front  →  [ elem1 | elem2 | elem3 ]  ← Add/Remove from back
     push/pop/poll          addFirst/removeLast        add/poll
```

**Best Implementation**: `ArrayDeque` (better than LinkedList)

```java
Deque<Integer> deque = new ArrayDeque<>();
deque.addFirst(1);      // Stack push
deque.removeFirst();    // Stack pop
deque.addLast(2);       // Queue add
deque.removeLast();     // Remove from back
```

---

## 6. Interview Questions & Answers

### Q1: When would you use LinkedList over ArrayList?

**Answer**:
Rarely in production. Only when you:
1. Frequently insert/delete at the beginning (O(1) vs O(n))
2. Implement a Queue/Deque (use ArrayDeque instead, actually)
3. Need stack operations

Even then, `ArrayDeque` is almost always better than LinkedList for queue/deque operations because it's cache-friendly and has less garbage creation.

**Interviewer is testing**: Do you understand algorithmic trade-offs and modern best practices?

---

### Q2: Why do you need to override both hashCode() and equals() for HashSet?

**Answer**:
HashSet uses a two-step lookup:
1. `hashCode()` determines which bucket to check
2. `equals()` checks for exact match within that bucket

If you override equals() without hashCode():
- Two objects that are equal might have different hash codes
- They'll be placed in different buckets
- HashSet will think they're different, breaking the contract

**Example**:
```java
Set<Person> set = new HashSet<>();
Person p1 = new Person("Alice");
Person p2 = new Person("Alice");  // Equal by equals(), different hash codes

set.add(p1);
set.contains(p2);  // false - BUG! Even though p1.equals(p2) is true
```

**Rule**: If `a.equals(b)` then `a.hashCode() == b.hashCode()`.

---

### Q3: HashMap is resizing itself when it reaches load factor 0.75. Why not 1.0?

**Answer**:
At load factor 1.0 (100% full):
- Collision probability increases dramatically (birthday paradox)
- O(1) operations degrade toward O(n)
- Wasted effort trying to find empty buckets

At 0.75:
- Statistical sweet spot between memory usage and collision probability
- ~75% full = ~25% empty buckets to absorb collisions
- Resizing before saturation prevents performance cliff

**Trade-off**: Higher load factor = less memory but more collisions. Lower load factor = more memory but better O(1) guarantees.

---

### Q4: What happens when you iterate over a HashMap and someone else modifies it?

**Answer**:
You get a `ConcurrentModificationException` (CME).

HashMap maintains a `modCount` counter. If the collection is structurally modified (add/remove, not value update) during iteration, the iterator detects this and fails fast.

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);

for (String key : map.keySet()) {
    if (key.equals("a")) {
        map.remove("a");  // ConcurrentModificationException!
    }
}
```

**Fix**: Use iterator's remove() method or create a copy:
```java
// Option 1: Iterator's remove
Iterator<String> it = map.keySet().iterator();
while (it.hasNext()) {
    String key = it.next();
    if (key.equals("a")) {
        it.remove();  // Safe
    }
}

// Option 2: Copy before iteration
new HashMap<>(map).forEach((k, v) -> map.remove(k));
```

---

### Q5: Why is ArrayDeque preferred over LinkedList for Queue operations?

**Answer**:
1. **Cache Locality**: ArrayDeque uses contiguous array; LinkedList scatters nodes
2. **Memory Overhead**: LinkedList needs extra pointers per node; ArrayDeque is just array
3. **Garbage Collection**: ArrayDeque creates less garbage
4. **GC Pressure**: Fewer objects = less GC pause time (critical for latency-sensitive systems)

Both are O(1) for queue operations, but ArrayDeque is faster in practice by 2-3x.

**Banking Context**: In high-frequency trading systems, 2-3x faster queue operations matter.

---

### Q6: TreeMap uses Red-Black trees. When would you use it over HashMap?

**Answer**:
When you need **sorted iteration or range queries**:

```java
TreeMap<String, Account> accounts = new TreeMap<>();
// Add account data...

// Get all accounts with names between "Alice" and "Zoe"
SortedMap<String, Account> range = accounts.subMap("Alice", "Zoe");

// Get first account (customer with alphabetically first name)
Account first = accounts.firstEntry();

// Get all accounts before "Mike"
SortedMap<String, Account> before = accounts.headMap("Mike");
```

**Trade-off**: O(log n) operations vs HashMap's O(1), but you get ordering.

HashMap is faster for random access; TreeMap is needed for sorted traversal.

---

### Q7: Is ConcurrentHashMap truly "thread-safe" for compound operations?

**Answer**:
Yes for **individual operations**, but NOT for **compound operations**.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("count", 0);

// Problem: These are NOT atomic together
if (!map.containsKey("count")) {
    map.put("count", 1);  // Another thread might execute between check and put
}

// Solution 1: Use atomic operations
map.putIfAbsent("count", 1);

// Solution 2: Use compute*() methods (atomic)
map.compute("count", (k, v) -> v == null ? 1 : v + 1);

// Solution 3: External synchronization for compound operations
synchronized(map) {
    if (!map.containsKey("count")) {
        map.put("count", 1);
    }
}
```

**Key Methods for Atomic Compound Operations**:
- `putIfAbsent(key, value)`
- `computeIfAbsent(key, function)`
- `compute(key, function)`
- `merge(key, value, function)`

---

### Q8: How does Java 8+ handle HashSet bucket collisions differently?

**Answer**:
**Before Java 8**: Always used linked lists for collisions:
- O(1) average case turns into O(n) under hash collision attacks
- Potential DoS vulnerability

**Java 8+**: Switches from linked list to Red-Black tree when bucket size > 8:
- Still O(log n) in worst case of bad hash function
- Thresholds: `TREEIFY_THRESHOLD = 8`, `UNTREEIFY_THRESHOLD = 6`
- Single bad hash code won't destroy performance

**Why matters**: Security against hash collision attacks and performance predictability.

---

### Q9: What's the difference between fail-fast and weakly-consistent iterators?

**Answer**:

**Fail-fast** (HashMap, ArrayList):
- Immediately throws `ConcurrentModificationException` if collection modified during iteration
- `modCount` is checked before each iteration
- Strict but safe

```java
List<String> list = new ArrayList<>();
list.add("a");
Iterator<String> it = list.iterator();
list.add("b");  // Structural modification
it.next();      // ConcurrentModificationException!
```

**Weakly-consistent** (ConcurrentHashMap):
- May or may not reflect recent modifications
- Doesn't throw CME
- Trades consistency for performance

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
// Modifications during iteration might not be visible, but won't throw
map.forEach((k, v) -> System.out.println(k + "=" + v));
```

**Interview Point**: ConcurrentHashMap sacrifices iteration consistency for better concurrent write performance. Trade-off between correctness and throughput.

---

### Q10: Why would you use a custom Comparator with TreeSet instead of implementing Comparable?

**Answer**:
1. **Multiple sort orders**: Sort the same class different ways:
```java
TreeSet<Person> byAge = new TreeSet<>((p1, p2) -> p1.age - p2.age);
TreeSet<Person> byName = new TreeSet<>((p1, p2) -> p1.name.compareTo(p2.name));
```

2. **Third-party classes**: Can't modify the class to add Comparable:
```java
TreeSet<Date> dates = new TreeSet<>(Comparator.comparing(Date::getTime));
```

3. **Flexibility**: Change sorting without changing the class definition

4. **Separation of concerns**: Sorting logic separate from domain logic

**Best Practice**: Prefer external Comparator over forcing Comparable into domain objects.

---

### Q11: How would you detect and prevent HashCode collisions in production?

**Answer**:
```java
// Monitor collision rate
Map<Integer, Integer> hashDistribution = new HashMap<>();
for (String item : dataSet) {
    int hash = item.hashCode() % capacity;
    hashDistribution.merge(hash, 1, Integer::sum);
}

// Calculate collision coefficient
int collisions = hashDistribution.values().stream()
    .filter(count -> count > 1)
    .count();
double collisionRate = (double) collisions / hashDistribution.size();

// Alert if collision rate > 5%
if (collisionRate > 0.05) {
    logger.warn("High hash collision rate: " + collisionRate);
}
```

**Prevention**:
- Use good hash functions (Java's built-in usually sufficient)
- Monitor load factor (resize before it exceeds 0.75)
- For security-sensitive: Use LinkedHashMap or TreeMap instead
- For performance: Profile actual data distribution

**Banking Context**: In transaction ID lookups, poor hash function could cause performance degradation under load.

---

### Q12: What's the difference between remove(Object) and remove(int) in ArrayList?

**Answer**:
```java
List<Integer> list = new ArrayList<>();
list.add(5);  // [5]
list.add(10); // [5, 10]
list.add(15); // [5, 10, 15]

list.remove(1);        // Removes at index 1 → [5, 15]
list.remove(Integer.valueOf(10));  // Removes value 10 → [5, 15]
```

If you have `List<Integer>`:
- `remove(1)` calls `remove(int index)` - removes at position 1
- `remove(Integer.valueOf(5))` calls `remove(Object)` - removes first occurrence of value 5

**Trap**: This confuses developers:
```java
List<Integer> list = new ArrayList<>();
list.add(5);
list.remove(5);  // Removes at index 5, NOT the value 5!
```

**Best Practice**: Be explicit with autoboxing:
```java
list.remove(Integer.valueOf(5));  // Clear intent: remove value
list.remove(5);                    // Clear intent: remove index
```

---

## 7. Collections Performance Quick Reference

| Operation | ArrayList | LinkedList | HashSet | TreeSet | HashMap | TreeMap | ConcurrentHashMap |
|-----------|-----------|------------|---------|---------|---------|---------|-------------------|
| Get | O(1) | O(n) | O(1)* | O(log n) | O(1)* | O(log n) | O(1)* |
| Add | O(1)* | O(1)** | O(1)* | O(log n) | O(1)* | O(log n) | O(1)* |
| Remove | O(n) | O(n)*** | O(1)* | O(log n) | O(1)* | O(log n) | O(1)* |
| Contains | O(n) | O(n) | O(1)* | O(log n) | O(1)* | O(log n) | O(1)* |

\* Average case; † Amortized; ‡ At end; ** At known position

---

## 8. Key Takeaways & Interview Checklist

### Must-Know Concepts

1. **Default to ArrayList** - unless you have specific reasons otherwise
2. **HashSet is for fast lookups** - not ordering
3. **TreeSet for sorted data** - accept O(log n) trade-off
4. **HashMap internals** - buckets, load factor, collision handling
5. **equals() AND hashCode()** - they're a package deal
6. **ConcurrentHashMap for concurrency** - not full thread-safety
7. **ArrayDeque over LinkedList** - for queue/deque operations
8. **Fail-fast vs weakly-consistent** - different iteration guarantees

### Red Flags If You Say...

- ❌ "Use LinkedList for everything" - Shows poor understanding
- ❌ "HashMap iteration order is unpredictable" - True, but missing the WHY
- ❌ "ConcurrentHashMap is fully thread-safe" - Wrong; only individual ops
- ❌ "TreeSet uses equals() for ordering" - Wrong; uses Comparable/Comparator
- ❌ "Just synchronize the collection" - Doesn't understand ConcurrentHashMap

### What Interviewers Want to Hear

- Understand **trade-offs** between collections
- Recognize **use cases** that require specific collections
- Know **time complexity** for each operation
- Grasp **concurrency implications** (thread-safety)
- Consider **real-world scenarios** (banking, high-frequency, etc.)

### Code Examples to Practice

```java
// 1. Building frequency map (HashMap use case)
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.put(word, freq.getOrDefault(word, 0) + 1);
}

// 2. Maintaining sorted unique values (TreeSet use case)
Set<Integer> unique = new TreeSet<>(values);

// 3. Checking duplicates (HashSet use case)
Set<String> seen = new HashSet<>();
for (String item : items) {
    if (!seen.add(item)) {
        System.out.println("Duplicate: " + item);
    }
}

// 4. LRU Cache (LinkedHashMap use case)
LinkedHashMap<String, String> lru = new LinkedHashMap<String, String>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > CAPACITY;
    }
};

// 5. Thread-safe counter (ConcurrentHashMap use case)
ConcurrentHashMap<String, Integer> counters = new ConcurrentHashMap<>();
counters.computeIfAbsent("requests", k -> 0);
counters.compute("requests", (k, v) -> v + 1);
```

---

## 9. Quick Decision Tree

```
Need to store unique values?
├─ Yes → Use Set
│   ├─ Need sorted order? → TreeSet (O(log n) ops)
│   ├─ Need insertion order? → LinkedHashSet (O(1) ops)
│   └─ Just need uniqueness? → HashSet (O(1) ops, fastest)
└─ No → Use List or Map

    Need key-value pairs?
    ├─ Yes → Use Map
    │   ├─ Need sorted keys? → TreeMap
    │   ├─ High concurrency writes? → ConcurrentHashMap
    │   └─ Default case? → HashMap
    └─ No → Use List
        ├─ Frequent random access? → ArrayList (default)
        ├─ Frequent insertions at start? → LinkedList (rare)
        └─ Queue/Deque behavior? → ArrayDeque (best)
```

---

## Summary

The Java Collections Framework is about **choosing the right tool for the job**. In interviews, you don't need to memorize APIs—you need to demonstrate:

1. **Understanding of data structures**: Linked lists, hash tables, trees
2. **Performance awareness**: Time complexity for common operations
3. **Real-world judgment**: When trade-offs make sense
4. **Thread-safety knowledge**: When synchronization matters
5. **Clean code practices**: Using collections idiomatically

Most importantly: **ArrayList and HashMap solve 80% of real-world problems**. Know them deeply, understand the alternatives, and be able to justify when you'd deviate from the defaults.

In your interview: "I'd start with ArrayList/HashMap because they're optimized for most use cases. But if I needed sorted order, I'd use TreeSet/TreeMap. If concurrency was critical, I'd use ConcurrentHashMap. And I'd only use LinkedList if..."—you can explain the specific reason.

That's senior-level thinking about collections.
