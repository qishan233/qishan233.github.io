+++
title = 'Fast Sort'
date = 2024-04-15T23:01:25+08:00
draft = false
summary = '快速排序的思考'
tags =["算法"]
+++
快速排序算法的核心思路是分治。

对于数组 A 以及待排序的范围 \[left, rigit]，每经过一次 partition，对于下标 x 处的元素，A\[left, x-1] 的元素均小于等于 A\[x]，对于 A\[x+1, right] 的元素均大于 A\[x]；这里 A\[x] 也被称为哨兵；实际上，可以选择数组中的任意值作为哨兵；

也就是说，每经过一次排序，数组 A 的有序性都增加了；当然，x+1 需要小于等于 right，x-1 需要大于等于 left，否则就是数组为空，而数组为空表示递归排序的结束；

所以，快速排序的代码实现核心是实现 partition 方法；核心思路是元素交换，但是怎么交换是另一个关键问题；按照双指针前进方向可以分为：

## 指针同向遍历
双指针同向遍历。快指针向前遍历，直到等于 right-1（这是哨兵为 A\[right] 的情况，如果哨兵不为 A\[rigiht] 那么就需要在 right 位置停下来）；慢指针指向小于等于哨兵的边界；快指针遇到大于哨兵的值则向前移动；遇到小于等于哨兵的值，则慢指针向前移动后（此时指向一个大于哨兵的值）和快指针进行交换；一般而言，会选择 A\[right] 作为哨兵值；

这种思路可以认为是快慢指针将整个数组分为了三个部分:小于等于哨兵、大于哨兵以及未检测；当未检测部分为空的时候，可以认为 partition 结束；而 partition 结束时需要返回的下标就是哨兵的位置，也就是第一、二部分的边界值；

```go
func partition(nums []int, start, end int) int {
	// 退出条件
	if start == end {
		return end
	}

	if start > end {
		return -1
	}

	// 哨兵就位
	x := nums[end]

	// 指针就位
	left := start - 1
	right := start

	// 双指针开始遍历
	for ; right < end; right++ {
		if nums[right] <= x {
			left++
			nums[left], nums[right] = nums[right], nums[left]
		}
	}

	// 保持循环不变性
	left++

	// 哨兵就位
	nums[left], nums[end] = nums[end], nums[left]

	return left
}
```


## 指针双向遍历
这种方法实际上更为复杂，但是有助于锻炼思维模式；left 指针指向 A\[left]，right 指针指向 A\[right]，同样以 A\[right] 作为哨兵；那么 left 遇到大于等于哨兵的值就停下来；right 遇到小于哨兵的值就停下来，然后交换 right 和 left 所在位置的元素；此时需要注意的是停止条件以及循环不变性：

left 指针的循环不变性：left-1 及之前的元素均小于哨兵；right 及之后的元素均大于等于哨兵；

停止条件而言，当 left == right 时，先观察循环不变性：
1. left-1 以及之前的元素，均小于哨兵；
2. right 以及之后的元素，均大于等于哨兵；
此时，看起来只需要将哨兵和 right 指针所在的元素进行交换即可完成一轮 partition；再来看看是怎么达到 left == right 的:
1. left 一直向右移动：如果 A\[left] < 哨兵 && left < right，则 left++；所以 A\[left] 表示遇到了大于等于哨兵的值，或者和 right 重逢了；
2. right 一直向左移动：如果 A\[right] >= 哨兵 && right > left，则 right --；所以 A\[right] 表示小于哨兵的值，或者和 left 重逢了；
3. 如果 left != right，那么就需要交换 left 和 right；之后需要 left++ 来保证循环不变性；
4. 如果 left == right，那么也可以交换一次 left 和 right；之后 left++ 来保证循环不变性；
5. 所以退出后，right 此时一定就是哨兵的位置，所以需要交换 right 和 哨兵的位置，并且返回 right；

从上面的分析来看，步骤 3 和 4 表示经过 步骤 1 和 2 后，总是要交换一次 right 和 left，并自增 left 来维持循环不变性；
```go
func partition2(nums []int, start, end int) int {
	// 退出条件
	if start == end {
		return end
	}

	if start > end {
		return -1
	}

	// 哨兵就位
	x := nums[end]

	// 指针就位
	left := start - 1
	right := start

	for left < right {
		// 左指针一直前进
		for ; left < right; left++ {
			if nums[left] >= x {
				break
			}
		}
		// 右指针一直前进
		for ; right > left; right-- {
			if nums[right] < x {
				break
			}
		}

		// 交换
		nums[left], nums[end] = nums[end], nums[left]

		// 保持循环不变性
		left++
	}

	nums[right], nums[end] = nums[end], nums[right]

	return right
}

```

综上，第一种双指针遍历实现要明显简单的，代码的圈复杂度也更低

其中第一种双指针的核心思路是整理：两个指针同向前进，快指针负责前进，慢指针则维护一个边界；而第二种双指针的核心思路则是交换，将元素放到它们应该去的区域，从而形成局部有序

