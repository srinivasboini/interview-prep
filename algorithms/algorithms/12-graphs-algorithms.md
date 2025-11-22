# Graphs: Advanced Algorithms - Complete Master Guide

## Overview
Advanced graph algorithms solve complex problems like shortest paths, minimum spanning trees, network flow, and strongly connected components. These algorithms are fundamental to routing, network design, and optimization problems.

**Key Insight**: These algorithms power GPS navigation, network routing, resource allocation, and financial systems.

For Senior/Staff Engineers, mastering graph algorithms means:
- Implementing Dijkstra's, Bellman-Ford, and Floyd-Warshall
- Understanding Union-Find with path compression
- Recognizing when to use each algorithm
- Discussing production trade-offs and optimizations

---

## Table of Contents
1. [Shortest Path Algorithms](#shortest-path-algorithms)
2. [Minimum Spanning Tree](#minimum-spanning-tree)
3. [Union-Find](#union-find)
4. [Advanced Patterns](#advanced-patterns)
5. [15+ Solved Problems](#solved-problems)
6. [Interview Questions & Answers](#interview-questions--answers)
7. [Banking & Production Context](#banking--production-context)

---

## Shortest Path Algorithms

### Dijkstra's Algorithm

**Problem**: Shortest path from source to all vertices (non-negative weights)

**Algorithm**: Greedy + Priority Queue

**Time**: O(E log V), **Space**: O(V)

```java
/**
 * Dijkstra's algorithm for shortest paths.
 * Time: O(E log V), Space: O(V)
 */
public int[] dijkstra(List<int[]>[] graph, int source) {
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[source] = 0;
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{source, 0});
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int u = curr[0];
        int d = curr[1];
        
        if (d > dist[u]) continue;  // Already processed
        
        for (int[] edge : graph[u]) {
            int v = edge[0];
            int weight = edge[1];
            
            if (dist[u] + weight < dist[v]) {
                dist[v] = dist[u] + weight;
                pq.offer(new int[]{v, dist[v]});
            }
        }
    }
    
    return dist;
}
```

### Bellman-Ford Algorithm

**Problem**: Shortest path with negative weights (detects negative cycles)

**Algorithm**: Relax all edges V-1 times

**Time**: O(V × E), **Space**: O(V)

```java
/**
 * Bellman-Ford algorithm.
 * Time: O(V × E), Space: O(V)
 */
public int[] bellmanFord(int[][] edges, int n, int source) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[source] = 0;
    
    // Relax edges V-1 times
    for (int i = 0; i < n - 1; i++) {
        for (int[] edge : edges) {
            int u = edge[0], v = edge[1], weight = edge[2];
            if (dist[u] != Integer.MAX_VALUE && dist[u] + weight < dist[v]) {
                dist[v] = dist[u] + weight;
            }
        }
    }
    
    // Check for negative cycles
    for (int[] edge : edges) {
        int u = edge[0], v = edge[1], weight = edge[2];
        if (dist[u] != Integer.MAX_VALUE && dist[u] + weight < dist[v]) {
            throw new IllegalArgumentException("Negative cycle detected");
        }
    }
    
    return dist;
}
```

### Floyd-Warshall Algorithm

**Problem**: All-pairs shortest paths

**Algorithm**: Dynamic programming

**Time**: O(V³), **Space**: O(V²)

```java
/**
 * Floyd-Warshall algorithm.
 * Time: O(V³), Space: O(V²)
 */
public int[][] floydWarshall(int[][] graph) {
    int n = graph.length;
    int[][] dist = new int[n][n];
    
    // Initialize
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            dist[i][j] = graph[i][j];
        }
    }
    
    // Try all intermediate vertices
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i][k] != Integer.MAX_VALUE && 
                    dist[k][j] != Integer.MAX_VALUE &&
                    dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }
    
    return dist;
}
```

---

## Minimum Spanning Tree

### Kruskal's Algorithm

**Problem**: Find MST using Union-Find

**Algorithm**: Sort edges, add if doesn't create cycle

**Time**: O(E log E), **Space**: O(V)

```java
/**
 * Kruskal's algorithm for MST.
 * Time: O(E log E), Space: O(V)
 */
public int kruskal(int n, int[][] edges) {
    Arrays.sort(edges, (a, b) -> a[2] - b[2]);  // Sort by weight
    
    UnionFind uf = new UnionFind(n);
    int mstCost = 0;
    int edgesUsed = 0;
    
    for (int[] edge : edges) {
        int u = edge[0], v = edge[1], weight = edge[2];
        
        if (uf.union(u, v)) {
            mstCost += weight;
            edgesUsed++;
            
            if (edgesUsed == n - 1) break;  // MST complete
        }
    }
    
    return edgesUsed == n - 1 ? mstCost : -1;
}
```

### Prim's Algorithm

**Problem**: Find MST using Priority Queue

**Algorithm**: Grow MST from source

**Time**: O(E log V), **Space**: O(V)

```java
/**
 * Prim's algorithm for MST.
 * Time: O(E log V), Space: O(V)
 */
public int prim(List<int[]>[] graph) {
    int n = graph.length;
    boolean[] inMST = new boolean[n];
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    
    pq.offer(new int[]{0, 0});  // {vertex, weight}
    int mstCost = 0;
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int u = curr[0];
        int weight = curr[1];
        
        if (inMST[u]) continue;
        
        inMST[u] = true;
        mstCost += weight;
        
        for (int[] edge : graph[u]) {
            int v = edge[0];
            int w = edge[1];
            if (!inMST[v]) {
                pq.offer(new int[]{v, w});
            }
        }
    }
    
    return mstCost;
}
```

---

## Union-Find

### Implementation with Optimizations

```java
/**
 * Union-Find with path compression and union by rank.
 * Time: O(α(n)) per operation (nearly constant)
 */
class UnionFind {
    private int[] parent;
    private int[] rank;
    private int components;
    
    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        components = n;
        
        for (int i = 0; i < n; i++) {
            parent[i] = i;
        }
    }
    
    public int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);  // Path compression
        }
        return parent[x];
    }
    
    public boolean union(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        
        if (rootX == rootY) return false;  // Already connected
        
        // Union by rank
        if (rank[rootX] < rank[rootY]) {
            parent[rootX] = rootY;
        } else if (rank[rootX] > rank[rootY]) {
            parent[rootY] = rootX;
        } else {
            parent[rootY] = rootX;
            rank[rootX]++;
        }
        
        components--;
        return true;
    }
    
    public boolean connected(int x, int y) {
        return find(x) == find(y);
    }
    
    public int getComponents() {
        return components;
    }
}
```

---

## Solved Problems

### Problem 1: Network Delay Time (Medium)

```java
/**
 * Dijkstra's algorithm application.
 * Time: O(E log V), Space: O(V + E)
 */
public int networkDelayTime(int[][] times, int n, int k) {
    List<int[]>[] graph = new ArrayList[n + 1];
    for (int i = 1; i <= n; i++) {
        graph[i] = new ArrayList<>();
    }
    
    for (int[] time : times) {
        graph[time[0]].add(new int[]{time[1], time[2]});
    }
    
    int[] dist = new int[n + 1];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[k] = 0;
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{k, 0});
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int u = curr[0];
        int d = curr[1];
        
        if (d > dist[u]) continue;
        
        for (int[] edge : graph[u]) {
            int v = edge[0];
            int weight = edge[1];
            
            if (dist[u] + weight < dist[v]) {
                dist[v] = dist[u] + weight;
                pq.offer(new int[]{v, dist[v]});
            }
        }
    }
    
    int maxDist = 0;
    for (int i = 1; i <= n; i++) {
        if (dist[i] == Integer.MAX_VALUE) return -1;
        maxDist = Math.max(maxDist, dist[i]);
    }
    
    return maxDist;
}
```

### Problem 2: Cheapest Flights Within K Stops (Medium)

```java
/**
 * Modified Dijkstra with stop limit.
 * Time: O(E log V), Space: O(V)
 */
public int findCheapestPrice(int n, int[][] flights, int src, int dst, int k) {
    List<int[]>[] graph = new ArrayList[n];
    for (int i = 0; i < n; i++) {
        graph[i] = new ArrayList<>();
    }
    
    for (int[] flight : flights) {
        graph[flight[0]].add(new int[]{flight[1], flight[2]});
    }
    
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{src, 0, 0});  // {node, cost, stops}
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int u = curr[0];
        int cost = curr[1];
        int stops = curr[2];
        
        if (u == dst) return cost;
        if (stops > k) continue;
        
        for (int[] edge : graph[u]) {
            int v = edge[0];
            int price = edge[1];
            
            if (cost + price < dist[v]) {
                dist[v] = cost + price;
                pq.offer(new int[]{v, cost + price, stops + 1});
            }
        }
    }
    
    return -1;
}
```

### Problem 3: Number of Connected Components (Medium)

```java
/**
 * Union-Find application.
 * Time: O(E × α(V)), Space: O(V)
 */
public int countComponents(int n, int[][] edges) {
    UnionFind uf = new UnionFind(n);
    
    for (int[] edge : edges) {
        uf.union(edge[0], edge[1]);
    }
    
    return uf.getComponents();
}
```

### Problem 4: Min Cost to Connect All Points (Medium)

```java
/**
 * MST using Kruskal's algorithm.
 * Time: O(n² log n), Space: O(n²)
 */
public int minCostConnectPoints(int[][] points) {
    int n = points.length;
    List<int[]> edges = new ArrayList<>();
    
    // Generate all edges
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            int dist = Math.abs(points[i][0] - points[j][0]) + 
                       Math.abs(points[i][1] - points[j][1]);
            edges.add(new int[]{i, j, dist});
        }
    }
    
    edges.sort((a, b) -> a[2] - b[2]);
    
    UnionFind uf = new UnionFind(n);
    int mstCost = 0;
    
    for (int[] edge : edges) {
        if (uf.union(edge[0], edge[1])) {
            mstCost += edge[2];
        }
    }
    
    return mstCost;
}
```

### Problem 5: Path with Maximum Probability (Medium)

```java
/**
 * Modified Dijkstra for maximum probability.
 * Time: O(E log V), Space: O(V + E)
 */
public double maxProbability(int n, int[][] edges, double[] succProb, int start, int end) {
    List<double[]>[] graph = new ArrayList[n];
    for (int i = 0; i < n; i++) {
        graph[i] = new ArrayList<>();
    }
    
    for (int i = 0; i < edges.length; i++) {
        int u = edges[i][0], v = edges[i][1];
        double prob = succProb[i];
        graph[u].add(new double[]{v, prob});
        graph[v].add(new double[]{u, prob});
    }
    
    double[] maxProb = new double[n];
    maxProb[start] = 1.0;
    
    PriorityQueue<double[]> pq = new PriorityQueue<>((a, b) -> 
        Double.compare(b[1], a[1]));  // Max heap
    pq.offer(new double[]{start, 1.0});
    
    while (!pq.isEmpty()) {
        double[] curr = pq.poll();
        int u = (int) curr[0];
        double prob = curr[1];
        
        if (u == end) return prob;
        if (prob < maxProb[u]) continue;
        
        for (double[] edge : graph[u]) {
            int v = (int) edge[0];
            double edgeProb = edge[1];
            
            if (prob * edgeProb > maxProb[v]) {
                maxProb[v] = prob * edgeProb;
                pq.offer(new double[]{v, maxProb[v]});
            }
        }
    }
    
    return 0.0;
}
```

---

## Interview Questions & Answers

### Q1: "When should you use Dijkstra vs Bellman-Ford vs Floyd-Warshall?"

**Model Answer:**
"I choose based on graph properties and requirements:

**Dijkstra's Algorithm**:
- **When**: Single-source shortest path, non-negative weights
- **Time**: O(E log V)
- **Use case**: GPS navigation, network routing

**Bellman-Ford**:
- **When**: Single-source, negative weights allowed, detect negative cycles
- **Time**: O(V × E)
- **Use case**: Currency arbitrage detection, financial modeling

**Floyd-Warshall**:
- **When**: All-pairs shortest paths, small graphs (V ≤ 400)
- **Time**: O(V³)
- **Use case**: Distance matrices, transitive closure

**Example - Banking**:
For payment routing between banks:
- Use Dijkstra if costs are always positive (transaction fees)
- Use Bellman-Ford if rebates exist (negative weights)
- Use Floyd-Warshall for small networks to precompute all routes

**Key trade-off**: Dijkstra is fastest but limited to non-negative weights. Bellman-Ford handles negative weights but slower. Floyd-Warshall computes all pairs but O(V³)."

### Q2: "Explain Union-Find optimizations and their impact."

**Model Answer:**
"Union-Find has two critical optimizations:

**1. Path Compression** (in find):
```java
public int find(int x) {
    if (parent[x] != x) {
        parent[x] = find(parent[x]);  // Flatten tree
    }
    return parent[x];
}
```
- Makes tree very flat
- Subsequent finds are nearly O(1)

**2. Union by Rank** (in union):
```java
if (rank[rootX] < rank[rootY]) {
    parent[rootX] = rootY;  // Attach smaller to larger
}
```
- Keeps tree balanced
- Prevents degeneration to linked list

**Without optimizations**: O(n) per operation
**With both**: O(α(n)) ≈ O(1) where α is inverse Ackermann

**Impact**:
For n=10⁶ operations:
- Without: 10⁶ × 10⁶ = 10¹² operations
- With: 10⁶ × 4 ≈ 4×10⁶ operations

**Production example**:
In network connectivity (Kruskal's MST), processing 1M edges:
- Without optimizations: Minutes
- With optimizations: Milliseconds

Both optimizations are essential—using only one gives O(log n), not O(α(n))."

### Q3: "How does Dijkstra's algorithm work and why does it fail with negative weights?"

**Model Answer:**
"Dijkstra's is a greedy algorithm using a priority queue:

**Algorithm**:
1. Start with source at distance 0
2. Always process nearest unvisited vertex
3. Relax all neighbors
4. Mark as visited (never revisit)

**Why it works** (non-negative weights):
- Once a vertex is processed, we've found its shortest path
- No future path can be shorter (all weights ≥ 0)

**Why it fails** (negative weights):
```
Example:
A --5--> B
A --2--> C --(-10)--> B

Dijkstra processes:
1. A (dist=0)
2. C (dist=2) 
3. B (dist=5) ✗ Wrong! Should be 2+(-10)=-8

Problem: B marked as visited at dist=5, 
never reconsidered when better path found via C.
```

**Solution for negative weights**: Use Bellman-Ford
- Relaxes all edges V-1 times
- Allows revisiting vertices
- Time: O(V×E) vs Dijkstra's O(E log V)

**Production context**:
In payment systems, Dijkstra works for routing (fees always positive). For arbitrage detection (negative cycles in currency exchange), we use Bellman-Ford."

---

## 🏦 Banking & Production Context

### Payment Routing Optimization

**Scenario**: Find cheapest route for international payments.

```java
/**
 * Find cheapest payment route considering fees and FX rates.
 */
class PaymentRouter {
    public double findCheapestRoute(int[][] routes, double[] fees, 
                                     String fromCurrency, String toCurrency) {
        // Build graph: nodes = currencies, edges = conversion routes
        Map<String, List<Edge>> graph = new HashMap<>();
        
        for (int i = 0; i < routes.length; i++) {
            String from = getCurrency(routes[i][0]);
            String to = getCurrency(routes[i][1]);
            double cost = fees[i];
            
            graph.computeIfAbsent(from, k -> new ArrayList<>())
                 .add(new Edge(to, cost));
        }
        
        // Dijkstra's algorithm
        Map<String, Double> minCost = new HashMap<>();
        PriorityQueue<State> pq = new PriorityQueue<>(
            (a, b) -> Double.compare(a.cost, b.cost)
        );
        
        pq.offer(new State(fromCurrency, 0.0));
        
        while (!pq.isEmpty()) {
            State curr = pq.poll();
            
            if (curr.currency.equals(toCurrency)) {
                return curr.cost;
            }
            
            if (minCost.containsKey(curr.currency)) continue;
            minCost.put(curr.currency, curr.cost);
            
            for (Edge edge : graph.getOrDefault(curr.currency, new ArrayList<>())) {
                if (!minCost.containsKey(edge.to)) {
                    pq.offer(new State(edge.to, curr.cost + edge.cost));
                }
            }
        }
        
        return -1;  // No route found
    }
}
```

### Network Redundancy Analysis

**Scenario**: Ensure banking network has no single point of failure.

```java
/**
 * Find critical connections (bridges) in network.
 */
class NetworkAnalyzer {
    private int time = 0;
    
    public List<List<Integer>> criticalConnections(int n, List<List<Integer>> connections) {
        List<Integer>[] graph = new ArrayList[n];
        for (int i = 0; i < n; i++) {
            graph[i] = new ArrayList<>();
        }
        
        for (List<Integer> conn : connections) {
            graph[conn.get(0)].add(conn.get(1));
            graph[conn.get(1)].add(conn.get(0));
        }
        
        int[] disc = new int[n];
        int[] low = new int[n];
        Arrays.fill(disc, -1);
        
        List<List<Integer>> bridges = new ArrayList<>();
        
        dfs(0, -1, disc, low, graph, bridges);
        
        return bridges;
    }
    
    private void dfs(int u, int parent, int[] disc, int[] low,
                     List<Integer>[] graph, List<List<Integer>> bridges) {
        disc[u] = low[u] = time++;
        
        for (int v : graph[u]) {
            if (v == parent) continue;
            
            if (disc[v] == -1) {
                dfs(v, u, disc, low, graph, bridges);
                low[u] = Math.min(low[u], low[v]);
                
                if (low[v] > disc[u]) {
                    bridges.add(Arrays.asList(u, v));
                }
            } else {
                low[u] = Math.min(low[u], disc[v]);
            }
        }
    }
}
```

---

## Key Takeaways

1. **Dijkstra**: Single-source shortest path, non-negative weights, O(E log V)
2. **Bellman-Ford**: Handles negative weights, detects cycles, O(V × E)
3. **Floyd-Warshall**: All-pairs shortest paths, O(V³)
4. **Union-Find**: Path compression + union by rank = O(α(n))
5. **Kruskal's MST**: Sort edges, use Union-Find, O(E log E)
6. **Prim's MST**: Grow from source, use priority queue, O(E log V)
7. **Production**: Payment routing, network analysis, fraud detection

---

**Next**: [Dynamic Programming: Introduction](13-dynamic-programming-intro.md)
