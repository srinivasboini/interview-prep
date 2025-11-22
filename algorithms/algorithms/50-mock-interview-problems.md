# Mock Interview Problems - Practice Set

## Overview
Realistic mock interview problems at varying difficulty levels for practice.

---

## Easy Level (30 min each)

### Problem 1: Valid Palindrome
**Question**: Given a string, determine if it's a palindrome (ignoring non-alphanumeric, case-insensitive).  
**Pattern**: Two pointers  
**Expected**: O(n) time, O(1) space

### Problem 2: Merge Two Sorted Lists
**Question**: Merge two sorted linked lists.  
**Pattern**: Two pointers  
**Expected**: O(n+m) time, O(1) space

### Problem 3: Maximum Depth of Binary Tree
**Question**: Find maximum depth of binary tree.  
**Pattern**: DFS  
**Expected**: O(n) time, O(h) space

---

## Medium Level (45 min each)

### Problem 4: LRU Cache
**Question**: Implement LRU cache with get() and put() in O(1).  
**Pattern**: HashMap + Doubly Linked List  
**Expected**: O(1) operations

### Problem 5: Course Schedule
**Question**: Can you finish all courses given prerequisites?  
**Pattern**: Topological sort  
**Expected**: O(V+E) time, O(V) space

### Problem 6: Longest Substring Without Repeating
**Question**: Find length of longest substring without repeating characters.  
**Pattern**: Sliding window  
**Expected**: O(n) time, O(min(m,n)) space

### Problem 7: Binary Tree Level Order Traversal
**Question**: Return level order traversal of binary tree.  
**Pattern**: BFS  
**Expected**: O(n) time, O(w) space

### Problem 8: Top K Frequent Elements
**Question**: Find k most frequent elements.  
**Pattern**: Heap  
**Expected**: O(n log k) time, O(n) space

---

## Hard Level (60 min each)

### Problem 9: Merge K Sorted Lists
**Question**: Merge k sorted linked lists.  
**Pattern**: Heap or divide-and-conquer  
**Expected**: O(n log k) time, O(k) space

### Problem 10: Minimum Window Substring
**Question**: Find smallest substring containing all characters of another string.  
**Pattern**: Sliding window  
**Expected**: O(m+n) time, O(1) space

### Problem 11: Word Ladder II
**Question**: Find all shortest transformation sequences.  
**Pattern**: BFS + backtracking  
**Expected**: O(n × m²) time

---

## System Design (60 min)

### Problem 12: Design URL Shortener
**Requirements**: Shorten URLs, redirect, analytics  
**Topics**: Hashing, database design, caching, scalability

### Problem 13: Design Rate Limiter
**Requirements**: Limit requests per user  
**Topics**: Sliding window, token bucket, distributed systems

### Problem 14: Design Trading System
**Requirements**: Order matching, real-time prices  
**Topics**: Order book, low latency, consistency

---

## Behavioral Questions

### Leadership
1. "Tell me about a time you mentored a junior engineer."
2. "Describe a technical decision you made that had significant impact."
3. "How do you handle disagreements with team members?"

### Problem-Solving
4. "Describe a complex bug you debugged."
5. "Tell me about a time you optimized system performance."
6. "How do you approach learning new technologies?"

### Project Management
7. "Describe a project you led from start to finish."
8. "How do you prioritize competing demands?"
9. "Tell me about a time you missed a deadline."

---

## Mock Interview Format

### 45-Minute Coding Interview
- **0-5 min**: Introductions, problem statement
- **5-10 min**: Clarify requirements, discuss approach
- **10-30 min**: Code solution
- **30-40 min**: Test, optimize
- **40-45 min**: Questions for interviewer

### 60-Minute System Design
- **0-5 min**: Understand requirements
- **5-15 min**: High-level design
- **15-40 min**: Deep dive into components
- **40-55 min**: Discuss trade-offs, scalability
- **55-60 min**: Questions

---

## Self-Evaluation Checklist
- [ ] Clarified requirements before coding
- [ ] Discussed approach and complexity
- [ ] Wrote clean, readable code
- [ ] Tested with multiple cases
- [ ] Handled edge cases
- [ ] Communicated thought process
- [ ] Asked clarifying questions
- [ ] Stayed calm under pressure
- [ ] Finished in time

---

**Next**: [DSA Master Guide](51-dsa-master-guide.md)
