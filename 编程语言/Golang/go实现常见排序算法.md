 Go 语言实现几种最常见和最经典的排序算法，包括：

1. **冒泡排序 (Bubble Sort)**
2. **选择排序 (Selection Sort)**
3. **插入排序 (Insertion Sort)**
4. **归并排序 (Merge Sort)**
5. **快速排序 (Quick Sort)**
6. **堆排序 (Heap Sort)**

每个算法都会有详细的注释来解释其核心思想和步骤。

------

### 1. 冒泡排序 (Bubble Sort)

**思想**：重复地遍历要排序的数列，一次比较两个元素，如果它们的顺序错误就把它们交换过来。遍历数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。

```go
package main

import "fmt"

// BubbleSort 冒泡排序
// 时间复杂度: O(n^2)
// 空间复杂度: O(1)
// 稳定性: 稳定
func BubbleSort(arr []int) {
	n := len(arr)
	if n <= 1 {
		return
	}

	// 外层循环控制遍历的轮数
	for i := 0; i < n; i++ {
		// 内存循环进行比较和交换
		// -i 是因为每轮过后，最大的元素会“冒泡”到末尾，无需再比较
		swapped := false
		for j := 0; j < n-1-i; j++ {
			// 如果前一个元素大于后一个元素，则交换
			if arr[j] > arr[j+1] {
				arr[j], arr[j+1] = arr[j+1], arr[j]
				swapped = true
			}
		}
		// 优化：如果在一轮中没有发生任何交换，说明数组已经有序，可以提前退出
		if !swapped {
			break
		}
	}
}
```

------

### 2. 选择排序 (Selection Sort)

**思想**：首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。

```go
package main

import "fmt"

// SelectionSort 选择排序
// 时间复杂度: O(n^2)
// 空间复杂度: O(1)
// 稳定性: 不稳定 (例如 [5, 8, 5, 2, 9]，第一个5会和2交换，导致相对位置改变)
func SelectionSort(arr []int) {
	n := len(arr)
	if n <= 1 {
		return
	}

	// 外层循环控制已排序部分的边界
	for i := 0; i < n; i++ {
		// 假设当前位置 i 的元素是最小的
		minIndex := i
		// 内层循环在未排序部分 [i+1, n-1] 中寻找真正的最小值
		for j := i + 1; j < n; j++ {
			if arr[j] < arr[minIndex] {
				minIndex = j // 更新最小值的索引
			}
		}
		// 将找到的最小值与当前位置 i 的元素交换
		if minIndex != i {
			arr[i], arr[minIndex] = arr[minIndex], arr[i]
		}
	}
}
```

------

### 3. 插入排序 (Insertion Sort)

**思想**：构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。就像打扑克牌时，一张一张地整理手中的牌。

```go
package main

import "fmt"

// InsertionSort 插入排序
// 时间复杂度: O(n^2) (平均), O(n) (最好情况，数组基本有序)
// 空间复杂度: O(1)
// 稳定性: 稳定
func InsertionSort(arr []int) {
	n := len(arr)
	if n <= 1 {
		return
	}

	// 从第二个元素开始，逐个将其插入到前面已排序的部分
	for i := 1; i < n; i++ {
		// 当前需要被插入的元素
		currentVal := arr[i]
		// 已排序部分的最后一个元素的索引
		j := i - 1

		// 从后向前扫描已排序部分，为 currentVal 找到插入位置
		// 如果已排序部分的元素大于 currentVal，则将其后移
		for j >= 0 && arr[j] > currentVal {
			arr[j+1] = arr[j]
			j--
		}

		// 找到了插入位置 (j+1)，放入 currentVal
		arr[j+1] = currentVal
	}
}
```

------

### 4. 归并排序 (Merge Sort)

**思想**：典型的**分治法 (Divide and Conquer)**。将数组递归地分成两半，直到每个子数组只有一个元素（天然有序），然后将这些有序的子数组两两合并（merge），最终得到一个完全有序的数组。

```go
package main

import "fmt"

// MergeSort 归并排序
// 时间复杂度: O(n log n)
// 空间复杂度: O(n) (需要额外的空间来合并)
// 稳定性: 稳定
func MergeSort(arr []int) []int {
	n := len(arr)
	if n <= 1 {
		return arr
	}

	// 1. 分解 (Divide)
	mid := n / 2
	left := MergeSort(arr[:mid])
	right := MergeSort(arr[mid:])

	// 2. 合并 (Merge)
	return merge(left, right)
}

// merge 函数负责将两个有序的子数组合并成一个有序数组
func merge(left, right []int) []int {
	result := make([]int, 0, len(left)+len(right))
	l, r := 0, 0

	for l < len(left) && r < len(right) {
		if left[l] <= right[r] {
			result = append(result, left[l])
			l++
		} else {
			result = append(result, right[r])
			r++
		}
	}

	// 将剩余的元素追加到结果中
	result = append(result, left[l:]...)
	result = append(result, right[r:]...)

	return result
}
```

