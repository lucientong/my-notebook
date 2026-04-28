# LeetCode 分类刷题指南

> 📅 最后更新：2026-04-27
>
> 🎯 定位：面试算法准备核心手册，覆盖高频题型 + 解题模板 + 代码实现
>
> 💻 代码语言：Go 为主，Python 为辅

---

## 📑 目录

1. [数组与双指针 ⭐⭐⭐](#1-数组与双指针-)
2. [滑动窗口 ⭐⭐⭐](#2-滑动窗口-)
3. [链表 ⭐⭐⭐](#3-链表-)
4. [栈与队列 ⭐⭐](#4-栈与队列-)
5. [二叉树 ⭐⭐⭐](#5-二叉树-)
6. [回溯 ⭐⭐⭐](#6-回溯-)
7. [动态规划 ⭐⭐⭐](#7-动态规划-)
8. [贪心 ⭐⭐](#8-贪心-)
9. [图算法 ⭐⭐](#9-图算法-)
10. [二分查找 ⭐⭐⭐](#10-二分查找-)
11. [堆/优先队列 ⭐⭐](#11-堆优先队列-)
12. [字符串 ⭐⭐](#12-字符串-)
13. [位运算 ⭐](#13-位运算-)
14. [高频面试题 TOP50 清单](#14-高频面试题-top50-清单)
15. [刷题策略与时间规划](#15-刷题策略与时间规划)

---

## 1. 数组与双指针 ⭐⭐⭐

### 1.1 核心思路总结

数组题目是面试中出现频率最高的类型。双指针是处理数组问题的核心技巧，分为：

| 指针类型 | 适用场景 | 典型题目 |
|---------|---------|---------|
| 对撞指针 | 有序数组、回文判断、盛水容器 | Two Sum II、接雨水 |
| 快慢指针 | 移除元素、去重 | 移除元素、删除重复项 |
| 左右指针 | 区间搜索、三数之和 | 三数之和、最接近的三数之和 |

**核心思想**：通过两个指针减少一层循环，将 O(n²) 优化到 O(n)。

### 1.2 解题模板

#### 对撞指针模板（Go）

```go
func twoPointerCollision(nums []int) {
    left, right := 0, len(nums)-1
    for left < right {
        // 根据条件移动指针
        if condition(nums[left], nums[right]) {
            // 找到结果
            break
        } else if needMoveLeft(nums[left], nums[right]) {
            left++
        } else {
            right--
        }
    }
}
```

#### 快慢指针模板（Go）

```go
func twoPointerFastSlow(nums []int) int {
    slow := 0
    for fast := 0; fast < len(nums); fast++ {
        if condition(nums[fast]) {
            nums[slow] = nums[fast]
            slow++
        }
    }
    return slow // slow 即为新数组长度
}
```

### 1.3 高频题详解

#### 题目1：LC1 两数之和 (Two Sum)

**难度**：Easy | **频率**：🔥🔥🔥🔥🔥

**思路**：使用哈希表存储已遍历的值及其索引，对于当前元素，查找 `target - nums[i]` 是否已在哈希表中。

```go
func twoSum(nums []int, target int) []int {
    m := make(map[int]int)
    for i, v := range nums {
        if j, ok := m[target-v]; ok {
            return []int{j, i}
        }
        m[v] = i
    }
    return nil
}
```

```python
def twoSum(nums: list[int], target: int) -> list[int]:
    seen = {}
    for i, v in enumerate(nums):
        if target - v in seen:
            return [seen[target - v], i]
        seen[v] = i
    return []
```

**复杂度分析**：
- 时间复杂度：O(n)，一次遍历
- 空间复杂度：O(n)，哈希表存储

---

#### 题目2：LC15 三数之和 (3Sum)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：排序 + 固定一个数 + 对撞双指针。关键在于去重：跳过重复的固定数和指针位置。

```go
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    result := [][]int{}
    n := len(nums)

    for i := 0; i < n-2; i++ {
        // 去重：跳过重复的固定数
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        // 剪枝：最小值已大于0，不可能有解
        if nums[i] > 0 {
            break
        }

        left, right := i+1, n-1
        for left < right {
            sum := nums[i] + nums[left] + nums[right]
            if sum == 0 {
                result = append(result, []int{nums[i], nums[left], nums[right]})
                // 去重：跳过重复的左右指针
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
    return result
}
```

```python
def threeSum(nums: list[int]) -> list[list[int]]:
    nums.sort()
    result = []
    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i - 1]:
            continue
        if nums[i] > 0:
            break
        left, right = i + 1, len(nums) - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                result.append([nums[i], nums[left], nums[right]])
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
    return result
```

**复杂度分析**：
- 时间复杂度：O(n²)，排序 O(n log n) + 双指针遍历 O(n²)
- 空间复杂度：O(log n)，排序所需空间

---

#### 题目3：LC42 接雨水 (Trapping Rain Water)

**难度**：Hard | **频率**：🔥🔥🔥🔥🔥

**思路**：双指针法。维护左右两侧的最大高度，每个位置能接的水 = min(leftMax, rightMax) - height[i]。从较矮的一侧计算并移动指针。

```go
func trap(height []int) int {
    left, right := 0, len(height)-1
    leftMax, rightMax := 0, 0
    water := 0

    for left < right {
        if height[left] < height[right] {
            if height[left] >= leftMax {
                leftMax = height[left]
            } else {
                water += leftMax - height[left]
            }
            left++
        } else {
            if height[right] >= rightMax {
                rightMax = height[right]
            } else {
                water += rightMax - height[right]
            }
            right--
        }
    }
    return water
}
```

```python
def trap(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    left_max = right_max = water = 0
    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                water += right_max - height[right]
            right -= 1
    return water
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

**其他解法对比**：

| 解法 | 时间 | 空间 | 说明 |
|------|------|------|------|
| 暴力法 | O(n²) | O(1) | 每个位置分别找左右最大值 |
| 前缀最大值 | O(n) | O(n) | 预计算 leftMax[] 和 rightMax[] |
| 双指针 | O(n) | O(1) | ✅ 最优解 |
| 单调栈 | O(n) | O(n) | 按层计算水量 |

---

#### 题目4：LC11 盛最多水的容器 (Container With Most Water)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：对撞双指针。面积 = min(height[left], height[right]) × (right - left)。每次移动较矮的一侧，因为移动较高侧不可能得到更大面积。

```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0

    for left < right {
        h := min(height[left], height[right])
        area := h * (right - left)
        if area > maxWater {
            maxWater = area
        }
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return maxWater
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

#### 题目5：LC27 移除元素 (Remove Element)

**难度**：Easy | **频率**：🔥🔥🔥

**思路**：快慢指针。快指针扫描所有元素，慢指针指向下一个要填入的位置。跳过等于 val 的元素。

```go
func removeElement(nums []int, val int) int {
    slow := 0
    for fast := 0; fast < len(nums); fast++ {
        if nums[fast] != val {
            nums[slow] = nums[fast]
            slow++
        }
    }
    return slow
}
```

```python
def removeElement(nums: list[int], val: int) -> int:
    slow = 0
    for fast in range(len(nums)):
        if nums[fast] != val:
            nums[slow] = nums[fast]
            slow += 1
    return slow
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

### 1.4 面试自查

- [ ] 两数之和的哈希表解法能秒写吗？
- [ ] 三数之和的去重逻辑能说清楚吗？
- [ ] 接雨水的双指针解法能画图讲解吗？
- [ ] 知道对撞指针和快慢指针的区别和适用场景吗？

---

## 2. 滑动窗口 ⭐⭐⭐

### 2.1 核心思路总结

滑动窗口本质是双指针的一种变体，维护一个**可变大小的窗口**在数组/字符串上滑动。

**适用场景**：
- 连续子数组/子字符串问题
- 满足某个条件的最长/最短子串
- 固定大小窗口的统计

**关键判断**：题目出现 "连续"、"子串"、"子数组" 时优先考虑滑动窗口。

| 窗口类型 | 说明 | 典型题 |
|---------|------|-------|
| 可变窗口（求最长） | 满足条件时右移右指针扩大窗口 | 最长无重复子串 |
| 可变窗口（求最短） | 满足条件时右移左指针收缩窗口 | 最小覆盖子串 |
| 固定窗口 | 窗口大小固定，同时移动左右 | 字符串排列 |

### 2.2 解题模板

#### 滑动窗口万能模板（Go）

```go
func slidingWindow(s string) int {
    window := make(map[byte]int) // 窗口内字符频次
    left := 0
    result := 0 // 根据题意初始化为 0 或 math.MaxInt32

    for right := 0; right < len(s); right++ {
        c := s[right]
        window[c]++ // 右指针字符入窗

        // 根据题意判断何时收缩窗口
        for windowNeedsShrink(window) {
            d := s[left]
            window[d]-- // 左指针字符出窗
            if window[d] == 0 {
                delete(window, d)
            }
            left++
        }

        // 在此处或收缩循环中更新结果
        result = max(result, right-left+1) // 求最长
        // result = min(result, right-left+1) // 求最短
    }
    return result
}
```

#### 滑动窗口万能模板（Python）

```python
def sliding_window(s: str) -> int:
    window = defaultdict(int)
    left = 0
    result = 0  # 或 float('inf')

    for right in range(len(s)):
        c = s[right]
        window[c] += 1  # 入窗

        while window_needs_shrink(window):
            d = s[left]
            window[d] -= 1  # 出窗
            if window[d] == 0:
                del window[d]
            left += 1

        result = max(result, right - left + 1)  # 求最长
    return result
```

### 2.3 高频题详解

#### 题目1：LC3 无重复字符的最长子串 (Longest Substring Without Repeating Characters)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：维护窗口内无重复字符。当新字符已在窗口中时，收缩左边界直到无重复。

```go
func lengthOfLongestSubstring(s string) int {
    window := make(map[byte]int)
    left := 0
    result := 0

    for right := 0; right < len(s); right++ {
        c := s[right]
        window[c]++

        // 窗口内有重复字符，收缩
        for window[c] > 1 {
            d := s[left]
            window[d]--
            left++
        }

        if right-left+1 > result {
            result = right - left + 1
        }
    }
    return result
}
```

```python
def lengthOfLongestSubstring(s: str) -> int:
    window = {}
    left = result = 0
    for right, c in enumerate(s):
        if c in window and window[c] >= left:
            left = window[c] + 1
        window[c] = right
        result = max(result, right - left + 1)
    return result
```

**复杂度分析**：
- 时间复杂度：O(n)，每个字符最多被访问两次
- 空间复杂度：O(min(m, n))，m 为字符集大小

---

#### 题目2：LC76 最小覆盖子串 (Minimum Window Substring)

**难度**：Hard | **频率**：🔥🔥🔥🔥🔥

**思路**：
1. 用 `need` 记录目标字符频次，`window` 记录窗口内字符频次
2. 扩展右边界直到窗口包含所有目标字符
3. 收缩左边界找最小窗口
4. 用 `valid` 计数器记录满足条件的字符种类数

```go
func minWindow(s string, t string) string {
    need := make(map[byte]int)
    window := make(map[byte]int)
    for i := 0; i < len(t); i++ {
        need[t[i]]++
    }

    left, right := 0, 0
    valid := 0           // 满足条件的字符种类数
    start, length := 0, len(s)+1 // 记录最小窗口

    for right < len(s) {
        c := s[right]
        right++
        // 入窗更新
        if _, ok := need[c]; ok {
            window[c]++
            if window[c] == need[c] {
                valid++
            }
        }

        // 收缩条件：所有目标字符都已覆盖
        for valid == len(need) {
            // 更新最小窗口
            if right-left < length {
                start = left
                length = right - left
            }
            d := s[left]
            left++
            // 出窗更新
            if _, ok := need[d]; ok {
                if window[d] == need[d] {
                    valid--
                }
                window[d]--
            }
        }
    }

    if length == len(s)+1 {
        return ""
    }
    return s[start : start+length]
}
```

```python
from collections import Counter

def minWindow(s: str, t: str) -> str:
    need = Counter(t)
    window = defaultdict(int)
    left = valid = 0
    start, length = 0, float('inf')

    for right in range(len(s)):
        c = s[right]
        if c in need:
            window[c] += 1
            if window[c] == need[c]:
                valid += 1

        while valid == len(need):
            if right - left + 1 < length:
                start = left
                length = right - left + 1
            d = s[left]
            left += 1
            if d in need:
                if window[d] == need[d]:
                    valid -= 1
                window[d] -= 1

    return "" if length == float('inf') else s[start:start + length]
```

**复杂度分析**：
- 时间复杂度：O(|S| + |T|)
- 空间复杂度：O(|S| + |T|)

---

#### 题目3：LC567 字符串的排列 (Permutation in String)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：固定窗口大小为 s1 的长度，判断窗口内字符频次是否与 s1 相同。

```go
func checkInclusion(s1 string, s2 string) bool {
    if len(s1) > len(s2) {
        return false
    }

    need := make(map[byte]int)
    window := make(map[byte]int)
    for i := 0; i < len(s1); i++ {
        need[s1[i]]++
    }

    left := 0
    valid := 0

    for right := 0; right < len(s2); right++ {
        c := s2[right]
        if _, ok := need[c]; ok {
            window[c]++
            if window[c] == need[c] {
                valid++
            }
        }

        // 窗口大小超过 s1 长度时收缩
        for right-left+1 > len(s1) {
            d := s2[left]
            if _, ok := need[d]; ok {
                if window[d] == need[d] {
                    valid--
                }
                window[d]--
            }
            left++
        }

        if valid == len(need) {
            return true
        }
    }
    return false
}
```

**复杂度分析**：
- 时间复杂度：O(n)，n 为 s2 长度
- 空间复杂度：O(1)，最多 26 个字符

---

#### 题目4：LC438 找到字符串中所有字母异位词

**难度**：Medium | **频率**：🔥🔥🔥

**思路**：与 LC567 几乎相同，只是需要收集所有满足条件的起始索引。

```go
func findAnagrams(s string, p string) []int {
    if len(p) > len(s) {
        return nil
    }

    need := make(map[byte]int)
    window := make(map[byte]int)
    for i := 0; i < len(p); i++ {
        need[p[i]]++
    }

    left, valid := 0, 0
    result := []int{}

    for right := 0; right < len(s); right++ {
        c := s[right]
        if _, ok := need[c]; ok {
            window[c]++
            if window[c] == need[c] {
                valid++
            }
        }

        for right-left+1 > len(p) {
            d := s[left]
            if _, ok := need[d]; ok {
                if window[d] == need[d] {
                    valid--
                }
                window[d]--
            }
            left++
        }

        if valid == len(need) {
            result = append(result, left)
        }
    }
    return result
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

### 2.4 滑动窗口解题模板总结

```
步骤1：定义窗口数据结构（哈希表/变量）
步骤2：右指针扩展窗口 → 更新窗口状态
步骤3：判断收缩条件 → 左指针收缩 → 更新窗口状态
步骤4：在合适时机更新答案（收缩前/收缩后/收缩中）
```

| 题目 | 窗口类型 | 收缩条件 | 更新时机 |
|------|---------|---------|---------|
| LC3 最长无重复子串 | 可变(最长) | 有重复字符 | 收缩后 |
| LC76 最小覆盖子串 | 可变(最短) | 包含所有目标 | 收缩中 |
| LC567 字符串排列 | 固定大小 | 窗口超过目标长度 | 收缩后 |
| LC209 最小子数组 | 可变(最短) | 和 >= target | 收缩中 |

### 2.5 面试自查

- [ ] 能默写滑动窗口模板吗？
- [ ] 知道何时在收缩前/收缩后/收缩中更新答案吗？
- [ ] 最小覆盖子串的 valid 计数器逻辑能讲清楚吗？

---

## 3. 链表 ⭐⭐⭐

### 3.1 核心思路总结

链表问题的核心技巧：

| 技巧 | 说明 | 典型题 |
|------|------|-------|
| 虚拟头节点 | 简化边界处理（头节点删除/插入） | 合并链表、删除节点 |
| 快慢指针 | 找中点、检测环、倒数第K个 | 环形链表、链表中点 |
| 递归 | 链表天然递归结构 | 反转链表、合并链表 |
| 哨兵节点 | 分离再合并 | 分隔链表 |

### 3.2 解题模板

#### 链表定义（Go）

```go
type ListNode struct {
    Val  int
    Next *ListNode
}
```

#### 虚拟头节点模板

```go
func withDummy(head *ListNode) *ListNode {
    dummy := &ListNode{Next: head}
    curr := dummy
    // 操作 curr.Next ...
    return dummy.Next
}
```

#### 快慢指针找中点

```go
func findMiddle(head *ListNode) *ListNode {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    return slow // 奇数个返回中点，偶数个返回后半段第一个
}
```

### 3.3 高频题详解

#### 题目1：LC206 反转链表 (Reverse Linked List)

**难度**：Easy | **频率**：🔥🔥🔥🔥🔥

**思路（迭代）**：维护 prev、curr、next 三个指针，逐个反转指向。

```go
// 迭代法
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head

    for curr != nil {
        next := curr.Next // 保存下一个
        curr.Next = prev  // 反转指向
        prev = curr       // prev 前进
        curr = next       // curr 前进
    }
    return prev
}

// 递归法
func reverseListRecursive(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseListRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```

```python
def reverseList(head):
    prev, curr = None, head
    while curr:
        curr.next, prev, curr = prev, curr, curr.next
    return prev
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：迭代 O(1)，递归 O(n)

---

#### 题目2：LC23 合并K个升序链表 (Merge k Sorted Lists)

**难度**：Hard | **频率**：🔥🔥🔥🔥🔥

**思路**：
- **方法1**：最小堆（优先队列），每次取最小节点
- **方法2**：分治合并，两两合并后递归

```go
// 方法1：最小堆
import "container/heap"

type MinHeap []*ListNode

func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool   { return h[i].Val < h[j].Val }
func (h MinHeap) Swap(i, j int)        { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{})  { *h = append(*h, x.(*ListNode)) }
func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func mergeKLists(lists []*ListNode) *ListNode {
    h := &MinHeap{}
    heap.Init(h)

    // 将所有链表头节点入堆
    for _, l := range lists {
        if l != nil {
            heap.Push(h, l)
        }
    }

    dummy := &ListNode{}
    curr := dummy

    for h.Len() > 0 {
        node := heap.Pop(h).(*ListNode)
        curr.Next = node
        curr = curr.Next
        if node.Next != nil {
            heap.Push(h, node.Next)
        }
    }
    return dummy.Next
}

// 方法2：分治合并
func mergeKListsDivide(lists []*ListNode) *ListNode {
    if len(lists) == 0 {
        return nil
    }
    if len(lists) == 1 {
        return lists[0]
    }
    mid := len(lists) / 2
    left := mergeKListsDivide(lists[:mid])
    right := mergeKListsDivide(lists[mid:])
    return mergeTwoLists(left, right)
}

func mergeTwoLists(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    curr := dummy
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            curr.Next = l1
            l1 = l1.Next
        } else {
            curr.Next = l2
            l2 = l2.Next
        }
        curr = curr.Next
    }
    if l1 != nil {
        curr.Next = l1
    } else {
        curr.Next = l2
    }
    return dummy.Next
}
```

**复杂度分析**：

| 方法 | 时间 | 空间 |
|------|------|------|
| 最小堆 | O(N log k) | O(k) |
| 分治合并 | O(N log k) | O(log k) |
| 逐一合并 | O(Nk) | O(1) |

N = 所有节点总数，k = 链表个数

---

#### 题目3：LC142 环形链表 II (Linked List Cycle II)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：快慢指针相遇后，将一个指针重置到 head，然后两个指针同速前进，再次相遇点即为环入口。

**数学证明**：设头到环入口距离 a，环入口到相遇点 b，相遇点到环入口 c。
快指针走 a+b+c+b = 2(a+b) → a = c

```go
func detectCycle(head *ListNode) *ListNode {
    slow, fast := head, head

    // 第一阶段：快慢指针找相遇点
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            // 第二阶段：找环入口
            slow = head
            for slow != fast {
                slow = slow.Next
                fast = fast.Next
            }
            return slow
        }
    }
    return nil // 无环
}
```

```python
def detectCycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            slow = head
            while slow != fast:
                slow = slow.next
                fast = fast.next
            return slow
    return None
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

#### 题目4：LC146 LRU 缓存 (LRU Cache)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：哈希表 + 双向链表。哈希表 O(1) 查找，双向链表 O(1) 插入删除，维护访问顺序。

```go
type LRUCache struct {
    capacity int
    cache    map[int]*DLinkedNode
    head     *DLinkedNode // 虚拟头
    tail     *DLinkedNode // 虚拟尾
}

type DLinkedNode struct {
    key, value int
    prev, next *DLinkedNode
}

func Constructor(capacity int) LRUCache {
    head := &DLinkedNode{}
    tail := &DLinkedNode{}
    head.next = tail
    tail.prev = head
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*DLinkedNode),
        head:     head,
        tail:     tail,
    }
}

func (l *LRUCache) Get(key int) int {
    if node, ok := l.cache[key]; ok {
        l.moveToHead(node)
        return node.value
    }
    return -1
}

func (l *LRUCache) Put(key int, value int) {
    if node, ok := l.cache[key]; ok {
        node.value = value
        l.moveToHead(node)
    } else {
        node := &DLinkedNode{key: key, value: value}
        l.cache[key] = node
        l.addToHead(node)
        if len(l.cache) > l.capacity {
            removed := l.removeTail()
            delete(l.cache, removed.key)
        }
    }
}

func (l *LRUCache) addToHead(node *DLinkedNode) {
    node.prev = l.head
    node.next = l.head.next
    l.head.next.prev = node
    l.head.next = node
}

func (l *LRUCache) removeNode(node *DLinkedNode) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (l *LRUCache) moveToHead(node *DLinkedNode) {
    l.removeNode(node)
    l.addToHead(node)
}

func (l *LRUCache) removeTail() *DLinkedNode {
    node := l.tail.prev
    l.removeNode(node)
    return node
}
```

**复杂度分析**：
- Get / Put 时间复杂度：O(1)
- 空间复杂度：O(capacity)

---

#### 题目5：LC148 排序链表 (Sort List)

**难度**：Medium | **频率**：🔥🔥🔥

**思路**：归并排序。找中点 → 断开 → 递归排序左右 → 合并。

```go
func sortList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }

    // 找中点并断开
    slow, fast := head, head.Next
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    mid := slow.Next
    slow.Next = nil

    // 递归排序
    left := sortList(head)
    right := sortList(mid)

    // 合并
    return mergeTwoLists(left, right)
}
```

**复杂度分析**：
- 时间复杂度：O(n log n)
- 空间复杂度：O(log n)，递归栈

### 3.4 面试自查

- [ ] 反转链表的迭代和递归都能写吗？
- [ ] LRU 缓存能在 20 分钟内手写完整吗？
- [ ] 环形链表 II 的数学证明能讲清楚吗？
- [ ] 链表归并排序的找中点为什么 fast 从 head.Next 开始？

---

## 4. 栈与队列 ⭐⭐

### 4.1 核心思路总结

| 数据结构 | 核心特点 | 常见应用 |
|---------|---------|---------|
| 栈 | LIFO | 括号匹配、单调栈、DFS |
| 队列 | FIFO | BFS、滑动窗口 |
| 单调栈 | 栈内元素单调 | 下一个更大/更小元素 |
| 单调队列 | 队列内元素单调 | 滑动窗口最值 |

### 4.2 解题模板

#### 单调栈模板（找下一个更大元素）

```go
func nextGreater(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    stack := []int{} // 存放索引

    // 从后往前遍历
    for i := n - 1; i >= 0; i-- {
        // 弹出所有不大于当前元素的
        for len(stack) > 0 && nums[stack[len(stack)-1]] <= nums[i] {
            stack = stack[:len(stack)-1]
        }
        if len(stack) == 0 {
            result[i] = -1 // 没有更大的
        } else {
            result[i] = stack[len(stack)-1]
        }
        stack = append(stack, i)
    }
    return result
}
```

#### 单调队列模板

```go
type MonotonicQueue struct {
    deque []int // 双端队列，存索引
}

func (q *MonotonicQueue) Push(nums []int, i int) {
    // 保持队列单调递减
    for len(q.deque) > 0 && nums[q.deque[len(q.deque)-1]] <= nums[i] {
        q.deque = q.deque[:len(q.deque)-1]
    }
    q.deque = append(q.deque, i)
}

func (q *MonotonicQueue) Pop(windowLeft int) {
    // 队头超出窗口范围则弹出
    if len(q.deque) > 0 && q.deque[0] < windowLeft {
        q.deque = q.deque[1:]
    }
}

func (q *MonotonicQueue) Max() int {
    return q.deque[0] // 队头就是最大值的索引
}
```

### 4.3 高频题详解

#### 题目1：LC20 有效的括号 (Valid Parentheses)

**难度**：Easy | **频率**：🔥🔥🔥🔥🔥

**思路**：遇到左括号入栈，遇到右括号检查栈顶是否匹配。

```go
func isValid(s string) bool {
    stack := []byte{}
    pairs := map[byte]byte{
        ')': '(',
        ']': '[',
        '}': '{',
    }

    for i := 0; i < len(s); i++ {
        c := s[i]
        if c == '(' || c == '[' || c == '{' {
            stack = append(stack, c)
        } else {
            if len(stack) == 0 || stack[len(stack)-1] != pairs[c] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    return len(stack) == 0
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(n)

---

#### 题目2：LC155 最小栈 (Min Stack)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：用两个栈，一个正常栈，一个辅助栈存储当前最小值。

```go
type MinStack struct {
    stack    []int
    minStack []int
}

func Constructor() MinStack {
    return MinStack{
        stack:    []int{},
        minStack: []int{math.MaxInt64},
    }
}

func (s *MinStack) Push(val int) {
    s.stack = append(s.stack, val)
    top := s.minStack[len(s.minStack)-1]
    if val < top {
        s.minStack = append(s.minStack, val)
    } else {
        s.minStack = append(s.minStack, top)
    }
}

func (s *MinStack) Pop() {
    s.stack = s.stack[:len(s.stack)-1]
    s.minStack = s.minStack[:len(s.minStack)-1]
}

func (s *MinStack) Top() int {
    return s.stack[len(s.stack)-1]
}

func (s *MinStack) GetMin() int {
    return s.minStack[len(s.minStack)-1]
}
```

**复杂度分析**：
- 所有操作时间复杂度：O(1)
- 空间复杂度：O(n)

---

#### 题目3：LC739 每日温度 (Daily Temperatures)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：单调递减栈。从前往后遍历，栈中存温度索引，遇到更高温度时弹栈计算等待天数。

```go
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    result := make([]int, n)
    stack := []int{} // 单调递减栈，存索引

    for i := 0; i < n; i++ {
        for len(stack) > 0 && temperatures[i] > temperatures[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[top] = i - top
        }
        stack = append(stack, i)
    }
    return result
}
```

```python
def dailyTemperatures(temperatures: list[int]) -> list[int]:
    n = len(temperatures)
    result = [0] * n
    stack = []  # 单调递减栈
    for i in range(n):
        while stack and temperatures[i] > temperatures[stack[-1]]:
            top = stack.pop()
            result[top] = i - top
        stack.append(i)
    return result
```

**复杂度分析**：
- 时间复杂度：O(n)，每个元素最多入栈出栈各一次
- 空间复杂度：O(n)

---

#### 题目4：LC239 滑动窗口最大值 (Sliding Window Maximum)

**难度**：Hard | **频率**：🔥🔥🔥🔥

**思路**：单调递减队列。维护一个双端队列，队头始终是当前窗口最大值的索引。

```go
func maxSlidingWindow(nums []int, k int) []int {
    deque := []int{}   // 单调递减队列，存索引
    result := []int{}

    for i := 0; i < len(nums); i++ {
        // 移除队头超出窗口的元素
        if len(deque) > 0 && deque[0] < i-k+1 {
            deque = deque[1:]
        }
        // 保持单调递减：移除队尾所有小于当前元素的
        for len(deque) > 0 && nums[deque[len(deque)-1]] <= nums[i] {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)

        // 窗口形成后记录最大值
        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(k)

---

#### 题目5：LC84 柱状图中最大的矩形 (Largest Rectangle in Histogram)

**难度**：Hard | **频率**：🔥🔥🔥🔥

**思路**：单调递增栈。对每个柱子找左右第一个更矮的柱子，确定以当前高度为矩形高时的最大宽度。

```go
func largestRectangleArea(heights []int) int {
    n := len(heights)
    stack := []int{} // 单调递增栈
    maxArea := 0

    for i := 0; i <= n; i++ {
        // 在末尾虚拟添加高度0，确保所有柱子都能出栈
        h := 0
        if i < n {
            h = heights[i]
        }

        for len(stack) > 0 && h < heights[stack[len(stack)-1]] {
            top := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            width := i
            if len(stack) > 0 {
                width = i - stack[len(stack)-1] - 1
            }
            area := heights[top] * width
            if area > maxArea {
                maxArea = area
            }
        }
        stack = append(stack, i)
    }
    return maxArea
}
```

```python
def largestRectangleArea(heights: list[int]) -> int:
    stack = []
    max_area = 0
    heights.append(0)  # 哨兵
    for i, h in enumerate(heights):
        while stack and h < heights[stack[-1]]:
            top = stack.pop()
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, heights[top] * width)
        stack.append(i)
    heights.pop()  # 还原
    return max_area
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(n)

### 4.4 面试自查

- [ ] 单调栈的核心思想能用一句话概括吗？
- [ ] 柱状图最大矩形为什么要在末尾加 0？
- [ ] 滑动窗口最大值的单调队列和单调栈有什么区别？

---

## 5. 二叉树 ⭐⭐⭐

### 5.1 核心思路总结

二叉树问题的两大思维模式：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **遍历** | 用 traverse 函数遍历整棵树，用外部变量记录结果 | 前/中/后序遍历变形 |
| **分解** | 将问题分解为子问题，利用返回值得到结果 | 最大深度、对称性判断 |

> **核心心法**：是否能通过遍历一遍得到答案？还是需要通过子问题（子树）的答案推导出原问题的答案？

### 5.2 遍历模板

#### 递归遍历（Go）

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// 前序遍历：根 → 左 → 右
func preorder(root *TreeNode, result *[]int) {
    if root == nil {
        return
    }
    *result = append(*result, root.Val)
    preorder(root.Left, result)
    preorder(root.Right, result)
}

// 中序遍历：左 → 根 → 右
func inorder(root *TreeNode, result *[]int) {
    if root == nil {
        return
    }
    inorder(root.Left, result)
    *result = append(*result, root.Val)
    inorder(root.Right, result)
}

// 后序遍历：左 → 右 → 根
func postorder(root *TreeNode, result *[]int) {
    if root == nil {
        return
    }
    postorder(root.Left, result)
    postorder(root.Right, result)
    *result = append(*result, root.Val)
}
```

#### 迭代遍历——统一模板（Go）

```go
// 前序遍历（迭代）
func preorderIterative(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    result := []int{}
    stack := []*TreeNode{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    return result
}

// 中序遍历（迭代）
func inorderIterative(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    curr := root

    for curr != nil || len(stack) > 0 {
        for curr != nil {
            stack = append(stack, curr)
            curr = curr.Left
        }
        curr = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, curr.Val)
        curr = curr.Right
    }
    return result
}

// 层序遍历（BFS）
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        size := len(queue)
        level := []int{}
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}
```

### 5.3 高频题详解

#### 题目1：LC104 二叉树的最大深度 (Maximum Depth of Binary Tree)

**难度**：Easy | **频率**：🔥🔥🔥🔥🔥

**思路**：分解子问题 —— 树的最大深度 = max(左子树深度, 右子树深度) + 1

```go
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
def maxDepth(root) -> int:
    if not root:
        return 0
    return max(maxDepth(root.left), maxDepth(root.right)) + 1
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(h)，h 为树高，最坏 O(n)

---

#### 题目2：LC98 验证二叉搜索树 (Validate Binary Search Tree)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：利用 BST 的中序遍历是升序的性质。或者递归传递 (min, max) 范围。

```go
// 方法1：递归 + 范围限制
func isValidBST(root *TreeNode) bool {
    return validate(root, math.MinInt64, math.MaxInt64)
}

func validate(node *TreeNode, min, max int) bool {
    if node == nil {
        return true
    }
    if node.Val <= min || node.Val >= max {
        return false
    }
    return validate(node.Left, min, node.Val) &&
           validate(node.Right, node.Val, max)
}

// 方法2：中序遍历
func isValidBSTInorder(root *TreeNode) bool {
    var prev *int
    var inorder func(node *TreeNode) bool
    inorder = func(node *TreeNode) bool {
        if node == nil {
            return true
        }
        if !inorder(node.Left) {
            return false
        }
        if prev != nil && node.Val <= *prev {
            return false
        }
        prev = &node.Val
        return inorder(node.Right)
    }
    return inorder(root)
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(h)

---

#### 题目3：LC236 二叉树的最近公共祖先 (Lowest Common Ancestor)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：后序遍历。如果当前节点是 p 或 q，返回当前节点。如果左右子树分别找到 p 和 q，当前节点就是 LCA。

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    // base case
    if root == nil || root == p || root == q {
        return root
    }

    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)

    // p 和 q 分别在左右子树
    if left != nil && right != nil {
        return root
    }
    // 都在左子树或都在右子树
    if left != nil {
        return left
    }
    return right
}
```

```python
def lowestCommonAncestor(root, p, q):
    if not root or root == p or root == q:
        return root
    left = lowestCommonAncestor(root.left, p, q)
    right = lowestCommonAncestor(root.right, p, q)
    if left and right:
        return root
    return left if left else right
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(h)

---

#### 题目4：LC297 二叉树的序列化与反序列化 (Serialize and Deserialize Binary Tree)

**难度**：Hard | **频率**：🔥🔥🔥🔥

**思路**：前序遍历序列化，用特殊字符标记空节点。反序列化时按相同顺序重建。

```go
type Codec struct{}

func (c *Codec) serialize(root *TreeNode) string {
    if root == nil {
        return "#"
    }
    return strconv.Itoa(root.Val) + "," +
           c.serialize(root.Left) + "," +
           c.serialize(root.Right)
}

func (c *Codec) deserialize(data string) *TreeNode {
    nodes := strings.Split(data, ",")
    idx := 0
    var build func() *TreeNode
    build = func() *TreeNode {
        if idx >= len(nodes) || nodes[idx] == "#" {
            idx++
            return nil
        }
        val, _ := strconv.Atoi(nodes[idx])
        idx++
        node := &TreeNode{Val: val}
        node.Left = build()
        node.Right = build()
        return node
    }
    return build()
}
```

**复杂度分析**：
- 序列化时间：O(n)
- 反序列化时间：O(n)
- 空间复杂度：O(n)

---

#### 题目5：LC437 路径总和 III (Path Sum III)

**难度**：Medium | **频率**：🔥🔥🔥

**思路**：前缀和 + 哈希表。类似数组的子数组和问题，在树上做前缀和。

```go
func pathSum(root *TreeNode, targetSum int) int {
    prefixSum := map[int64]int{0: 1} // 前缀和 -> 出现次数
    var dfs func(node *TreeNode, currSum int64) int
    dfs = func(node *TreeNode, currSum int64) int {
        if node == nil {
            return 0
        }
        currSum += int64(node.Val)
        count := prefixSum[currSum-int64(targetSum)]
        prefixSum[currSum]++
        count += dfs(node.Left, currSum)
        count += dfs(node.Right, currSum)
        prefixSum[currSum]-- // 回溯
        return count
    }
    return dfs(root, 0)
}
```

```python
def pathSum(root, targetSum: int) -> int:
    prefix = {0: 1}

    def dfs(node, curr_sum):
        if not node:
            return 0
        curr_sum += node.val
        count = prefix.get(curr_sum - targetSum, 0)
        prefix[curr_sum] = prefix.get(curr_sum, 0) + 1
        count += dfs(node.left, curr_sum)
        count += dfs(node.right, curr_sum)
        prefix[curr_sum] -= 1  # 回溯
        return count

    return dfs(root, 0)
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(n)

### 5.4 面试自查

- [ ] 三种遍历的递归和迭代都能写吗？
- [ ] 能用一句话区分"遍历思维"和"分解思维"吗？
- [ ] LCA 的递归逻辑能画图讲清楚吗？
- [ ] 路径总和 III 的前缀和为什么需要回溯？

---

## 6. 回溯 ⭐⭐⭐

### 6.1 核心思路总结

回溯 = 暴力穷举 + 剪枝。本质是在决策树上做 DFS。

**回溯三要素**：
1. **路径**：已经做出的选择
2. **选择列表**：当前可以做的选择
3. **结束条件**：到达决策树底层，路径加入结果

**回溯 vs 动态规划**：
| | 回溯 | 动态规划 |
|--|------|---------|
| 求什么 | 所有解/排列/组合 | 最优解/方案数 |
| 重叠子问题 | 无（或不利用） | 有（且利用） |
| 时间复杂度 | 指数级 | 多项式级 |

### 6.2 解题模板

```go
var result [][]int

func backtrack(path []int, choices []int, /* 其他参数 */) {
    // 满足结束条件
    if isEnd(path) {
        // 注意：必须拷贝 path
        tmp := make([]int, len(path))
        copy(tmp, path)
        result = append(result, tmp)
        return
    }

    for i, choice := range choices {
        // 剪枝
        if !isValid(choice) {
            continue
        }
        // 做选择
        path = append(path, choice)
        // 递归（注意下一层的选择列表）
        backtrack(path, nextChoices(choices, i), /* ... */)
        // 撤销选择
        path = path[:len(path)-1]
    }
}
```

```python
def backtrack(path, choices):
    if is_end(path):
        result.append(path[:])  # 拷贝
        return
    for i, choice in enumerate(choices):
        if not is_valid(choice):
            continue
        path.append(choice)
        backtrack(path, next_choices(choices, i))
        path.pop()  # 撤销
```

### 6.3 高频题详解

#### 题目1：LC46 全排列 (Permutations)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：标准回溯。用 visited 数组标记已使用的元素。

```go
func permute(nums []int) [][]int {
    result := [][]int{}
    path := []int{}
    used := make([]bool, len(nums))

    var backtrack func()
    backtrack = func() {
        if len(path) == len(nums) {
            tmp := make([]int, len(path))
            copy(tmp, path)
            result = append(result, tmp)
            return
        }

        for i := 0; i < len(nums); i++ {
            if used[i] {
                continue
            }
            used[i] = true
            path = append(path, nums[i])
            backtrack()
            path = path[:len(path)-1]
            used[i] = false
        }
    }

    backtrack()
    return result
}
```

```python
def permute(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(path, used):
        if len(path) == len(nums):
            result.append(path[:])
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            used[i] = True
            path.append(nums[i])
            backtrack(path, used)
            path.pop()
            used[i] = False

    backtrack([], [False] * len(nums))
    return result
```

**复杂度分析**：
- 时间复杂度：O(n × n!)
- 空间复杂度：O(n)

---

#### 题目2：LC39 组合总和 (Combination Sum)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：允许重复选择，下一轮从当前索引开始（不是 i+1）。排序后可剪枝。

```go
func combinationSum(candidates []int, target int) [][]int {
    sort.Ints(candidates)
    result := [][]int{}
    path := []int{}

    var backtrack func(start, remain int)
    backtrack = func(start, remain int) {
        if remain == 0 {
            tmp := make([]int, len(path))
            copy(tmp, path)
            result = append(result, tmp)
            return
        }

        for i := start; i < len(candidates); i++ {
            if candidates[i] > remain {
                break // 剪枝：排序后后面的更大，不可能满足
            }
            path = append(path, candidates[i])
            backtrack(i, remain-candidates[i]) // 可重复选，从 i 开始
            path = path[:len(path)-1]
        }
    }

    backtrack(0, target)
    return result
}
```

**复杂度分析**：
- 时间复杂度：O(n^(T/M))，T 为 target，M 为最小候选值
- 空间复杂度：O(T/M)，递归深度

---

#### 题目3：LC51 N 皇后 (N-Queens)

**难度**：Hard | **频率**：🔥🔥🔥🔥

**思路**：逐行放置皇后，用三个集合记录列、主对角线、副对角线的占用情况。

```go
func solveNQueens(n int) [][]string {
    result := [][]string{}
    board := make([][]byte, n)
    for i := range board {
        board[i] = make([]byte, n)
        for j := range board[i] {
            board[i][j] = '.'
        }
    }

    cols := make(map[int]bool)      // 列
    diag1 := make(map[int]bool)     // 主对角线 (row - col)
    diag2 := make(map[int]bool)     // 副对角线 (row + col)

    var backtrack func(row int)
    backtrack = func(row int) {
        if row == n {
            solution := make([]string, n)
            for i := range board {
                solution[i] = string(board[i])
            }
            result = append(result, solution)
            return
        }

        for col := 0; col < n; col++ {
            if cols[col] || diag1[row-col] || diag2[row+col] {
                continue
            }
            board[row][col] = 'Q'
            cols[col] = true
            diag1[row-col] = true
            diag2[row+col] = true

            backtrack(row + 1)

            board[row][col] = '.'
            cols[col] = false
            diag1[row-col] = false
            diag2[row+col] = false
        }
    }

    backtrack(0)
    return result
}
```

**复杂度分析**：
- 时间复杂度：O(n!)
- 空间复杂度：O(n)

---

#### 题目4：LC78 子集 (Subsets)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：每个元素都有"选"或"不选"两种选择。或用回溯，每次递归都记录当前 path。

```go
func subsets(nums []int) [][]int {
    result := [][]int{}
    path := []int{}

    var backtrack func(start int)
    backtrack = func(start int) {
        // 每个节点都是一个子集
        tmp := make([]int, len(path))
        copy(tmp, path)
        result = append(result, tmp)

        for i := start; i < len(nums); i++ {
            path = append(path, nums[i])
            backtrack(i + 1)
            path = path[:len(path)-1]
        }
    }

    backtrack(0)
    return result
}
```

```python
def subsets(nums: list[int]) -> list[list[int]]:
    result = []
    def backtrack(start, path):
        result.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()
    backtrack(0, [])
    return result
```

**复杂度分析**：
- 时间复杂度：O(n × 2^n)
- 空间复杂度：O(n)

---

#### 题目5：LC79 单词搜索 (Word Search)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：在网格上做回溯 DFS，上下左右四个方向搜索。

```go
func exist(board [][]byte, word string) bool {
    m, n := len(board), len(board[0])
    dirs := [4][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

    var backtrack func(i, j, k int) bool
    backtrack = func(i, j, k int) bool {
        if k == len(word) {
            return true
        }
        if i < 0 || i >= m || j < 0 || j >= n || board[i][j] != word[k] {
            return false
        }

        temp := board[i][j]
        board[i][j] = '#' // 标记已访问

        for _, d := range dirs {
            if backtrack(i+d[0], j+d[1], k+1) {
                return true
            }
        }

        board[i][j] = temp // 回溯
        return false
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if backtrack(i, j, 0) {
                return true
            }
        }
    }
    return false
}
```

**复杂度分析**：
- 时间复杂度：O(m × n × 3^L)，L 为单词长度（每步最多3个方向）
- 空间复杂度：O(L)，递归深度

### 6.4 回溯问题分类对比

| 题型 | 起始索引 | 可否重复选 | 关键去重 |
|------|---------|----------|---------|
| 子集 | start | 否 | 自然去重 |
| 组合 | start | 否 | 自然去重 |
| 组合总和(可重复) | i (当前) | 是 | 排序+剪枝 |
| 组合总和II(有重复元素) | i+1 | 否 | `i>start && nums[i]==nums[i-1]` |
| 全排列 | 0 | 否 | used 数组 |
| 全排列II(有重复) | 0 | 否 | used + 排序去重 |

### 6.5 面试自查

- [ ] 回溯模板能默写吗？
- [ ] 子集、组合、排列的区别能一句话说清吗？
- [ ] N 皇后的对角线判断逻辑能讲清楚吗？
- [ ] 什么时候从 i 开始，什么时候从 i+1 开始？

---

## 7. 动态规划 ⭐⭐⭐

### 7.1 核心思路总结

**动态规划五步法**：
1. **定义状态**：dp[i] 或 dp[i][j] 代表什么？
2. **状态转移方程**：dp[i] 和 dp[i-1] 的关系
3. **初始化**：base case
4. **遍历顺序**：从小到大 or 从大到小
5. **返回值**：dp[n] 还是 dp[n-1]

**DP 分类**：

| 类型 | 典型题目 | 状态定义 |
|------|---------|---------|
| 线性 DP | 爬楼梯、打家劫舍 | dp[i] = 前 i 个元素的最优解 |
| 序列 DP | LIS、编辑距离 | dp[i][j] = 两个序列前 i,j 的最优 |
| 背包 DP | 0-1 背包、零钱兑换 | dp[i][w] = 前 i 个物品容量 w |
| 区间 DP | 戳气球、回文子串 | dp[i][j] = 区间 [i,j] 的最优 |
| 树形 DP | 打家劫舍 III | dp[node] = 以 node 为根的最优 |

### 7.2 状态转移模板

#### 一维 DP

```go
// dp[i] 依赖 dp[i-1] 和 dp[i-2]
dp := make([]int, n+1)
dp[0] = base0
dp[1] = base1
for i := 2; i <= n; i++ {
    dp[i] = transition(dp[i-1], dp[i-2])
}
return dp[n]
```

#### 二维 DP

```go
// dp[i][j] 依赖 dp[i-1][j], dp[i][j-1], dp[i-1][j-1]
dp := make([][]int, m+1)
for i := range dp {
    dp[i] = make([]int, n+1)
}
// 初始化第0行第0列...
for i := 1; i <= m; i++ {
    for j := 1; j <= n; j++ {
        dp[i][j] = transition(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
    }
}
return dp[m][n]
```

#### 背包模板

```go
// 0-1 背包
dp := make([]int, capacity+1)
for i := 0; i < n; i++ {
    for w := capacity; w >= weight[i]; w-- { // 逆序！
        dp[w] = max(dp[w], dp[w-weight[i]]+value[i])
    }
}

// 完全背包
dp := make([]int, capacity+1)
for i := 0; i < n; i++ {
    for w := weight[i]; w <= capacity; w++ { // 正序！
        dp[w] = max(dp[w], dp[w-weight[i]]+value[i])
    }
}
```

### 7.3 高频题详解

#### 题目1：LC70 爬楼梯 (Climbing Stairs)

**难度**：Easy | **频率**：🔥🔥🔥🔥🔥

**思路**：dp[i] = dp[i-1] + dp[i-2]，本质是斐波那契数列。

```go
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }
    prev, curr := 1, 2
    for i := 3; i <= n; i++ {
        prev, curr = curr, prev+curr
    }
    return curr
}
```

```python
def climbStairs(n: int) -> int:
    if n <= 2:
        return n
    prev, curr = 1, 2
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    return curr
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)（空间优化后）

---

#### 题目2：LC300 最长递增子序列 (Longest Increasing Subsequence)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：
- **O(n²) DP**：dp[i] = 以 nums[i] 结尾的 LIS 长度
- **O(n log n) 贪心+二分**：维护一个 tails 数组，tails[i] = 长度为 i+1 的 LIS 的最小末尾

```go
// 方法1：O(n²) DP
func lengthOfLIS(nums []int) int {
    n := len(nums)
    dp := make([]int, n)
    for i := range dp {
        dp[i] = 1
    }
    maxLen := 1

    for i := 1; i < n; i++ {
        for j := 0; j < i; j++ {
            if nums[i] > nums[j] {
                if dp[j]+1 > dp[i] {
                    dp[i] = dp[j] + 1
                }
            }
        }
        if dp[i] > maxLen {
            maxLen = dp[i]
        }
    }
    return maxLen
}

// 方法2：O(n log n) 贪心 + 二分
func lengthOfLISOptimal(nums []int) int {
    tails := []int{} // tails[i] = 长度为 i+1 的 LIS 的最小末尾

    for _, num := range nums {
        // 二分查找 num 应该插入的位置
        left, right := 0, len(tails)
        for left < right {
            mid := left + (right-left)/2
            if tails[mid] < num {
                left = mid + 1
            } else {
                right = mid
            }
        }
        if left == len(tails) {
            tails = append(tails, num) // 扩展
        } else {
            tails[left] = num // 替换
        }
    }
    return len(tails)
}
```

```python
import bisect

def lengthOfLIS(nums: list[int]) -> int:
    tails = []
    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

**复杂度分析**：

| 方法 | 时间 | 空间 |
|------|------|------|
| DP | O(n²) | O(n) |
| 贪心+二分 | O(n log n) | O(n) |

---

#### 题目3：LC72 编辑距离 (Edit Distance)

**难度**：Hard | **频率**：🔥🔥🔥🔥🔥

**思路**：经典二维 DP。dp[i][j] = word1 前 i 个字符转换为 word2 前 j 个字符的最少操作数。

状态转移：
- 若 word1[i-1] == word2[j-1]：dp[i][j] = dp[i-1][j-1]（无需操作）
- 否则：dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
  - dp[i-1][j] + 1：删除
  - dp[i][j-1] + 1：插入
  - dp[i-1][j-1] + 1：替换

```go
func minDistance(word1 string, word2 string) int {
    m, n := len(word1), len(word2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }

    // 初始化
    for i := 0; i <= m; i++ {
        dp[i][0] = i
    }
    for j := 0; j <= n; j++ {
        dp[0][j] = j
    }

    // 状态转移
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = 1 + minThree(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
            }
        }
    }
    return dp[m][n]
}

func minThree(a, b, c int) int {
    if a < b {
        if a < c {
            return a
        }
        return c
    }
    if b < c {
        return b
    }
    return c
}
```

```python
def minDistance(word1: str, word2: str) -> int:
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1])
    return dp[m][n]
```

**复杂度分析**：
- 时间复杂度：O(m × n)
- 空间复杂度：O(m × n)，可优化到 O(min(m, n))

---

#### 题目4：LC322 零钱兑换 (Coin Change)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：完全背包问题。dp[i] = 凑成金额 i 所需的最少硬币数。

```go
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := range dp {
        dp[i] = amount + 1 // 初始化为不可能的大值
    }
    dp[0] = 0

    for i := 1; i <= amount; i++ {
        for _, coin := range coins {
            if coin <= i && dp[i-coin]+1 < dp[i] {
                dp[i] = dp[i-coin] + 1
            }
        }
    }

    if dp[amount] > amount {
        return -1
    }
    return dp[amount]
}
```

```python
def coinChange(coins: list[int], amount: int) -> int:
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

**复杂度分析**：
- 时间复杂度：O(amount × n)，n 为硬币种类数
- 空间复杂度：O(amount)

---

#### 题目5：背包问题模板总结

##### 0-1 背包（每个物品只能选一次）

```go
// 问题：n个物品，重量weight[]，价值value[]，背包容量W
// dp[w] = 容量为w时的最大价值
func knapsack01(weight, value []int, W int) int {
    dp := make([]int, W+1)
    for i := 0; i < len(weight); i++ {
        for w := W; w >= weight[i]; w-- { // ⚠️ 逆序遍历
            if dp[w-weight[i]]+value[i] > dp[w] {
                dp[w] = dp[w-weight[i]] + value[i]
            }
        }
    }
    return dp[W]
}
```

##### 完全背包（每个物品可选无限次）

```go
func knapsackComplete(weight, value []int, W int) int {
    dp := make([]int, W+1)
    for i := 0; i < len(weight); i++ {
        for w := weight[i]; w <= W; w++ { // ⚠️ 正序遍历
            if dp[w-weight[i]]+value[i] > dp[w] {
                dp[w] = dp[w-weight[i]] + value[i]
            }
        }
    }
    return dp[W]
}
```

##### 背包问题变种对照

| 问题 | 背包类型 | 求什么 | 状态转移 |
|------|---------|-------|---------|
| LC416 分割等和子集 | 0-1 背包 | 能否装满 | `dp[w] = dp[w] \|\| dp[w-nums[i]]` |
| LC322 零钱兑换 | 完全背包 | 最少物品数 | `dp[w] = min(dp[w], dp[w-coin]+1)` |
| LC518 零钱兑换II | 完全背包 | 组合数 | `dp[w] += dp[w-coin]` |
| LC474 一和零 | 多维0-1背包 | 最多物品数 | 二维容量 |
| LC494 目标和 | 0-1 背包 | 方案数 | `dp[w] += dp[w-nums[i]]` |

### 7.4 面试自查

- [ ] DP 五步法能脱口而出吗？
- [ ] 0-1 背包和完全背包的遍历顺序区别能解释吗？
- [ ] 编辑距离的 DP 表能手画吗？
- [ ] 零钱兑换为什么初始化为 amount+1？

---

## 8. 贪心 ⭐⭐

### 8.1 核心思路总结

贪心算法：每步做出当前看起来最优的选择，期望全局最优。

**贪心 vs DP**：
- 贪心：每步最优 → 全局最优（需要证明贪心选择性质）
- DP：所有子问题最优 → 全局最优（保证正确性）

**常见贪心策略**：
1. 排序后贪心
2. 区间调度（按结束时间排序）
3. 跳跃/覆盖（维护最远可达）

### 8.2 高频题详解

#### 题目1：LC55 跳跃游戏 (Jump Game)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：维护当前能到达的最远位置。如果某一步的位置超过了最远可达，则无法到达终点。

```go
func canJump(nums []int) bool {
    maxReach := 0
    for i := 0; i < len(nums); i++ {
        if i > maxReach {
            return false
        }
        if i+nums[i] > maxReach {
            maxReach = i + nums[i]
        }
    }
    return true
}
```

**LC45 跳跃游戏 II**（最少跳跃次数）：

```go
func jump(nums []int) int {
    jumps := 0
    curEnd := 0   // 当前跳跃的边界
    farthest := 0 // 能到达的最远位置

    for i := 0; i < len(nums)-1; i++ {
        if i+nums[i] > farthest {
            farthest = i + nums[i]
        }
        if i == curEnd { // 到达当前跳跃边界，必须跳
            jumps++
            curEnd = farthest
        }
    }
    return jumps
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

#### 题目2：LC435 无重叠区间 / 区间调度 (Non-overlapping Intervals)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：经典区间调度问题。按结束时间排序，贪心选择结束最早的区间，最大化不重叠区间数。

```go
func eraseOverlapIntervals(intervals [][]int) int {
    if len(intervals) == 0 {
        return 0
    }

    // 按结束时间排序
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][1] < intervals[j][1]
    })

    count := 1 // 不重叠区间数
    end := intervals[0][1]

    for i := 1; i < len(intervals); i++ {
        if intervals[i][0] >= end {
            count++
            end = intervals[i][1]
        }
    }

    return len(intervals) - count
}
```

**复杂度分析**：
- 时间复杂度：O(n log n)
- 空间复杂度：O(log n)，排序

**区间问题套路**：

| 问题 | 排序依据 | 贪心策略 |
|------|---------|---------|
| 最多不重叠区间 | 结束时间 | 选结束最早的 |
| 最少移除使不重叠 | 结束时间 | n - 最多不重叠 |
| 合并区间 | 开始时间 | 比较 end 与下一个 start |
| 插入区间 | 开始时间 | 分三段处理 |

---

#### 题目3：LC135 分发糖果 (Candy)

**难度**：Hard | **频率**：🔥🔥🔥

**思路**：两次遍历。从左到右确保右边比左边大的孩子多得糖，从右到左确保左边比右边大的孩子多得糖。

```go
func candy(ratings []int) int {
    n := len(ratings)
    candies := make([]int, n)
    for i := range candies {
        candies[i] = 1
    }

    // 从左到右
    for i := 1; i < n; i++ {
        if ratings[i] > ratings[i-1] {
            candies[i] = candies[i-1] + 1
        }
    }

    // 从右到左
    for i := n - 2; i >= 0; i-- {
        if ratings[i] > ratings[i+1] && candies[i] <= candies[i+1] {
            candies[i] = candies[i+1] + 1
        }
    }

    total := 0
    for _, c := range candies {
        total += c
    }
    return total
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(n)

---

#### 题目4：LC134 加油站 (Gas Station)

**难度**：Medium | **频率**：🔥🔥🔥

**思路**：如果总油量 >= 总消耗，则一定有解。从某站出发如果到某站油量为负，则起点一定在该站之后。

```go
func canCompleteCircuit(gas []int, cost []int) int {
    totalTank := 0
    currTank := 0
    start := 0

    for i := 0; i < len(gas); i++ {
        diff := gas[i] - cost[i]
        totalTank += diff
        currTank += diff
        if currTank < 0 {
            start = i + 1  // 起点设为下一站
            currTank = 0
        }
    }

    if totalTank < 0 {
        return -1
    }
    return start
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

### 8.3 面试自查

- [ ] 跳跃游戏 II 的 BFS 思维方式能讲清吗？
- [ ] 区间调度为什么按结束时间排序而非开始时间？
- [ ] 加油站的"如果总油量够就一定有解"能证明吗？

---

## 9. 图算法 ⭐⭐

### 9.1 核心思路总结

| 算法 | 用途 | 数据结构 | 时间复杂度 |
|------|------|---------|----------|
| BFS | 最短路径（无权图）、层序遍历 | 队列 | O(V+E) |
| DFS | 连通性、路径搜索、拓扑排序 | 栈/递归 | O(V+E) |
| 拓扑排序 | DAG 的线性排序、依赖关系 | BFS(Kahn)/DFS | O(V+E) |
| Dijkstra | 单源最短路径（非负权） | 优先队列 | O((V+E)logV) |
| 并查集 | 连通分量、判环 | 数组 | O(α(n))≈O(1) |

### 9.2 BFS/DFS 模板

#### BFS 模板（Go）

```go
func bfs(graph [][]int, start int) {
    visited := make(map[int]bool)
    queue := []int{start}
    visited[start] = true

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        // 处理 node

        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }
}
```

#### DFS 模板（Go）

```go
func dfs(graph [][]int, node int, visited map[int]bool) {
    if visited[node] {
        return
    }
    visited[node] = true
    // 处理 node

    for _, neighbor := range graph[node] {
        dfs(graph, neighbor, visited)
    }
}
```

#### 拓扑排序模板（Kahn's BFS）

```go
func topologicalSort(numNodes int, edges [][]int) []int {
    inDegree := make([]int, numNodes)
    adj := make([][]int, numNodes)
    for _, e := range edges {
        adj[e[0]] = append(adj[e[0]], e[1])
        inDegree[e[1]]++
    }

    queue := []int{}
    for i := 0; i < numNodes; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    order := []int{}
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        order = append(order, node)
        for _, next := range adj[node] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    if len(order) != numNodes {
        return nil // 有环
    }
    return order
}
```

### 9.3 高频题详解

#### 题目1：LC200 岛屿数量 (Number of Islands)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：遍历网格，遇到 '1' 就启动 DFS/BFS 将整个岛标记为已访问，计数加一。

```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }
    m, n := len(grid), len(grid[0])
    count := 0

    var dfs func(i, j int)
    dfs = func(i, j int) {
        if i < 0 || i >= m || j < 0 || j >= n || grid[i][j] != '1' {
            return
        }
        grid[i][j] = '0' // 标记已访问
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                count++
                dfs(i, j)
            }
        }
    }
    return count
}
```

```python
def numIslands(grid: list[list[str]]) -> int:
    if not grid:
        return 0
    m, n = len(grid), len(grid[0])
    count = 0

    def dfs(i, j):
        if i < 0 or i >= m or j < 0 or j >= n or grid[i][j] != '1':
            return
        grid[i][j] = '0'
        dfs(i + 1, j)
        dfs(i - 1, j)
        dfs(i, j + 1)
        dfs(i, j - 1)

    for i in range(m):
        for j in range(n):
            if grid[i][j] == '1':
                count += 1
                dfs(i, j)
    return count
