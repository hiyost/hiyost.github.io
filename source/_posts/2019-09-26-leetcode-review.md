---
title: leetcode题解持续更新
date: 2019-09-26 09:40:54
tags: leetcode
---

## 搜索相关

### [301. 删除无效的括号](https://leetcode-cn.com/problems/remove-invalid-parentheses/)

#### 题目

删除最小数量的无效括号，使得输入的字符串有效，返回所有可能的结果。

说明: 输入可能包含了除 ( 和 ) 以外的字符。

示例 1:

```
输入: "()())()"
输出: ["()()()", "(())()"]
```


示例 2:

```
输入: "(a)())()"
输出: ["(a)()()", "(a())()"]
```


示例 3:

```
输入: ")("
输出: [""]
```

<!-- more -->

#### 题解

```go
// 注意这道题目中可能存在的场景：
// "()())()"、"(a)())()"、")x("、"((("、")(f"、""
// 算法使用深度优先
var ret []string
func removeInvalidParentheses(s string) []string {
    // 先统计需要删除的左括号和右括号的数量
    left, right := 0, 0
    for i:=0; i< len(s); i++ {
        if s[i] == '(' {
            left ++
        } else if s[i] == ')' {
            if left == 0 {
                right ++
            } else {
                left --
            }
        }
    }
    // 使用深度优先算法计算所有可能的值
    ret = []string{}
    dfs(s, 0, left, right)
    if len(ret) == 0 {
        ret = []string{""}
    }
    return ret
}
func dfs(s string, start, left, right int) {
    if left == 0 && right == 0 {
        if isValid(s) {
            ret = append(ret, s)
        }
        return
    }
    for i := start; i < len(s) ; i++ {
        // 防止重复，如果这次处理的字符跟上次一样则跳过（得到的结果是一样的）
        if i > start  && s[i] == s[i-1] {
            continue
        }
        if s[i] == '(' && left > 0 {
            dfs(s[:i]+s[i+1:], i, left-1, right)
        } else if s[i] == ')' && right > 0 {
            dfs(s[:i]+s[i+1:], i, left, right-1)
        }
    }
}
// 判断是否为满足要求的字符串
func isValid(s string) bool {
    left := 0
    for i:=0; i< len(s); i++ {
        if s[i] == '(' {
            left ++
        } else if s[i] == ')' {
            left --
            if left < 0 {
                return false
            }
        }
    }
    return left == 0 && len(s) != 0
}
```



### [239. 滑动窗口最大值](https://leetcode-cn.com/problems/sliding-window-maximum/)

#### 题目

给定一个数组 nums，有一个大小为 k 的滑动窗口从数组的最左侧移动到数组的最右侧。你只可以看到在滑动窗口内的 k 个数字。滑动窗口每次只向右移动一位。

返回滑动窗口中的最大值。

 

示例:

```
输入: nums = [1,3,-1,-3,5,3,6,7], 和 k = 3
输出: [3,3,5,5,6,7] 
解释: 

  滑动窗口的位置                最大值

------

[1  3  -1] -3  5  3  6  7       3
 1 [3  -1  -3] 5  3  6  7       3
 1  3 [-1  -3  5] 3  6  7       5
 1  3  -1 [-3  5  3] 6  7       5
 1  3  -1  -3 [5  3  6] 7       6
 1  3  -1  -3  5 [3  6  7]      7
```


提示：

你可以假设 k 总是有效的，在输入数组不为空的情况下，1 ≤ k ≤ 输入数组的大小。

 

进阶：

你能在线性时间复杂度内解决此题吗？

#### 题解

暴力求解

```go
func maxSlidingWindow(nums []int, k int) []int {
    if len(nums) == 0 {
        return []int{}
    }
    ret := make([]int, len(nums) - k +1)
    for i := k-1; i < len(nums); i++ {
        maxN := nums[i-k+1]
        for j := i-k+2; j <= i; j++ {
            if nums[j] > maxN {
                maxN = nums[j]
            }
        }
        ret[i-k+1] = maxN
    }
    return ret
}
```

线性时间的话需要有一个变量保存上一次最大值和第二大值的索引，如果过了最大值则将新加进来的元素跟第二大的值进行比较，并且需要重新刷新最大值和第二大值



### [128. 最长连续序列](https://leetcode-cn.com/problems/longest-consecutive-sequence/)

#### 题目

给定一个未排序的整数数组，找出最长连续序列的长度。

要求算法的时间复杂度为 O(n)。

示例:

```
输入: [100, 4, 200, 1, 3, 2]
输出: 4
解释: 最长连续序列是 [1, 2, 3, 4]。它的长度为 4。
```



#### 题解

