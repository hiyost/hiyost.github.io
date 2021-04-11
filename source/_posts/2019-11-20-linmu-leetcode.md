---
title: leetcode课程记录（讲师：林沐）
date: 2019-12-04 17:06:01
tags: leetcode
---

讲师：林沐

配合视频食用更佳哦！B站链接：https://www.bilibili.com/video/av36288901

## 1、链表

### [206. 反转链表](https://leetcode-cn.com/problems/reverse-linked-list/)

**题目**

反转一个单链表。

示例:

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

进阶:
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？

<!-- more -->

**题解**

简单地翻转

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    var pre *ListNode
    for head != nil {
        tmp := head.Next
        head.Next = pre
        pre = head
        head = tmp
    }
    return pre
}
```



### [92. 反转链表 II](https://leetcode-cn.com/problems/reverse-linked-list-ii/)



**题目**

反转从位置 m 到 n 的链表。请使用一趟扫描完成反转。

说明:
1 ≤ m ≤ n ≤ 链表长度。

示例:

```
输入: 1->2->3->4->5->NULL, m = 2, n = 4
输出: 1->4->3->2->5->NULL
```



**题解**

先找到m所在的位置，然后进行反转，注意要对m=1的情况进行特殊处理

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseBetween(head *ListNode, m int, n int) *ListNode {
    pre, cur, i := &ListNode{Next: head}, head, 1
    for ; i < m && cur != nil; i++ {
        pre = cur
        cur = cur.Next
    }
    // 要反转的部分的尾巴
    tail, nHead := cur, pre
    for ; i <= n && cur != nil; i++ {
        tmp := cur.Next
        cur.Next = pre
        pre = cur
        cur = tmp
    }
    nHead.Next, tail.Next = pre, cur
    // 对于 m = 1的情况，head已经被翻转，这时应该返回nHead.Next
    if m == 1 {
        return nHead.Next
    }
    return head
}
```



### [160. 相交链表](https://leetcode-cn.com/problems/intersection-of-two-linked-lists/)



**题目**

编写一个程序，找到两个单链表相交的起始节点。

如下面的两个链表：

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_statement.png)

在节点 c1 开始相交。

 

示例 1：

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_1.png)

```
输入：intersectVal = 8, listA = [4,1,8,4,5], listB = [5,0,1,8,4,5], skipA = 2, skipB = 3
输出：Reference of the node with value = 8
输入解释：相交节点的值为 8 （注意，如果两个列表相交则不能为 0）。从各自的表头开始算起，链表 A 为 [4,1,8,4,5]，链表 B 为 [5,0,1,8,4,5]。在 A 中，相交节点前有 2 个节点；在 B 中，相交节点前有 3 个节点。
```


示例 2：

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_2.png)

```
输入：intersectVal = 2, listA = [0,9,1,2,4], listB = [3,2,4], skipA = 3, skipB = 1
输出：Reference of the node with value = 2
输入解释：相交节点的值为 2 （注意，如果两个列表相交则不能为 0）。从各自的表头开始算起，链表 A 为 [0,9,1,2,4]，链表 B 为 [3,2,4]。在 A 中，相交节点前有 3 个节点；在 B 中，相交节点前有 1 个节点。
```


示例 3：

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_3.png)

```
输入：intersectVal = 0, listA = [2,6,4], listB = [1,5], skipA = 3, skipB = 2
输出：null
输入解释：从各自的表头开始算起，链表 A 为 [2,6,4]，链表 B 为 [1,5]。由于这两个链表不相交，所以 intersectVal 必须为 0，而 skipA 和 skipB 可以是任意值。
解释：这两个链表不相交，因此返回 null。
```


注意：

- 如果两个链表没有交点，返回 null.

- 在返回结果后，两个链表仍须保持原有的结构。

- 可假定整个链表结构中没有循环。

- 程序尽量满足 O(n) 时间复杂度，且仅用 O(1) 内存。



**题解**

先计算两个链表的长度，然后计算将其对齐，再同时移动直到找到交点

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    // 先计算两个链表的长度
    lenghtA, lenghtB := calculateLength(headA), calculateLength(headB)
    // 保证B是较长的那个链表
    if lenghtA > lenghtB {
        lenghtA, lenghtB, headA, headB = lenghtB, lenghtA, headB, headA
    }
    // 将A和B两个链表对齐
    for ; lenghtB - lenghtA > 0; lenghtB, headB = lenghtB -1, headB.Next {}
    // 同时将AB链表往后移动，知道遇到同一个节点
    for ; lenghtB > 0 && headA != headB;  lenghtB, headB, headA = lenghtB -1, headB.Next, headA.Next {}
    return headA
}

func calculateLength(head *ListNode) int {
    n := 0
    for ; head != nil; head, n = head.Next, n+1 {}
    return n
}
```



### [141. 环形链表](https://leetcode-cn.com/problems/linked-list-cycle/)



**题目**

给定一个链表，判断链表中是否有环。

为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。

 

示例 1：

```
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)

示例 2：

```
输入：head = [1,2], pos = 0
输出：true
解释：链表中有一个环，其尾部连接到第一个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

示例 3：

```
输入：head = [1], pos = -1
输出：false
解释：链表中没有环。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)


进阶：

你能用 O(1)（即，常量）内存解决此问题吗？



**题解**

使用快慢双指针

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func hasCycle(head *ListNode) bool {
    pre := &ListNode{Next: head}
    for fast, slow := head, pre; slow != nil && fast != nil && fast.Next != nil; slow, fast = slow.Next, fast.Next.Next {
        if fast == slow {
            return true
        }
    }
    return false
}
```



### [142. 环形链表 II](https://leetcode-cn.com/problems/linked-list-cycle-ii/)





**题目**

给定一个链表，返回链表开始入环的第一个节点。 如果链表无环，则返回 null。

为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。

说明：不允许修改给定的链表。

 

示例 1：

```
输入：head = [3,2,0,-4], pos = 1
输出：tail connects to node index 1
解释：链表中有一个环，其尾部连接到第二个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)


示例 2：

```
输入：head = [1,2], pos = 0
输出：tail connects to node index 0
解释：链表中有一个环，其尾部连接到第一个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

示例 3：

```
输入：head = [1], pos = -1
输出：no cycle
解释：链表中没有环。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)

进阶：
你是否可以不用额外空间解决此题？





**题解**

相比于141多了个寻找交点的地方，这个过程可以使用数学的方法证明交点的位置位于在找到meet点之后head和meet点的相遇位置：

![142](2019-12-04-leetcode-linmu/142.png)

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func detectCycle(head *ListNode) *ListNode {
    start := true
    for fast, slow := head, head; slow != nil && fast != nil && fast.Next != nil; slow, fast = slow.Next, fast.Next.Next {
        if !start && fast == slow {
            // 找到交点
            for ; fast != head; fast, head = fast.Next, head.Next {}
            return head
        }
        start = false
    }
    return nil
}
```





### [86. 分隔链表](https://leetcode-cn.com/problems/partition-list/)



**题目**

给定一个链表和一个特定值 x，对链表进行分隔，使得所有小于 x 的节点都在大于或等于 x 的节点之前。

你应当保留两个分区中每个节点的初始相对位置。

示例:

```
输入: head = 1->4->3->2->5->2, x = 3
输出: 1->2->2->4->3->5
```





**题解**

使用两个head来处理分隔的左右两边的链表

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func partition(head *ListNode, x int) *ListNode {
    left, right := &ListNode{}, &ListNode{}
    leftHead, rightHead := left, right
    for ; head != nil; head = head.Next {
        if head.Val < x {
            left.Next = head
            left = head
        } else {
            right.Next = head
            right = head
        }
    }
    left.Next = rightHead.Next
    right.Next = nil
    return leftHead.Next
}
```



### [21. 合并两个有序链表](https://leetcode-cn.com/problems/merge-two-sorted-lists/)



**题目**

将两个有序链表合并为一个新的有序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 

示例：

```
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```



**题解**

递归完成，每次找最小的

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    if l1 == nil {
        return l2
    }
    if l2 == nil {
        return l1
    }
    if l1.Val > l2.Val {
        l1, l2 = l2, l1
    }
    l1.Next = mergeTwoLists(l1.Next, l2)
    return l1
}
```





### [23. 合并K个排序链表](https://leetcode-cn.com/problems/merge-k-sorted-lists/)



**题目**

合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

示例:

```
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6
```



**题解**

这个题目首先可以使用暴力的方法求解，也就是每次遍历每个链表，取其中最小的那个，只不过这样到后面很多链表都取完了，一个循环里面每个链表还是会遍历一遍，浪费时间。更好的方式是通过归并排序，也就是二分的思路，将n个链表分成二叉树两两归并。

- 暴力解法

  ```go
  /**
   * Definition for singly-linked list.
   * type ListNode struct {
   *     Val int
   *     Next *ListNode
   * }
   */
  func mergeKLists(lists []*ListNode) *ListNode {
      // 一个迭代里面只处理一个最小的node，然后递归
      minVal, index :=  65535, 0
      for i, list := range lists {
          if list != nil && list.Val < minVal {
              minVal = list.Val
              index = i
          }
      }
      if minVal == 65535 && index == 0{
          return nil
      }
      head := lists[index]
      lists[index] = lists[index].Next
      head.Next = mergeKLists(lists)
      return head
  }
  ```

  

- 归并排序

  ```go
  /**
   * Definition for singly-linked list.
   * type ListNode struct {
   *     Val int
   *     Next *ListNode
   * }
   */
  func mergeKLists(lists []*ListNode) *ListNode {
      n := len(lists)
      if n > 2 {
          return mergeKLists([]*ListNode{mergeKLists(lists[:n/2]), mergeKLists(lists[n/2:])})
      } else if n == 1 {
          return lists[0]
      } else if n == 2 {
          if lists[0] == nil {
              return lists[1]
          }
          if lists[1] == nil {
              return lists[0]
          }
          a, b := lists[0], lists[1]
          if a.Val > b.Val {
              a, b = lists[1], lists[0]
          }
          // 保证cur永远是比b更小的node
          cur := a
          for ; cur.Next != nil; cur = cur.Next {
              if cur.Next.Val > b.Val {
                  cur.Next, b = b, cur.Next
              }
          }
          cur.Next = b
          return a
      }
      return nil
  }
  ```

  



## 2、栈、队列和堆



### [225. 用队列实现栈](https://leetcode-cn.com/problems/implement-stack-using-queues/)



太简单，不解答了



### [232. 用栈实现队列](https://leetcode-cn.com/problems/implement-queue-using-stacks/)



太简单，不解答了



### [155. 最小栈](https://leetcode-cn.com/problems/min-stack/)



太简单，不解答了（双栈）



### [224. 基本计算器](https://leetcode-cn.com/problems/basic-calculator/)*



了解一下使用双栈的解法



### [215. 数组中的第K个最大元素](https://leetcode-cn.com/problems/kth-largest-element-in-an-array/)



**题目**

在未排序的数组中找到第 k 个最大的元素。请注意，你需要找的是数组排序后的第 k 个最大的元素，而不是第 k 个不同的元素。

示例 1:

```
输入: [3,2,1,5,6,4] 和 k = 2
输出: 5
```


示例 2:

```
输入: [3,2,3,1,2,4,5,5,6] 和 k = 4
输出: 4
```


说明:

你可以假设 k 总是有效的，且 1 ≤ k ≤ 数组的长度。



**题解**

方法一：

一种经典的解法是使用堆，维护一个元素个数为k的最小堆，最后只需要返回堆的head就可以了

```go
// 使用最大堆(每一个根节点比叶子节点都要大)来实现
import "container/heap"
type maxHeap []int
func (m maxHeap) Len() int { return len(m) }
func (m maxHeap) Less(i, j int) bool { return m[i] < m[j] }
func (m *maxHeap) Swap(i, j int) { (*m)[i], (*m)[j] = (*m)[j], (*m)[i]}
func (m *maxHeap) Push(x interface{}) {
	(*m) = append((*m), x.(int))
}
func (m *maxHeap) Pop() interface{} {
	n := len(*m)
	x := (*m)[n-1]
	(*m) = (*m)[:n-1]
	return x
}
func (m maxHeap) Top() int { return m[0] }
// 用来确定maxHeap实现了heap.Interface（包含Len、Less、Swap、Push、Pop5个函数）
var _ = heap.Interface(&maxHeap{})