```

**复杂度分析**：
- 时间复杂度：O(m × n)
- 空间复杂度：O(m × n)，最坏递归深度

---

#### 题目2：LC207 课程表 (Course Schedule)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：课程依赖就是有向图，能否完成所有课程 = 图中是否有环 = 能否拓扑排序。

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    inDegree := make([]int, numCourses)
    adj := make([][]int, numCourses)

    for _, p := range prerequisites {
        adj[p[1]] = append(adj[p[1]], p[0])
        inDegree[p[0]]++
    }

    queue := []int{}
    for i := 0; i < numCourses; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    count := 0
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        count++
        for _, next := range adj[node] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    return count == numCourses
}
```

**LC210 课程表 II**（返回修课顺序）：只需将 count++ 改为收集 order。

**复杂度分析**：
- 时间复杂度：O(V + E)
- 空间复杂度：O(V + E)

---

#### 题目3：Dijkstra 最短路径模板

**适用**：非负权图的单源最短路径。

```go
import "container/heap"

type Edge struct {
    to, weight int
}

type Item struct {
    node, dist int
}

type PQ []Item

func (pq PQ) Len() int            { return len(pq) }
func (pq PQ) Less(i, j int) bool  { return pq[i].dist < pq[j].dist }
func (pq PQ) Swap(i, j int)       { pq[i], pq[j] = pq[j], pq[i] }
func (pq *PQ) Push(x interface{}) { *pq = append(*pq, x.(Item)) }
func (pq *PQ) Pop() interface{} {
    old := *pq
    n := len(old)
    x := old[n-1]
    *pq = old[:n-1]
    return x
}

func dijkstra(graph [][]Edge, src int, n int) []int {
    dist := make([]int, n)
    for i := range dist {
        dist[i] = math.MaxInt64
    }
    dist[src] = 0

    pq := &PQ{{src, 0}}
    heap.Init(pq)

    for pq.Len() > 0 {
        curr := heap.Pop(pq).(Item)
        if curr.dist > dist[curr.node] {
            continue // 已找到更短路径，跳过
        }

        for _, edge := range graph[curr.node] {
            newDist := curr.dist + edge.weight
            if newDist < dist[edge.to] {
                dist[edge.to] = newDist
                heap.Push(pq, Item{edge.to, newDist})
            }
        }
    }
    return dist
}
```

