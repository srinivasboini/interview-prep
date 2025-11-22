# Pattern: Tree DFS - Complete Guide

## Overview
Depth-first search using recursion or stack for tree traversal.

**Time**: O(n), **Space**: O(h) where h is height

---

## Template
```java
public void dfs(TreeNode root) {
    if (root == null) return;
    
    // Preorder: process root first
    System.out.println(root.val);
    dfs(root.left);
    dfs(root.right);
}
```

---

## Problems

### 1. Path Sum (Easy)
```java
public boolean hasPathSum(TreeNode root, int targetSum) {
    if (root == null) return false;
    if (root.left == null && root.right == null) {
        return root.val == targetSum;
    }
    
    return hasPathSum(root.left, targetSum - root.val) ||
           hasPathSum(root.right, targetSum - root.val);
}
```

### 2. All Paths (Medium)
```java
public List<List<Integer>> allPaths(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    dfs(root, new ArrayList<>(), result);
    return result;
}

private void dfs(TreeNode node, List<Integer> path, List<List<Integer>> result) {
    if (node == null) return;
    
    path.add(node.val);
    
    if (node.left == null && node.right == null) {
        result.add(new ArrayList<>(path));
    } else {
        dfs(node.left, path, result);
        dfs(node.right, path, result);
    }
    
    path.remove(path.size() - 1);
}
```

---

## 🏦 Banking Context
**Scenario**: Find all approval paths in org hierarchy.  
**Solution**: DFS to explore all paths from root to leaves.

---

**Next**: [Pattern: Two Heaps](34-pattern-two-heaps.md)