func findKthLargest(nums []int, k int) int {
	h := maxHeap{}
	for _, num := range nums {
		if h.Len() < k || num >= h.Top() {
			heap.Push(&h, num)
		}
		if h.Len() > k {
			heap.Pop(&h)
		}
	}
	return heap.Pop(&h).(int)
}
```



### [295. 数据流的中位数](https://leetcode-cn.com/problems/find-median-from-data-stream/)



**题目**

中位数是有序列表中间的数。如果列表长度是偶数，中位数则是中间两个数的平均值。

例如，

[2,3,4] 的中位数是 3

[2,3] 的中位数是 (2 + 3) / 2 = 2.5

设计一个支持以下两种操作的数据结构：

- `void addNum(int num)` - 从数据流中添加一个整数到数据结构中。

- `double findMedian()` - 返回目前所有元素的中位数。

示例：

```
addNum(1)
addNum(2)
findMedian() -> 1.5
addNum(3) 
findMedian() -> 2
```


进阶:

1. 如果数据流中所有整数都在 0 到 100 范围内，你将如何优化你的算法？
2. 如果数据流中 99% 的整数都在 0 到 100 范围内，你将如何优化你的算法？



**题解**

同时使用大小堆来实现

```go
// minHeap实现小堆（根节点均比叶子节点小）
type minHeap []int
func (m minHeap) Len() int { return len(m) }
func (m minHeap) Less(i, j int) bool { return m[i] < m[j] }
func (m *minHeap) Swap(i, j int) { (*m)[i], (*m)[j] = (*m)[j], (*m)[i]}
func (m *minHeap) Push(x interface{}) {
	(*m) = append((*m), x.(int))
}
func (m *minHeap) Pop() interface{} {
	n := len(*m)
	x := (*m)[n-1]
	(*m) = (*m)[:n-1]
	return x
}
func (m minHeap) Top() int {
	if m.Len() == 0 {return 0}
	return m[0]
}
var _ = heap.Interface(&minHeap{})

// maxHeap实现大堆（根节点均比叶子节点大）
type maxHeap []int
func (m maxHeap) Len() int { return len(m) }
func (m maxHeap) Less(i, j int) bool { return m[i] > m[j] }
func (m *maxHeap) Swap(i, j int) { (*m)[i], (*m)[j] = (*m)[j], (*m)[i]}
func (m *maxHeap) Push(x interface{}) {
	(*m) = append((*m), x.(int))
}
func (m *maxHeap) Pop() interface{} {
	n := len(*m)
	x := (*m)[n-1]
	(*m) = (*m)[:n-1]
	return x
}
func (m maxHeap) Top() int {
	if m.Len() == 0 {return 0}
	return m[0]
}
var _ = heap.Interface(&maxHeap{})


type MedianFinder struct {
	left maxHeap
	right minHeap
}


/** initialize your data structure here. */
func Constructor() MedianFinder {
	return MedianFinder{
		left: maxHeap{},
		right: minHeap{},
	}
}


func (this *MedianFinder) AddNum(num int) {
	if num >= this.right.Top() {
		heap.Push(&this.right, num)
	} else {
		heap.Push(&this.left, num)
	}
	if this.right.Len() == this.left.Len() + 2 {
		num = heap.Pop(&this.right).(int)
		heap.Push(&this.left,num)
	} else if this.right.Len() + 2 == this.left.Len() {
		num = heap.Pop(&this.left).(int)
		heap.Push(&this.right, num)
	}
}


func (this *MedianFinder) FindMedian() float64 {
	if this.right.Len() > this.left.Len() {
		return  float64(this.right.Top())
	} else if this.right.Len() < this.left.Len() {
		return float64(this.left.Top())
	} else {
		return (float64(this.right.Top() + this.left.Top()))/2.0
	}
}
/*
func main() {
	obj := Constructor()
	obj.AddNum(1)
	fmt.Println(obj.FindMedian())
	obj.AddNum(2)
	fmt.Println(obj.FindMedian())
	obj.AddNum(3)
	fmt.Println(obj.FindMedian())
}
*/
```





## 3、贪心算法



### [455. 分发饼干](https://leetcode-cn.com/problems/assign-cookies/)



**题目**

假设你是一位很棒的家长，想要给你的孩子们一些小饼干。但是，每个孩子最多只能给一块饼干。对每个孩子 i ，都有一个胃口值 gi ，这是能让孩子们满足胃口的饼干的最小尺寸；并且每块饼干 j ，都有一个尺寸 sj 。如果 sj >= gi ，我们可以将这个饼干 j 分配给孩子 i ，这个孩子会得到满足。你的目标是尽可能满足越多数量的孩子，并输出这个最大数值。

注意：

你可以假设胃口值为正。
一个小朋友最多只能拥有一块饼干。

示例 1:

```
输入: [1,2,3], [1,1]

输出: 1

解释: 
你有三个孩子和两块小饼干，3个孩子的胃口值分别是：1,2,3。
虽然你有两块小饼干，由于他们的尺寸都是1，你只能让胃口值是1的孩子满足。
所以你应该输出1。
```


示例 2:

```
输入: [1,2], [1,2,3]

输出: 2

解释: 
你有两个孩子和三块小饼干，2个孩子的胃口值分别是1,2。
你拥有的饼干数量和尺寸都足以让所有孩子满足。
所以你应该输出2.
```



**题解**

先将g和s均排序，然后每次取剩下的最小的饼干，看是否满足还未分配的胃口最小的孩子，不满足则继续取更大的饼干，满足则计数。

```go
import "sort"
func findContentChildren(g []int, s []int) int {
    sort.Ints(g)
    sort.Ints(s)
    gLen, sLen, i, j := len(g), len(s), 0, 0
    for i < gLen && j < sLen {
        // s[j]满足分发给g[i]时，递增i即可
        if s[j] >= g[i] {
            i++
        }
        // 无论是否满足都要递增j（每个饼干尝试一次）
        j++
    }
    return i
}
```





### [376. 摆动序列](https://leetcode-cn.com/problems/wiggle-subsequence/)



**题目**

如果连续数字之间的差严格地在正数和负数之间交替，则数字序列称为摆动序列。第一个差（如果存在的话）可能是正数或负数。少于两个元素的序列也是摆动序列。

例如， [1,7,4,9,2,5] 是一个摆动序列，因为差值 (6,-3,5,-7,3) 是正负交替出现的。相反, [1,4,7,2,5] 和 [1,7,4,5,5] 不是摆动序列，第一个序列是因为它的前两个差值都是正数，第二个序列是因为它的最后一个差值为零。

给定一个整数序列，返回作为摆动序列的最长子序列的长度。 通过从原始序列中删除一些（也可以不删除）元素来获得子序列，剩下的元素保持其原始顺序。

示例 1:

```
输入: [1,7,4,9,2,5]
输出: 6 
解释: 整个序列均为摆动序列。
```


示例 2:

```
输入: [1,17,5,10,13,15,10,5,16,8]
输出: 7
解释: 这个序列包含几个长度为 7 摆动序列，其中一个可为[1,17,10,13,10,16,8]。
```


示例 3:

```
输入: [1,2,3,4,5,6,7,8,9]
输出: 2
```


**进阶:**
你能否用 O(n) 时间复杂度完成此题?



**题解**

一个变量记录当前变化方向，变化方向发生变化时计数加1

```go
func wiggleMaxLength(nums []int) int {
    if len(nums) < 2 {
        return len(nums)
    }
    // result保存结果，up保存当前的变化方向，true为上升，false为下降
    result, up := 1, true
    for i, _ := range nums[1:] {
        // 相等时直接跳过
        if nums[i+1] == nums[i] {
            continue
        }
        // 第一次有变化方向时用up记录
        if result == 1 {
            up, result = nums[i+1] > nums[i], result+1
            continue
        }
        if (nums[i+1] > nums[i]) != up {
            result++
            up = nums[i+1] > nums[i]
        }
    }
    return result
}
```





### [402. 移掉K位数字](https://leetcode-cn.com/problems/remove-k-digits/)



**题目**

给定一个以字符串表示的非负整数 num，移除这个数中的 k 位数字，使得剩下的数字最小。

注意:

- num 的长度小于 10002 且 ≥ k。

- num 不会包含任何前导零。

示例 1 :

```
输入: num = "1432219", k = 3
输出: "1219"
解释: 移除掉三个数字 4, 3, 和 2 形成一个新的最小的数字 1219。
```


示例 2 :

```
输入: num = "10200", k = 1
输出: "200"
解释: 移掉首位的 1 剩下的数字为 200. 注意输出不能有任何前导零。
```


示例 3 :

```
输入: num = "10", k = 2
输出: "0"
解释: 从原数字移除所有的数字，剩余为空就是0。
```



**题解**

使用单调递增栈来处理，最小的结果中每一位数字一定是单调递增的

```go
func removeKdigits(num string, k int) string {
    result, n := make([]byte, 0), len(num)
    if k >= n {
        return "0"
    }
    for i := 0; i < n; i++ {
        // result栈顶大于当前需要入栈的值时，将栈中大于该值的元素根据k出栈
        for len(result) > 0 && num[i] < result[len(result)-1] && k > 0 {
            result = result[:len(result)-1]
            k--
        }
        // 将当前值入栈，当然栈为空时不能将0入栈
        if !(len(result) == 0 && num[i] == '0' && i < n-1) {
            result = append(result, num[i])
        }
    }
	return string(result[:len(result)-k])
}
```





### [55. 跳跃游戏](https://leetcode-cn.com/problems/jump-game/)



**题目**

给定一个非负整数数组，你最初位于数组的第一个位置。

数组中的每个元素代表你在该位置可以跳跃的最大长度。

判断你是否能够到达最后一个位置。

示例 1:

```
输入: [2,3,1,1,4]
输出: true
解释: 我们可以先跳 1 步，从位置 0 到达 位置 1, 然后再从位置 1 跳 3 步到达最后一个位置。
```


示例 2:

```
输入: [3,2,1,0,4]
输出: false
解释: 无论怎样，你总会到达索引为 3 的位置。但该位置的最大跳跃长度是 0 ， 所以你永远不可能到达最后一个位置。
```



**题解**

从后往前遍历，每次更新最前面的跳跃点，其实跳跃游戏2更能体现贪心

```go
func canJump(nums []int) bool {
    last := len(nums)-1
    for i := len(nums)-2; i>=0; i-- {
        if i+nums[i] >= last && i < last {
            last = i
        }
    }
	return last == 0
}
```





### [45. 跳跃游戏 II](https://leetcode-cn.com/problems/jump-game-ii/)



**题目**

给定一个非负整数数组，你最初位于数组的第一个位置。

数组中的每个元素代表你在该位置可以跳跃的最大长度。

你的目标是使用最少的跳跃次数到达数组的最后一个位置。

示例:

```
输入: [2,3,1,1,4]
输出: 2
解释: 跳到最后一个位置的最小跳跃数是 2。
     从下标为 0 跳到下标为 1 的位置，跳 1 步，然后跳 3 步到达数组的最后一个位置。