**LC743 网络延迟时间** 就是直接应用 Dijkstra 的题目。

**复杂度分析**：
- 时间复杂度：O((V + E) log V)
- 空间复杂度：O(V + E)

#### 题目4：LC695 岛屿的最大面积 (Max Area of Island)

**难度**：Medium | **频率**：🔥🔥🔥

```go
func maxAreaOfIsland(grid [][]int) int {
    m, n := len(grid), len(grid[0])
    maxArea := 0

    var dfs func(i, j int) int
    dfs = func(i, j int) int {
        if i < 0 || i >= m || j < 0 || j >= n || grid[i][j] != 1 {
            return 0
        }
        grid[i][j] = 0
        return 1 + dfs(i+1, j) + dfs(i-1, j) + dfs(i, j+1) + dfs(i, j-1)
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == 1 {
                area := dfs(i, j)
                if area > maxArea {
                    maxArea = area
                }
            }
        }
    }
    return maxArea
}
```

**复杂度分析**：
- 时间复杂度：O(m × n)
- 空间复杂度：O(m × n)

### 9.4 面试自查

- [ ] BFS 和 DFS 分别用什么数据结构？
- [ ] 拓扑排序 BFS(Kahn) 和 DFS 两种实现都会写吗？
- [ ] Dijkstra 为什么不能处理负权边？
- [ ] 岛屿问题的 DFS 沉岛思路能讲清楚吗？

