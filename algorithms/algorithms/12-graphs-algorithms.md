# Graphs: Algorithms

## Overview
Beyond basic traversal, these algorithms solve specific graph problems like shortest paths, connectivity, and ordering.

## Key Algorithms

### 1. Topological Sort (Kahn's Algorithm)
*   **Problem**: Order tasks with dependencies (Course Schedule).
*   **Graph Type**: Directed Acyclic Graph (DAG).
*   **Method**: BFS using "Indegree".
*   **Complexity**: O(V + E).

### 2. Dijkstra's Algorithm
*   **Problem**: Shortest path in **weighted** graph (non-negative).
*   **Method**: BFS + Priority Queue (Greedy).
*   **Complexity**: O(E log V).

### 3. Union-Find (Disjoint Set)
*   **Problem**: Connected components, Cycle detection in undirected graph.
*   **Operations**: `find(x)`, `union(x, y)`.
*   **Complexity**: O(α(N)) (Inverse Ackermann function - nearly constant).

## Interview Problems

### Problem 1: Course Schedule (Medium)
**Pattern**: Topological Sort (Cycle Detection)

```java
/**
 * Can you finish all courses given prerequisites?
 * Time: O(V + E)
 * Space: O(V + E)
 */
public boolean canFinish(int numCourses, int[][] prerequisites) {
    int[] indegree = new int[numCourses];
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());
    
    for (int[] pr : prerequisites) {
        adj.get(pr[1]).add(pr[0]);
        indegree[pr[0]]++;
    }
    
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (indegree[i] == 0) queue.offer(i);
    }
    
    int count = 0;
    while (!queue.isEmpty()) {
        int curr = queue.poll();
        count++;
        
        for (int neighbor : adj.get(curr)) {
            indegree[neighbor]--;
            if (indegree[neighbor] == 0) queue.offer(neighbor);
        }
    }
    
    return count == numCourses;
}
```

### Problem 2: Number of Islands (Medium)
**Pattern**: DFS / BFS (Connected Components)

```java
/**
 * Count number of islands ('1') surrounded by water ('0').
 * Time: O(M * N)
 * Space: O(M * N)
 */
public int numIslands(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;
    
    int count = 0;
    int m = grid.length;
    int n = grid[0].length;
    
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1') {
                count++;
                dfs(grid, i, j);
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] == '0') {
        return;
    }
    
    grid[i][j] = '0'; // Mark as visited (sink the island)
    
    dfs(grid, i+1, j);
    dfs(grid, i-1, j);
    dfs(grid, i, j+1);
    dfs(grid, i, j-1);
}
```

### Problem 3: Network Delay Time (Medium)
**Pattern**: Dijkstra's Algorithm

```java
/**
 * Find time for signal to reach all nodes.
 * Time: O(E log V)
 */
public int networkDelayTime(int[][] times, int n, int k) {
    Map<Integer, List<int[]>> graph = new HashMap<>();
    for (int[] t : times) {
        graph.computeIfAbsent(t[0], x -> new ArrayList<>()).add(new int[]{t[1], t[2]});
    }
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{k, 0});
    
    Map<Integer, Integer> dist = new HashMap<>();
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int u = curr[0];
        int d = curr[1];
        
        if (dist.containsKey(u)) continue;
        dist.put(u, d);
        
        if (graph.containsKey(u)) {
            for (int[] edge : graph.get(u)) {
                int v = edge[0];
                int w = edge[1];
                if (!dist.containsKey(v)) {
                    pq.offer(new int[]{v, d + w});
                }
            }
        }
    }
    
    if (dist.size() != n) return -1;
    
    int maxDist = 0;
    for (int d : dist.values()) maxDist = Math.max(maxDist, d);
    return maxDist;
}
```

## 🏦 Banking Context: Settlement Systems
*   **Scenario**: Inter-bank settlements.
*   **Problem**: Bank A owes B, B owes C, C owes A.
*   **Algorithm**: **Max Flow** or **Graph Simplification** to net off the debts so fewer physical transfers are needed.

## Common Pitfalls
1.  **Dijkstra with Negative Weights**: Dijkstra fails. Use Bellman-Ford.
2.  **Union-Find Complexity**: Without "Path Compression" and "Union by Rank", it degrades to O(N). Always implement both optimizations.

---
**Next**: [Dynamic Programming: Introduction](13-dynamic-programming-intro.md)
