# LeetCode Pattern Guide

Language: English | [中文](../算法与数据结构/03-LeetCode分类刷题指南.md)

> Purpose: prepare for foreign-company coding interviews by practicing high-frequency problem families, explaining invariants in English, and writing clean Go/Python solutions.

---

## Table of Contents

1. [Arrays and Two Pointers](#1-arrays-and-two-pointers)
2. [Sliding Window](#2-sliding-window)
3. [Linked Lists](#3-linked-lists)
4. [Stacks and Queues](#4-stacks-and-queues)
5. [Binary Trees](#5-binary-trees)
6. [Backtracking](#6-backtracking)
7. [Dynamic Programming](#7-dynamic-programming)
8. [Greedy](#8-greedy)
9. [Graphs](#9-graphs)
10. [Binary Search](#10-binary-search)
11. [Heap and Priority Queue](#11-heap-and-priority-queue)
12. [Strings](#12-strings)
13. [Bit Manipulation](#13-bit-manipulation)
14. [Top Interview List](#14-top-interview-list)
15. [Practice Plan](#15-practice-plan)

---

## 1. Arrays and Two Pointers

### Problem Pattern

Use two pointers when the problem involves a sorted array, pair/triple search, palindrome check, in-place removal, or a container/rain-water boundary.

### Invariant

Every pointer move discards candidates that can no longer beat or satisfy the current answer.

### LC15 3Sum

Complexity: `O(n^2)` time, excluding output; `O(log n)` or `O(n)` sorting space depending on implementation.

```go
import "sort"

func threeSum(nums []int) [][]int {
	sort.Ints(nums)
	ans := [][]int{}
	for i := 0; i < len(nums)-2; i++ {
		if i > 0 && nums[i] == nums[i-1] {
			continue
		}
		if nums[i] > 0 {
			break
		}
		left, right := i+1, len(nums)-1
		for left < right {
			sum := nums[i] + nums[left] + nums[right]
			if sum == 0 {
				ans = append(ans, []int{nums[i], nums[left], nums[right]})
				for left < right && nums[left] == nums[left+1] {
					left++
				}
				for left < right && nums[right] == nums[right-1] {
					right--
				}
				left++
				right--
			} else if sum < 0 {
				left++
			} else {
				right--
			}
		}
	}
	return ans
}
```

```python
def three_sum(nums: list[int]) -> list[list[int]]:
    nums.sort()
    ans: list[list[int]] = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        if nums[i] > 0:
            break
        left, right = i + 1, len(nums) - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                ans.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    return ans
```

### Pitfalls

- Skip duplicates at all three positions.
- Sorting changes indices, so do not use this pattern if original indices are required.
- For overflow-prone languages, use a wider integer type when summing.

### Interview Self-Check

- Can you explain why moving the smaller pointer is safe?
- Can you handle `[-1, -1, 0, 1, 1]` without duplicate answers?

---

## 2. Sliding Window

### Problem Pattern

Use sliding window for contiguous subarray or substring problems with a constraint: at most/at least/exactly some count, no duplicates, or covering all required characters.

### Invariant

The window `[left, right]` always represents the current candidate range, and `left` only moves forward to restore or tighten the constraint.

### LC3 Longest Substring Without Repeating Characters

Complexity: `O(n)` time, `O(min(n, alphabet))` space.

```go
func lengthOfLongestSubstring(s string) int {
	last := make(map[byte]int)
	left, ans := 0, 0
	for right := 0; right < len(s); right++ {
		ch := s[right]
		if p, ok := last[ch]; ok && p >= left {
			left = p + 1
		}
		last[ch] = right
		if right-left+1 > ans {
			ans = right - left + 1
		}
	}
	return ans
}
```

```python
def length_of_longest_substring(s: str) -> int:
    last: dict[str, int] = {}
    left = ans = 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1
        last[ch] = right
        ans = max(ans, right - left + 1)
    return ans
```

### Pitfalls

- Move `left` to `max(left, last[ch] + 1)`, not blindly to `last[ch] + 1`.
- For Unicode strings in Go, switch from byte iteration to rune iteration if the problem requires characters beyond ASCII.

### Interview Self-Check

- Can you state the window invariant before coding?
- Can you test `"abba"` and explain why the answer is `2`?

---

## 3. Linked Lists

### Problem Pattern

Use linked-list techniques for reversal, fast/slow pointers, dummy head, cycle detection, merge, and LRU cache.

### Invariant

For fast/slow pointers, the distance relation between the pointers encodes the answer: midpoint, cycle entrance, or kth-from-end.

### LC142 Linked List Cycle II

Complexity: `O(n)` time, `O(1)` space.

```go
type ListNode struct {
	Val  int
	Next *ListNode
}

func detectCycle(head *ListNode) *ListNode {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			p := head
			for p != slow {
				p = p.Next
				slow = slow.Next
			}
			return p
		}
	}
	return nil
}
```

```python
class ListNode:
    def __init__(self, val: int = 0, next: "ListNode | None" = None):
        self.val = val
        self.next = next


def detect_cycle(head: ListNode | None) -> ListNode | None:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            p = head
            while p is not slow:
                p = p.next
                slow = slow.next
            return p
    return None
```

### Pitfalls

- Use identity comparison for nodes, not value comparison.
- For merge problems, a dummy node makes the code simpler and safer.

---

## 4. Stacks and Queues

### Problem Pattern

Use monotonic stacks for next greater/smaller element and histogram-like boundaries. Use queues for BFS and monotonic queues.

### LC739 Daily Temperatures

Invariant: the stack stores indices whose temperatures have not found a warmer future day, in decreasing temperature order.

Complexity: `O(n)` time, `O(n)` space.

```go
func dailyTemperatures(temperatures []int) []int {
	ans := make([]int, len(temperatures))
	stack := []int{}
	for i, t := range temperatures {
		for len(stack) > 0 && t > temperatures[stack[len(stack)-1]] {
			idx := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			ans[idx] = i - idx
		}
		stack = append(stack, i)
	}
	return ans
}
```

```python
def daily_temperatures(temperatures: list[int]) -> list[int]:
    ans = [0] * len(temperatures)
    stack: list[int] = []
    for i, t in enumerate(temperatures):
        while stack and t > temperatures[stack[-1]]:
            idx = stack.pop()
            ans[idx] = i - idx
        stack.append(i)
    return ans
```

### Pitfalls

- Store indices, not values, when the answer needs distance.
- The nested loop is still linear because each index leaves the stack once.

---

## 5. Binary Trees

### Problem Pattern

Use preorder for passing state down, inorder for BST ordering, and postorder when the parent answer depends on child answers.

### LC98 Validate Binary Search Tree

Invariant: every node must be strictly inside its `(low, high)` valid range.

Complexity: `O(n)` time, `O(h)` stack.

```go
func isValidBST(root *TreeNode) bool {
	var dfs func(*TreeNode, *int, *int) bool
	dfs = func(node *TreeNode, low, high *int) bool {
		if node == nil {
			return true
		}
		if low != nil && node.Val <= *low {
			return false
		}
		if high != nil && node.Val >= *high {
			return false
		}
		return dfs(node.Left, low, &node.Val) && dfs(node.Right, &node.Val, high)
	}
	return dfs(root, nil, nil)
}
```

```python
def is_valid_bst(root: TreeNode | None) -> bool:
    def dfs(node: TreeNode | None, low: int | None, high: int | None) -> bool:
        if not node:
            return True
        if low is not None and node.val <= low:
            return False
        if high is not None and node.val >= high:
            return False
        return dfs(node.left, low, node.val) and dfs(node.right, node.val, high)

    return dfs(root, None, None)
```

### Pitfalls

- Local checks such as `left < root < right` are not enough.
- Duplicates are usually invalid in LeetCode BST unless the prompt explicitly allows them.

---

## 6. Backtracking

### Problem Pattern

Use backtracking when you must enumerate decisions: subsets, combinations, permutations, phone number letters, N-Queens, and word search.

### LC46 Permutations

Invariant: `path` contains a valid partial permutation and `used[i]` tells whether `nums[i]` is already in the path.

Complexity: `O(n * n!)` time to copy all answers, `O(n)` stack excluding output.

```go
func permute(nums []int) [][]int {
	ans := [][]int{}
	path := []int{}
	used := make([]bool, len(nums))
	var dfs func()
	dfs = func() {
		if len(path) == len(nums) {
			ans = append(ans, append([]int(nil), path...))
			return
		}
		for i, v := range nums {
			if used[i] {
				continue
			}
			used[i] = true
			path = append(path, v)
			dfs()
			path = path[:len(path)-1]
			used[i] = false
		}
	}
	dfs()
	return ans
}
```

```python
def permute(nums: list[int]) -> list[list[int]]:
    ans: list[list[int]] = []
    path: list[int] = []
    used = [False] * len(nums)

    def dfs() -> None:
        if len(path) == len(nums):
            ans.append(path.copy())
            return
        for i, v in enumerate(nums):
            if used[i]:
                continue
            used[i] = True
            path.append(v)
            dfs()
            path.pop()
            used[i] = False

    dfs()
    return ans
```

### Pitfalls

- Copy `path`; otherwise all answers may reference the same mutable list/slice.
- For duplicate input, sort first and skip `i > 0 and nums[i] == nums[i-1] and not used[i-1]`.

---

## 7. Dynamic Programming

### Problem Pattern

Use DP when the same subproblem is reused and the answer can be built from smaller states.

### LC300 Longest Increasing Subsequence

Invariant: `tails[i]` is the smallest possible tail value for an increasing subsequence of length `i + 1`.

Complexity: `O(n log n)` time, `O(n)` space.

```go
import "sort"

func lengthOfLIS(nums []int) int {
	tails := []int{}
	for _, v := range nums {
		i := sort.Search(len(tails), func(i int) bool {
			return tails[i] >= v
		})
		if i == len(tails) {
			tails = append(tails, v)
		} else {
			tails[i] = v
		}
	}
	return len(tails)
}
```

```python
from bisect import bisect_left


def length_of_lis(nums: list[int]) -> int:
    tails: list[int] = []
    for v in nums:
        i = bisect_left(tails, v)
        if i == len(tails):
            tails.append(v)
        else:
            tails[i] = v
    return len(tails)
```

### Pitfalls

- `tails` is not the actual subsequence; it is a compact state representation.
- Use `bisect_left` for strictly increasing and `bisect_right` for non-decreasing variants.

---

## 8. Greedy

### Problem Pattern

Use greedy when a locally optimal choice can be proven safe by exchange argument or staying-ahead reasoning.

### LC435 Non-overlapping Intervals

Invariant: after sorting by end time, choosing the earliest finishing compatible interval leaves maximum room for future intervals.

Complexity: `O(n log n)` time, `O(1)` extra space excluding sorting.

```go
import "sort"

func eraseOverlapIntervals(intervals [][]int) int {
	if len(intervals) == 0 {
		return 0
	}
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i][1] < intervals[j][1]
	})
	kept, end := 1, intervals[0][1]
	for i := 1; i < len(intervals); i++ {
		if intervals[i][0] >= end {
			kept++
			end = intervals[i][1]
		}
	}
	return len(intervals) - kept
}
```

```python
def erase_overlap_intervals(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0
    intervals.sort(key=lambda x: x[1])
    kept, end = 1, intervals[0][1]
    for start, finish in intervals[1:]:
        if start >= end:
            kept += 1
            end = finish
    return len(intervals) - kept
```

### Pitfalls

- Sorting by start time is natural but not always correct for interval scheduling.
- Be explicit about whether touching intervals count as overlap.

---

## 9. Graphs

### Problem Pattern

Use DFS/BFS for connected components, grid traversal, shortest path in unweighted graphs, and topological sorting.

### LC200 Number of Islands

Invariant: once a land cell is visited, all land cells in the same connected component will be visited by the same DFS/BFS.

Complexity: `O(mn)` time, `O(mn)` worst-case stack or queue space.

```go
func numIslands(grid [][]byte) int {
	if len(grid) == 0 {
		return 0
	}
	m, n := len(grid), len(grid[0])
	dirs := [][2]int{{1, 0}, {-1, 0}, {0, 1}, {0, -1}}
	var dfs func(int, int)
	dfs = func(r, c int) {
		if r < 0 || r >= m || c < 0 || c >= n || grid[r][c] != '1' {
			return
		}
		grid[r][c] = '0'
		for _, d := range dirs {
			dfs(r+d[0], c+d[1])
		}
	}
	count := 0
	for r := 0; r < m; r++ {
		for c := 0; c < n; c++ {
			if grid[r][c] == '1' {
				count++
				dfs(r, c)
			}
		}
	}
	return count
}
```

```python
def num_islands(grid: list[list[str]]) -> int:
    if not grid:
        return 0
    m, n = len(grid), len(grid[0])

    def dfs(r: int, c: int) -> None:
        if r < 0 or r >= m or c < 0 or c >= n or grid[r][c] != "1":
            return
        grid[r][c] = "0"
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            dfs(r + dr, c + dc)

    count = 0
    for r in range(m):
        for c in range(n):
            if grid[r][c] == "1":
                count += 1
                dfs(r, c)
    return count
```

### Pitfalls

- Mutating the grid is fine only if the prompt allows it.
- Recursion may overflow on a huge grid; switch to iterative BFS/DFS if needed.

---

## 10. Binary Search

### Problem Pattern

Use binary search for sorted arrays or any monotonic predicate over an answer space.

### Lower Bound

Invariant: the answer is always inside `[lo, hi)`.

Complexity: `O(log n)` time, `O(1)` space.

```go
func lowerBound(nums []int, target int) int {
	lo, hi := 0, len(nums)
	for lo < hi {
		mid := lo + (hi-lo)/2
		if nums[mid] < target {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}
```

```python
def lower_bound(nums: list[int], target: int) -> int:
    lo, hi = 0, len(nums)
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo
```

### Pitfalls

- Decide whether the search interval is closed `[lo, hi]` or half-open `[lo, hi)` before coding.
- In answer-space binary search, write the predicate first.

---

## 11. Heap and Priority Queue

### Problem Pattern

Use heaps for Top K, median stream, k-way merge, and shortest path.

### LC347 Top K Frequent Elements

Invariant: the heap stores at most `k` frequency candidates; the root is the weakest candidate among the current Top K.

Complexity: `O(n log k)` time, `O(n)` frequency map space and `O(k)` heap space.

```go
import "container/heap"

type Pair struct {
	num  int
	freq int
}

type PairHeap []Pair

func (h PairHeap) Len() int           { return len(h) }
func (h PairHeap) Less(i, j int) bool { return h[i].freq < h[j].freq }
func (h PairHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PairHeap) Push(x any)        { *h = append(*h, x.(Pair)) }
func (h *PairHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}

func topKFrequent(nums []int, k int) []int {
	freq := map[int]int{}
	for _, v := range nums {
		freq[v]++
	}
	h := &PairHeap{}
	heap.Init(h)
	for num, count := range freq {
		heap.Push(h, Pair{num: num, freq: count})
		if h.Len() > k {
			heap.Pop(h)
		}
	}
	ans := []int{}
	for h.Len() > 0 {
		ans = append(ans, heap.Pop(h).(Pair).num)
	}
	return ans
}
```

```python
from collections import Counter
import heapq


def top_k_frequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)
    heap: list[tuple[int, int]] = []
    for num, count in freq.items():
        heapq.heappush(heap, (count, num))
        if len(heap) > k:
            heapq.heappop(heap)
    return [num for _, num in heap]
```

### Pitfalls

- The returned order may not be sorted unless the prompt requires it.
- Bucket sort can be `O(n)` when frequencies are bounded by `n`.

---

## 12. Strings

### Problem Pattern

Use frequency maps for anagrams, center expansion or DP for palindromes, KMP for exact pattern search, and trie for prefix-heavy workloads.

### LC49 Group Anagrams

Invariant: all strings with the same character-count signature belong to the same group.

Complexity: `O(n * L)` time for lowercase English counting, `O(n * L)` space.

```go
func groupAnagrams(strs []string) [][]string {
	groups := map[[26]int][]string{}
	for _, s := range strs {
		var key [26]int
		for i := 0; i < len(s); i++ {
			key[s[i]-'a']++
		}
		groups[key] = append(groups[key], s)
	}
	ans := [][]string{}
	for _, group := range groups {
		ans = append(ans, group)
	}
	return ans
}
```

```python
from collections import defaultdict


def group_anagrams(strs: list[str]) -> list[list[str]]:
    groups: dict[tuple[int, ...], list[str]] = defaultdict(list)
    for s in strs:
        count = [0] * 26
        for ch in s:
            count[ord(ch) - ord("a")] += 1
        groups[tuple(count)].append(s)
    return list(groups.values())
```

### Pitfalls

- Count signatures assume a known alphabet; for arbitrary Unicode, sort or use a general counter key.
- For KMP, define the prefix function precisely before implementing.

---

## 13. Bit Manipulation

### Problem Pattern

Use XOR for cancellation, masks for subsets, and `n & (n - 1)` for clearing the lowest set bit.

### LC338 Counting Bits

Invariant: `bits[i] = bits[i >> 1] + (i & 1)`.

Complexity: `O(n)` time, `O(n)` space.

```go
func countBits(n int) []int {
	ans := make([]int, n+1)
	for i := 1; i <= n; i++ {
		ans[i] = ans[i>>1] + (i & 1)
	}
	return ans
}
```

```python
def count_bits(n: int) -> list[int]:
    ans = [0] * (n + 1)
    for i in range(1, n + 1):
        ans[i] = ans[i >> 1] + (i & 1)
    return ans
```

### Pitfalls

- Python integers are unbounded; Go integer width matters.
- For negative numbers, clarify whether the problem uses two's-complement fixed width.

---

## 14. Top Interview List

### Must-Solve Ten

| Problem | Pattern | Target Time |
|---|---|---|
| LC1 Two Sum | Hash table | 3 minutes |
| LC20 Valid Parentheses | Stack | 3 minutes |
| LC206 Reverse Linked List | Pointers | 5 minutes |
| LC15 3Sum | Sort + two pointers | 10 minutes |
| LC3 Longest Substring | Sliding window | 6 minutes |
| LC200 Number of Islands | DFS/BFS | 8 minutes |
| LC53 Maximum Subarray | DP/Kadane | 5 minutes |
| LC236 LCA | Tree postorder | 6 minutes |
| LC146 LRU Cache | Hash + doubly linked list | 15 minutes |
| LC42 Trapping Rain Water | Two pointers/stack | 10 minutes |

### Pattern Buckets

| Bucket | Representative Problems |
|---|---|
| Array/hash | 1, 15, 49, 56, 128 |
| Sliding window | 3, 76, 438, 567 |
| Linked list | 21, 23, 141, 142, 146, 206 |
| Tree | 98, 102, 104, 124, 236, 297 |
| Backtracking | 17, 39, 46, 51, 78, 79 |
| DP | 53, 70, 72, 300, 322, 518 |
| Graph | 200, 207, 210, 433, 695 |
| Binary search | 33, 34, 69, 153, 162, 410 |
| Heap | 23, 215, 295, 347, 373 |
| Monotonic stack | 42, 84, 739 |

---

## 15. Practice Plan

### 15.1 Single-Problem Method

1. Read the prompt and constraints.
2. Spend five minutes on examples and brute force.
3. Identify the pattern and invariant.
4. Code once in Go and once in Python.
5. Run through edge cases verbally.
6. Re-solve after one day and one week.

### 15.2 One-Week Sprint

| Day | Topics |
|---|---|
| Day 1 | Array, hash table, two pointers |
| Day 2 | Linked list, stack, queue |
| Day 3 | Tree DFS/BFS, binary search |
| Day 4 | Sliding window, backtracking |
| Day 5 | DP, greedy |
| Day 6 | Graph, heap, monotonic stack |
| Day 7 | Mock interview and Top 10 review |

### 15.3 Interview Speaking Checklist

- Clarify whether input can be empty, contain duplicates, or exceed memory.
- Say the invariant before coding.
- Use meaningful variable names.
- Test at least one normal case and one boundary case.
- Explain why the total time is linear or logarithmic, not just state it.
- Mention tradeoffs if another solution is possible.