---

## 10. 二分查找 ⭐⭐⭐

### 10.1 核心思路总结

二分查找的本质：在有序（或部分有序）空间中，通过每次排除一半来缩小搜索范围。

**三个关键细节**：
1. 循环条件：`left <= right` vs `left < right`
2. 中点计算：`mid = left + (right-left)/2`（防溢出）
3. 边界更新：`left = mid + 1` vs `left = mid`

### 10.2 三种模板

#### 模板1：标准二分（找精确值）

```go
// left <= right，区间 [left, right]
func binarySearch(nums []int, target int) int {
    left, right := 0, len(nums)-1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1 // 未找到
}
```

#### 模板2：找左边界（第一个 >= target 的位置）

```go
// left < right，区间 [left, right)
func lowerBound(nums []int, target int) int {
    left, right := 0, len(nums)
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid // 不是 mid-1
        }
    }
    return left
}
```

#### 模板3：找右边界（最后一个 <= target 的位置）

```go
func upperBound(nums []int, target int) int {
    left, right := 0, len(nums)
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] <= target {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return left - 1
}
```

#### 三种模板对比

| 模板 | 循环条件 | 返回值 | 用途 |
|------|---------|--------|------|
| 标准 | `left <= right` | mid / -1 | 找精确值 |
| 左边界 | `left < right` | left | 第一个 >= target |
| 右边界 | `left < right` | left - 1 | 最后一个 <= target |