```

**说明:**

假设你总是可以到达数组的最后一个位置。



**题解**

贪心，每次跳跃到自己的范围内下次能跳到达的最远的那个点

```go
func jump(nums []int) int {
    cur, n, step := 0, len(nums), 0
    if n < 2 {
        return 0
    }
	for cur + nums[cur] < n-1 {
        // 贪心，每次跳跃到自己的范围内下次能跳到达的最远的那个点
		max, next := cur, cur
		for i:=cur+1; i <= cur + nums[cur]; i++ {
			if i + nums[i] > max {
				max, next = i + nums[i], i
			}
		}
		cur, step = next, step + 1
	}
	return step + 1
}
```





### [452. 用最少数量的箭引爆气球](https://leetcode-cn.com/problems/minimum-number-of-arrows-to-burst-balloons/)



**题目**



在二维空间中有许多球形的气球。对于每个气球，提供的输入是水平方向上，气球直径的开始和结束坐标。由于它是水平的，所以y坐标并不重要，因此只要知道开始和结束的x坐标就足够了。开始坐标总是小于结束坐标。平面内最多存在104个气球。

一支弓箭可以沿着x轴从不同点完全垂直地射出。在坐标x处射出一支箭，若有一个气球的直径的开始和结束坐标为 xstart，xend， 且满足  xstart ≤ x ≤ xend，则该气球会被引爆。可以射出的弓箭的数量没有限制。 弓箭一旦被射出之后，可以无限地前进。我们想找到使得所有气球全部被引爆，所需的弓箭的最小数量。

Example:

```
输入:
[[10,16], [2,8], [1,6], [7,12]]

输出:
2

解释:
对于该样例，我们可以在x = 6（射爆[2,8],[1,6]两个气球）和 x = 11（射爆另外两个气球）。
```



**题解**



先排序，后遍历每一个区间，当重叠区间已经无法满足后面的区间时新增一杆枪

```go
type array [][]int
func (a array) Len() int { return len(a) }
func (a array) Less(i, j int) bool { return a[i][0] < a[j][0] }
func (a array) Swap(i, j int) { a[i][0], a[i][1], a[j][0], a[j][1] = a[j][0], a[j][1], a[i][0], a[i][1] }
var _ = sort.Interface(array{})

func findMinArrowShots(points [][]int) int {
	if len(points) < 2 {
		return len(points)
	}
	sort.Sort(array(points))
    // curR记录的是当前射击区间的右边界
	curR, step := points[0][1], 1
	for _, v := range points[1:] {
		if v[0] <= curR {
            // 当前区间的右区间变小时更新右区间
            if v[1] < curR {
                curR = v[1]
            }
			continue
		}
        // 当前区间无法满足时加一杆枪并更新右区间
		curR, step = v[1], step+1
	}
	return step
}
```







## 4、递归、回溯与分治



### [78. 子集](https://leetcode-cn.com/problems/subsets/)



**题目**

给定一组**不含重复元素**的整数数组 `nums`，返回该数组所有可能的子集（幂集）。

说明：解集不能包含重复的子集。

示例:

```
输入: nums = [1,2,3]
输出:
[
  [3],
  [1],
  [2],
  [1,2,3],
  [1,3],
  [2,3],
  [1,2],
  []
]
```



**题解**

使用递归的方式，先算出子集的所有子集，然后遍历添加，在子集的基础上，可以选择增加当前的数字或者不增加。

```go
func subsets(nums []int) [][]int {
    n, result := len(nums), [][]int{}
    if n == 0 {
        return append(result, []int{})
    }
    for _, v := range subsets(nums[1:]) {
        result = append(result, v)
        result = append(result, append([]int{nums[0]}, v...))
    }
    return result
}
```





### [90. 子集 II](https://leetcode-cn.com/problems/subsets-ii/)



**题目**

给定一个可能包含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。

说明：解集不能包含重复的子集。

示例:

```
输入: [1,2,2]
输出:
[
  [2],
  [1],
  [1,2,2],
  [2,2],
  [1,2],
  []
]
```





**题解**

相比于子集I，这个题目中输入的数组可能会重复，因此需要去重，

```go
func subsetsWithDup(nums []int) [][]int {
    // 先排序，以便将相等的放在一起
    sort.Ints(nums)
	return subsets(nums)
}

func subsets(nums []int) [][]int {
	n, result := len(nums), [][]int{}
	if n == 0 {
		return append(result, []int{})
	}
	for _, v := range subsets(nums[1:]) {
        // 如果nums[0] == nums[1] == v[0]，那么就不要append v，因为subsets(nums[1:])中肯定也有v[1:]了，后面append([]int{nums[0]}, v...)会与之重复
		if !( n>1 && nums[0] == nums[1] && len(v) > 0 && nums[0] == v[0]) {
			result = append(result, v)
		}
		result = append(result, append([]int{nums[0]}, v...))
	}
	return result
}
```



### [39. 组合总和](https://leetcode-cn.com/problems/combination-sum/)



**题目**

给定一个无重复元素的数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的数字可以无限制重复被选取。

说明：

- 所有数字（包括 target）都是正整数。

- 解集不能包含重复的组合。 

示例 1:

```
输入: candidates = [2,3,6,7], target = 7,
所求解集为:
[
  [7],
  [2,2,3]
]
```


示例 2:

```
输入: candidates = [2,3,5], target = 8,
所求解集为:
[
  [2,2,2,2],
  [2,3,3],
  [3,5]
]
```



**题解**

通过递归遍历剪枝实现

```go
func combinationSum(candidates []int, target int) [][]int {
    if target <= 0 {
        return [][]int{}
    }
    result, n := [][]int{}, len(candidates)
    for i:=0; i<n; i++ {
        // 如果当前数字就能满足要求，就直接加入result中
        if target == candidates[i] {
            result = append(result, []int{candidates[i]})
        }
        // 如果当前的数字与后面的候选数组中能组合出target，则也加入result中
        // 这里使用candidates[i:]是因为当前数字是可以重复的
        if can := combinationSum(candidates[i:], target-candidates[i]); len(can) > 0 {
            for _, v := range can {
                result = append(result, append([]int{candidates[i]}, v...))
            }
        }
    }
    return result
}
```



### [40. 组合总和 II](https://leetcode-cn.com/problems/combination-sum-ii/)



**题目**



给定一个数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的每个数字在每个组合中只能使用一次。

说明：

所有数字（包括目标数）都是正整数。
解集不能包含重复的组合。 
示例 1:

```
输入: candidates = [10,1,2,7,6,1,5], target = 8,
所求解集为:
[
  [1, 7],
  [1, 2, 5],
  [2, 6],
  [1, 1, 6]
]
```


示例 2:

```
输入: candidates = [2,5,2,1,2], target = 5,
所求解集为:
[
  [1,2,2],
  [5]
]
```



**题解**



由于这题的条件中所给的数组中元素是可以重复的，因此要先做个排序，然后逻辑复用39.组合总和就可以了。

```go
func combinationSum2(candidates []int, target int) [][]int {
    sort.Ints(candidates)
    return combinationSum(candidates, target)
}
func combinationSum(candidates []int, target int) [][]int {
    if target <= 0 {
        return [][]int{}
    }
    result, n := [][]int{}, len(candidates)
    for i:=0; i<n; i++ {
        // 如果当前的数字是重复数字则跳过
        if i > 0 && candidates[i] == candidates[i-1] {
            continue
        }
        // 如果当前数字就能满足要求，就直接加入result中
        if target == candidates[i] {
            result = append(result, []int{candidates[i]})
        }
        // 如果当前的数字与后面的候选数组中能组合出target，则也加入result中
        // 这里使用candidates[i+1:]是因为当前数字是不能重复的
        if can := combinationSum(candidates[i+1:], target-candidates[i]); len(can) > 0 {
            for _, v := range can {
                result = append(result, append([]int{candidates[i]}, v...))
            }
        }
    }
    return result
}
```



### [46. 全排列](https://leetcode-cn.com/problems/permutations/)

**题目**

给定一个 没有重复 数字的序列，返回其所有可能的全排列。

示例:

```
输入: [1,2,3]
输出:
[
  [1,2,3],
  [1,3,2],
  [2,1,3],
  [2,3,1],
  [3,1,2],
  [3,2,1]
]
```

**题解**

同样使用递归的方式去求解，先算出子数组的结果

```go
func permute(nums []int) [][]int {
    if len(nums) < 2 {
        return [][]int{nums}
    }
    result := [][]int{}
    for _, v := range permute(nums[1:]) {
        // 将nums[0]插入到v的任何一个位置生成一个result记录
        for i := 0; i <= len(v); i++ {
            r := append([]int{}, v[:i]...)
            r = append(r, nums[0])
            r = append(r, v[i:]...)
            result = append(result, r)
        }
    }
    return result
}
```



### [22. 括号生成](https://leetcode-cn.com/problems/generate-parentheses/)



**题目**

给出 n 代表生成括号的对数，请你写出一个函数，使其能够生成所有可能的并且有效的括号组合。

例如，给出 n = 3，生成结果为：

```
[
  "((()))",
  "(()())",
  "(())()",
  "()(())",
  "()()()"
]
```



**题解**

通过递归实现，每次可以选择增加一个左括号或者右括号，通过计数器来判断是否是满足条件的，满足条件时加入到结果中。注意因为要append到slice中，需要使用slice指针。

```go
func generateParenthesis(n int) []string {
    if n == 0 {
        return []string{}
    }
    result := []string{}
    generate(n, 0, "", &result)
    return result
}
func generate(n, left int, item string, result *[]string) {
    if left == 0 && len(item) == 2*n {
        *result = append(*result, item)
        return
    }
    if left < 0 || left > n || len(item) > 2*n {
        return
    }
    if left < n {
        generate(n, left+1, item+"(", result)
    }
    if left > 0 {
        generate(n, left-1, item+")", result)
    }
}
```





### [51. N皇后](https://leetcode-cn.com/problems/n-queens/)



**题目**



n 皇后问题研究的是如何将 n 个皇后放置在 n×n 的棋盘上，并且使皇后彼此之间不能相互攻击。

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/12/8-queens.png)

上图为 8 皇后问题的一种解法。

给定一个整数 n，返回所有不同的 n 皇后问题的解决方案。

每一种解法包含一个明确的 n 皇后问题的棋子放置方案，该方案中 'Q' 和 '.' 分别代表了皇后和空位。

示例:

```
输入: 4
输出: [
 [".Q..",  // 解法 1
  "...Q",
  "Q...",
  "..Q."],

 ["..Q.",  // 解法 2
  "Q...",
  "...Q",
  ".Q.."]
]
解释: 4 皇后问题存在两个不同的解法。
```



**题解**



dfs，每次在新行加入一个Q之后，查找这个Q导致需要变成'.'的点，dfs完成之后恢复这些点

```go
func solveNQueens(n int) [][]string {
	ret := [][]string{}
	board := [][]byte{}
	for i:=0; i < n; i++ {
		board = append(board, make([]byte, n))
	}
	// 第一行的Q所在的位置
	for i:=0; i < n; i++ {
		ret = dfs(0, i, board, ret)
	}
	return ret
}
 
