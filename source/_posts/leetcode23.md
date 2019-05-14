---
title: leetcode23
date: 2019-05-14 21:26:33
tags:
---
<meta name="referrer" content="no-referrer" />

一道关于链表操作，难度为高级的题目。解法有多种，不同解法运算效率不一。

## 题目描述
合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

**示例:**
```
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6
```

## 常规解法
  构造一个虚拟头节点，把链表中每一个非空的节点值拿出来，找出最小值，头节点下一个值指向最小值节点，头节点前进。节点前进，直到数组里的所有节点都为空
  
  这个做法比较常规，也是这题的第一直觉，时间复杂度为O(n*k)，n为数组长度，k是最长节点长度

```go
func findMin(lists []*ListNode) *ListNode {
    n := len(lists)
    i := 0
    j := -1
    // 随便取一个非空节点
    for ;i<n;i++ {
        if lists[i] != nil {
            j = i
            break
        }
    }
    // 找最小值非空节点
    for i=i+1;i<n;i++{
        if lists[i] != nil && lists[i].Val < lists[j].Val {
            j = i
        }
    }
    if j != -1 {
        p := lists[j]
        lists[j] = p.Next
        return p
    }
    return nil
}

func mergeKLists(lists []*ListNode) *ListNode {
    dummy := new(ListNode)
    cur := dummy
    var p *ListNode
    for {
        p = findMin(lists)
        if p == nil {
            break
        }
        cur.Next = p
        cur = p
    }
    cur.Next = nil
    return dummy.Next
}
```
  golang运行500ms

## 小顶堆
 上一个解法，时间复杂度有点高，golang需500ms。发现每次寻找最小值节点都需要遍历数组所有节点导致复杂度提升，
 
 对于上一个解法改良，使用小顶堆解决获取最小值问题。时间复杂度O(k*logn),n为数组长度，k是最长节点长度
 
 数组每一个节点构建一个小顶堆，找到堆顶的节点，头节点指向最小值节点，最小值节点下一个节点放到堆顶重新构建堆，若下一值为空，则堆容量-1，直到堆空


 
```go
func fixHeap(heap []*ListNode, start, hig int) []*ListNode {
	i := start
	next := i * 2 + 1
	p := heap[start]
	for next < hig {
		if next + 1 < hig && heap[next].Val > heap[next+1].Val {
			next++
		}
		if heap[next].Val >= p.Val {
			break
		}
		heap[i] = heap[next]
		i = next
		next = i * 2 + 1
	}
	heap[i] = p
	return heap
}


func mergeKLists(lists []*ListNode) *ListNode {
	n := len(lists)
	if n == 0 {
		return nil
	}
	dummy := new(ListNode)
	heap := make([]*ListNode, 0, n)
	cur := dummy
	// 节点填入堆
	for i:=0;i<n;i++ {
		if lists[i] != nil {
			heap =append(heap, lists[i])
		}
	}
	lheap := len(heap)
	// 构建堆
	for i:= lheap / 2 - 1;i>=0;i--{
		heap = fixHeap(heap, i, lheap)
	}
	var p *ListNode
	for lheap > 0 {
		p = heap[0]
		cur.Next = p
		cur = p
		p = p.Next
		if p != nil {
			// 堆顶为节点下一值
			heap[0] = p
		} else {
			// 堆容量-1
			lheap--
			heap[0] = heap[lheap]
		}
		// 重新修复堆
		heap = fixHeap(heap, 0, lheap)
	}
	cur.Next = nil
	return dummy.Next
}

```
  运行时间大大减少，golang运行16ms

## 二分法
  其实是可以将一部分节点先合并起来，然后再每一个节点在合并。这里用是二分法，递归进行将数组一半节点合并成有序链表，然后将两个有序链表再合成一个新的有序链表
  
  这个方法比上面的方法更简单，时间复杂度O(k*logn),n是数组长度，k是每个节点平均长度。
  
```go
func nextNode(cur, node *ListNode) (*ListNode, *ListNode) {
    cur.Next = node
    return node, node.Next
}

func mergeKLists(lists []*ListNode) *ListNode {
	n := len(lists)
	if n == 0 {
		return nil
    } else if n == 1 {
        return lists[0]
    } else {
        mid := n / 2
        node1 := mergeKLists(lists[:mid])
        node2 := mergeKLists(lists[mid:])
        dummy := new(ListNode)
        cur := dummy
        for node1 != nil && node2 != nil {
            if node1.Val < node2.Val {
                cur, node1 = nextNode(cur, node1)
            } else {
                cur, node2 = nextNode(cur, node2)
            }
        }
        for node1 != nil {
            cur, node1 = nextNode(cur, node1)
        }
        for node2 != nil {
            cur, node2 = nextNode(cur, node2)
        }
        cur.Next = nil
        return dummy.Next
    }
	
}
```
  运行时间更是比上一个减少，golang8ms