### 10.3 高频题详解

#### 题目1：LC33 搜索旋转排序数组 (Search in Rotated Sorted Array)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：二分查找，每次判断哪一半是有序的，然后决定在哪一半继续搜索。

```go
func search(nums []int, target int) int {
    left, right := 0, len(nums)-1

    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        }

        // 左半段有序
        if nums[left] <= nums[mid] {
            if target >= nums[left] && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else { // 右半段有序
            if target > nums[mid] && target <= nums[right] {
                left = mid + 1
            } else {
                right = mid - 1
            }
        }
    }
    return -1
}
```

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        if nums[left] <= nums[mid]:  # 左半有序
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:  # 右半有序
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

**复杂度分析**：
- 时间复杂度：O(log n)
- 空间复杂度：O(1)

---

#### 题目2：LC162 寻找峰值 (Find Peak Element)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：二分查找。如果 nums[mid] < nums[mid+1]，峰值在右半部分；否则在左半部分。

```go
func findPeakElement(nums []int) int {
    left, right := 0, len(nums)-1
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] < nums[mid+1] {
            left = mid + 1 // 峰值在右边
        } else {
            right = mid // 峰值在左边（含 mid）
        }
    }
    return left
}
```

**复杂度分析**：
- 时间复杂度：O(log n)
- 空间复杂度：O(1)