func dfs(i, j int, board [][]byte, ret [][]string) [][]string {
	board[i][j] = 'Q'
	// 当前已经到达最后一行则说明满足要求并记录到最终的结果中
	if i >= len(board)-1 {
		s := make([]string, len(board))
		for x:=0; x<len(board); x++ {
			s[x] = string(board[x])
		}
		ret = append(ret, s)
		board[i][j] = ' '
		return ret
	}
	cur := [][]int{}
	// 找到当前需要去掉的点并记录下来，方便后面恢复
	for x:=i; x<len(board); x++{
		for y:=0; y<len(board); y++{
			if board[x][y] != '.' && board[x][y] != 'Q' && (x == i || y == j || x-y == i-j || x+y == i+j) {
				cur=append(cur, []int{x, y})
				board[x][y] = '.'
			}
		}
	}
	for y:=0; y<len(board); y++ {
		if board[i+1][y] != '.' {
			ret = dfs(i+1, y, board, ret)
		}
	}
	// 恢复当前去掉的点并去掉Q
	for _, v := range cur {
		board[v[0]][v[1]] = 0
	}
	board[i][j] = ' '
	return ret
}
```





### [315. 计算右侧小于当前元素的个数](https://leetcode-cn.com/problems/count-of-smaller-numbers-after-self/)



**题目**



给定一个整数数组 nums，按要求返回一个新数组 counts。数组 counts 有该性质： counts[i] 的值是  nums[i] 右侧小于 nums[i] 的元素的数量。

示例:

```
输入: [5,2,6,1]
输出: [2,1,1,0] 
解释:
5 的右侧有 2 个更小的元素 (2 和 1).
2 的右侧仅有 1 个更小的元素 (1).
6 的右侧有 1 个更小的元素 (1).
1 的右侧有 0 个更小的元素.
```



**题解**



方法一（归并排序）：

利用归并排序的思路，从后往前遍历数组，每次将当前元素插入到新建的一个排序好的数组中，插入的位置就是当前这个元素右侧小于该元素的个数。

```go
func countSmaller(nums []int) []int {
    n := len(nums)
    if n == 0 {
        return nums
    }
    sortedNums, ret := make([]int, n), make([]int, n)
    count(nums, ret, sortedNums, n-1)
    return ret
}
func count(nums, ret, sortedNums []int, i int) {
    if i < 0 {
        return
    }
    // 插入之后最后一个（最大）元素的索引
    j := len(nums) -1 - i
    // 寻找可以插入的位置，其对应的位置的索引就是右侧小于该值的个数
    for k:=0; k < j; k ++ {
        if sortedNums[k] >= nums[i] {
            ret[i] = k
            for l := j; l > k; l -- {
                sortedNums[l] = sortedNums[l-1]
            }
            sortedNums[k] = nums[i]
            count(nums, ret, sortedNums, i-1)
            return
        }
    }
    // 如果找不到则说明是当前最大的，放在最后
    sortedNums[j], ret[i] = nums[i], j
    count(nums, ret, sortedNums, i-1)
    return
}
```

方法二（树状数组）：

首先要弄清楚树状数组的概念，树状数组的目的在于更加平衡地进行前缀和的更新和查询（普通的做法中更新的时间复杂度为O(1)，查询的复杂度为O(n)，使用树状数组更新和查询的时间复杂度均为为O(log n)），对于大规模的更新和查询而言，其时间效率是大大提升的。





## 5、二叉树与图



### [112. 路径总和](https://leetcode-cn.com/problems/path-sum/)



**题目**



给定一个二叉树和一个目标和，判断该树中是否存在根节点到叶子节点的路径，这条路径上所有节点值相加等于目标和。

说明: 叶子节点是指没有子节点的节点。

示例: 
给定如下二叉树，以及目标和 sum = 22，

              5
             / \
            4   8
           /   / \
          11  13  4
         /  \      \
        7    2      1
返回 true, 因为存在目标和为 22 的根节点到叶子节点的路径 5->4->11->2。



**题解**



深度优先遍历即可。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func hasPathSum(root *TreeNode, sum int) bool {
    if root == nil {
        return false
    }
    // 到达叶子节点
    if root.Left == nil && root.Right == nil {
        return root.Val == sum
    }
    return hasPathSum(root.Left, sum-root.Val) || hasPathSum(root.Right, sum-root.Val)
}
```



### [113. 路径总和 II](https://leetcode-cn.com/problems/path-sum-ii/)



**题目**



给定一个二叉树和一个目标和，找到所有从根节点到叶子节点路径总和等于给定目标和的路径。

说明: 叶子节点是指没有子节点的节点。

示例:
给定如下二叉树，以及目标和 sum = 22，

              5
             / \
            4   8
           /   / \
          11  13  4
         /  \    / \
        7    2  5   1
返回:

```
[
   [5,4,11,2],
   [5,8,4,5]
]
```



**题解**



深度优先遍历时记录路径即可。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func pathSum(root *TreeNode, sum int) [][]int {
    result := [][]int{}
    if root == nil {
        return result
    }
    sumPath(root, sum, []int{}, &result)
    return result
}

func sumPath(root *TreeNode, sum int, item []int, result *[][]int) {
    item = append(item, root.Val)
    if root.Left == nil && root.Right == nil {
        if sum == root.Val {
            *result = append(*result, append([]int{}, item...))
        }
    }
    if root.Left != nil {
        sumPath(root.Left, sum - root.Val, item, result)
    }
    if root.Right != nil {
        sumPath(root.Right, sum - root.Val, item, result)
    }
}
```





### [236. 二叉树的最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)



**题目**



给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。

百度百科中最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”

例如，给定如下二叉树:  root = [3,5,1,6,2,0,8,null,null,7,4]

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/15/binarytree.png)

 

示例 1:

```
输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 1
输出: 3
解释: 节点 5 和节点 1 的最近公共祖先是节点 3。
```


示例 2:

```
输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 4
输出: 5
解释: 节点 5 和节点 4 的最近公共祖先是节点 5。因为根据定义最近公共祖先节点可以为节点本身。
```



**题解**



我的烂算法：

```go
/**
 * Definition for TreeNode.
 * type TreeNode struct {
 *     Val int
 *     Left *ListNode
 *     Right *ListNode
 * }
 */
 func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
     pPath := findPath(root, p)
     qPath := findPath(root, q)
     i := 0
     for ; i<len(pPath) && i<len(qPath) && pPath[i] == qPath[i]; i++ {}
     return pPath[i-1]
}
// 找到节点到root的路径
func findPath(root, node *TreeNode) []*TreeNode {
    if root == nil || (root.Left == nil && root.Right == nil && root != node) {
        return []*TreeNode{}
    }
    result := []*TreeNode{root}
    if root == node {
        return result
    }
    if root.Left != nil {
        if path := findPath(root.Left, node); len(path) > 0 {
            return append(result, path...)
        }
    }
    if root.Right != nil {
        if path := findPath(root.Right, node); len(path) > 0 {
            return append(result, path...)
        }
    }
    return []*TreeNode{}
}
```

优秀的算法：

```go
/**
 * Definition for TreeNode.
 * type TreeNode struct {
 *     Val int
 *     Left *ListNode
 *     Right *ListNode
 * }
 */
 func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
     if root == nil || p == root || q == root {
         return root
     }
     left, right := lowestCommonAncestor(root.Left, p, q), lowestCommonAncestor(root.Right, p, q)
     if left != nil && right != nil {
         return root
     } else if left != nil {
         return left
     } else if right != nil {
         return right
     }
     return nil
}
```



### [114. 二叉树展开为链表](https://leetcode-cn.com/problems/flatten-binary-tree-to-linked-list/)



**题目**



给定一个二叉树，原地将它展开为链表。

例如，给定二叉树

```
    1
   / \
  2   5
 / \   \
3   4   6
```


将其展开为：

```
1
 \
  2
   \
    3
     \
      4
       \
        5
         \
          6
```



**题解**



将右子树移动到左子树的最右边，然后将左子树移动有右子树的位置，递归遍历即可

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func flatten(root *TreeNode)  {
    if root == nil {
        return
    }
    left, right := root.Left, root.Right
    if left == nil {
        flatten(root.Right)
        return
    }
    for left.Right != nil {
        left = left.Right
    }
    left.Right, root.Right = right, root.Left
    root.Left = nil
    flatten(root.Right)
}
```





### [199. 二叉树的右视图](https://leetcode-cn.com/problems/binary-tree-right-side-view/)



**题目**



给定一棵二叉树，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

示例:

```
输入: [1,2,3,null,5,null,4]
输出: [1, 3, 4]
解释:

   1            <---
 /   \
2     3         <---
 \     \
  5     4       <---
```



**题解**



通过队列来进行层序遍历

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
 type levelNode struct {
     node *TreeNode
     level int
 }
