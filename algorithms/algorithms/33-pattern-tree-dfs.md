# Pattern: Tree DFS

## Overview
Recursion to explore all paths.

## Interview Problems

### Problem: Path Sum II (Medium)
**Pattern**: DFS with Backtracking

```java
/**
 * Find all paths summing to target.
 * Time: O(N)
 * Space: O(N)
 */
public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
    List<List<Integer>> res = new ArrayList<>();
    dfs(root, targetSum, new ArrayList<>(), res);
    return res;
}

private void dfs(TreeNode node, int target, List<Integer> currentPath, List<List<Integer>> res) {
    if (node == null) return;
    
    currentPath.add(node.val);
    
    if (node.left == null && node.right == null && target == node.val) {
        res.add(new ArrayList<>(currentPath));
    } else {
        dfs(node.left, target - node.val, currentPath, res);
        dfs(node.right, target - node.val, currentPath, res);
    }
    
    currentPath.remove(currentPath.size() - 1); // Backtrack
}
```

---
**Next**: [Pattern: Two Heaps](34-pattern-two-heaps.md)
