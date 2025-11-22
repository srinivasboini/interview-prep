# Searching Algorithms - Complete Master Guide

## Overview
Searching is the process of finding a specific element or determining its presence in a data structure. The choice of searching algorithm depends on whether the data is sorted, the data structure type, and performance requirements.

**Key Insight**: Use binary search on sorted data for O(log n), hash tables for O(1) average case, or BFS/DFS for graphs/trees.

For Senior/Staff Engineers, mastering searching means:
- Understanding linear vs binary search trade-offs
- Implementing binary search correctly (avoiding off-by-one errors)
- Knowing when to use BFS vs DFS
- Discussing production applications (database indexing, caching, search engines)

---

## Table of Contents
1. [Linear Search](#linear-search)
2. [Binary Search](#binary-search)
3. [Graph/Tree Search](#graphtree-search)
4. [Advanced Search](#advanced-search)
5. [10+ Solved Problems](#solved-problems)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Linear Search

### Implementation

```java
/**
 * Linear search - check each element.
 * Time: O(n), Space: O(1)
 */
public int linearSearch(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == target) {
            return i;
        }
    }
    return -1;
}
```

**When to use**:
- Unsorted data
- Small datasets (n < 100)
- Single search operation

---

## Binary Search

### Standard Binary Search

```java
/**
 * Binary search on sorted array.
 * Time: O(log n), Space: O(1)
 */
public int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return -1;
}
```

### Binary Search Variants

**Find first occurrence**:
```java
public int findFirst(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    int result = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            right = mid - 1;  // Continue searching left
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return result;
}
```

**Find last occurrence**:
```java
public int findLast(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    int result = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            left = mid + 1;  // Continue searching right
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return result;
}
```

---

## Graph/Tree Search

### Breadth-First Search (BFS)

```java
/**
 * BFS for graphs.
 * Time: O(V + E), Space: O(V)
 */
public void bfs(int start, List<List<Integer>> graph) {
    boolean[] visited = new boolean[graph.size()];
    Queue<Integer> queue = new LinkedList<>();
    
    queue.offer(start);
    visited[start] = true;
    
    while (!queue.isEmpty()) {
        int node = queue.poll();
        System.out.println(node);
        
        for (int neighbor : graph.get(node)) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue.offer(neighbor);
            }
        }
    }
}
```

### Depth-First Search (DFS)

```java
/**
 * DFS for graphs.
 * Time: O(V + E), Space: O(V)
 */
public void dfs(int node, List<List<Integer>> graph, boolean[] visited) {
    visited[node] = true;
    System.out.println(node);
    
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) {
            dfs(neighbor, graph, visited);
        }
    }
}
```

---

## Advanced Search

### Exponential Search

```java
/**
 * Exponential search - find range then binary search.
 * Time: O(log n), Space: O(1)
 */
public int exponentialSearch(int[] arr, int target) {
    if (arr[0] == target) return 0;
    
    int i = 1;
    while (i < arr.length && arr[i] <= target) {
        i *= 2;
    }
    
    return binarySearch(arr, target, i / 2, Math.min(i, arr.length - 1));
}

private int binarySearch(int[] arr, int target, int left, int right) {
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    
    return -1;
}
```

### Interpolation Search

```java
/**
 * Interpolation search - better for uniformly distributed data.
 * Time: O(log log n) average, O(n) worst, Space: O(1)
 */
public int interpolationSearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    
    while (left <= right && target >= arr[left] && target <= arr[right]) {
        if (left == right) {
            return arr[left] == target ? left : -1;
        }
        
        int pos = left + ((target - arr[left]) * (right - left)) / 
                        (arr[right] - arr[left]);
        
        if (arr[pos] == target) {
            return pos;
        } else if (arr[pos] < target) {
            left = pos + 1;
        } else {
            right = pos - 1;
        }
    }
    
    return -1;
}
```

---

## Solved Problems

### Problem 1: Search in Rotated Sorted Array (Medium)

```java
/**
 * Binary search in rotated sorted array.
 * Time: O(log n), Space: O(1)
 */
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (nums[mid] == target) {
            return mid;
        }
        
        if (nums[left] <= nums[mid]) {
            // Left half is sorted
            if (nums[left] <= target && target < nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        } else {
            // Right half is sorted
            if (nums[mid] < target && target <= nums[right]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    
    return -1;
}
```

### Problem 2: Find Peak Element (Medium)

```java
/**
 * Find peak element using binary search.
 * Time: O(log n), Space: O(1)
 */
public int findPeakElement(int[] nums) {
    int left = 0, right = nums.length - 1;
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (nums[mid] > nums[mid + 1]) {
            right = mid;
        } else {
            left = mid + 1;
        }
    }
    
    return left;
}
```

### Problem 3: Search a 2D Matrix (Medium)

```java
/**
 * Search in sorted 2D matrix.
 * Time: O(log(m×n)), Space: O(1)
 */
public boolean searchMatrix(int[][] matrix, int target) {
    int m = matrix.length, n = matrix[0].length;
    int left = 0, right = m * n - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        int midValue = matrix[mid / n][mid % n];
        
        if (midValue == target) {
            return true;
        } else if (midValue < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return false;
}
```

---

## Interview Questions & Answers

### Q1: "Why is binary search O(log n)?"

**Model Answer:**
"Binary search is O(log n) because it halves the search space with each comparison:

**Analysis**:
```
Initial size: n
After 1 comparison: n/2
After 2 comparisons: n/4
After 3 comparisons: n/8
...
After k comparisons: n/2^k

When n/2^k = 1:
2^k = n
k = log₂(n)
```

**Example**: n = 1,000,000
```
log₂(1,000,000) ≈ 20 comparisons
vs linear search: 1,000,000 comparisons (worst case)
```

**Production impact**:
In database indexing (B-tree):
- 1 billion records
- Binary search: ~30 disk reads
- Linear search: 1 billion disk reads

This is why databases use B-tree indexes for O(log n) lookups."

### Q2: "When should you use BFS vs DFS?"

**Model Answer:**
"I choose based on the problem requirements:

**Use BFS when**:
- Finding shortest path (unweighted graph)
- Level-order traversal needed
- Exploring nearby nodes first
- Example: Social network (friends of friends)

**Use DFS when**:
- Exploring all paths
- Detecting cycles
- Topological sorting
- Memory constrained (O(h) vs O(w))
- Example: Maze solving, dependency resolution

**Comparison**:
| Aspect | BFS | DFS |
|--------|-----|-----|
| Data structure | Queue | Stack/Recursion |
| Space | O(w) width | O(h) height |
| Shortest path | ✓ | ✗ |
| All paths | ✗ | ✓ |
| Cycle detection | ✓ | ✓ |

**Production example**:
In banking:
- BFS: Find shortest transaction path between accounts
- DFS: Detect circular dependencies in payment chains"

### Q3: "What are common binary search pitfalls?"

**Model Answer:**
"Common binary search mistakes:

**1. Integer overflow**:
```java
// Wrong
int mid = (left + right) / 2;  // Overflow if left + right > MAX_INT

// Correct
int mid = left + (right - left) / 2;
```

**2. Infinite loop**:
```java
// Wrong
while (left < right) {
    int mid = (left + right) / 2;
    if (arr[mid] < target) {
        left = mid;  // Infinite loop if left = mid
    }
}

// Correct
while (left < right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] < target) {
        left = mid + 1;  // Always make progress
    }
}
```

**3. Off-by-one errors**:
```java
// Finding first occurrence
while (left <= right) {  // <= not <
    if (arr[mid] == target) {
        result = mid;
        right = mid - 1;  // Continue left, not mid
    }
}
```

**4. Wrong boundary conditions**:
```java
// Wrong
int left = 0, right = arr.length;  // right should be length - 1

// Correct
int left = 0, right = arr.length - 1;
```

**Best practice**: Test with:
- Empty array
- Single element
- Two elements
- Target at boundaries
- Target not present"

---

## 🏦 Banking & Production Context

### Customer Search System

**Scenario**: Multi-tier search for customer lookup.

```java
/**
 * Customer search with caching and database fallback.
 */
class CustomerSearchSystem {
    private Map<String, Customer> cache = new ConcurrentHashMap<>();
    private DatabaseIndex dbIndex;
    
    public Customer searchCustomer(String customerId) {
        // Tier 1: Cache (O(1))
        Customer customer = cache.get(customerId);
        if (customer != null) {
            return customer;
        }
        
        // Tier 2: Database B-tree index (O(log n))
        customer = dbIndex.search(customerId);
        if (customer != null) {
            cache.put(customerId, customer);
            return customer;
        }
        
        return null;
    }
    
    public List<Customer> fuzzySearch(String query) {
        // Tier 3: Elasticsearch for fuzzy matching
        return elasticsearchClient.search(query);
    }
}

/**
 * B-tree index simulation for database.
 */
class DatabaseIndex {
    private TreeMap<String, Customer> index = new TreeMap<>();
    
    public Customer search(String id) {
        return index.get(id);  // O(log n) Red-Black tree
    }
    
    public List<Customer> rangeSearch(String start, String end) {
        return new ArrayList<>(index.subMap(start, end).values());
    }
}
```

---

## Key Takeaways

1. **Linear Search**: O(n), works on unsorted data
2. **Binary Search**: O(log n), requires sorted data
3. **BFS**: Queue, shortest path, O(V+E)
4. **DFS**: Stack/recursion, all paths, O(V+E)
5. **Pitfalls**: Integer overflow, off-by-one, infinite loops
6. **Production**: Database B-tree indexes, caching, search engines
7. **Trade-offs**: Time vs space, sorted vs unsorted, exact vs fuzzy

---

**Next**: [Pattern: Sliding Window](26-pattern-sliding-window.md)
