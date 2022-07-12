## 关于面试的一道代码题

当时没写出来。。。面完去上了个厕所就想出来了。。。。。。。。。

题目大意:
> 在一个从大到小的数组的中找到一个数第一次出现和最后一次出现的坐标。

实现:
```go
func main() {

	fmt.Println(match([]int{0, 1, 1, 2, 3, 4, 17}, 1)) // 1 2
	fmt.Println(match([]int{0, 1, 1, 2, 3, 4, 17}, 3)) // 4 4
}

func match(ints []int, target int) []int {
	var left = -1
	var right = -1

	for i, x := range ints {
		if left == -1 {
			if x == target {
				left = i
				right = i
			}
		}

		if x == target {
			right = i
		}
	}

	return []int{left, right}
}
```