```go
// 用map做hashset，每次都将改元素所有相邻的元素都找出来
func longestConsecutive(nums []int) int {
    set := make(map[int]struct{})
    for _,v := range nums {
        set[v]=struct{}{}
    }
    max := 0
    for _,v := range nums {
        if _, ok := set[v]; ok {
            // 包含本元素的连续序列长度，已经遍历过的元素从hashset中删除
            l := 1
            // 向后搜索
            for cur := v+1; ; cur++ {
                if _, ok := set[cur]; !ok {
                    break
                }
                l++
                delete(set, cur)
            }
            // 向前搜索
            for cur := v-1; ; cur-- {
                if _, ok := set[cur]; !ok {
                    break
                }
                l++
                delete(set, cur)
            }
            delete(set, v)
            if l > max {
                max = l
            }
        }
    }
    return max
}
```

### [1147. 段式回文](https://leetcode-cn.com/problems/longest-chunked-palindrome-decomposition/)

#### 题目

段式回文 其实与 一般回文 类似，只不过是最小的单位是 一段字符 而不是 单个字母。

举个例子，对于一般回文 "abcba" 是回文，而 "volvo" 不是，但如果我们把 "volvo" 分为 "vo"、"l"、"vo" 三段，则可以认为 “(vo)(l)(vo)” 是段式回文（分为 3 段）。

 

给你一个字符串 text，在确保它满足段式回文的前提下，请你返回 段 的 最大数量 k。

如果段的最大数量为 k，那么存在满足以下条件的 `a_1, a_2, ..., a_k`：

- 每个 a_i 都是一个非空字符串；

- 将这些字符串首位相连的结果 `a_1 + a_2 + ... + a_k` 和原始字符串 text 相同；

- 对于所有`1 <= i <= k`，都有 `a_i = a_{k+1 - i}`。


示例 1：

```
输入：text = "ghiabcdefhelloadamhelloabcdefghi"
输出：7
解释：我们可以把字符串拆分成 "(ghi)(abcdef)(hello)(adam)(hello)(abcdef)(ghi)"。
```


示例 2：

```
输入：text = "merchant"
输出：1
解释：我们可以把字符串拆分成 "(merchant)"。
```


示例 3：

```
输入：text = "antaprezatepzapreanta"
输出：11
解释：我们可以把字符串拆分成 "(a)(nt)(a)(pre)(za)(tpe)(za)(pre)(a)(nt)(a)"。
```


示例 4：

```
输入：text = "aaa"
输出：3
解释：我们可以把字符串拆分成 "(a)(a)(a)"。
```


提示：

- text 仅由小写英文字符组成。

- 1 <= text.length <= 1000

#### 题解

```go
// 递归求解，每次从后往前遍历，如果能找到一个回文段则+2并去掉这两个回文段继续遍历
func longestDecomposition(text string) int {
    if len(text) == 0 {
        return 0
    }
	for last := len(text)-1; len(text) > 1 && last >= len(text)/2; last -- {
		if text[:len(text)-last] == text[last:] {
			return 2 + longestDecomposition(text[len(text)-last:last])
		}
	}
    // 找不到回文段则将整体作为回文段
	return 1
}
```

### [76. 最小覆盖子串](https://leetcode-cn.com/problems/minimum-window-substring/)

#### 题目

给你一个字符串 S、一个字符串 T，请在字符串 S 里面找出：包含 T 所有字母的最小子串。

示例：

```
输入: S = "ADOBECODEBANC", T = "ABC"
输出: "BANC"
```


说明：

- 如果 S 中不存这样的子串，则返回空字符串 ""。
- 如果 S 中存在这样的子串，我们保证它是唯一的答案。



#### 题解

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



### [124. 二叉树中的最大路径和](https://leetcode-cn.com/problems/binary-tree-maximum-path-sum/)

#### 题目

给定一个非空二叉树，返回其最大路径和。

本题中，路径被定义为一条从树中任意节点出发，达到任意节点的序列。该路径至少包含一个节点，且不一定经过根节点。

示例 1:

```
输入: [1,2,3]

   1
  / \
 2   3

输出: 6
```


示例 2:

```
输入: [-10,9,20,null,null,15,7]

   -10
   / \
  9  20
    /  \
   15   7

输出: 42
```



#### 题解

主要思路是递归。最开始一直没有想明白的是每次递归的函数里面应该返回什么？是这个问题要求解的最大路径和吗？其实不是。可以思考一下，对于每一个root而言，其最大路径和需要考虑的是root节点、左节点的一条边路径和右节点的一条边路径，因此这个**递归函数要返回的是这个root节点的最大路径的一条边**，这样才能递归下去。但是这样问题也来了：怎么计算和递归最大路径和呢？这里**使用指针将我们要求解的值传到递归的函数里面去**，一旦有更大的路径和就更新这个值。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func maxPathSum(root *TreeNode) int {
    max := -1<<31
    subPathMax(root, &max)
    return max
}
// subPathMax函数计算的是当前这个root中取left+root、right+root、root中的最大值
// 对于题目中的问题，需要算的是left+root+right的最大值，所以通过指针传进去病更新max
func subPathMax(root *TreeNode, max *int) int {
    if root == nil {
        return 0
    }
    left := Max(subPathMax(root.Left, max), 0)
    right := Max(subPathMax(root.Right, max), 0)
    *max = Max(*max, root.Val+left+right)
    return root.Val+Max(left, right)
}
// 
func Max(i, j int) int {
    if i > j {
        return i
    }
    return j
}
```

### [980. 不同路径 III](https://leetcode-cn.com/problems/unique-paths-iii/)

#### 题目

在二维网格 grid 上，有 4 种类型的方格：

- 1 表示起始方格。且只有一个起始方格。
- 2 表示结束方格，且只有一个结束方格。
- 0 表示我们可以走过的空方格。
- -1 表示我们无法跨越的障碍。

返回在四个方向（上、下、左、右）上行走时，从起始方格到结束方格的不同路径的数目，每一个无障碍方格都要通过一次。



示例 1：


```
输入：[[1,0,0,0],[0,0,0,0],[0,0,2,-1]]
输出：2
解释：我们有以下两条路径：