---

#### 题目3：LC34 在排序数组中查找元素的第一个和最后一个位置 (Find First and Last Position)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：两次二分，分别找左边界和右边界。

```go
func searchRange(nums []int, target int) []int {
    return []int{findFirst(nums, target), findLast(nums, target)}
}

func findFirst(nums []int, target int) int {
    left, right := 0, len(nums)-1
    result := -1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            result = mid
            right = mid - 1 // 继续往左找
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return result
}

func findLast(nums []int, target int) int {
    left, right := 0, len(nums)-1
    result := -1
    for left <= right {
        mid := left + (right-left)/2
        if nums[mid] == target {
            result = mid
            left = mid + 1 // 继续往右找
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return result
}
```

```python
def searchRange(nums: list[int], target: int) -> list[int]:
    def find_first():
        left, right = 0, len(nums) - 1
        result = -1
        while left <= right:
            mid = (left + right) // 2
            if nums[mid] == target:
                result = mid
                right = mid - 1
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1
        return result

    def find_last():
        left, right = 0, len(nums) - 1
        result = -1
        while left <= right:
            mid = (left + right) // 2
            if nums[mid] == target:
                result = mid
                left = mid + 1
            elif nums[mid] < target:
                left = mid + 1
            else:
                right = mid - 1
        return result

    return [find_first(), find_last()]
```

**复杂度分析**：
- 时间复杂度：O(log n)
- 空间复杂度：O(1)

---

#### 题目4：LC153 寻找旋转排序数组中的最小值

**难度**：Medium | **频率**：🔥🔥🔥

```go
func findMin(nums []int) int {
    left, right := 0, len(nums)-1
    for left < right {
        mid := left + (right-left)/2
        if nums[mid] > nums[right] {
            left = mid + 1 // 最小值在右半部分
        } else {
            right = mid // 最小值在左半部分（含 mid）
        }
    }
    return nums[left]
}
```

**复杂度分析**：
- 时间复杂度：O(log n)
- 空间复杂度：O(1)

### 10.4 二分查找常见陷阱

| 陷阱 | 说明 | 解决 |
|------|------|------|
| 整数溢出 | `(left+right)/2` 溢出 | 用 `left + (right-left)/2` |
| 死循环 | `left = mid` 不收缩 | 检查 mid 计算和更新逻辑 |
| 边界条件 | off-by-one | 明确搜索区间是开还是闭 |
| 返回值 | 找不到目标时返回什么 | 明确题意要求 |

### 10.5 面试自查

- [ ] 三种二分模板能默写吗？
- [ ] 搜索旋转数组的判断逻辑能画图讲清楚吗？
- [ ] `left <= right` 和 `left < right` 的区别能解释吗？
- [ ] 能解释为什么找左边界时 right = mid 而不是 mid - 1 吗？

---

## 11. 堆/优先队列 ⭐⭐

### 11.1 核心思路总结

堆是一种完全二叉树，常用于维护动态集合中的最值。

| 操作 | 时间复杂度 |
|------|----------|
| 插入 | O(log n) |
| 删除堆顶 | O(log n) |
| 获取堆顶 | O(1) |
| 建堆 | O(n) |

**典型场景**：
- 求第 K 大/小 → 大小为 K 的堆
- 合并多个有序序列 → 最小堆
- 动态维护中位数 → 一个大顶堆 + 一个小顶堆

### 11.2 Go 的堆使用

```go
import "container/heap"

// 最小堆
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
```

### 11.3 高频题详解

#### 题目1：LC347 前 K 个高频元素 (Top K Frequent Elements)

**难度**：Medium | **频率**：🔥🔥🔥🔥

**思路**：统计频次 + 最小堆维护 K 个最高频元素。也可用桶排序 O(n)。

```go
func topKFrequent(nums []int, k int) []int {
    // 统计频次
    freq := make(map[int]int)
    for _, num := range nums {
        freq[num]++
    }

    // 桶排序：buckets[i] 存放出现 i 次的元素
    n := len(nums)
    buckets := make([][]int, n+1)
    for num, count := range freq {
        buckets[count] = append(buckets[count], num)
    }

    result := []int{}
    for i := n; i >= 0 && len(result) < k; i-- {
        result = append(result, buckets[i]...)
    }
    return result[:k]
}
```

```python
from collections import Counter

def topKFrequent(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)
    # 桶排序
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in count.items():
        buckets[freq].append(num)
    result = []
    for i in range(len(nums), -1, -1):
        result.extend(buckets[i])
        if len(result) >= k:
            return result[:k]
    return result[:k]
```

**复杂度分析**：

| 方法 | 时间 | 空间 |
|------|------|------|
| 最小堆 | O(n log k) | O(n) |
| 桶排序 | O(n) | O(n) |

---

#### 题目2：LC295 数据流的中位数 (Find Median from Data Stream)

**难度**：Hard | **频率**：🔥🔥🔥🔥🔥

**思路**：用两个堆 —— 大顶堆存较小的一半，小顶堆存较大的一半。中位数从堆顶获取。

```go
type MedianFinder struct {
    small *MaxHeap // 大顶堆：较小的一半
    large *MinHeap // 小顶堆：较大的一半
}

// MaxHeap
type MaxHeap []int
func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] }
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MaxHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MaxHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

// MinHeap (同上 IntHeap)
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func ConstructorMF() MedianFinder {
    s := &MaxHeap{}
    l := &MinHeap{}
    heap.Init(s)
    heap.Init(l)
    return MedianFinder{small: s, large: l}
}

func (mf *MedianFinder) AddNum(num int) {
    // 先加入大顶堆
    heap.Push(mf.small, num)
    // 把大顶堆最大值移到小顶堆
    heap.Push(mf.large, heap.Pop(mf.small))

    // 保持大顶堆元素数 >= 小顶堆
    if mf.small.Len() < mf.large.Len() {
        heap.Push(mf.small, heap.Pop(mf.large))
    }
}

func (mf *MedianFinder) FindMedian() float64 {
    if mf.small.Len() > mf.large.Len() {
        return float64((*mf.small)[0])
    }
    return float64((*mf.small)[0]+(*mf.large)[0]) / 2.0
}
```

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []  # 大顶堆（取负）
        self.large = []  # 小顶堆

    def addNum(self, num: int) -> None:
        heapq.heappush(self.small, -num)
        heapq.heappush(self.large, -heapq.heappop(self.small))
        if len(self.small) < len(self.large):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self) -> float:
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

**复杂度分析**：
- AddNum：O(log n)
- FindMedian：O(1)
- 空间：O(n)

---

#### 题目3：LC373 查找和最小的 K 对数字 / LC378 有序矩阵中第 K 小的元素

**难度**：Medium | **频率**：🔥🔥🔥

**思路**：最小堆 + BFS 式扩展。与合并 K 个有序链表思路类似。

```go
// LC378: 有序矩阵中第K小的元素
func kthSmallest(matrix [][]int, k int) int {
    n := len(matrix)
    h := &PQMatrix{}
    for i := 0; i < n && i < k; i++ {
        heap.Push(h, [3]int{matrix[i][0], i, 0})
    }

    for k > 1 {
        item := heap.Pop(h).([3]int)
        row, col := item[1], item[2]
        if col+1 < n {
            heap.Push(h, [3]int{matrix[row][col+1], row, col + 1})
        }
        k--
    }
    return heap.Pop(h).([3]int)[0]
}

type PQMatrix [][3]int
func (pq PQMatrix) Len() int            { return len(pq) }
func (pq PQMatrix) Less(i, j int) bool  { return pq[i][0] < pq[j][0] }
func (pq PQMatrix) Swap(i, j int)       { pq[i], pq[j] = pq[j], pq[i] }
func (pq *PQMatrix) Push(x interface{}) { *pq = append(*pq, x.([3]int)) }
func (pq *PQMatrix) Pop() interface{} {
    old := *pq
    n := len(old)
    x := old[n-1]
    *pq = old[:n-1]
    return x
}
```

**复杂度分析**：
- 时间复杂度：O(k log n)
- 空间复杂度：O(n)

### 11.4 面试自查

- [ ] Go 的 container/heap 接口能快速实现吗？
- [ ] 数据流中位数的双堆方案能讲清楚吗？
- [ ] Top K 问题用最小堆还是最大堆？为什么？

---

## 12. 字符串 ⭐⭐

### 12.1 核心思路总结

| 技巧 | 典型问题 |
|------|---------|
| 双指针/中心扩展 | 回文串、反转 |
| 哈希表 | 字母异位词、字符计数 |
| KMP | 字符串匹配 |
| Trie（前缀树） | 前缀搜索、自动补全 |

### 12.2 高频题详解

#### 题目1：LC5 最长回文子串 (Longest Palindromic Substring)

**难度**：Medium | **频率**：🔥🔥🔥🔥🔥

**思路**：中心扩展法。以每个字符（和每两个相邻字符）为中心向外扩展。

```go
func longestPalindrome(s string) string {
    if len(s) < 2 {
        return s
    }

    start, maxLen := 0, 1

    expand := func(left, right int) {
        for left >= 0 && right < len(s) && s[left] == s[right] {
            if right-left+1 > maxLen {
                start = left
                maxLen = right - left + 1
            }
            left--
            right++
        }
    }

    for i := 0; i < len(s); i++ {
        expand(i, i)   // 奇数长度回文
        expand(i, i+1) // 偶数长度回文
    }

    return s[start : start+maxLen]
}
```

```python
def longestPalindrome(s: str) -> str:
    start, max_len = 0, 1

    def expand(left, right):
        nonlocal start, max_len
        while left >= 0 and right < len(s) and s[left] == s[right]:
            if right - left + 1 > max_len:
                start = left
                max_len = right - left + 1
            left -= 1
            right += 1

    for i in range(len(s)):
        expand(i, i)
        expand(i, i + 1)

    return s[start:start + max_len]
```

