# Pattern: Topological Sort

## Overview
Linear ordering of vertices in DAG.

## Interview Problems

### Problem: Alien Dictionary (Hard)
**Pattern**: Topological Sort

```java
/**
 * Derive order of letters in alien language.
 * Time: O(C) where C is total chars.
 */
public String alienOrder(String[] words) {
    Map<Character, List<Character>> adj = new HashMap<>();
    Map<Character, Integer> counts = new HashMap<>();
    for (String w : words) {
        for (char c : w.toCharArray()) {
            counts.put(c, 0);
            adj.putIfAbsent(c, new ArrayList<>());
        }
    }
    
    for (int i = 0; i < words.length - 1; i++) {
        String w1 = words[i], w2 = words[i+1];
        if (w1.length() > w2.length() && w1.startsWith(w2)) return "";
        for (int j = 0; j < Math.min(w1.length(), w2.length()); j++) {
            if (w1.charAt(j) != w2.charAt(j)) {
                adj.get(w1.charAt(j)).add(w2.charAt(j));
                counts.put(w2.charAt(j), counts.get(w2.charAt(j)) + 1);
                break;
            }
        }
    }
    
    StringBuilder sb = new StringBuilder();
    Queue<Character> q = new LinkedList<>();
    for (char c : counts.keySet()) {
        if (counts.get(c) == 0) q.offer(c);
    }
    
    while (!q.isEmpty()) {
        char c = q.poll();
        sb.append(c);
        for (char next : adj.get(c)) {
            counts.put(next, counts.get(next) - 1);
            if (counts.get(next) == 0) q.offer(next);
        }
    }
    
    if (sb.length() < counts.size()) return "";
    return sb.toString();
}
```

---
**Next**: [Problem Lists](40-leetcode-easy-problems.md)