1. (0,0),(0,1),(0,2),(0,3),(1,3),(1,2),(1,1),(1,0),(2,0),(2,1),(2,2)
2. (0,0),(1,0),(2,0),(2,1),(1,1),(0,1),(0,2),(0,3),(1,3),(1,2),(2,2)
```


示例 2：

```
输入：[[1,0,0,0],[0,0,0,0],[0,0,0,2]]
输出：4
解释：我们有以下四条路径： 

1. (0,0),(0,1),(0,2),(0,3),(1,3),(1,2),(1,1),(1,0),(2,0),(2,1),(2,2),(2,3)
2. (0,0),(0,1),(1,1),(1,0),(2,0),(2,1),(2,2),(1,2),(0,2),(0,3),(1,3),(2,3)
3. (0,0),(1,0),(2,0),(2,1),(2,2),(1,2),(1,1),(0,1),(0,2),(0,3),(1,3),(2,3)
4. (0,0),(1,0),(2,0),(2,1),(1,1),(0,1),(0,2),(0,3),(1,3),(1,2),(2,2),(2,3)
```


示例 3：

```
输入：[[0,1],[2,0]]
输出：0
解释：
没有一条路能完全穿过每一个空的方格一次。
请注意，起始和结束方格可以位于网格中的任意位置。
```


提示：

- `1 <= grid.length * grid[0].length <= 20`

#### 题解

还算是比较简单的dfs，注意题目中说到的几个满足条件：

- 从起点出发能到达终点

- 所有空格都要访问到

因此在dfs的时候要有地方可以记录已经访问过的空格个数

```go
func uniquePathsIII(grid [][]int) int {
    start, end, m, n, emp := 0, 0, len(grid), len(grid[0]), 0
    // isVisited改成一维数组
    isVisited := make([]bool, m*n)
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == 1 {
                start = i*n+j
            } else if grid[i][j] == 2 {
                end = i*n+j
            } else if grid[i][j] == 0 {
                // 这里统计出空方格的个数，用于判断是否每个方格都走完了
                // 注意这里没有统计end，所以后面实施时emp+1
                emp++
            } else {
                // 将障碍方格置为访问过
                isVisited[i*n+j] = true
            }
        }
    }
    num := 0
    isVisited[start] = true
    dfs(grid, start, end, emp+1, isVisited, &num)
    return num
}
func dfs(grid [][]int, start, end, emp int, isVisited []bool, num *int) {
	if start == end {
		// 如果所有空方格都访问过，则说明是一条
		if emp <= 0 {
			*num ++
		}
		return
	}
	nexts := [][]int{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}
	i, j := start/len(grid[0]), start%len(grid[0])
	for _, n := range nexts {
		nStart := (i + n[0])*len(grid[0]) + j + n[1]
		// 跳过超出范围或者已经访问过（包含阻碍方格）
		if i + n[0] < 0 || i + n[0] >= len(grid) || j + n[1] < 0 || j + n[1] >= len(grid[0]) || isVisited[nStart] {
			continue
		}
		isVisited[nStart] = true
		dfs(grid, nStart, end, emp-1, isVisited, num)
        // 注意isVisited是大家共享的，用过之后要改回来
		isVisited[nStart] = false
	}
}
```



### [773. 滑动谜题](https://leetcode-cn.com/problems/sliding-puzzle/)

#### 题目

在一个 2 x 3 的板上（board）有 5 块砖瓦，用数字 1~5 来表示, 以及一块空缺用 0 来表示.

一次移动定义为选择 0 与一个相邻的数字（上下左右）进行交换.

最终当板 board 的结果是 [[1,2,3],[4,5,0]] 谜板被解开。

给出一个谜板的初始状态，返回最少可以通过多少次移动解开谜板，如果不能解开谜板，则返回 -1 。

示例：

```
输入：board = [[1,2,3],[4,0,5]]
输出：1
解释：交换 0 和 5 ，1 步完成
```

```
输入：board = [[1,2,3],[5,4,0]]
输出：-1
解释：没有办法完成谜板
```

```
输入：board = [[4,1,2],[5,0,3]]
输出：5
解释：
最少完成谜板的最少移动次数是 5 ，
一种移动路径:
尚未移动: [[4,1,2],[5,0,3]]
移动 1 次: [[4,1,2],[0,5,3]]
移动 2 次: [[0,1,2],[4,5,3]]
移动 3 次: [[1,0,2],[4,5,3]]
移动 4 次: [[1,2,0],[4,5,3]]
移动 5 次: [[1,2,3],[4,5,0]]
```

```
输入：board = [[3,2,4],[1,5,0]]
输出：14
```

提示：

- `board` 是一个如上所述的 2 x 3 的数组.
- `board[i][j]` 是一个 [0, 1, 2, 3, 4, 5] 的排列.

#### 题解

典型的bfs，按层搜索

```go
import "strconv"
func slidingPuzzle(board [][]int) int {
    queue, set := []string{}, map[string]int{}
    b := ""
    for i:=0; i<2; i++ {
        for j:=0; j<3; j++ {
            b += strconv.Itoa(board[i][j])
            if board[i][j] == 0 {
                // 将0的位置记录在b的开头
                b = strconv.Itoa(i*3+j) + b
            }
        }
    }
    queue = append(queue, b)
    set[b[1:]] = 0
    // 记录每个位置可以移动的下一个位置
    next := [][]int{{3, 1}, {4, 0, 2}, {5,1}, {0, 4}, {1,3,5}, {2, 4}}
    for len(queue) > 0 {
        // pop出队列中的第一个
        // zero是cur中0所在的索引
        ind := int(queue[0][0] - '0')
        cur := queue[0][1:] // 去掉索引
        if cur == "123450" {
            return set[cur]
        }
        queue = queue[1:]
        for _, n := range next[ind] {
            i, j := ind, n
            if i > j {
                i, j = j, i
            }
            nString := cur[:i] + cur[j:j+1] + cur[i+1:j] + cur[i:i+1] + cur[j+1:]
            // 如果之前的操作已经处理过，则跳过
            if _, ok := set[nString]; ok {
                continue
            }
            // 否则记录步数并将新字符加入到队列中
            set[nString] = set[cur] + 1
            queue = append(queue, strconv.Itoa(n)+nString)
        }
    }
    return -1
}
```



## 排序相关

### [406. 根据身高重建队列](https://leetcode-cn.com/problems/queue-reconstruction-by-height/)

#### 题目

假设有打乱顺序的一群人站成一个队列。 每个人由一个整数对(h, k)表示，其中h是这个人的身高，k是排在这个人前面且身高大于或等于h的人数。 编写一个算法来重建这个队列。

注意：
总人数少于1100人。

示例

```
输入:
[[7,0], [4,4], [7,1], [5,0], [6,1], [5,2]]