func rightSideView(root *TreeNode) []int {
    if root == nil {
        return []int{}
    }
    result, queue := []int{root.Val}, []levelNode{levelNode{node: root, level: 1}}
    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]
        if cur.level > len(result) {
            result = append(result, cur.node.Val)
        }
        // result[cur.level-1]的值最终是这一层最右侧的节点的值
        result[cur.level-1] = cur.node.Val
        if cur.node.Left != nil {
            queue = append(queue, levelNode{node: cur.node.Left, level: cur.level + 1})
        }
        if cur.node.Right != nil {
            queue = append(queue, levelNode{node: cur.node.Right, level: cur.level + 1})
        }
    }
    return result
}
```



### [207. 课程表](https://leetcode-cn.com/problems/course-schedule/)



**题目**



现在你总共有 n 门课需要选，记为 0 到 n-1。

在选修某些课程之前需要一些先修课程。 例如，想要学习课程 0 ，你需要先完成课程 1 ，我们用一个匹配来表示他们: [0,1]

给定课程总量以及它们的先决条件，判断是否可能完成所有课程的学习？

示例 1:

```
输入: 2, [[1,0]] 
输出: true
解释: 总共有 2 门课程。学习课程 1 之前，你需要完成课程 0。所以这是可能的。
```


示例 2:

```
输入: 2, [[1,0],[0,1]]
输出: false
解释: 总共有 2 门课程。学习课程 1 之前，你需要先完成课程 0；并且学习课程 0 之前，你还应先完成课程 1。这是不可能的。
```


说明:

1. 输入的先决条件是由边缘列表表示的图形，而不是邻接矩阵。详情请参见图的表示法。

2. 你可以假定输入的先决条件中没有重复的边。

提示:

1. 这个问题相当于查找一个循环是否存在于有向图中。如果存在循环，则不存在拓扑排序，因此不可能选取所有课程进行学习。

2. 通过 DFS 进行拓扑排序 - 一个关于Coursera的精彩视频教程（21分钟），介绍拓扑排序的基本概念。
   拓扑排序也可以通过 BFS 完成。



**题解**



我的效率较低的算法

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    pre, isVisited := make([][]int, numCourses), make([]int, numCourses)
    for _, v := range prerequisites {
        pre[v[0]] = append(pre[v[0]], v[1])
    }
    fmt.Println(pre)
    for i, _ := range isVisited {
        // isVisited[i] == 1表明已经搜索过，== 0表示没有搜索过，== -1表示正在处理
        if isVisited[i] == 1 {
            continue
        }
        if dfs(i, pre, &isVisited) {
            return false
        }
        fmt.Println(isVisited)
    }
    return true
}
func dfs(cur int, pre [][]int, isVisited *[]int) bool {
    (*isVisited)[cur] = -1
    for _, v := range pre[cur] {
        if (*isVisited)[v] == -1 || ((*isVisited)[v] == 0 && dfs(v, pre, isVisited)) {
            // 说明有环，则返回true，其他情况不返回
            return true
        }
    }
    (*isVisited)[cur] = 1
    return false
}
```

  效率最高的算法

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    inNums := make([]int, numCourses)
    record := make([][]int, numCourses)
    for i:=0; i<len(prerequisites); i++ {
        toLearn, requisite := prerequisites[i][0], prerequisites[i][1]
        inNums[toLearn]++
        record[requisite] = append(record[requisite], toLearn)
    }
    
    for i:=0; i<len(inNums); {
        if inNums[i] == 0 {
            inNums[i] = -1
            for _, canLearn := range record[i] {
                inNums[canLearn]--
            }
            i = 0
            continue
        }
        i++
    }
    
    for i:=0; i<len(inNums); i++ {
        if inNums[i] != -1 {
            return false
        }
    }
    return true
}
```





## 6、二分查找与二叉排序树



### [35. 搜索插入位置](https://leetcode-cn.com/problems/search-insert-position/)



**题目**



给定一个排序数组和一个目标值，在数组中找到目标值，并返回其索引。如果目标值不存在于数组中，返回它将会被按顺序插入的位置。

你可以假设数组中无重复元素。

示例 1:

```
输入: [1,3,5,6], 5
输出: 2
```

示例 2:

```
输入: [1,3,5,6], 2
输出: 1
```


示例 3:

```
输入: [1,3,5,6], 7
输出: 4
```


示例 4:

```
输入: [1,3,5,6], 0
输出: 0
```



**题解**



这个题目使用暴力比二分效率更高啊

```go
func searchInsert(nums []int, target int) int {
    left, right := 0, len(nums) - 1
    if target > nums[right] {
        return right+1
    }
    if target <= nums[left] {
        return left
    }
    // 二分搜索
    for right > left+1 {
        mid := left + (right-left)/2
        if nums[mid] == target {
            return mid
        } else if nums[mid] > target {
            right = mid
        } else {
            left = mid
        }
    }
    return right
}
```



### [34. 在排序数组中查找元素的第一个和最后一个位置](https://leetcode-cn.com/problems/find-first-and-last-position-of-element-in-sorted-array/)*





### [33. 搜索旋转排序数组](https://leetcode-cn.com/problems/search-in-rotated-sorted-array/)*





### [315. 计算右侧小于当前元素的个数](https://leetcode-cn.com/problems/count-of-smaller-numbers-after-self/)



**题目**



给定一个整数数组 nums，按要求返回一个新数组 counts。数组 counts 有该性质： counts[i] 的值是  nums[i] 右侧小于 nums[i] 的元素的数量。

示例:

```
输入: [5,2,6,1]
输出: [2,1,1,0] 
解释:
5 的右侧有 2 个更小的元素 (2 和 1).
2 的右侧仅有 1 个更小的元素 (1).
6 的右侧有 1 个更小的元素 (1).
1 的右侧有 0 个更小的元素.
```



**题解**



方法一（归并排序）：

利用归并排序的思路，从后往前遍历数组，每次将当前元素插入到新建的一个排序好的数组中，插入的位置就是当前这个元素右侧小于该元素的个数。

```go
func countSmaller(nums []int) []int {
    n := len(nums)
    if n == 0 {
        return nums
    }
    sortedNums, ret := make([]int, n), make([]int, n)
    count(nums, ret, sortedNums, n-1)
    return ret
}
func count(nums, ret, sortedNums []int, i int) {
    if i < 0 {
        return
    }
    // 插入之后最后一个（最大）元素的索引
    j := len(nums) -1 - i
    // 寻找可以插入的位置，其对应的位置的索引就是右侧小于该值的个数
    for k:=0; k < j; k ++ {
        if sortedNums[k] >= nums[i] {
            ret[i] = k
            for l := j; l > k; l -- {
                sortedNums[l] = sortedNums[l-1]
            }
            sortedNums[k] = nums[i]
            count(nums, ret, sortedNums, i-1)
            return
        }
    }
    // 如果找不到则说明是当前最大的，放在最后
    sortedNums[j], ret[i] = nums[i], j
    count(nums, ret, sortedNums, i-1)
    return
}
```

方法二（树状数组）：

首先要弄清楚树状数组的概念，树状数组的目的在于更加平衡地进行前缀和的更新和查询（普通的做法中更新的时间复杂度为O(1)，查询的复杂度为O(n)，使用树状数组更新和查询的时间复杂度均为为O(log n)），对于大规模的更新和查询而言，其时间效率是大大提升的。





## 7、哈希表与字符串



### [409. 最长回文串](https://leetcode-cn.com/problems/longest-palindrome/)



**题目**



给定一个包含大写字母和小写字母的字符串，找到通过这些字母构造成的最长的回文串。

在构造过程中，请注意区分大小写。比如 "Aa" 不能当做一个回文字符串。

注意:
假设字符串的长度不会超过 1010。

示例 1:

```
输入:
"abccccdd"

输出:
7

解释:
我们可以构造的最长的回文串是"dccaccd", 它的长度是 7。
```



**题解**



用hash表记录每个字母的个数

```go
func longestPalindrome(s string) int {
    hash, n := make([]int, 256), len(s)
    for i:=0; i<n; i++ {
        hash[s[i]-'a'] ++
    }
    sum := 0
    for _, v := range hash {
        if v%2 == 0 {
            sum += v
        } else {
            sum += v-1
        }
    }
    if sum < n {
        sum ++
    }
    return sum
}
```



### [290. 单词规律](https://leetcode-cn.com/problems/word-pattern/)



**题目**



给定一种规律 pattern 和一个字符串 str ，判断 str 是否遵循相同的规律。

这里的 遵循 指完全匹配，例如， pattern 里的每个字母和字符串 str 中的每个非空单词之间存在着双向连接的对应规律。

示例1:

```
输入: pattern = "abba", str = "dog cat cat dog"
输出: true
```


示例 2:

```
输入:pattern = "abba", str = "dog cat cat fish"
输出: false
```


示例 3:

```
输入: pattern = "aaaa", str = "dog cat cat dog"
输出: false
```


示例 4:

```
输入: pattern = "abba", str = "dog dog dog dog"
输出: false
```


**说明:**
你可以假设 pattern 只包含小写字母， str 包含了由单个空格分隔的小写字母。    



**题解**



用两个hash表来建立对应关系

```go
func wordPattern(pattern string, str string) bool {
	data := strings.Fields(str)
    if len(data) != len(pattern) {
        return false
    }
	b2s, s2b := make(map[byte]string), make(map[string]byte)
	for i, v := range data {
		if m, ok := b2s[pattern[i]]; ok {
			if m != v {
				return false
			}
		} else {
			b2s[pattern[i]] = v
		}
		if m, ok := s2b[v]; ok {
			if m != pattern[i] {
				return false
			}
		} else {
			s2b[v] = pattern[i]
		}
	}
	return true
}
```





### [49. 字母异位词分组](https://leetcode-cn.com/problems/group-anagrams/)



**题目**



给定一个字符串数组，将字母异位词组合在一起。字母异位词指字母相同，但排列不同的字符串。

示例:

```
输入: ["eat", "tea", "tan", "ate", "nat", "bat"],
输出:
[
  ["ate","eat","tea"],
  ["nat","tan"],
  ["bat"]
]
```


说明：

所有输入均为小写字母。
不考虑答案输出的顺序。



**题解**



注意：由于go语言中是没有自带byte数组的排序，所以需要自行实现排序的interface。

```go
type Byte []byte
func (b Byte) Len() int { return len(b) }
func (b Byte) Less(i, j int) bool { return b[i] < b[j] }
func (b Byte) Swap(i, j int) { b[i], b[j] = b[j], b[i] }
// 先用hash表来保存字母相同的字符串，然后组合成二维字符串数组
func groupAnagrams(strs []string) [][]string {
    hash := make(map[string][]string)
    for _, v := range strs {
        b := Byte(v)
        sort.Sort(b)
        s := string(b)
        if _, ok := hash[s]; ok {
            hash[s] = append(hash[s], v)
        } else {
            hash[s] = []string{v}
        }
    }
    result := [][]string{}
    for _, v := range hash {
        result = append(result, v)
    }
    return result
}
```





### [3. 无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)



**题目**



给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。

示例 1:

输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
示例 2:

```
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```


示例 3:

```
输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```



**题解**



左右双指针移动（20ms）

```go
func lengthOfLongestSubstring(s string) int {
	hash := make(map[byte]int)
	left, right, max, n := 0, 0, 0, len(s)
	for ; right < n; right++ {
		// 先移动右指针，直到有一个发生重复
		for ; ; right++ {
			if right == n {
				return maximum(max, right - left)
			}
			if v, ok := hash[s[right]]; ok {
				hash[s[right]]++
				if v == 1 {
					max = maximum(max, right - left)
					break
				}
			} else {
				hash[s[right]] = 1
			}
		}
		// fmt.Println(s[left:right])
		// 再移动左指针，直到消除这个重复
		for ; hash[s[right]] > 1; left++ {
			hash[s[left]]--
		}
	}
	return max
}

