# Algorithms and Data Structures

Language: English | [中文](../算法与数据结构/01-算法与数据结构.md)

> Interview goal: explain the pattern, state the invariant, implement the core algorithm in Go and Python, then analyze complexity and edge cases.

---

## Table of Contents

1. [How To Read This Document](#0-how-to-read-this-document)
2. [Arrays and Hash Tables](#1-arrays-and-hash-tables)
3. [Linked Lists](#2-linked-lists)
4. [Stacks and Queues](#3-stacks-and-queues)
5. [Trees](#4-trees)
6. [Heaps](#5-heaps)
7. [Graphs](#6-graphs)
8. [Sorting](#7-sorting)
9. [Dynamic Programming](#8-dynamic-programming)
10. [Backtracking](#9-backtracking)
11. [Greedy Algorithms](#10-greedy-algorithms)
12. [Bit Manipulation](#11-bit-manipulation)
13. [Advanced Patterns](#12-advanced-patterns)
14. [Interview Self-Check](#13-interview-self-check)

---

## 0. How To Read This Document

### 0.1 Standard Section Shape

For interview preparation, each important topic should be practiced in this order:

| Step | What To Say |
|---|---|
| Problem pattern | "This problem asks for ..." |
| Invariant | "At every iteration, I maintain ..." |
| Complexity | "Each element is processed ... times." |
| Implementation | Write concise Go and Python code. |
| Pitfalls | Mention off-by-one, empty input, duplicate values, overflow, recursion depth. |
| Self-check | Answer the most likely follow-up questions. |

### 0.2 Code Convention

- Go examples use standard-library-only templates whenever possible.
- Python examples use `collections`, `heapq`, `bisect`, and `functools` when they simplify the core idea.
- Complexity is written as `O(...)` time and `O(...)` space.

---

## 1. Arrays and Hash Tables

### 1.1 Pattern

Arrays are best when you need random access, contiguous scanning, two pointers, prefix sums, or in-place updates. Hash tables are best when you need fast membership, counting, grouping, or mapping a value to its latest index.

### 1.2 Invariant

For a one-pass hash-table algorithm, the invariant is usually: before processing `nums[i]`, the map contains exactly the information from indices `< i` needed to decide whether `nums[i]` completes an answer.

### 1.3 Two Sum

Complexity: `O(n)` time, `O(n)` space.

```go
func twoSum(nums []int, target int) []int {
	seen := make(map[int]int)
	for i, v := range nums {
		if j, ok := seen[target-v]; ok {
			return []int{j, i}
		}
		seen[v] = i
	}
	return nil
}
```

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    seen: dict[int, int] = {}
    for i, v in enumerate(nums):
        if target - v in seen:
            return [seen[target - v], i]
        seen[v] = i
    return []
```

### 1.4 Pitfalls

- Insert the current value after checking the complement, otherwise the same element can be used twice.
- For grouping problems, use a normalized key, such as sorted characters or a 26-count tuple.
- Hash-table average lookup is `O(1)`, but collision-heavy cases can degrade depending on implementation.

---

## 2. Linked Lists

### 2.1 Pattern

Linked-list problems are pointer-manipulation problems. Use a dummy head when deleting, inserting, merging, or building a new list, because it removes special handling for the real head.

### 2.2 Invariant

For iterative reversal: `prev` always points to the already reversed prefix, and `cur` points to the first node not yet reversed.

### 2.3 Reverse Linked List

Complexity: `O(n)` time, `O(1)` extra space.

```go
type ListNode struct {
	Val  int
	Next *ListNode
}

func reverseList(head *ListNode) *ListNode {
	var prev *ListNode
	cur := head
	for cur != nil {
		next := cur.Next
		cur.Next = prev
		prev = cur
		cur = next
	}
	return prev
}
```

```python
class ListNode:
    def __init__(self, val: int = 0, next: "ListNode | None" = None):
        self.val = val
        self.next = next


def reverse_list(head: ListNode | None) -> ListNode | None:
    prev, cur = None, head
    while cur:
        nxt = cur.next
        cur.next = prev
        prev = cur
        cur = nxt
    return prev
```

### 2.4 Pitfalls

- Save `next` before rewiring `cur.next`.
- For cycle detection, the fast pointer must check both `fast` and `fast.Next`.
- For LRU cache, combine a hash table for lookup and a doubly linked list for `O(1)` movement.

---

## 3. Stacks and Queues

### 3.1 Pattern

Stacks model nested structure, undo behavior, DFS, and monotonic constraints. Queues model FIFO processing, BFS, and streaming windows.

### 3.2 Valid Parentheses

Invariant: the stack contains unmatched opening brackets in the order they must be closed.

Complexity: `O(n)` time, `O(n)` space.

```go
func isValid(s string) bool {
	pairs := map[byte]byte{')': '(', ']': '[', '}': '{'}
	stack := []byte{}
	for i := 0; i < len(s); i++ {
		ch := s[i]
		if ch == '(' || ch == '[' || ch == '{' {
			stack = append(stack, ch)
			continue
		}
		if len(stack) == 0 || stack[len(stack)-1] != pairs[ch] {
			return false
		}
		stack = stack[:len(stack)-1]
	}
	return len(stack) == 0
}
```

```python
def is_valid(s: str) -> bool:
    pairs = {")": "(", "]": "[", "}": "{"}
    stack: list[str] = []
    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif not stack or stack[-1] != pairs[ch]:
            return False
        else:
            stack.pop()
    return not stack
```

### 3.3 Pitfalls

- A stack can prove linear time even with a nested `while` if each element is pushed and popped at most once.
- A queue implemented by slicing in Go can retain memory; for long-running services use an index-based queue or ring buffer.

---

## 4. Trees

### 4.1 DFS Order

Preorder, inorder, and postorder are not decided by the base case. They are decided by where the core processing of `root` happens relative to recursive calls.

| Order | Processing Position | Typical Use |
|---|---|---|
| Preorder | before children | pass state from top to bottom, serialize |
| Inorder | between left and right | BST sorted order |
| Postorder | after children | tree DP, height, LCA |

### 4.2 Maximum Depth

Pattern: tree DP by postorder.

Invariant: each recursive call returns the correct depth of its subtree.

Complexity: `O(n)` time, `O(h)` recursion stack.

```go
type TreeNode struct {
	Val   int
	Left  *TreeNode
	Right *TreeNode
}

func maxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}
	left := maxDepth(root.Left)
	right := maxDepth(root.Right)
	if left > right {
		return left + 1
	}
	return right + 1
}
```

```python
class TreeNode:
    def __init__(self, val: int = 0, left: "TreeNode | None" = None, right: "TreeNode | None" = None):
        self.val = val
        self.left = left
        self.right = right


def max_depth(root: TreeNode | None) -> int:
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

### 4.3 Lowest Common Ancestor

The LCA of a binary tree is postorder because the current node can decide only after seeing whether the target nodes appear in the left and right subtrees.

Complexity: `O(n)` time, `O(h)` stack.

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	if root == nil || root == p || root == q {
		return root
	}
	left := lowestCommonAncestor(root.Left, p, q)
	right := lowestCommonAncestor(root.Right, p, q)
	if left != nil && right != nil {
		return root
	}
	if left != nil {
		return left
	}
	return right
}
```

```python
def lowest_common_ancestor(root: TreeNode | None, p: TreeNode, q: TreeNode) -> TreeNode | None:
    if root is None or root is p or root is q:
        return root
    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)
    if left and right:
        return root
    return left or right
```

### 4.4 Pitfalls

- Do not call a function preorder just because the null check appears first.
- Recursive depth can be `O(n)` for a skewed tree.
- BST validation needs lower and upper bounds, not only local parent-child comparison.

---

## 5. Heaps

### 5.1 Pattern

Use a heap when you repeatedly need the minimum or maximum among changing candidates, especially Top K, streaming median, k-way merge, and Dijkstra.

### 5.2 Invariant

For Top K largest with a min-heap of size `k`: after processing each number, the heap contains the largest `k` numbers seen so far, and the heap root is the smallest among those candidates.

### 5.3 Top K Largest

Complexity: `O(n log k)` time, `O(k)` space.

```go
import "container/heap"

type IntMinHeap []int

func (h IntMinHeap) Len() int           { return len(h) }
func (h IntMinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntMinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntMinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *IntMinHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}

func topKLargest(nums []int, k int) []int {
	h := &IntMinHeap{}
	heap.Init(h)
	for _, v := range nums {
		if h.Len() < k {
			heap.Push(h, v)
		} else if k > 0 && v > (*h)[0] {
			heap.Pop(h)
			heap.Push(h, v)
		}
	}
	return []int(*h)
}
```

```python
import heapq


def top_k_largest(nums: list[int], k: int) -> list[int]:
    heap: list[int] = []
    for v in nums:
        if len(heap) < k:
            heapq.heappush(heap, v)
        elif k > 0 and v > heap[0]:
            heapq.heapreplace(heap, v)
    return heap
```

### 5.4 Pitfalls

- A heap is not fully sorted; only the root is guaranteed to be the current extreme.
- Python `heapq` is a min-heap; simulate a max-heap by storing negative values.
- Go's `container/heap` requires pointer receiver methods for `Push` and `Pop`.

---

## 6. Graphs

### 6.1 Representation

Use an adjacency list for sparse graphs: `O(V + E)` space. Use an adjacency matrix only when the graph is dense or when `O(1)` edge lookup is more important than memory.

### 6.2 BFS For Unweighted Shortest Path

Invariant: when a node is dequeued for the first time, its distance from the start is minimal. Therefore, BFS `visited` should not be backtracked for ordinary shortest-path search.

Complexity: `O(V + E)` time, `O(V)` space.

```go
func shortestPath(graph [][]int, start int) []int {
	dist := make([]int, len(graph))
	for i := range dist {
		dist[i] = -1
	}
	dist[start] = 0
	queue := []int{start}
	for head := 0; head < len(queue); head++ {
		node := queue[head]
		for _, nei := range graph[node] {
			if dist[nei] == -1 {
				dist[nei] = dist[node] + 1
				queue = append(queue, nei)
			}
		}
	}
	return dist
}
```

```python
from collections import deque


def shortest_path(graph: list[list[int]], start: int) -> list[int]:
    dist = [-1] * len(graph)
    dist[start] = 0
    q = deque([start])
    while q:
        node = q.popleft()
        for nei in graph[node]:
            if dist[nei] == -1:
                dist[nei] = dist[node] + 1
                q.append(nei)
    return dist
```

### 6.3 Pitfalls

- If edge weights differ, use Dijkstra or another weighted shortest-path algorithm.
- If the state includes extra dimensions, `visited` must include those dimensions.
- If the question asks for all paths, use backtracking instead of a global BFS visited set.

---

## 7. Sorting

### 7.1 Pattern

Sorting is often the setup step that creates monotonic structure for two pointers, greedy selection, interval merging, or binary search.

### 7.2 Quicksort Partition

Invariant: after partition, elements before the pivot position are `<= pivot`, and elements after it are `> pivot`.

Average complexity: `O(n log n)` time, `O(log n)` recursion stack. Worst case: `O(n^2)` time.

```go
func quickSort(nums []int) {
	var sortRange func(int, int)
	sortRange = func(lo, hi int) {
		if lo >= hi {
			return
		}
		pivot := nums[hi]
		i := lo
		for j := lo; j < hi; j++ {
			if nums[j] <= pivot {
				nums[i], nums[j] = nums[j], nums[i]
				i++
			}
		}
		nums[i], nums[hi] = nums[hi], nums[i]
		sortRange(lo, i-1)
		sortRange(i+1, hi)
	}
	sortRange(0, len(nums)-1)
}
```

```python
def quick_sort(nums: list[int]) -> None:
    def sort_range(lo: int, hi: int) -> None:
        if lo >= hi:
            return
        pivot = nums[hi]
        i = lo
        for j in range(lo, hi):
            if nums[j] <= pivot:
                nums[i], nums[j] = nums[j], nums[i]
                i += 1
        nums[i], nums[hi] = nums[hi], nums[i]
        sort_range(lo, i - 1)
        sort_range(i + 1, hi)

    sort_range(0, len(nums) - 1)
```

### 7.3 Pitfalls

- Randomize or use median-of-three to reduce worst-case risk.
- Quicksort is usually not stable.
- Merge sort is preferred when stability matters.

---

## 8. Dynamic Programming

### 8.1 Pattern

Use DP when the problem has overlapping subproblems and optimal substructure. Say the state definition before the transition.

### 8.2 Coin Change

State: `dp[x]` is the minimum number of coins needed to make amount `x`.

Transition: for each coin `c`, `dp[x] = min(dp[x], dp[x-c] + 1)`.

Complexity: `O(amount * number_of_coins)` time, `O(amount)` space.

```go
func coinChange(coins []int, amount int) int {
	const inf = int(1e9)
	dp := make([]int, amount+1)
	for i := 1; i <= amount; i++ {
		dp[i] = inf
	}
	for x := 1; x <= amount; x++ {
		for _, c := range coins {
			if x >= c && dp[x-c]+1 < dp[x] {
				dp[x] = dp[x-c] + 1
			}
		}
	}
	if dp[amount] == inf {
		return -1
	}
	return dp[amount]
}
```

```python
def coin_change(coins: list[int], amount: int) -> int:
    inf = 10**9
    dp = [inf] * (amount + 1)
    dp[0] = 0
    for x in range(1, amount + 1):
        for c in coins:
            if x >= c:
                dp[x] = min(dp[x], dp[x - c] + 1)
    return -1 if dp[amount] == inf else dp[amount]
```

### 8.3 Pitfalls

- Do not start coding before defining `dp[i]` in plain English.
- Check base cases such as amount `0`, empty input, and impossible states.
- For knapsack, iteration order determines whether an item can be reused.

---

## 9. Backtracking

### 9.1 Pattern

Backtracking enumerates a decision tree. It is used for subsets, combinations, permutations, N-Queens, word search, and constraint satisfaction.

### 9.2 Invariant

`path` contains the exact choices from root to the current recursion node. Before returning, the function must undo the latest choice.

### 9.3 Subsets

Complexity: `O(n * 2^n)` time to materialize all answers, `O(n)` recursion depth excluding output.

```go
func subsets(nums []int) [][]int {
	ans := [][]int{}
	path := []int{}
	var dfs func(int)
	dfs = func(start int) {
		snapshot := append([]int(nil), path...)
		ans = append(ans, snapshot)
		for i := start; i < len(nums); i++ {
			path = append(path, nums[i])
			dfs(i + 1)
			path = path[:len(path)-1]
		}
	}
	dfs(0)
	return ans
}
```

```python
def subsets(nums: list[int]) -> list[list[int]]:
    ans: list[list[int]] = []
    path: list[int] = []

    def dfs(start: int) -> None:
        ans.append(path.copy())
        for i in range(start, len(nums)):
            path.append(nums[i])
            dfs(i + 1)
            path.pop()

    dfs(0)
    return ans
```

### 9.4 Pitfalls

- Always copy `path` when storing an answer.
- For permutations, use a `used` array or swap in place.
- For duplicate values, sort first and skip repeated choices at the same recursion depth.

---

## 10. Greedy Algorithms

### 10.1 Pattern

A greedy algorithm is valid only when a local choice can be proven not to hurt the global optimum. In interviews, mention the proof idea: exchange argument, staying ahead, or domination.

### 10.2 Jump Game

Invariant: `farthest` is the farthest index reachable after scanning positions up to `i`.

Complexity: `O(n)` time, `O(1)` space.

```go
func canJump(nums []int) bool {
	farthest := 0
	for i, jump := range nums {
		if i > farthest {
			return false
		}
		if i+jump > farthest {
			farthest = i + jump
		}
	}
	return true
}
```

```python
def can_jump(nums: list[int]) -> bool:
    farthest = 0
    for i, jump in enumerate(nums):
        if i > farthest:
            return False
        farthest = max(farthest, i + jump)
    return True
```

### 10.3 Pitfalls

- Greedy intuition is not enough; provide a short correctness argument.
- If a counterexample exists, switch to DP.
- Sorting is often the hidden first step in interval greedy problems.

---

## 11. Bit Manipulation

### 11.1 Pattern

Bit manipulation is common in set compression, parity, masks, XOR cancellation, and low-level optimization.

### 11.2 Single Number

Invariant: XOR cancels equal pairs because `a ^ a = 0` and `a ^ 0 = a`.

Complexity: `O(n)` time, `O(1)` space.

```go
func singleNumber(nums []int) int {
	ans := 0
	for _, v := range nums {
		ans ^= v
	}
	return ans
}
```

```python
def single_number(nums: list[int]) -> int:
    ans = 0
    for v in nums:
        ans ^= v
    return ans
```

### 11.3 Hamming Weight

Invariant: `n & (n - 1)` removes the lowest set bit.

Complexity: `O(number_of_set_bits)` time, `O(1)` space.

```go
func hammingWeight(n uint32) int {
	count := 0
	for n != 0 {
		n &= n - 1
		count++
	}
	return count
}
```

```python
def hamming_weight(n: int) -> int:
    count = 0
    while n:
        n &= n - 1
        count += 1
    return count
```

---

## 12. Advanced Patterns

### 12.1 Prefix Sum

Use prefix sums when range queries are frequent and the array is immutable between queries.

Complexity: build `O(n)`, query `O(1)`, space `O(n)`.

```go
type NumArray struct {
	prefix []int
}

func NewNumArray(nums []int) NumArray {
	prefix := make([]int, len(nums)+1)
	for i, v := range nums {
		prefix[i+1] = prefix[i] + v
	}
	return NumArray{prefix: prefix}
}

func (a NumArray) SumRange(left, right int) int {
	return a.prefix[right+1] - a.prefix[left]
}
```

```python
class NumArray:
    def __init__(self, nums: list[int]):
        self.prefix = [0]
        for v in nums:
            self.prefix.append(self.prefix[-1] + v)

    def sum_range(self, left: int, right: int) -> int:
        return self.prefix[right + 1] - self.prefix[left]
```

### 12.2 Monotonic Stack

Use a monotonic stack for next greater or smaller element, histogram area, and some rain-water variants.

Complexity: `O(n)` time because each index is pushed and popped at most once.

```go
func nextGreater(nums []int) []int {
	ans := make([]int, len(nums))
	for i := range ans {
		ans[i] = -1
	}
	stack := []int{}
	for i, v := range nums {
		for len(stack) > 0 && v > nums[stack[len(stack)-1]] {
			idx := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			ans[idx] = v
		}
		stack = append(stack, i)
	}
	return ans
}
```

```python
def next_greater(nums: list[int]) -> list[int]:
    ans = [-1] * len(nums)
    stack: list[int] = []
    for i, v in enumerate(nums):
        while stack and v > nums[stack[-1]]:
            ans[stack.pop()] = v
        stack.append(i)
    return ans
```

### 12.3 Union-Find

Use union-find for dynamic connectivity, connected components, redundant edges, Kruskal MST, and account merge.

Complexity: amortized `O(alpha(n))` per operation, effectively constant.

```go
type UnionFind struct {
	parent []int
	rank   []int
}

func NewUnionFind(n int) *UnionFind {
	parent := make([]int, n)
	rank := make([]int, n)
	for i := range parent {
		parent[i] = i
	}
	return &UnionFind{parent: parent, rank: rank}
}

func (u *UnionFind) Find(x int) int {
	if u.parent[x] != x {
		u.parent[x] = u.Find(u.parent[x])
	}
	return u.parent[x]
}

func (u *UnionFind) Union(a, b int) bool {
	ra, rb := u.Find(a), u.Find(b)
	if ra == rb {
		return false
	}
	if u.rank[ra] < u.rank[rb] {
		ra, rb = rb, ra
	}
	u.parent[rb] = ra
	if u.rank[ra] == u.rank[rb] {
		u.rank[ra]++
	}
	return true
}
```

```python
class UnionFind:
    def __init__(self, n: int):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x: int) -> int:
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, a: int, b: int) -> bool:
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False
        if self.rank[ra] < self.rank[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra
        if self.rank[ra] == self.rank[rb]:
            self.rank[ra] += 1
        return True
```

### 12.4 Trie

Use a trie for prefix lookup, autocomplete, word search, dictionary matching, and bitwise maximum XOR.

Complexity: insert/search `O(L)` where `L` is word length.

```go
type TrieNode struct {
	children map[rune]*TrieNode
	isEnd    bool
}

type Trie struct {
	root *TrieNode
}

func NewTrie() *Trie {
	return &Trie{root: &TrieNode{children: map[rune]*TrieNode{}}}
}

func (t *Trie) Insert(word string) {
	node := t.root
	for _, ch := range word {
		if node.children[ch] == nil {
			node.children[ch] = &TrieNode{children: map[rune]*TrieNode{}}
		}
		node = node.children[ch]
	}
	node.isEnd = true
}

func (t *Trie) StartsWith(prefix string) bool {
	node := t.root
	for _, ch := range prefix {
		if node.children[ch] == nil {
			return false
		}
		node = node.children[ch]
	}
	return true
}
```

```python
class TrieNode:
    def __init__(self):
        self.children: dict[str, TrieNode] = {}
        self.is_end = False


class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def starts_with(self, prefix: str) -> bool:
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return False
            node = node.children[ch]
        return True
```

---

## 13. Interview Self-Check

### 13.1 Core Questions

- Why is array random access `O(1)`?
- Why does a hash table usually provide `O(1)` lookup, and when can it degrade?
- Why does LRU need both a hash table and a doubly linked list?
- How do you distinguish preorder, inorder, and postorder in a real problem?
- Why is BFS first visit shortest only for unweighted graphs?
- Why does Top K use a min-heap of size `k`?
- What must be true before using greedy?
- How do you define a DP state before writing code?
- Why is a monotonic stack still linear when it has a nested `while`?
- What is the difference between backtracking `visited` and BFS `visited`?

### 13.2 English Answer Template

Use this for most coding rounds:

1. "I will first clarify constraints and edge cases."
2. "The brute-force solution is ..., but it costs ..."
3. "The optimized pattern is ... because ..."
4. "The invariant is ..."
5. "Now I will implement it."
6. "Let's test empty input, duplicates, and boundary values."
7. "The time complexity is ... and the space complexity is ..."

### 13.3 Common Edge Cases

| Topic | Edge Cases |
|---|---|
| Array | empty input, one element, duplicates, sorted/reverse-sorted |
| Linked list | empty list, one node, cycle, head deletion |
| Tree | empty tree, skewed tree, duplicate values in BST questions |
| Graph | disconnected graph, self-loop, duplicate edges, weighted edges |
| DP | impossible state, zero amount, first row/column initialization |
| Backtracking | duplicate choices, path copy, restoring state |
| Heap | `k = 0`, `k > n`, duplicate priorities |