输出:
[[5,0], [7,0], [5,2], [6,1], [4,4], [7,1]]
```

#### 题解

- 排序+按索引插入：排序的时候需要降序排序，并且对于h相同的元素，k值大的应该排在后面，降序排序完成之后，遍历这个序列，如果需要插入的这个元素的k值是小于需要返回的数组ret的长度，就需要将他插入到ret的索引为k的位置，否则加入到末尾就可以了

- 由于go语言并不支持链表，同时也不支持二维数组的排序，这里需要自行实现

```go
import "sort"

// 这里实现一下[][]int二维数组的降序排列（实现Len、Less、Swap这三个接口，sort默认是升序）
// 由于是降序，所以Less()中是p[i][0] > p[j][0]
// 需要注意的是，默认sort默认是升序的，Less()应该是p[i][0] < p[j][0]
type People [][]int
func (p People) Len() int { return len(p) }
func (p People) Less(i, j int) bool { return p[i][0] > p[j][0] || ( p[i][0] == p[j][0] && p[i][1] < p[j][1]) }
func (p People) Swap(i, j int) { p[i][0], p[i][1], p[j][0], p[j][1] = p[j][0], p[j][1], p[i][0], p[i][1] }

func reconstructQueue(people [][]int) [][]int {
    // 先对其进行降序排列（注意h相同时，k大的在后面）
    sort.Sort(People(people))
    var ret [][]int
    for _, p := range people {
        // 如果当前这个元素p需要insert的index是末尾，则直接append
        if len(ret) <= p[1] {
            ret = append(ret, p)
        } else {
            // 否则插入到p[1]这个位置
            tmp := append([][]int{}, ret[p[1]:]...) //先保存末尾
            ret = append(ret[:p[1]], p) // 再将p插入
            ret = append(ret, tmp...)
        }
    }
    return ret
}
```

### [899. 有序队列](https://leetcode-cn.com/problems/orderly-queue/)

#### 题目

给出了一个由小写字母组成的字符串 S。然后，我们可以进行任意次数的移动。

在每次移动中，我们选择前 K 个字母中的一个（从左侧开始），将其从原位置移除，并放置在字符串的末尾。

返回我们在任意次数的移动之后可以拥有的按字典顺序排列的最小字符串。

 

示例 1：

```
输入：S = "cba", K = 1
输出："acb"
解释：
在第一步中，我们将第一个字符（“c”）移动到最后，获得字符串 “bac”。
在第二步中，我们将第一个字符（“b”）移动到最后，获得最终结果 “acb”。
```


示例 2：

```
输入：S = "baaca", K = 3
输出："aaabc"
解释：
在第一步中，我们将第一个字符（“b”）移动到最后，获得字符串 “aacab”。
在第二步中，我们将第三个字符（“c”）移动到最后，获得最终结果 “aaabc”。
```


提示：

- 1 <= K <= S.length <= 1000

- S 只由小写字母组成。



#### 题解

```go
import (
    "strings"
    "sort"
)
// 注意go语言中的string中无法Swap，所以这里用[]byte
type String []byte
func (s String) Len() int {return len(s)}
func (s String) Less(i, j int) bool {return s[i] < s[j]}
func (s String) Swap(i, j int) {s[i], s[j] = s[j], s[i]}
func orderlyQueue(S string, K int) string {
    ret := S
    if K == 1 {
        // K=1时，S只能不停循环，因此循环到S最小就好
        for i:=1; i < len(S); i++ {
            if strings.Compare(S[i:]+S[:i], ret) < 0 {
                ret = S[i:]+S[:i]
            }
        }
    } else {
        // K>1时，S可以通过冒泡算法实现排序
        ss := String(S)
        sort.Sort(ss)
        ret = string(ss)
    }
    return ret
}
```



### [23. 合并K个排序链表](https://leetcode-cn.com/problems/merge-k-sorted-lists/)

#### 题目

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

#### 题解

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

  

## 动态规划相关

### [72. 编辑距离](https://leetcode-cn.com/problems/edit-distance/)

#### 题目

给定两个单词 word1 和 word2，计算出将 word1 转换成 word2 所使用的最少操作数 。

你可以对一个单词进行如下三种操作：

插入一个字符
删除一个字符
替换一个字符
示例 1:

```
输入: word1 = "horse", word2 = "ros"
输出: 3
解释: 
horse -> rorse (将 'h' 替换为 'r')
rorse -> rose (删除 'r')
rose -> ros (删除 'e')
```


示例 2:

```
输入: word1 = "intention", word2 = "execution"
输出: 5
解释: 
intention -> inention (删除 't')
inention -> enention (将 'i' 替换为 'e')
enention -> exention (将 'n' 替换为 'x')
exention -> exection (将 'n' 替换为 'c')
exection -> execution (插入 'u')
```

#### 题解

```go
// 动态规划，假设dp[i][j]为截取word1中的前i个字符和word2中的前j个字符进行转换的最小距离
// 那么dp[i][j]跟dp[i-1][j-1]、dp[i-1][j]、dp[i][j-1]有关，详情请看代码注释