func maximum(i, j int) int {
    if i > j {
        return i
    }
    return j
}
```

0ms的实现

```go
func lengthOfLongestSubstring(s string) int {
    // 定义游标尺寸大小,游标的左边位置
    window,start := 0,0

    // 循环字符串
    for key := 0; key < len(s); key++ {
        // 查看当前字符串是否在游标内
        isExist := strings.Index(string(s[start:key]), string(s[key]));
        
        // 如果不存在游标内部,游标长度重新计算并赋值
        if (isExist == -1) {
            if (key - start + 1 > window) {
                window = key - start + 1
            }
        } else { //存在，游标开始位置更换为重复字符串位置的下一个位置
            start = start + 1 + isExist
        }
    }
    return window
}
```





### [187. 重复的DNA序列](https://leetcode-cn.com/problems/repeated-dna-sequences/)



**题目**



所有 DNA 都由一系列缩写为 A，C，G 和 T 的核苷酸组成，例如：“ACGAATTCCG”。在研究 DNA 时，识别 DNA 中的重复序列有时会对研究非常有帮助。

编写一个函数来查找 DNA 分子中所有出现超过一次的 10 个字母长的序列（子串）。

 

示例：

```
输入：s = "AAAAACCCCCAAAAACCCCCCAAAAAGGGTTT"
输出：["AAAAACCCCC", "CCCCCAAAAA"]
```



**题解**



最简单的方法就是直接使用hash表来记录

```go
func findRepeatedDnaSequences(s string) []string {
    n := len(s)
    if n <= 10 {
        return []string{}
    }
    hash := make(map[string]int)
    for i:=0; i<=n-10; i++ {
        if _, ok := hash[s[i:i+10]]; ok {
            hash[s[i:i+10]]++
        } else {
            hash[s[i:i+10]] = 1
        }
    }
    result := []string{}
    for k, v := range hash {
        if v > 1 {
            result = append(result, k)
        }
    }
    return result
}
```

通过重新编码来提高查询的效率

```go
func findRepeatedDnaSequences(s string) []string {
    if len(s) < 10 {
        return []string{}
    }

    mt := map[byte]byte {
        'A' : 0x0,
        'C' : 0x1,
        'G' : 0x2,
        'T' : 0x3,
    }
    seen := make([]byte, 1 << 20) //
    ans := []string{}
    cur := uint32(0)
    for i := 0; i < 9; i++ {
        cur = ((cur << 2) & 0xFFFFF) | uint32(mt[s[i]])
    }
    for i := 9; i < len(s); i++ {
        cur = ((cur << 2) & 0xFFFFF) | uint32(mt[s[i]])
        if seen[cur] == 0 {
            seen[cur] = 1
        } else if seen[cur] == 1 {
            ans = append(ans, string(s[i-9:i+1])) //
            seen[cur] = 2
        }
    }
    return ans
}
```





### [76. 最小覆盖子串](https://leetcode-cn.com/problems/minimum-window-substring/)



**题目**



给你一个字符串 S、一个字符串 T，请在字符串 S 里面找出：包含 T 所有字母的最小子串。

示例：

```
输入: S = "ADOBECODEBANC", T = "ABC"
输出: "BANC"
```

说明：

- 如果 S 中不存这样的子串，则返回空字符串 ""。
- 如果 S 中存在这样的子串，我们保证它是唯一的答案。



**题解**



使用两个指针来进行搜索，从前往后慢慢搜索满足要求的子串，后面的指针用来找到满足要求的子串，前面的指针用来移动到长度最小。注意这里没有用字符比较的方式进行匹配，而是通过一个计数器来判断，具体请看代码。

```go
func minWindow(s string, t string) string {
    tMap, sMap := make([]int, 58), make([]int, 58)
    for _, i := range t {
        tMap[i-'A']++
    }
    head, tail, minL, minR := 0, 0, 0, len(s)+1
    for fulfill := 0; tail < len(s); tail++ {
        // t中不存在这个字符则不用管
        if tMap[s[tail]-'A'] == 0 {
            continue
        }
        sMap[s[tail]-'A']++
        if sMap[s[tail]-'A'] <= tMap[s[tail]-'A'] {
            // 字符只有小于或等于t这个字符串中该字符的个数时才产生作用
            fulfill++
        }
        if fulfill == len(t) {
            // 满足当前要求，则向右移动head，直到遇到第一个使子串不满足要求为止
            // 即s[head:tail+1]满足要求，s[head+1:tail+1]不满足，为当前的最短子串
            for ;tMap[s[head]-'A'] == 0 || sMap[s[head]-'A'] > tMap[s[head]-'A']; head++ {
                if tMap[s[head]-'A'] >= 0 {
                    sMap[s[head]-'A']--
                }
            }
            if tail-head < minR-minL {
                 minR, minL = tail,head
            }
            // head右移一位，这样s[head:tail+1]又不满足要求了
            sMap[s[head]-'A']--
            head++
            fulfill--
        }
    }
    if minR== len(s)+1 {
        return ""
    }
    return s[minL:minR+1]
}
```





## 8、搜索





### [200. 岛屿数量](https://leetcode-cn.com/problems/number-of-islands/)



**题目**



给定一个由 '1'（陆地）和 '0'（水）组成的的二维网格，计算岛屿的数量。一个岛被水包围，并且它是通过水平方向或垂直方向上相邻的陆地连接而成的。你可以假设网格的四个边均被水包围。

示例 1:

```
输入:
11110
11010
11000
00000

输出: 1
```


示例 2:

```
输入:
11000
11000
00100
00011

输出: 3
```



**题解**



简单的dfs，每次遇到岛屿的时候把这个岛屿所有的陆地都染色。

```go
func numIslands(grid [][]byte) int {
    sum := 0
    for i, v := range grid {
        for j, _ := range v {
            if grid[i][j] == '1' {
                sum ++
                sameIsland(grid, i, j)
            }
        }
    }
    return sum
}

func sameIsland(grid [][]byte, i, j int) {
    near := [][]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}
    grid[i][j] = '2'
    for _, v := range near {
        if i+v[0] >= len(grid) || i+v[0] < 0 || j+v[1] >= len(grid[0]) || j+v[1]<0 || grid[i+v[0]][j+v[1]] != '1' {
            continue
        }
        sameIsland(grid, i+v[0], j+v[1])
    }
}
```





### [127. 单词接龙](https://leetcode-cn.com/problems/word-ladder/)



**题目**



给定两个单词（beginWord 和 endWord）和一个字典，找到从 beginWord 到 endWord 的最短转换序列的长度。转换需遵循如下规则：

每次转换只能改变一个字母。
转换过程中的中间单词必须是字典中的单词。
说明:

如果不存在这样的转换序列，返回 0。
所有单词具有相同的长度。
所有单词只由小写字母组成。
字典中不存在重复的单词。
你可以假设 beginWord 和 endWord 是非空的，且二者不相同。
示例 1:

```
输入:
beginWord = "hit",
endWord = "cog",
wordList = ["hot","dot","dog","lot","log","cog"]

输出: 5

解释: 一个最短转换序列是 "hit" -> "hot" -> "dot" -> "dog" -> "cog",
     返回它的长度 5。
```


示例 2:

```
输入:
beginWord = "hit"
endWord = "cog"
wordList = ["hot","dot","dog","lot","log"]

输出: 0

解释: endWord "cog" 不在字典中，所以无法进行转换。
```



**题解**



这题用dfs会超时，所以最好使用bfs

```go
// dfs，会超时
import "math"
func ladderLength(beginWord string, endWord string, wordList []string) int {
    n := len(wordList)
    for i, v := range wordList {
        if v == endWord {
            break
        }
        if i == n - 1 {
            return 0
        }
    }
    num := math.MaxInt64
    dfs(beginWord, endWord, 1, &num, wordList)
    if num == math.MaxInt64 {
        return 0
    }
    return num
}
func dfs(beginWord, endWord string, step int, num *int, wordList []string) {
    if beginWord == endWord {
        if step < *num {
            *num = step
        }
        return
    }
    for i, v := range wordList {
        if canConvert(beginWord, v) {
            list := append([]string{}, wordList[:i]...)
            list = append(list, wordList[i+1:]...)
            dfs(v, endWord, step+1, num, list)
        }
    }
}
func canConvert(cur, next string) bool {
    if len(cur) != len(next) {
        return false
    }
    sum, n := 0, len(cur)
    for i:=0; i < n; i++ {
        if cur[i] != next[i] {
            sum++
        }
    }
    if sum == 1 {
        return true
    }
    return false
}
```

188ms

```go
// bfs
import "math"
func ladderLength(beginWord string, endWord string, wordList []string) int {
    if beginWord == endWord {
        return 0
    }
    n := len(wordList)
    for i, v := range wordList {
        if v == endWord {
            break
        }
        if i == n - 1 {
            return 0
        }
    }
    list := make([][]int, n)
    for i:=0; i<n; i++ {
        for j:=i+1; j<n; j++ {
            if canConvert(wordList[i], wordList[j]) {
                list[i] = append(list[i], j)
                list[j] = append(list[j], i)
            }
        }
    }
    num := math.MaxInt64
    for i:=0; i<n; i++ {
        if canConvert(beginWord, wordList[i]) {
            bfs(i, wordList, list, &num, endWord)
        }
    }
    if num == math.MaxInt64 {
        return 0
    }
    return num
}
func bfs(i int, wordList []string, list [][]int, num *int, endWord string) {
    if wordList[i] == endWord {
        *num = 2
    }
    isVisited := make([]bool, len(wordList))
    isVisited[i] = true
    queue := [][]int{{i, 2}}
    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]
        for _, v := range list[cur[0]] {
            if !isVisited[v] {
                if wordList[v] == endWord {
                    if cur[1]+1 < *num {
                        *num = cur[1]+1
                    }
                    return
                }
                isVisited[v] = true
                queue = append(queue, []int{v, cur[1]+1})
            }
        }
    }
}

func canConvert(cur, next string) bool {
    if len(cur) != len(next) {
        return false
    }
    sum, n := 0, len(cur)
    for i:=0; i < n; i++ {
        if cur[i] != next[i] {
            sum++
        }
    }
    if sum == 1 {
        return true
    }
    return false
}
```



排名第一的算法16ms

```go
func ladderLength(beginWord string, endWord string, wordList []string) int {
	wordDict := make(map[string]struct{})
	for _, word := range wordList {
		wordDict[word] = struct{}{}
	}
	// 提前返回
	if _, ok := wordDict[endWord]; !ok {
		return 0
	}

	res := 1
	src, dst := map[string]struct{}{}, map[string]struct{}{}
	src[beginWord] = struct{}{}
	dst[endWord] = struct{}{}

	found := false

	ok := false
	for len(src) != 0 && len(dst) != 0 && !found {
		res++
		// 始终遍历 小数组
		if len(src) > len(dst) {
			src, dst = dst, src
		}
		for w := range src {
			delete(wordDict, w)
		}

		newSrc := make(map[string]struct{})
		for word := range src {
			bytes := []byte(word) // 转成 []byte，然后修改值
			for i := 0; i < len(bytes); i++ {
				for j := 0; j < 26; j++ {
					bytes[i] = byte(j) + 'a' // 修改为 a-z
					target := string(bytes) // 转成 string
					if _, ok = dst[target]; ok { // 已经连通
						return res
					} else {
						if _, ok = wordDict[target]; ok {
							newSrc[target] = struct{}{}
						}
					}
				}
				// 恢复原来的值
				bytes[i] = word[i]
			}
		}
		src = newSrc
	}

	return 0
}
```

双向bfs了解一下。



### [126. 单词接龙 II](https://leetcode-cn.com/problems/word-ladder-ii/)*







### [473. 火柴拼正方形](https://leetcode-cn.com/problems/matchsticks-to-square/)



**题目**



还记得童话《卖火柴的小女孩》吗？现在，你知道小女孩有多少根火柴，请找出一种能使用所有火柴拼成一个正方形的方法。不能折断火柴，可以把火柴连接起来，并且每根火柴都要用到。

输入为小女孩拥有火柴的数目，每根火柴用其长度表示。输出即为是否能用所有的火柴拼成正方形。

示例 1:

```
输入: [1,1,2,2,2]
输出: true

解释: 能拼成一个边长为2的正方形，每边两根火柴。
```


示例 2:

```
输入: [3,3,3,3,4]
输出: false

解释: 不能用所有火柴拼成一个正方形。
```


注意:

- 给定的火柴长度和在 0 到 10^9之间。 

- 火柴数组的长度不超过15。



**题解**



可以使用建议的dfs进行深度遍历，遍历之前对其进行排序，从长到短进行遍历

```go
func makesquare(nums []int) bool {
    sum, n := sumOf(nums), len(nums)
    if sum%4 != 0 || n < 4 {
        return false
    }
    // per保存的是每个边的边长，count保存的是当前每个边已经加入的火柴的总长度
    per, count := sum/4, make([]int, 4)
    sort.Ints(nums)
    if nums[n-1] > per {
        return false
    }
    return dfs(nums, count, per, n-1)
}
func dfs(nums, count []int, per, i int) bool {
    if i < 0 {
        return true
    }
    for j, _ := range count {
        if nums[i] <= per - count[j] {
            count[j] += nums[i]
            if dfs(nums, count, per, i-1) {
                return true
            }
            // 失败后需要回退
            count[j] -= nums[i]
        }
    }
    return false
}
func sumOf(nums []int) int {
    sum := 0
    for _, v := range nums {
        sum += v
    }
    return sum
}
```





### [407. 接雨水 II](https://leetcode-cn.com/problems/trapping-rain-water-ii/)*





## 9、动态规划





### [70. 爬楼梯](https://leetcode-cn.com/problems/climbing-stairs/)



**题目**



假设你正在爬楼梯。需要 n 阶你才能到达楼顶。

每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？

注意：给定 n 是一个正整数。

示例 1：

```
输入： 2
输出： 2
解释： 有两种方法可以爬到楼顶。
   
