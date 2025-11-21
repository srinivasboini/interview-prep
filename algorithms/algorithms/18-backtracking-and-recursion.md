# Backtracking and Recursion

## Overview
Backtracking is an algorithmic paradigm that tries different solutions until finds a solution that "works". It's essentially **DFS on a state space tree**.

## Fundamentals

### The Template
```java
void backtrack(Result res, State current, Input input) {
    if (isGoal(current)) {
        res.add(new State(current));
        return;
    }
    
    for (Choice choice : input.getChoices()) {
        if (isValid(choice)) {
            current.add(choice);      // Make choice
            backtrack(res, current, input); // Explore
            current.remove(choice);   // Undo choice (Backtrack)
        }
    }
}
```

## Interview Problems

### Problem 1: Permutations (Medium)
**Pattern**: Backtracking

```java
/**
 * Generate all permutations.
 * Time: O(N!)
 * Space: O(N)
 */
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> list = new ArrayList<>();
    backtrack(list, new ArrayList<>(), nums);
    return list;
}

private void backtrack(List<List<Integer>> list, List<Integer> tempList, int[] nums){
    if(tempList.size() == nums.length){
        list.add(new ArrayList<>(tempList));
    } else{
        for(int i = 0; i < nums.length; i++){ 
            if(tempList.contains(nums[i])) continue; // Element already exists, skip
            tempList.add(nums[i]);
            backtrack(list, tempList, nums);
            tempList.remove(tempList.size() - 1);
        }
    }
}
```

### Problem 2: N-Queens (Hard)
**Pattern**: Constraint Satisfaction

```java
/**
 * Place N queens on NxN board.
 * Time: O(N!)
 */
// Implementation involves tracking cols, diagonals, and anti-diagonals sets.
```

## 🏦 Banking Context: Portfolio Construction
*   **Scenario**: Selecting a subset of assets that satisfy complex regulatory constraints (e.g., "No more than 5% in Tech", "Must have 10% in Bonds").
*   **Algorithm**: **Backtracking** (Constraint Satisfaction) to find valid portfolios.

---
**Next**: [Greedy Algorithms](19-greedy-algorithms.md)