------

### 5. 快速排序 (Quick Sort)

**思想**：也是**分治法**。通过一趟排序将要排序的数据分割成独立的两部分，其中一部分的所有数据都比另外一部分的所有数据都要小，然后再按此方法对这两部分数据分别进行快速排序，整个排序过程可以递归进行，以此达到整个数据变成有序序列。

```go
package main

import "fmt"

// QuickSort 快速排序
// 时间复杂度: O(n log n) (平均), O(n^2) (最坏情况)
// 空间复杂度: O(log n) (递归栈深度)
// 稳定性: 不稳定
// quickSort 函数接受一个整数切片 arr 作为参数，用于对切片进行快速排序。
func quickSort(arr []int) []int {
	// 如果切片长度小于等于 1，表示已经排序好了，直接返回原切片。
	if len(arr) <= 1 {
		return arr
	}

	// 选择第一个元素作为基准值（pivot）。
	pivot := arr[0]
	// 创建两个空切片，用于存放比基准值小的元素和比基准值大的元素。
	var left, right []int

	// 遍历切片中除基准值外的元素。
	for _, v := range arr[1:] {
		// 如果元素小于等于基准值，将其加入 left 切片。
		if v <= pivot {
			left = append(left, v)
		} else {
			// 否则，将其加入 right 切片。
			right = append(right, v)
		}
	}

	// 递归对 left 和 right 切片进行快速排序。
	left = quickSort(left)
	right = quickSort(right)

	// 将 left 切片、基准值和 right 切片合并成一个有序切片，并返回。
	return append(append(left, pivot), right...)
}
```

------

### 6. 堆排序 (Heap Sort)

**思想**：利用**堆**这种数据结构所设计的一种排序算法。堆是一个近似完全二叉树的结构，并同时满足**堆的性质**：即子结点的键值或索引总是小于（或者大于）它的父节点。

1. 将待排序序列构造成一个大顶堆（或小顶堆）。
2. 将堆顶元素与末尾元素交换，此时末尾就是最大值。
3. 将剩余 `n-1` 个元素重新调整成一个大顶堆，再重复步骤2，直到整个序列有序。

```go
package main

import "fmt"

// HeapSort 堆排序
// 时间复杂度: O(n log n)
// 空间复杂度: O(1)
// 稳定性: 不稳定
func HeapSort(arr []int) {
	n := len(arr)
	if n <= 1 {
		return
	}

	// 1. 构建大顶堆
	// 从最后一个非叶子节点开始，从下至上，从右至左调整
	for i := n/2 - 1; i >= 0; i-- {
		heapify(arr, n, i)
	}

	// 2. 排序
	// 逐个将堆顶元素（最大值）与末尾元素交换，并重新调整堆
	for i := n - 1; i > 0; i-- {
		// 将堆顶元素移到末尾
		arr[0], arr[i] = arr[i], arr[0]
		// 对剩余的元素重新进行堆化
		heapify(arr, i, 0)
	}
}

// heapify 函数负责将以 i 为根的子树调整为大顶堆
// n 是堆的大小
func heapify(arr []int, n, i int) {
	// 初始化最大值为根节点
	largest := i
	// 左子节点和右子节点的索引
	left := 2*i + 1
	right := 2*i + 2

	// 如果左子节点存在且大于根节点
	if left < n && arr[left] > arr[largest] {
		largest = left
	}
	// 如果右子节点存在且大于当前最大值
	if right < n && arr[right] > arr[largest] {
		largest = right
	}

	// 如果最大值不是根节点
	if largest != i {
		// 交换根节点和最大值
		arr[i], arr[largest] = arr[largest], arr[i]
		// 递归地调整受影响的子树
		heapify(arr, n, largest)
	}
}

// 主函数用于测试
func main() {
	testArr := []int{64, 34, 25, 12, 22, 11, 90}
	
	// 选择一个排序算法来测试
	// BubbleSort(testArr)
	// SelectionSort(testArr)
	// InsertionSort(testArr)
	// testArr = MergeSort(testArr)
	// QuickSort(testArr)
	HeapSort(testArr)

	fmt.Println("Sorted array:", testArr)
}
```