**复杂度分析**：
- 时间复杂度：O(n²)
- 空间复杂度：O(1)

**其他解法**：

| 方法 | 时间 | 空间 | 说明 |
|------|------|------|------|
| 暴力 | O(n³) | O(1) | 枚举所有子串 |
| DP | O(n²) | O(n²) | dp[i][j] 表示 s[i..j] 是否回文 |
| 中心扩展 | O(n²) | O(1) | ✅ 面试首选 |
| Manacher | O(n) | O(n) | 最优但复杂 |

---

#### 题目2：KMP 字符串匹配 (LC28 实现 strStr)

**难度**：Easy (题目) / Medium (KMP) | **频率**：🔥🔥🔥

**KMP 核心思想**：利用已匹配的信息，避免重复比较。构建 next 数组（部分匹配表）。

```go
// 构建 next 数组（前缀函数）
func buildNext(pattern string) []int {
    n := len(pattern)
    next := make([]int, n)
    next[0] = 0
    j := 0

    for i := 1; i < n; i++ {
        for j > 0 && pattern[i] != pattern[j] {
            j = next[j-1]
        }
        if pattern[i] == pattern[j] {
            j++
        }
        next[i] = j
    }
    return next
}

// KMP 搜索
func strStr(haystack string, needle string) int {
    if len(needle) == 0 {
        return 0
    }
    next := buildNext(needle)
    j := 0

    for i := 0; i < len(haystack); i++ {
        for j > 0 && haystack[i] != needle[j] {
            j = next[j-1]
        }
        if haystack[i] == needle[j] {
            j++
        }
        if j == len(needle) {
            return i - len(needle) + 1
        }
    }
    return -1
}
```

```python
def strStr(haystack: str, needle: str) -> int:
    if not needle:
        return 0

    # 构建 next 数组
    n = len(needle)
    next_arr = [0] * n
    j = 0
    for i in range(1, n):
        while j > 0 and needle[i] != needle[j]:
            j = next_arr[j - 1]
        if needle[i] == needle[j]:
            j += 1
        next_arr[i] = j

    # KMP 搜索
    j = 0
    for i in range(len(haystack)):
        while j > 0 and haystack[i] != needle[j]:
            j = next_arr[j - 1]
        if haystack[i] == needle[j]:
            j += 1
        if j == len(needle):
            return i - n + 1
    return -1
```

**复杂度分析**：
- 时间复杂度：O(m + n)
- 空间复杂度：O(n)，next 数组

---

#### 题目3：LC14 最长公共前缀 (Longest Common Prefix)

**难度**：Easy | **频率**：🔥🔥🔥

**思路**：纵向扫描。比较每个字符串的同一位置字符。

```go
func longestCommonPrefix(strs []string) string {
    if len(strs) == 0 {
        return ""
    }

    for i := 0; i < len(strs[0]); i++ {
        c := strs[0][i]
        for j := 1; j < len(strs); j++ {
            if i >= len(strs[j]) || strs[j][i] != c {
                return strs[0][:i]
            }
        }
    }
    return strs[0]
}
```

**复杂度分析**：
- 时间复杂度：O(S)，S 为所有字符串长度之和
- 空间复杂度：O(1)

#### 题目4：LC49 字母异位词分组 (Group Anagrams)

**难度**：Medium | **频率**：🔥🔥🔥🔥

```go
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)
    for _, s := range strs {
        // 用排序后的字符串作为 key
        b := []byte(s)
        sort.Slice(b, func(i, j int) bool { return b[i] < b[j] })
        key := string(b)
        groups[key] = append(groups[key], s)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}
```

**复杂度分析**：
- 时间复杂度：O(n × k log k)，k 为最长字符串长度
- 空间复杂度：O(n × k)

### 12.3 面试自查

- [ ] 最长回文子串的中心扩展法能快速写出吗？
- [ ] KMP 的 next 数组构建过程能讲清楚吗？
- [ ] 字母异位词分组还有什么 key 的设计方式？（计数数组做 key）

---

## 13. 位运算 ⭐

### 13.1 核心思路总结

| 运算 | 符号 | 常用技巧 |
|------|------|---------|
| 与 AND | `&` | 取位、判断奇偶、清除最低位的1 |
| 或 OR | `\|` | 设位 |
| 异或 XOR | `^` | 不进位加法、找唯一数、交换 |
| 取反 NOT | `~` | 翻转所有位 |
| 左移 | `<<` | 乘以 2^n |
| 右移 | `>>` | 除以 2^n |

**常用位运算技巧**：

```
n & (n-1)       // 清除最低位的 1
n & (-n)        // 获取最低位的 1
n ^ n = 0       // 任何数异或自身为 0
n ^ 0 = n       // 任何数异或 0 为自身
```

### 13.2 高频题详解

#### 题目1：LC136 只出现一次的数字 (Single Number)

**难度**：Easy | **频率**：🔥🔥🔥🔥

**思路**：全部异或。相同数字异或为 0，最终剩下的就是只出现一次的数。

```go
func singleNumber(nums []int) int {
    result := 0
    for _, num := range nums {
        result ^= num
    }
    return result
}
```

```python
def singleNumber(nums: list[int]) -> int:
    result = 0
    for num in nums:
        result ^= num
    return result
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

**扩展**：LC137 只出现一次的数字 II（其他出现三次）、LC260 只出现一次的数字 III（两个数各出现一次）。

---

#### 题目2：LC231 2 的幂 (Power of Two)

**难度**：Easy | **频率**：🔥🔥🔥

**思路**：2 的幂在二进制中只有一个 1。用 `n & (n-1) == 0` 判断。

```go
func isPowerOfTwo(n int) bool {
    return n > 0 && n&(n-1) == 0
}
```

**复杂度分析**：
- 时间复杂度：O(1)
- 空间复杂度：O(1)

---

#### 题目3：LC461 汉明距离 (Hamming Distance)

**难度**：Easy | **频率**：🔥🔥🔥

**思路**：两数异或后统计 1 的个数。

```go
func hammingDistance(x int, y int) int {
    xor := x ^ y
    count := 0
    for xor != 0 {
        count++
        xor &= xor - 1 // 清除最低位的 1
    }
    return count
}
```

```python
def hammingDistance(x: int, y: int) -> int:
    xor = x ^ y
    count = 0
    while xor:
        count += 1
        xor &= xor - 1
    return count
```

**复杂度分析**：
- 时间复杂度：O(1)，最多 32 位
- 空间复杂度：O(1)

---

#### 题目4：LC191 位 1 的个数 (Number of 1 Bits)

```go
func hammingWeight(num uint32) int {
    count := 0
    for num != 0 {
        count++
        num &= num - 1
    }
    return count
}
```

#### 题目5：LC338 比特位计数 (Counting Bits)

```go
func countBits(n int) []int {
    dp := make([]int, n+1)
    for i := 1; i <= n; i++ {
        dp[i] = dp[i&(i-1)] + 1 // i 去掉最低位的1后的值 + 1
    }
    return dp
}
```

### 13.3 面试自查

- [ ] `n & (n-1)` 的作用能脱口而出吗？
- [ ] 异或的三条性质能说清楚吗？
- [ ] 只出现一次的数字三道题的区别是什么？

---

## 14. 高频面试题 TOP50 清单

### 14.1 按难度分类

#### Easy（15 题）

| # | 题号 | 题目 | 分类 | 重要度 |
|---|------|------|------|--------|
| 1 | LC1 | 两数之和 | 数组/哈希 | ⭐⭐⭐ |
| 2 | LC20 | 有效的括号 | 栈 | ⭐⭐⭐ |
| 3 | LC21 | 合并两个有序链表 | 链表 | ⭐⭐⭐ |
| 4 | LC70 | 爬楼梯 | DP | ⭐⭐⭐ |
| 5 | LC104 | 二叉树最大深度 | 树 | ⭐⭐ |
| 6 | LC121 | 买卖股票最佳时机 | 贪心/DP | ⭐⭐⭐ |
| 7 | LC136 | 只出现一次的数字 | 位运算 | ⭐⭐ |
| 8 | LC141 | 环形链表 | 链表 | ⭐⭐ |
| 9 | LC155 | 最小栈 | 栈 | ⭐⭐ |
| 10 | LC160 | 相交链表 | 链表 | ⭐⭐ |
| 11 | LC206 | 反转链表 | 链表 | ⭐⭐⭐ |
| 12 | LC226 | 翻转二叉树 | 树 | ⭐⭐ |
| 13 | LC234 | 回文链表 | 链表 | ⭐⭐ |
| 14 | LC283 | 移动零 | 数组 | ⭐⭐ |
| 15 | LC543 | 二叉树的直径 | 树 | ⭐⭐ |

#### Medium（25 题）

| # | 题号 | 题目 | 分类 | 重要度 |
|---|------|------|------|--------|
| 1 | LC3 | 无重复字符最长子串 | 滑动窗口 | ⭐⭐⭐ |
| 2 | LC5 | 最长回文子串 | 字符串/DP | ⭐⭐⭐ |
| 3 | LC11 | 盛最多水的容器 | 双指针 | ⭐⭐⭐ |
| 4 | LC15 | 三数之和 | 双指针 | ⭐⭐⭐ |
| 5 | LC33 | 搜索旋转排序数组 | 二分 | ⭐⭐⭐ |
| 6 | LC34 | 排序数组查找元素 | 二分 | ⭐⭐⭐ |
| 7 | LC39 | 组合总和 | 回溯 | ⭐⭐⭐ |
| 8 | LC46 | 全排列 | 回溯 | ⭐⭐⭐ |
| 9 | LC48 | 旋转图像 | 数组 | ⭐⭐ |
| 10 | LC49 | 字母异位词分组 | 哈希 | ⭐⭐⭐ |
| 11 | LC53 | 最大子数组和 | DP/分治 | ⭐⭐⭐ |
| 12 | LC56 | 合并区间 | 排序 | ⭐⭐⭐ |
| 13 | LC78 | 子集 | 回溯 | ⭐⭐⭐ |
| 14 | LC98 | 验证二叉搜索树 | 树 | ⭐⭐⭐ |
| 15 | LC102 | 二叉树层序遍历 | 树/BFS | ⭐⭐⭐ |
| 16 | LC142 | 环形链表 II | 链表 | ⭐⭐⭐ |
| 17 | LC146 | LRU 缓存 | 设计 | ⭐⭐⭐ |
| 18 | LC148 | 排序链表 | 链表 | ⭐⭐ |
| 19 | LC200 | 岛屿数量 | 图/DFS | ⭐⭐⭐ |
| 20 | LC207 | 课程表 | 拓扑排序 | ⭐⭐⭐ |
| 21 | LC236 | 最近公共祖先 | 树 | ⭐⭐⭐ |
| 22 | LC300 | 最长递增子序列 | DP | ⭐⭐⭐ |
| 23 | LC322 | 零钱兑换 | DP | ⭐⭐⭐ |
| 24 | LC347 | 前 K 个高频元素 | 堆/哈希 | ⭐⭐⭐ |
| 25 | LC739 | 每日温度 | 单调栈 | ⭐⭐⭐ |

#### Hard（10 题）

| # | 题号 | 题目 | 分类 | 重要度 |
|---|------|------|------|--------|
| 1 | LC23 | 合并K个升序链表 | 堆/链表 | ⭐⭐⭐ |
| 2 | LC42 | 接雨水 | 双指针/栈 | ⭐⭐⭐ |
| 3 | LC51 | N 皇后 | 回溯 | ⭐⭐ |
| 4 | LC72 | 编辑距离 | DP | ⭐⭐⭐ |
| 5 | LC76 | 最小覆盖子串 | 滑动窗口 | ⭐⭐⭐ |
| 6 | LC84 | 柱状图最大矩形 | 单调栈 | ⭐⭐⭐ |
| 7 | LC124 | 二叉树最大路径和 | 树 | ⭐⭐⭐ |
| 8 | LC239 | 滑动窗口最大值 | 单调队列 | ⭐⭐⭐ |
| 9 | LC295 | 数据流中位数 | 堆 | ⭐⭐⭐ |
| 10 | LC297 | 二叉树序列化 | 树 | ⭐⭐ |

### 14.2 按公司分类（高频出题）

#### 字节跳动 TOP10

| 题号 | 题目 | 分类 |
|------|------|------|
| LC3 | 无重复字符最长子串 | 滑动窗口 |
| LC15 | 三数之和 | 双指针 |
| LC25 | K 个一组翻转链表 | 链表 |
| LC42 | 接雨水 | 双指针 |
| LC46 | 全排列 | 回溯 |
| LC53 | 最大子数组和 | DP |
| LC146 | LRU 缓存 | 设计 |
| LC200 | 岛屿数量 | DFS |
| LC206 | 反转链表 | 链表 |
| LC236 | 最近公共祖先 | 树 |

#### 腾讯 TOP10

| 题号 | 题目 | 分类 |
|------|------|------|
| LC1 | 两数之和 | 哈希 |
| LC5 | 最长回文子串 | 字符串 |
| LC15 | 三数之和 | 双指针 |
| LC21 | 合并两个有序链表 | 链表 |
| LC33 | 搜索旋转排序数组 | 二分 |
| LC56 | 合并区间 | 排序 |
| LC102 | 二叉树层序遍历 | 树 |
| LC146 | LRU 缓存 | 设计 |
| LC215 | 数组中第K大元素 | 堆/快排 |
| LC322 | 零钱兑换 | DP |

#### 阿里巴巴 TOP10

| 题号 | 题目 | 分类 |
|------|------|------|
| LC3 | 无重复字符最长子串 | 滑动窗口 |
| LC23 | 合并K个升序链表 | 堆 |
| LC42 | 接雨水 | 双指针 |
| LC72 | 编辑距离 | DP |
| LC76 | 最小覆盖子串 | 滑动窗口 |
| LC124 | 二叉树最大路径和 | 树 |
| LC146 | LRU 缓存 | 设计 |
| LC200 | 岛屿数量 | DFS |
| LC295 | 数据流中位数 | 堆 |
| LC300 | 最长递增子序列 | DP |

#### Google/Meta TOP10

| 题号 | 题目 | 分类 |
|------|------|------|
| LC1 | Two Sum | Hash |
| LC42 | Trapping Rain Water | Two Pointers |
| LC56 | Merge Intervals | Sort |
| LC76 | Minimum Window Substring | Sliding Window |
| LC124 | Binary Tree Max Path Sum | Tree |
| LC146 | LRU Cache | Design |
| LC200 | Number of Islands | DFS |
| LC236 | Lowest Common Ancestor | Tree |
| LC295 | Find Median from Data Stream | Heap |
| LC297 | Serialize and Deserialize BT | Tree |

### 14.3 必须秒杀的 10 道题

> 这些题目在面试中出现频率极高，必须做到闭眼也能写出来。

1. **LC1 两数之和** — 哈希表，3 分钟内写完
2. **LC206 反转链表** — 迭代 + 递归，5 分钟
3. **LC20 有效括号** — 栈，3 分钟
4. **LC15 三数之和** — 排序 + 双指针 + 去重，10 分钟
5. **LC146 LRU 缓存** — 哈希表 + 双向链表，15 分钟
6. **LC3 无重复字符最长子串** — 滑动窗口，5 分钟
7. **LC200 岛屿数量** — DFS 沉岛，5 分钟
8. **LC53 最大子数组和** — Kadane 算法，3 分钟
9. **LC236 最近公共祖先** — 后序遍历，5 分钟
10. **LC42 接雨水** — 双指针，8 分钟

---

## 15. 刷题策略与时间规划

### 15.1 刷题方法论

#### 单题学习法（推荐）

```
第1遍：读题 → 思考5分钟 → 看不出思路就看题解 → 理解后自己写
第2遍：隔天不看题解，独立写出来
第3遍：一周后再写一遍，追求最优解
```

#### 分类刷题法

```
1. 选一个分类（如双指针）
2. 先学模板和核心思想
3. 按难度 Easy → Medium → Hard 刷 5-10 题
4. 总结该分类的套路和变形
5. 切换到下一个分类
```

#### 做题流程

```
1. 读题（2分钟）：明确输入输出、边界条件
2. 思考（5-10分钟）：确定思路、数据结构、算法
3. 编码（10-15分钟）：写出代码
4. 测试（3分钟）：用例子验证、考虑边界
5. 优化（5分钟）：能否降低时间/空间复杂度
```

### 15.2 时间规划

#### 1 周紧急突击计划（每天 4-6 小时）

| 天数 | 主题 | 重点题目（题号） |
|------|------|----------------|
| Day 1 | 数组 + 双指针 + 哈希 | 1, 15, 42, 11, 283 |
| Day 2 | 链表 + 栈 | 206, 146, 20, 155, 142 |
| Day 3 | 二叉树 | 104, 98, 236, 102, 226 |
| Day 4 | 二分 + 滑动窗口 | 33, 34, 3, 76, 567 |
| Day 5 | 回溯 + DFS/BFS | 46, 78, 200, 79, 207 |
| Day 6 | 动态规划 | 70, 300, 72, 322, 53 |
| Day 7 | 综合复习 + 模拟面试 | TOP10 必须秒杀题 |

#### 2 周标准计划（每天 3-4 小时）

| 周 | 天数 | 主题 | 题目数 |
|---|------|------|--------|
| W1 | Day 1-2 | 数组 + 双指针 | 8 题 |
| W1 | Day 3-4 | 链表 + 栈与队列 | 8 题 |
| W1 | Day 5 | 二叉树基础 | 5 题 |
| W1 | Day 6-7 | 二叉树进阶 + 二分查找 | 8 题 |
| W2 | Day 1-2 | 滑动窗口 + 回溯 | 8 题 |
| W2 | Day 3-4 | 动态规划 | 8 题 |
| W2 | Day 5 | 图算法 + 贪心 | 5 题 |
| W2 | Day 6 | 堆 + 字符串 + 位运算 | 5 题 |
| W2 | Day 7 | 全面复习 + 模拟 | TOP50 |

#### 1 月深度计划（每天 2-3 小时）

| 周 | 重点 | 目标 |
|---|------|------|
| Week 1 | 基础数据结构 | 数组、链表、栈、队列，掌握基本操作，40 题 |
| Week 2 | 树 + 搜索 | 二叉树、BST、二分查找、DFS/BFS，35 题 |
| Week 3 | 高级算法 | 回溯、DP、贪心、图算法，40 题 |
| Week 4 | 综合提升 | 堆、字符串、位运算、模拟面试，35 题 |

### 15.3 面试答题框架

```
1. 确认题意（1-2分钟）
   - 复述问题
   - 确认输入输出格式
   - 问边界条件：空输入？负数？重复？