func minDistance(word1 string, word2 string) int {
    // 初始化dp数组，注意数组的长度要比输入的字符串的长度长1个单位
    var dp [][]int
    for i:=0; i < len(word1)+1; i++ {
        sli := make([]int, len(word2)+1)
        dp = append(dp, sli)
        // 注意当word2截取的长度为0时，word1只需将自己的所有字符删掉即可
        dp[i][0] = i
    }
    for j:=0; j <= len(word2); j++ {
        // 注意当word1截取的长度为0时，word1只需将word2的所有字符即可
        dp[0][j] = j
    }
    // 注意这里i是word1中截取的长度，j是word2中截取的长度，所以都从1开始算起
    for i:=1; i < len(word1)+1; i++ {
        for j:=1; j < len(word2)+1; j++{
            if word1[i-1] == word2[j-1] {
                // 如果word1[i-1]（第i个字符）和word2[j-1]相同，则不需要做任何操作
                dp[i][j] = dp[i-1][j-1]
            } else {
                // 否则取最小代价的方式
                dp[i][j] = min(min(dp[i][j-1], dp[i-1][j]), dp[i-1][j-1]) +1
            }
        }
    }
    return dp[len(word1)][len(word2)]
}

// go中没有实现int类型的min函数，所以只能自己实现
func min(i, j int) int {
    if i > j {
        return j
    }
    return i
}
```



### [960. 删列造序 III](https://leetcode-cn.com/problems/delete-columns-to-make-sorted-iii/)

#### 题目

给定由 N 个小写字母字符串组成的数组 A，其中每个字符串长度相等。

选取一个删除索引序列，对于 A 中的每个字符串，删除对应每个索引处的字符。

比如，有 `A = ["babca","bbazb"]`，删除索引序列 `{0, 1, 4}`，删除后 `A` 为`["bc","az"]`。

假设，我们选择了一组删除索引 D，那么在执行删除操作之后，最终得到的数组的行中的每个元素都是按字典序排列的。

清楚起见，`A[0]` 是按字典序排列的（即，`A[0][0] <= A[0][1] <= ... <= A[0][A[0].length - 1]`），A[1] 是按字典序排列的（即，`A[1][0] <= A[1][1] <= ... <= A[1][A[1].length - 1]`），依此类推。

请你返回 `D.length` 的最小可能值。

 

示例 1：

```
输入：["babca","bbazb"]
输出：3
解释：
删除 0、1 和 4 这三列后，最终得到的数组是 A = ["bc", "az"]。
这两行是分别按字典序排列的（即，A[0][0] <= A[0][1] 且 A[1][0] <= A[1][1]）。
注意，A[0] > A[1] —— 数组 A 不一定是按字典序排列的。
```


示例 2：

```
输入：["edcba"]
输出：4
解释：如果删除的列少于 4 列，则剩下的行都不会按字典序排列。
```


示例 3：

```
输入：["ghi","def","abc"]
输出：0
解释：所有行都已按字典序排列。
```


提示：

- `1 <= A.length <= 100`
- `1 <= A[i].length <= 100`



#### 题解

```go
// 动态规划，可以按照最长子序列来做，先计算最长子序列，然后用原始序列的长度减去最长子序列的长度就得到答案
// 转移方程： d[i] = max{d[i-1], d[k]+1: k为0~i-1并且s[i]>s[k]}
// d[i]表示的是以s[i]结尾的子序列的最大长度
func minDeletionSize(A []string) int {
    d := make([]int, len(A[0]))
    d[0] = 1 // d[0]其实值为1
    for i:=1; i<len(A[0]); i++ {
        d[i] = 1
        for j := 0; j < i; j++ {
            if cmp(A, j, i) && d[j] + 1 > d[i] {
                d[i] = d[j] + 1
            }
        }
    }
    max := 0
    for _,v := range d {
        if v > max {
            max = v
        }
    }
    return len(A[0]) - max
}
// 只有A中所有字符串的s[i]>=s[j]时才会返回true
func cmp(A []string, j, i int) bool {
    for _, v := range A {
        if v[i] < v[j] {
            return false
        }
    }
    return true
}
```





### [312. 戳气球](https://leetcode-cn.com/problems/burst-balloons/)

#### 题目

有 n 个气球，编号为0 到 n-1，每个气球上都标有一个数字，这些数字存在数组 nums 中。

现在要求你戳破所有的气球。每当你戳破一个气球 i 时，你可以获得 nums[left] * nums[i] * nums[right] 个硬币。 这里的 left 和 right 代表和 i 相邻的两个气球的序号。注意当你戳破了气球 i 后，气球 left 和气球 right 就变成了相邻的气球。

求所能获得硬币的最大数量。

说明:

- 你可以假设 nums[-1] = nums[n] = 1，但注意它们不是真实存在的所以并不能被戳破。

- 0 ≤ n ≤ 500, 0 ≤ nums[i] ≤ 100

  示例:

```
输入: [3,1,5,8]
输出: 167 
解释: nums = [3,1,5,8] --> [3,5,8] -->   [3,8]   -->  [8]  --> []
     coins =  3*1*5      +  3*5*8    +  1*3*8      + 1*8*1   = 167
