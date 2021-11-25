---
title: "Quick Sort 快速排序"
date: 2021-11-25T11:49:38+08:00
draft: false
tags: ["go", "quicksort", "快速排序"]
categories: ["技术"]
---


## 基础的快速排序算法思想

首先需要确定一个基准值，通常的做法是选择待排序区间的头部元素作为基准值。如下图所示，第一轮基准值选择为8。

![快排选择基准值](/images/quick-sort-1.png)

确定基准值之后，记为b，就需要确定基准值在待排序区间的位置（partition 操作），满足如下条件：`小于基准值的元素全部位于基准值的左边，大于基准值的元素全部位于基准值的右边。`

确定的过程如下：先从待排区间的右边开始遍历，找到小于基准值的值k，将其与基准值b互换位置；

然后从待排区间的左边遍历，找到大于基准值的值k2， 并将其与基准值b互换位置；

直到左右index 相遇，本轮遍历结束；

![确定基准值的位置](/images/quick-sort-2.png)

一轮遍历结束之后，对基准值的左右两侧递归地进行上述遍历过程。


## 对partition 操作的优化

在一轮遍历中，分别把左右两边的值与基准值互换的过程，可以优化为在遍历的过程中对左右两边的值直接进行互换，然后在一轮遍历结束后再对基准值进行互换。

如下图所示：首先是将6和10的位置互换，然后再将7和基准值8的位置互换，这样第一轮遍历就结束了。

![partition操作优化](/images/quick-sort-3.png)

接下来再分别对基准值左右两侧递归地执行上述过程。


## go 代码实现

```go
func swap(arr []int, i, j int) {
	arr[i], arr[j] = arr[j], arr[i]
}

func partition(arr []int, left, right int) int {
	benchmark := arr[left]
	i := left
	j := right
	for i < j {
		for arr[j] >= benchmark && i < j {
			j--
		}
		for arr[i] <= benchmark && i < j {
			i++
		}
		if i < j {
			swap(arr, i, j)
		}
	}
	swap(arr, left, i)
	return i
}

func quickSort(arr []int, left, right int) {
	if left > right {
		return
	}
	index := partition(arr, left, right)
	quickSort(arr, left, index-1)
	quickSort(arr, index+1, right)
}
```
