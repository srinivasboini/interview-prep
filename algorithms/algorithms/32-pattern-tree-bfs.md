# Pattern: Tree BFS

## Overview
Level-by-level traversal using a Queue.

## Interview Problems

### Problem: Binary Tree Zigzag Level Order Traversal (Medium)
**Pattern**: BFS with Flag

```java
/**
 * Zigzag traversal.
 * Time: O(n)
 * Space: O(n)
 */
public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    boolean leftToRight = true;
    
    while (!q.isEmpty()) {
        int size = q.size();
        LinkedList<Integer> level = new LinkedList<>();
        
        for (int i = 0; i < size; i++) {
            TreeNode node = q.poll();
            if (leftToRight) level.addLast(node.val);
            else level.addFirst(node.val);
            
            if (node.left != null) q.offer(node.left);
            if (node.right != null) q.offer(node.right);
        }
        res.add(level);
        leftToRight = !leftToRight;
    }
    return res;
}
```

---
**Next**: [Pattern: Tree DFS](33-pattern-tree-dfs.md)