```



#### 题解

```go
// 使用动态规划来完成
// 对于[i, j]区间内（不包含i和j）的气球的最大收益，我们假设k是最后一个戳破的气球
// 则转移方程为d[i][j]=max{d[i][k-1]+d[k+1][j]+nums[k]*nums[i]*nums[j]，k从i+1到j-1}
func maxCoins(nums []int) int {
    // 在nums前后分别加一个1的元素
    le := len(nums) + 2
    nu := make([]int, le)
    nu[0], nu[le-1] = 1, 1
    for i:=1 ; i<le-1; i++{
        nu[i] = nums[i-1]
    }
    // dp[i][j]表示nums中戳破i到j中间（不包含i和j）这些气球的最大收益，因此本题的返回值是dp[0][le]
    var dp [][]int
    for i:=0; i < le; i++ {
        sl := make([]int, le)
        dp = append(dp, sl)
    }
    // i表示的是长度，对于这个问题而言，我们要先求出小区间内的最大分数，才能推到大区间的最大分数
    for i := 2; i < le; i ++ {
        for left := 0; left < le - i; left++ {
            // 
            right := left + i
            for k := left+1; k < right; k++ {
                value := dp[left][k] + dp[k][right] + nu[left]*nu[k]*nu[right]
                if value > dp[left][right] {
                    dp[left][right] = value
                }
            }
        }
    }
    return dp[0][le-1]
}
```



### [546. 移除盒子](https://leetcode-cn.com/problems/remove-boxes/)

#### 题目

给出一些不同颜色的盒子，盒子的颜色由数字表示，即不同的数字表示不同的颜色。
你将经过若干轮操作去去掉盒子，直到所有的盒子都去掉为止。每一轮你可以移除具有相同颜色的连续 k 个盒子（k >= 1），这样一轮之后你将得到 k*k 个积分。
当你将所有盒子都去掉之后，求你能获得的最大积分和。

示例 1：
输入:

```
[1, 3, 2, 2, 2, 3, 4, 3, 1]
```


输出:

```
23
```


解释:

```
[1, 3, 2, 2, 2, 3, 4, 3, 1] 
----> [1, 3, 3, 4, 3, 1] (3*3=9 分) 
----> [1, 3, 3, 3, 1] (1*1=1 分) 
----> [1, 1] (3*3=9 分) 
----> [] (2*2=4 分)
```


提示：盒子的总数 n 不会超过 100。

#### 题解

解法一：暴力求解（dfs），会超时，时间复杂度为O(n!)，使用[3,8,8,5,5,3,9,2,4,4,6,5,8,4,8,6,9,6,2,8,6,4,1,9,5,3,10,5,3,3,9,8,8,6,5,3,7,4,9,6,3,9,4,3,5,10,7,6,10,7]这个长度为50的用例在桌面云上跑了一下午都没跑完，50!=3e64，这是因为有太多重复的计算，特别是算到后面newBox只有几个元素的时候很多情况会重复，在这个基础上可以通过map来存储已经算过的数组，但是会消耗较多空间。

```go
func removeBoxes(boxes []int) int {
    max := 0
    for i:=0; i<len(boxes); i++ {
        // 如果当前盒子跟之前的盒子颜色相同，则结果跟上次算出来的结果一样，直接跳过
        if i>0 && boxes[i] == boxes[i-1] {
            continue
        }
        // 如果i是同颜色盒子的起点，那么往后找所有跟i相同颜色的盒子，k-1是终点
        k := i+1
        // 将k遍历到末尾
        for ; k<len(boxes) && boxes[k] == boxes[i]; k++ {}
        // 建立一个新slice计算去掉i相邻的通颜色的盒子之后的分数
		newBox := append([]int{}, boxes[:i]...)
		newBox = append(newBox, boxes[k:]...)
        if point := (k-i)*(k-i) + removeBoxes(newBox); point>max {
            max = point
        }
    }
    return max
}
```

解法二：动态规划

这个题目的dp转移方程理解起来比较困难，我们先给出结果，以题目中的示例为例

- 转移方程的定义

  对于`[1, 3, 2, 2, 2, 3, 4, 3, 1]`这个数组`boxes`，我们假设`dp[i][j][k]`表示的是`boxes[i:j]`这个子序列的最大积分，而k表示j右侧有k个**与之相邻**的boxes[j]，那么我们最终要求的解就是`dp[0][n-1][0]`，其中n是boxes的长度。

- 转移方程的状态表述

  为了求解`dp[i][j][k]`，我们需要根据`boxes[i:j]`中的情况和j后面的与`boxes[j]`相同的**相邻**盒子个数。首先暂且不管k是怎么算出来的，我们先看`boxes[i:j]`中的情况：

  - 假如`boxes[i:j-1]`中有跟`boxes[j]`相同的盒子`boxes[m]`，那么我们认为可以先去掉`boxes[m+1:j-1]`（即计算`dp[m+1][j-1][0]`，之所以k是0，是因为这一段不考虑j-1后面是否有跟j-1重复的，注意`boxes[m+1:j-1]`中可能还会跟`boxes[j]`相同的盒子，我们都会遍历到），然后再计算`dp[i][m][k+1]`获得积分，这样`dp[i][j][k]=dp[i][m][k+1]+dp[m+1][j-1][0]`，当前我们也可以直接先去掉右边所有与`boxes[j]`相同的**相邻**盒子并得到积分`(k+1)*(k+1)`，这个跟下面的情况相同
  - 假如`boxes[i:j-1]`中没有有跟`boxes[j]`相同的盒子，那么我们可以直接去掉`box[j]`和右边所有与`boxes[j]`相同的**相邻**盒子并得到积分`(k+1)*(k+1)`，这样`dp[i][j][k]=dp[i][j-1][0]+(k+1)*(k+1)`，这里之所以`dp[i][j-1][0]`中k=0是因为j-1后面已经没有其他盒子了

  总结下来`dp[i][j][k]`总共有两种情况：

  `dp[i][j][k]=max{dp[i][m][k+1]+dp[m+1][j-1][0], dp[i][j-1][0]+(k+1)*(k+1)}`

针对这个方程，我们可以使用题目中的示例演算一下

```go
func removeBoxes(boxes []int) int {
    dp := [][][]int{}
    for i:=0; i< len(boxes); i++ {
        dp1 := [][]int{}
        for j:=0; j< len(boxes); j++ {
            dp1 = append(dp1, make([]int, len(boxes)))
        }
        dp = append(dp, dp1)
    }
    return maxPoint(0, len(boxes)-1, 0, dp, boxes)
}
func maxPoint(i, j, k int, dp [][][]int, boxes []int) int {
    if dp[i][j][k] > 0 || i > j {
        return dp[i][j][k]
    }
    if i == j {
        dp[i][j][k] = (k+1)*(k+1)
        return dp[i][j][k]
    }
    max := maxPoint(i, j-1, 0, dp, boxes) + (k+1)*(k+1)
    for m:=i; m<j; m++ {
        if boxes[m] != boxes[j] {
            continue
        }
        if cur := maxPoint(i, m, k+1, dp, boxes) + maxPoint(m+1, j-1, 0, dp, boxes); cur > max {
            max = cur
        }
    }
    dp[i][j][k] = max
    return max
}
```



### [174. 地下城游戏](https://leetcode-cn.com/problems/dungeon-game/)

#### 题目

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

#### 题解

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



## 图论相关



### [778. 水位上升的泳池中游泳](https://leetcode-cn.com/problems/swim-in-rising-water/)

#### 题目

在一个 N x N 的坐标方格 grid 中，每一个方格的值 grid[i][j] 表示在位置 (i,j) 的平台高度。

现在开始下雨了。当时间为 t 时，此时雨水导致水池中任意位置的水位为 t 。你可以从一个平台游向四周相邻的任意一个平台，但是前提是此时水位必须同时淹没这两个平台。假定你可以瞬间移动无限距离，也就是默认在方格内部游动是不耗时的。当然，在你游泳的时候你必须待在坐标方格里面。

你从坐标方格的左上平台 (0，0) 出发。最少耗时多久你才能到达坐标方格的右下平台 (N-1, N-1)？

示例 1:

> 输入: [[0,2],[1,3]]
> 输出: 3
> 解释:
> 时间为0时，你位于坐标方格的位置为 (0, 0)。
> 此时你不能游向任意方向，因为四个相邻方向平台的高度都大于当前时间为 0 时的水位。
>
> 等时间到达 3 时，你才可以游向平台 (1, 1). 因为此时的水位是 3，坐标方格中的平台没有比水位 3 更高的，所以你可以游向坐标方格中的任意位置


示例2:

> 输入: [[0,1,2,3,4],[24,23,22,21,5],[12,13,14,15,16],[11,17,18,19,20],[10,9,8,7,6]]
> 输入: 16
> 解释:
>  **0  1  2  3  4**
> 24 23 22 21  **5**
> **12 13 14 15 16**
> **11** 17 18 19 20
> **10  9  8  7  6**
>
> 最终的路线用加粗进行了标记。
> 我们必须等到时间为 16，此时才能保证平台 (0, 0) 和 (4, 4) 是连通的


提示:

- `2 <= N <= 50`.
- `grid[i][j]` 位于区间 `[0, ..., N*N - 1]` 内。

#### 题解

本题实际上是一个变形的最短路径问题，使用dijkstra就可以实现了，注意dijkstra算法的基本思路：

- 计算新加入到最短路径中的节点k相邻的所有未加入的节点到起始点的距离，如果经过节点k到达这个节点的距离更短，则更新这个节点的最小距离
- 遍历所有未加入最短路径的节点，找到距离起始节点最近的那个节点并加入到最短路径中，然后重复上一步

注意上述循环可以反过来，只要控制好循环之前的初始化就好了

```go
func swimInWater(grid [][]int) int {
    n, m := len(grid), len(grid[0])
    // 初始化dist和isVisited
    // dist[i][j]表示[i][j]到[0][0]的距离
    dist := [][]int{}
    // isVisited[i][j]表示[i][j]是否已经加入路径中
    isVisited := [][]bool{}
    for i:=0; i<n; i++{
        dist = append(dist, make([]int, m))
        isVisited = append(isVisited, make([]bool, m))
        for j:=0; j < m; j++ {
            dist[i][j] = 65535
        }
    }
    dist[0][0] = grid[0][0]
    isVisited[0][0]=true
    curi, curj := 0, 0
    // 以下是dijkstra算法的改造
    for curi!=n-1 || curj!=m-1 {
        // 先更新刚加入路径的节点所带来的影响，更新该节点周围节点的路径
        arounds := [][]int{{0,-1},{0,1},{-1,0},{1,0}}
        for _, around := range arounds {
            nexti, nextj := curi+around[0], curj+around[1]
            if nexti<n && nexti>=0 && nextj<m && nextj>=0 && !isVisited[nexti][nextj] {
				dist[nexti][nextj]=dist[curi][curj]
				// 对于水位更高的地方，水位等于当前的 否则等于当前的
				if grid[nexti][nextj] > dist[curi][curj] {
					dist[nexti][nextj]=grid[nexti][nextj]
                }
            }
        }
        minDist := 65535
        // 先找到还未加入路径且离起始点最近的节点
        for i:=0; i<n; i++{
            for j:=0; j < m; j++ {
                if !isVisited[i][j] && dist[i][j] < minDist {
                    minDist, curi, curj = dist[i][j], i, j
                }
            }
        }
        // 将新加入的节点标记为已经加入路径
        isVisited[curi][curj]=true
    }
    return dist[n-1][m-1]
}
```