1. 1 阶 + 1 阶
2. 2 阶
```


示例 2：
```
输入： 3
输出： 3
解释： 有三种方法可以爬到楼顶。

1.  1 阶 + 1 阶 + 1 阶
2.  1 阶 + 2 阶
3.  2 阶 + 1 阶
```



**题解**



```go
// 动态规划
// 转移方程：d[i] = d[i-1] + d[i-2]
// 也就是d[i]是基于d[i-1]走一个台阶上来或者基于d[i-2]走两个台阶上来
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }
    d := make([]int, n)
    d[0], d[1] = 1, 2
    for i := 2; i < n; i++ {
        d[i] = d[i-1] + d[i-2]
    }
    return d[n-1]
}
```





### [198. 打家劫舍](https://leetcode-cn.com/problems/house-robber/)



**题目**



你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警。

给定一个代表每个房屋存放金额的非负整数数组，计算你在不触动警报装置的情况下，能够偷窃到的最高金额。

示例 1:

```
输入: [1,2,3,1]
输出: 4
解释: 偷窃 1 号房屋 (金额 = 1) ，然后偷窃 3 号房屋 (金额 = 3)。
     偷窃到的最高金额 = 1 + 3 = 4 。
```


示例 2:

```
输入: [2,7,9,3,1]
输出: 12
解释: 偷窃 1 号房屋 (金额 = 2), 偷窃 3 号房屋 (金额 = 9)，接着偷窃 5 号房屋 (金额 = 1)。
     偷窃到的最高金额 = 2 + 9 + 1 = 12 。
```



**题解**



```go
// 动态规划
// 转移方程： d[i] = max{list[i] + d[i-2], d[i-1]}，由于只使用到了前两次的数据，因此只需要有两个变量就可以了
func rob(nums []int) int {
    pre, cur := 0, 0
    for _, v := range nums {
        pre, cur = cur, max(cur, pre + v)
    }
    return cur
}

func max(i, j int) int {
    if i>j {
        return i
    }
    return j
}
```





### [53. 最大子序和](https://leetcode-cn.com/problems/maximum-subarray/)



**题目**



给定一个整数数组 nums ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

示例:

```
输入: [-2,1,-3,4,-1,2,1,-5,4],
输出: 6
解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。
```

**进阶:**

如果你已经实现复杂度为 O(n) 的解法，尝试使用更为精妙的分治法求解。



**题解**



```go
// 使用动态规划
// d[i]表示以i结束的子数组的最大和，但是这里只依赖d[i-1]，因此可以使用一个变量sum来表示
func maxSubArray(nums []int) int {
    max, sum := nums[0], nums[0]
    for _, v := range nums[1:] {
        if sum < 0 {
            sum = v
        } else {
            sum += v
        }
        if sum > max {
            max = sum
        }
    }
    return max
}
```





### [322. 零钱兑换](https://leetcode-cn.com/problems/coin-change/)



**题目**



给定不同面额的硬币 coins 和一个总金额 amount。编写一个函数来计算可以凑成总金额所需的最少的硬币个数。如果没有任何一种硬币组合能组成总金额，返回 -1。

示例 1:

```
输入: coins = [1, 2, 5], amount = 11
输出: 3 
解释: 11 = 5 + 5 + 1
```


示例 2:

```
输入: coins = [2], amount = 3
输出: -1
```


说明:
你可以认为每种硬币的数量是无限的。



**题解**



动态规划

```go
// 动态规划，dp[i]中保存i的最小钞票张数，那么d[i]就是min{d[i-v]+1, 其中v是小于i的钞票面额}
func coinChange(coins []int, amount int) int {
	dp := make([]int, amount+1)
	for i, _ := range dp {
		dp[i] = -1
	}
	sort.Ints(coins)
	for _, v := range coins {
		if v > amount {
			break
		}
		dp[v] = 1
	}
    dp[0] = 0
	for i := coins[0]+1; i < amount+1; i++ {
		if dp[i] == 1 {
			continue
		}
		for _, v := range coins {
			if v > i {
				break
			}
			if dp[i-v] != -1 && (dp[i] == -1 || dp[i-v]+1 < dp[i]) {
				dp[i] = dp[i-v] + 1
			}
		}
	}
	return dp[amount]
}
```





### [120. 三角形最小路径和](https://leetcode-cn.com/problems/triangle/)（经典）



**题目**



给定一个三角形，找出自顶向下的最小路径和。每一步只能移动到下一行中相邻的结点上。

例如，给定三角形：

```
[
     [2],
    [3,4],
   [6,5,7],
  [4,1,8,3]
]
```


自顶向下的最小路径和为 11（即，2 + 3 + 5 + 1 = 11）。

说明：

- 如果你可以只使用 O(n) 的额外空间（n 为三角形的总行数）来解决这个问题，那么你的算法会很加分。



**题解**



```go
// 动态规划，dp[j][i]表示第j行第i列的最小路径，dp[j][i]=min{dp[j-1][i], dp[j-1][i-1]}
// 由于dp[j][i]只与dp的第j-1行有关，因此可以将dp改成1维
func minimumTotal(triangle [][]int) int {
    if len(triangle) == 0 {
        return 0
    }
    dp := make([]int, len(triangle))
    for _, v := range triangle {
        // 初始化第一行
		if len(v) == 1 {
			dp[0] = v[0]
			continue
		}
        dp[len(v)-1] = dp[len(v)-2] + v[len(v)-1]
        for i:=len(v)-2; i>0; i-- {
            dp[i] = v[i] + min(dp[i], dp[i-1])
        }
        dp[0] = dp[0] + v[0]
    }
	minStep := dp[0]
	for _, v := range dp[1:] {
		minStep = min(minStep, v)
	}
	return minStep
}
func min(i, j int) int {
    if i<j {
        return i
    }
    return j
}
```





### [300. 最长上升子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)



**题目**



给定一个无序的整数数组，找到其中最长上升子序列的长度。

示例:

```
输入: [10,9,2,5,3,7,101,18]
输出: 4 
解释: 最长的上升子序列是 [2,3,7,101]，它的长度是 4。
```


说明:

- 可能会有多种最长上升子序列的组合，你只需要输出对应的长度即可。 

- 你算法的时间复杂度应该为 O(n2) 。

**进阶:** 你能将算法的时间复杂度降低到 O(n log n) 吗?



**题解**



常规的动态规划

```go
// 动态规划，dp[i]表示以i结尾的最长长度，dp[i]=max{dp[j]+1, 0<=j<i && nums[j]<nums[i]}
func lengthOfLIS(nums []int) int {
    if len(nums) < 2 {
        return len(nums)
    }
    dp := make([]int, len(nums))
    dp[0] = 1
    for i, _ := range nums[1:] {
        dp[i+1] = 1
        for j:=0; j <= i; j++ {
            if nums[i+1] > nums[j] && dp[j] + 1 > dp[i+1] {
                dp[i+1] = dp[j] + 1
            }
        }
    }
    max := 0
    for _, v := range dp {
        if v > max {
            max = v
        }
    }
    return max
}
```

通过栈和二分查找

```go
func lengthOfLIS(nums []int) int {
    // O(nlogn)
    rec := make([]int, len(nums)) // 记录和更新当前每个位置可能的最小值
    res := 0 // 当前最长长度，其实也是rec的长度
    for i := 0; i < len(nums); i++ {
        // 二分查找
        left, right := 0, res
        for left < right {
            mid := (left+right)/2
            if rec[mid] < nums[i] {
                left = mid+1
            } else {
                right = mid
            }
        }
        if right == res {
            res += 1
            rec[right] = nums[i]
        } else {
            rec[left] = nums[i]
        }
    }
    return res
}
```





### [64. 最小路径和](https://leetcode-cn.com/problems/minimum-path-sum/)



**题目**



给定一个包含非负整数的 m x n 网格，请找出一条从左上角到右下角的路径，使得路径上的数字总和为最小。

说明：每次只能向下或者向右移动一步。

示例:

```
输入:
[
  [1,3,1],
  [1,5,1],
  [4,2,1]
]
输出: 7
解释: 因为路径 1→3→1→1→1 的总和最小。
```



**题解**



```go
// 动态规划，dp[i][j]表示从[0, 0]到[i, j]的最小路径和，转移方程dp[i][j]=min{dp[i][j-1], dp[i-1][j]}+grid[i][j]
// 由于dp[i][j]只跟dp[i-1][]相关，因此可以简化成一维，转移方程dp[j]=min{dp[j-1], dp[j]}+grid[i][j]
func minPathSum(grid [][]int) int {
    if len(grid) == 0 {
        return 0
    }
    m, n := len(grid), len(grid[0])
    dp := make([]int, n)
    dp[0] = grid[0][0]
    for i, _ := range dp[1:] {
        dp[i+1] = grid[0][i+1] + dp[i]
    }
    for i:=1; i<m; i++ {
        dp[0] += grid[i][0]
        for j:=1; j<n; j++ {
            dp[j] = min(dp[j-1], dp[j]) + grid[i][j]
        }
    }
    return dp[n-1]
}
func min(i, j int) int {
    if i<j {
        return i
    }
    return j
}
```





### [174. 地下城游戏](https://leetcode-cn.com/problems/dungeon-game/)



**题目**



一些恶魔抓住了公主（P）并将她关在了地下城的右下角。地下城是由 M x N 个房间组成的二维网格。我们英勇的骑士（K）最初被安置在左上角的房间里，他必须穿过地下城并通过对抗恶魔来拯救公主。

骑士的初始健康点数为一个正整数。如果他的健康点数在某一时刻降至 0 或以下，他会立即死亡。

有些房间由恶魔守卫，因此骑士在进入这些房间时会失去健康点数（若房间里的值为负整数，则表示骑士将损失健康点数）；其他房间要么是空的（房间里的值为 0），要么包含增加骑士健康点数的魔法球（若房间里的值为正整数，则表示骑士将增加健康点数）。

为了尽快到达公主，骑士决定每次只向右或向下移动一步。

 

编写一个函数来计算确保骑士能够拯救到公主所需的最低初始健康点数。

例如，考虑到如下布局的地下城，如果骑士遵循最佳路径 右 -> 右 -> 下 -> 下，则骑士的初始健康点数至少为 7。

| -2 (K) | -3   | 3      |
| ------ | ---- | ------ |
| -5     | -10  | 1      |
| 10     | 30   | -5 (P) |

说明:

骑士的健康点数没有上限。

任何房间都可能对骑士的健康点数造成威胁，也可能增加骑士的健康点数，包括骑士进入的左上角房间以及公主被监禁的右下角房间。





**题解**



这道题目如果通过正向的思维（从骑士的位置走到公主的位置）来暴力求解会产生一定的问题：

- 骑士在`[i,j]`位置时可以选择从`[i-1,j]`或者`[i,j-1]`过来，在选择时应该以什么策略进行判断？`d[i][j]`里面是不是还需要记录初始最小血量和当前的血量？
- 这里既需要考虑初始血量最小，又需要考虑有些房间是可以加血的，那么在选择下一步的时候到底应该选初始血量小的还是走过去之后当前血量更大的？

以上问题其实没有办法判断，无法使用贪心的方式来决策。那么我们可以换一个思路，从后往前看：假如骑士从`[i,j]`这个位置出发到达终点那么最少需要多少初始血量：

- 我们可以从后往前看，我们假设`d[i][j]`表示的是骑士从`[i,j]`这个位置出发所需要的初始血量，当`d[i][j]`是负数时表示初始需要血量，且后面缺少的血量为`1-d[i][j]`，`d[i][j]`是0则表示初始不需要血
- 我们知道骑士的下一步只能走右边或者下面，因此`d[i][j]`只与`d[i][j+1]`（右）和`d[i+1][j]`（下）有关，因此我们要选的是`d[i][j+1]`和`d[i+1][j]`中血量最大的那个，这样对于`d[i][j]`是最有利的，因此转移方程就变成了`d[i][j]= max{d[i][j+1], d[i+1][j]} + l[i][j]`，如果算下来`d[i][j]>0`，那么就说明骑士从`[i,j]`这个位置出发到达终点是不需要血量的，也就是`d[i][j]=0`

```go
// 动态规划
// dp[j] = max{dp[j], dp[j+1]} + dungeon[i][j]
// 其中i表示房间的行数，j表示列数，dp[j]表示骑士剩余的血量
func calculateMinimumHP(dungeon [][]int) int {
    n, m := len(dungeon), len(dungeon[0])
    dp:=make([]int, m)
    // 初始化血量
    dp[m-1]=0
    for i:=0;i<m-1;i++{
        dp[i]=-65535
    }
    for i:=n-1;i>=0;i--{
        // 对于最后一列，只跟下面的一行的状态有关
        dp[m-1]+=dungeon[i][m-1]
        if dp[m-1]>0 {
            dp[m-1]=0
        }
        // 对于其他列，需要同时考虑下面一行和右边一列的状态，两者取血量最大的那个
        for j:=m-2;j>=0;j--{
            if dp[j+1]>dp[j] {
                dp[j]=dp[j+1]
            }
            dp[j]+=dungeon[i][j]
            if dp[j]>0 {
                dp[j]=0
            }
        }
    }
    return 1-dp[0]
}
```





## 10、复杂数据结构





### [208. 实现 Trie (前缀树)](https://leetcode-cn.com/problems/implement-trie-prefix-tree/)



**题目**



实现一个 Trie (前缀树)，包含 insert, search, 和 startsWith 这三个操作。

示例:

```
Trie trie = new Trie();

