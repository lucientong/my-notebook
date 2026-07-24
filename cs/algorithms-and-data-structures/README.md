# Algorithms and Data Structures Interview Guide

Language: English | [中文](../算法与数据结构/)

> Purpose: an English interview-prep version of the Chinese algorithms and system-design notes, optimized for foreign-company coding interviews and system-design rounds.

---

## How To Use This Guide

Use this directory as a practical interview playbook, not as a textbook.

1. Start with `01-Algorithms-and-Data-Structures.md` to build the common vocabulary: data structures, invariants, complexity, and template-level reasoning.
2. Use `03-LeetCode-Pattern-Guide.md` for coding practice. Each pattern should be practiced until you can explain the invariant before writing code.
3. Use `02-System-Design-Interview.md` for system-design rounds and coding-adjacent design problems such as rate limiting, consistent hashing, Snowflake IDs, pagination, idempotency, and API gateways.
4. Revisit the Chinese source notes when you need deeper Chinese explanations or local context.

---

## Difficulty Layers

| Layer | Goal | Typical Topics | Interview Target |
|---|---|---|---|
| Foundation | Speak clearly about data structures and complexity | Array, hash table, linked list, stack, queue, tree, heap | Easy/Medium coding rounds |
| Pattern | Recognize the problem family quickly | Two pointers, sliding window, binary search, DFS/BFS, backtracking | Medium coding rounds |
| Optimization | Justify why the chosen algorithm is optimal | DP, greedy proof, monotonic stack, heap Top K, union-find | Medium/Hard coding rounds |
| System Thinking | Connect algorithms to production tradeoffs | consistent hashing, rate limiting, pagination, idempotency, cache strategy | System-design and staff-level discussion |

---

## Problem Pattern Map

| Signal In The Problem | Primary Pattern | Invariant To Say Out Loud |
|---|---|---|
| Sorted array, pair/triple, remove in-place | Two pointers | Every pointer move discards states that cannot improve the answer. |
| Longest/shortest substring or subarray with a constraint | Sliding window | The window is always the smallest or largest range satisfying the current constraint. |
| Search over a monotonic predicate | Binary search | The answer space is split into false/true or true/false regions. |
| All combinations, permutations, subsets, boards | Backtracking | `path` is the current partial decision; undo exactly what you choose. |
| Unweighted shortest path | BFS | The first time a state is dequeued, its distance is minimal. |
| Tree answer depends on child answers | Postorder DFS | Compute left and right sub-results before processing the current node. |
| Local choice must be proven globally optimal | Greedy | Prove exchange argument or domination, not just intuition. |
| State depends on earlier overlapping states | Dynamic programming | Define state, transition, base cases, and iteration order. |
| Next greater/smaller or range boundary | Monotonic stack | Each element is pushed and popped at most once. |
| Top K or streaming extrema | Heap | Keep only the best `k` candidates seen so far. |
| Connectivity under merges | Union-find | Each component has one representative; path compression preserves membership. |

---

## Document Map

| File | Chinese Source | Focus |
|---|---|---|
| [01-Algorithms-and-Data-Structures.md](./01-Algorithms-and-Data-Structures.md) | [01-算法与数据结构.md](../算法与数据结构/01-算法与数据结构.md) | Core data structures, algorithm templates, invariants, complexity, pitfalls |
| [02-System-Design-Interview.md](./02-System-Design-Interview.md) | [02-系统设计面试专题.md](../算法与数据结构/02-系统设计面试专题.md) | System-design interview framework, estimation, common services, coding-adjacent algorithms |
| [03-LeetCode-Pattern-Guide.md](./03-LeetCode-Pattern-Guide.md) | [03-LeetCode分类刷题指南.md](../算法与数据结构/03-LeetCode分类刷题指南.md) | LeetCode patterns, Go/Python templates, high-frequency problem list, practice plan |

---

## Recommended Reading Paths

### One-Week Sprint

| Day | Focus | Output |
|---|---|---|
| Day 1 | Hash table, two pointers, linked list | Solve Two Sum, 3Sum, Reverse List without notes |
| Day 2 | Stack, queue, binary tree DFS/BFS | Explain preorder/inorder/postorder and BFS visited rules |
| Day 3 | Sliding window and binary search | Write minimum window and lower bound templates |
| Day 4 | Backtracking and graph search | Write subsets/permutations/number of islands |
| Day 5 | Dynamic programming and greedy | Explain state definition vs greedy proof |
| Day 6 | Heap, monotonic stack, union-find, trie | Solve Top K, Daily Temperatures, connectivity |
| Day 7 | Mock interview | Speak in English using the framework below |

### One-Month Track

| Week | Focus | Suggested Goal |
|---|---|---|
| Week 1 | Foundation data structures | 30-40 Easy/Medium problems |
| Week 2 | Trees, binary search, graph traversal | 25-35 Medium problems |
| Week 3 | DP, backtracking, greedy | 30-40 Medium/Hard problems |
| Week 4 | System design + mock interviews | 3 coding mocks and 2 design mocks |

---

## English Interview Answer Framework

For coding questions, use this structure:

1. Clarify the input, output, constraints, and edge cases.
2. State the brute-force approach and why it is not enough.
3. Identify the pattern and invariant.
4. Walk through a small example.
5. Implement in Go or Python.
6. Test normal, boundary, duplicate, empty, and stress cases.
7. State time and space complexity.

Useful phrases:

| Situation | English Expression |
|---|---|
| Clarifying | "Before jumping into code, I want to confirm a few constraints." |
| Choosing a pattern | "This looks like a sliding-window problem because we need a contiguous range under a constraint." |
| Stating invariant | "The invariant is that the current window never contains duplicate characters." |
| Discussing tradeoff | "This uses extra space to reduce the time complexity from quadratic to linear." |
| Handling edge cases | "For an empty input, the loop never runs, so the default return value is correct." |
| Complexity | "Each element enters and leaves the data structure at most once, so the total time is linear." |

---

## Maintenance Rules

- Keep file numbers aligned with the Chinese directory.
- Add the language link at the top of every English document.
- When adding code examples for algorithms or coding-adjacent logic, provide both Go and Python versions.
- Prefer the section shape: problem pattern -> invariant -> complexity -> Go implementation -> Python implementation -> pitfalls -> interview self-check.
- Do not update global indexes from this directory-specific task. Update only this folder unless a parent task coordinates shared index changes.
- Avoid unfinished sections once a document is considered complete.

---

## Local Chinese Links

- [中文算法目录](../算法与数据结构/)
- [中文 01：算法与数据结构](../算法与数据结构/01-算法与数据结构.md)
- [中文 02：系统设计面试专题](../算法与数据结构/02-系统设计面试专题.md)
- [中文 03：LeetCode 分类刷题指南](../算法与数据结构/03-LeetCode分类刷题指南.md)