2. 讨论思路（3-5分钟）
   - 先说暴力解法及其复杂度
   - 再说优化思路
   - 与面试官确认方向

3. 编码实现（10-15分钟）
   - 写清楚变量名
   - 边写边解释
   - 处理边界条件

4. 测试验证（3-5分钟）
   - 用示例走一遍代码
   - 测试边界用例
   - 分析时间空间复杂度
```

### 15.4 常见面试追问及应对

| 追问 | 应对策略 |
|------|---------|
| "能优化吗？" | 先分析当前瓶颈（时间 or 空间），用空间换时间或反之 |
| "如果数据量很大呢？" | 考虑外部排序、分治、流式处理、MapReduce |
| "如果是多线程环境？" | 加锁、CAS、读写锁、无锁数据结构 |
| "如何测试？" | 正常用例、边界用例、异常用例、压力测试 |
| "时间复杂度是多少？" | 明确说出最好/最坏/平均，解释推导过程 |

### 15.5 心态管理

```
✅ 做到：
- 每道题限时，超时就看题解（不要死磕）
- 重理解不重数量（50题吃透 > 200题浅尝）
- 建立错题本，定期复习
- 模拟面试环境练习（白板/共享屏幕）

❌ 避免：
- 只刷 Easy 找自信
- 看完题解就算"会了"（必须自己写一遍）
- 追求一次通过（debug 能力也很重要）
- 刷题时开多个标签页（专注！）
```

### 15.6 推荐刷题顺序总览

```
阶段1：基础篇（第1-2周）
├── 数组 + 双指针（10题）
├── 链表（8题）
├── 栈与队列（6题）
└── 哈希表（5题）

阶段2：进阶篇（第2-3周）
├── 二叉树（10题）
├── 二分查找（6题）
├── 滑动窗口（5题）
└── BFS/DFS（6题）

阶段3：高级篇（第3-4周）
├── 动态规划（12题）
├── 回溯（8题）
├── 贪心（5题）
└── 堆/优先队列（4题）

阶段4：冲刺篇（最后几天）
├── 高频 TOP50 复习
├── 字符串 + 位运算（5题）
├── 模拟面试（2-3次）
└── 查漏补缺
```

---

## 附录 A：算法复杂度速查表

### 排序算法

| 算法 | 平均 | 最好 | 最坏 | 空间 | 稳定 |
|------|------|------|------|------|------|
| 冒泡 | O(n²) | O(n) | O(n²) | O(1) | ✅ |
| 选择 | O(n²) | O(n²) | O(n²) | O(1) | ❌ |
| 插入 | O(n²) | O(n) | O(n²) | O(1) | ✅ |
| 归并 | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |
| 快排 | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |
| 堆排 | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ |
| 计数 | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ |

### 数据结构操作

| 数据结构 | 查找 | 插入 | 删除 | 空间 |
|---------|------|------|------|------|
| 数组 | O(1) | O(n) | O(n) | O(n) |
| 链表 | O(n) | O(1) | O(1) | O(n) |
| 哈希表 | O(1) | O(1) | O(1) | O(n) |
| BST | O(log n) | O(log n) | O(log n) | O(n) |
| 堆 | O(1)顶 | O(log n) | O(log n) | O(n) |
| Trie | O(m) | O(m) | O(m) | O(字符集×节点) |

---

## 附录 B：Go 语言刷题常用技巧

### 常用数据结构初始化

```go
// 切片（动态数组）
arr := make([]int, n)          // 长度 n，默认值 0
arr := make([]int, 0, n)       // 长度 0，容量 n

// 二维切片
grid := make([][]int, m)
for i := range grid {
    grid[i] = make([]int, n)
}

// 哈希表
m := make(map[int]int)
m := map[string][]int{}       // 值为切片的 map

// 栈（用切片模拟）
stack := []int{}
stack = append(stack, x)       // push
top := stack[len(stack)-1]     // peek
stack = stack[:len(stack)-1]   // pop

// 队列（用切片模拟）
queue := []int{}
queue = append(queue, x)       // enqueue
front := queue[0]              // peek
queue = queue[1:]              // dequeue

// 集合（用 map 模拟）
set := make(map[int]bool)
set[x] = true                 // add
delete(set, x)                // remove
_, exists := set[x]           // contains
```

### 常用工具函数

```go
// Go 1.21+ 内置 min/max
// 旧版本需自定义
func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}

// 排序
sort.Ints(arr)                // 升序
sort.Slice(arr, func(i, j int) bool {
    return arr[i] > arr[j]    // 降序
})

// 字符串 <-> 整数
strconv.Itoa(123)             // int -> string
strconv.Atoi("123")           // string -> int

// 字符串操作
strings.Split(s, ",")
strings.Join(arr, ",")
strings.Contains(s, "abc")
```

### 常用数学常量

```go
import "math"

math.MaxInt32    // 2147483647
math.MinInt32    // -2147483648
math.MaxInt64
math.MinInt64
math.MaxFloat64
```

---

## 附录 C：Python 刷题常用技巧

### 常用数据结构

```python
from collections import defaultdict, Counter, deque
from heapq import heappush, heappop, heapify
from bisect import bisect_left, bisect_right
from functools import lru_cache
import math

# 默认字典
d = defaultdict(int)       # 默认值 0
d = defaultdict(list)      # 默认值 []

# 计数器
cnt = Counter("aabbc")     # {'a': 2, 'b': 2, 'c': 1}

# 双端队列
dq = deque()
dq.append(x)               # 右端入
dq.appendleft(x)           # 左端入
dq.pop()                   # 右端出
dq.popleft()               # 左端出

# 堆（最小堆）
h = []
heappush(h, x)
top = heappop(h)
# 最大堆：取负
heappush(h, -x)
top = -heappop(h)

# 排序
arr.sort()                          # 原地升序
arr.sort(key=lambda x: x[1])       # 按第二元素
arr.sort(reverse=True)              # 降序
sorted_arr = sorted(arr)            # 返回新列表

# 二分查找
pos = bisect_left(arr, x)          # 第一个 >= x
pos = bisect_right(arr, x)         # 第一个 > x

# 记忆化递归
@lru_cache(maxsize=None)
def dp(i, j):
    ...
```

### 常用技巧

```python
# 无穷大
float('inf')
float('-inf')

# 整数最大值（Python 无溢出问题）
# 直接用 float('inf') 即可

# 列表推导
dp = [0] * n
dp = [[0] * n for _ in range(m)]  # ⚠️ 不要用 [[0]*n]*m

# 解包
a, b = b, a                       # 交换
a, b = b, a + b                   # 同时更新

# enumerate
for i, v in enumerate(arr):
    ...

# zip
for a, b in zip(arr1, arr2):
    ...
```

---

> 💡 **最后的建议**：算法面试的核心不在于"做了多少题"，而在于"掌握了多少种解题思路"。每个分类吃透 5-10 道经典题，理解背后的思想，远比盲目刷 300 道题有效。
>
> 祝面试顺利！🎯