trie.insert("apple");
trie.search("apple");   // 返回 true
trie.search("app");     // 返回 false
trie.startsWith("app"); // 返回 true
trie.insert("app");   
trie.search("app");     // 返回 true
```


说明:

- 你可以假设所有的输入都是由小写字母 a-z 构成的。 

- 保证所有输入均为非空字符串。





**题解**



```go
type Trie struct {
    child [26]*Trie
    end   bool
}


/** Initialize your data structure here. */
func Constructor() Trie {
    return Trie{}
}


/** Inserts a word into the trie. */
func (this *Trie) Insert(word string)  {
    n := len(word)
    for i:=0; i<n; i++ {
        index := word[i] - 'a'
        // 当前字符不在Trie树中时，增加这个节点
        if this.child[index] == nil {
            this.child[index] = &Trie{}
        }
        // this指向下一级，进入下一个循环
        this = this.child[index]
    }
    // 将最后一个字符标记为End
    this.end = true
}


/** Returns if the word is in the trie. */
func (this *Trie) Search(word string) bool {
    n := len(word)
    for i:=0; i<n; i++ {
        index := word[i] - 'a'
        // 当前字符不在Trie树中时，返回false
        if this.child[index] == nil {
            return false
        }
        // this指向下一级，进入下一个循环
        this = this.child[index]
    }
    // 返回末尾单词的标记
    return this.end
}


/** Returns if there is any word in the trie that starts with the given prefix. */
func (this *Trie) StartsWith(prefix string) bool {
    n := len(prefix)
    for i:=0; i<n; i++ {
        index := prefix[i] - 'a'
        // 当前字符不在Trie树中时，返回false
        if this.child[index] == nil {
            return false
        }
        // this指向下一级，进入下一个循环
        this = this.child[index]
    }
    return true
}


/**
 * Your Trie object will be instantiated and called as such:
 * obj := Constructor();
 * obj.Insert(word);
 * param_2 := obj.Search(word);
 * param_3 := obj.StartsWith(prefix);
 */
```





### [211. 添加与搜索单词 - 数据结构设计](https://leetcode-cn.com/problems/add-and-search-word-data-structure-design/)



**题目**



设计一个支持以下两种操作的数据结构：

```
void addWord(word)
bool search(word)
```


search(word) 可以搜索文字或正则表达式字符串，字符串只包含字母 . 或 a-z 。 . 可以表示任何一个字母。

示例:

```
addWord("bad")
addWord("dad")
addWord("mad")
search("pad") -> false
search("bad") -> true
search(".ad") -> true
search("b..") -> true
```


说明:

你可以假设所有单词都是由小写字母 a-z 组成的。





**题解**



由于存在'.'，所以遇到时需要通过遍历来查找后面是否有匹配，这里用的是dfs

```go
type WordDictionary struct {
    root *Trie
}

type Trie struct {
    child [26]*Trie
    end   bool
}


/** Initialize your data structure here. */
func Constructor() WordDictionary {
    return WordDictionary{
        root: &Trie{},
    }
}


/** Adds a word into the data structure. */
func (this *WordDictionary) AddWord(word string)  {
    root, n := this.root, len(word)
    for i:=0; i<n; i++ {
        index := word[i] - 'a'
        if root.child[index] == nil {
            root.child[index] = &Trie{}
        }
        root = root.child[index]
    }
    root.end = true
}


/** Returns if the word is in the data structure. A word could contain the dot character '.' to represent any one letter. */
func (this *WordDictionary) Search(word string) bool {
    return dfs(this.root, word)
}

func dfs(root *Trie, word string) bool {
    n := len(word)
    if n == 0 {
        return root != nil && root.end
    }
    if word[0] == '.' {
        for i:=0; i<26; i++ {
            if root.child[i] != nil && dfs(root.child[i], word[1:]) {
                return true
            }
        }
        return false
    }
    index := word[0] - 'a'
    if root.child[index] == nil {
        return false
    }
    if n == 1 {
        return root.child[index].end
    }
    return dfs(root.child[index], word[1:])
}


/**
 * Your WordDictionary object will be instantiated and called as such:
 * obj := Constructor();
 * obj.AddWord(word);
 * param_2 := obj.Search(word);
 */
```





### [547. 朋友圈](https://leetcode-cn.com/problems/friend-circles/)（并查集）



**题目**



班上有 N 名学生。其中有些人是朋友，有些则不是。他们的友谊具有是传递性。如果已知 A 是 B 的朋友，B 是 C 的朋友，那么我们可以认为 A 也是 C 的朋友。所谓的朋友圈，是指所有朋友的集合。

给定一个 N * N 的矩阵 M，表示班级中学生之间的朋友关系。如果M[i][j] = 1，表示已知第 i 个和 j 个学生互为朋友关系，否则为不知道。你必须输出所有学生中的已知的朋友圈总数。

示例 1:

```
输入: 
[[1,1,0],
 [1,1,0],
 [0,0,1]]
输出: 2 
说明：已知学生0和学生1互为朋友，他们在一个朋友圈。
第2个学生自己在一个朋友圈。所以返回2。
```


示例 2:

```
输入: 
[[1,1,0],
 [1,1,1],
 [0,1,1]]
输出: 1
说明：已知学生0和学生1互为朋友，学生1和学生2互为朋友，所以学生0和学生2也是朋友，所以他们三个在一个朋友圈，返回1。
```


注意：

N 在[1,200]的范围内。
对于所有学生，有M[i][i] = 1。
如果有M[i][j] = 1，则有M[j][i] = 1。





**题解**



并查集

```go
func findCircleNum(M [][]int) int {
    n := len(M)
    if n < 2 {
        return n
    }
    head, sum := make([]int, n), n
    // 初始化head，每个人的head都是自己
    for i, _ := range head {
        head[i] = i
    }
    for i:=0; i<n; i++ {
        for j:=i+1; j<n; j++ {
            if M[i][j] == 1 {
                if x, y := find(head, i), find(head, j); x != y {
                    join(head, j, x)
                    sum --
                }
            }
        }
    }
    return sum
}
// 找到i对应的head
func find(head []int, i int) int {
    for head[i] != i {
        i = head[i]
    }
    return i
}
// 把i这条线路上的所有节点的head都改成j
func join(head []int, i, j int) {
    // 如果i自己不是head，则将i的head也join进来
    if head[i] != i {
        join(head, head[i], j)
    }
    head[i] = j
}
```





### [307. 区域和检索 - 数组可修改](https://leetcode-cn.com/problems/range-sum-query-mutable/)（树状数组）



**题目**



给定一个整数数组  nums，求出数组从索引 i 到 j  (i ≤ j) 范围内元素的总和，包含 i,  j 两点。

update(i, val) 函数可以通过将下标为 i 的数值更新为 val，从而对数列进行修改。

示例:

```
Given nums = [1, 3, 5]

sumRange(0, 2) -> 9
update(1, 2)
sumRange(0, 2) -> 8
```


说明:

- 数组仅可以在 update 函数下进行修改。 

- 你可以假设 update 函数与 sumRange 函数的调用次数是均匀分布的。



**题解**



树状数组

```go
// 树状数组问题
// 注意几个关键操作就行：
// 1. i的低位为i&(-i)
// 2. i的父节点为i + i&(-i)
// 3. 树状数组是从1开始索引的，因此需要做一下转换
type NumArray struct {
    items []int
    tree []int
}


func Constructor(nums []int) NumArray {
    n := len(nums)
    array := NumArray{
        items: make([]int, n+1),
        tree:  make([]int, n+1),
    }
	for i:=1; i<n+1; i++ {
		array.items[i] = nums[i-1]
		array.tree[i] = nums[i-1] + array.tree[i-1]
	}
    // 这里的初始化一定要理解
	for i:=n; i>0; i-- {
		array.tree[i] -= array.tree[i-(i&(-i))]
	}
    return array
}


func (this *NumArray) Update(i int, val int)  {
	i++
	diff := val - this.items[i]
	this.items[i] = val
	for i<len(this.tree) {
		this.tree[i] += diff
		i += i&(-i)
	}
}


func (this *NumArray) SumRange(i int, j int) int {
	sum, i, j := 0, i+1, j+1
	for i--; i > 0; i -= i&(-i) {
		sum -= this.tree[i]
	}
	for ; j > 0; j -= j&(-j) {
		sum += this.tree[j]
	}
	return sum
}


/**
 * Your NumArray object will be instantiated and called as such:
 * obj := Constructor(nums);
 * obj.Update(i,val);
 * param_2 := obj.SumRange(i,j);
 */
```