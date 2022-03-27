---
title: "MapReduce in Go"
date: "2022-03-27T21:19:14+08:00"
toc: true
draft: false
tags:
    - Go Generic
    - MapReduce
---

## 简单版Generic Map

```go
func Map[T any](data []T, f func(T) T) []T {
	res := make([]T, len(data))
	for i, v := range data {
		res[i] = f(v)
	}
	return res
}
```


我们可以下面的代码测试这个简易的Map函数

```go
func TestMap(t *testing.T) {
	square := func(x int) int {
		return x * x
	}
	nums := []int{1, 2, 3, 4, 5}
	squareArr := Map(nums, square)
	for i, num := range nums {
		if squareArr[i] != num*num {
			t.Errorf("Expected %d, got %d", num*num, squareArr[i])
		}
	}

	upCase := func(s string) string {
		return strings.ToUpper(s)
	}
	strs := []string{"abc", "def", "ghi"}
	upperStrs := Map(strs, upCase)
	for i, str := range strs {
		if upperStrs[i] != strings.ToUpper(str) {
			t.Errorf("Expected %s, got %s", strings.ToUpper(str), upperStrs[i])
		}
	}
}
```


## 简单版 Generic Reduce

```go
func Reduce[T any](data []T, f func(T, T) T) T {
	res := data[0]
	for i := 1; i < len(data); i++ {
		res = f(res, data[i])
	}
	return res
}
```

我们可以下面的代码测试这个简易的Reduce函数

```go
func TestReduce(t *testing.T) {
	sum := func(x, y int) int {
		return x + y
	}
	nums := []int{1, 2, 3, 4, 5}
	sumArr := Reduce(nums, sum)
	if sumArr != 15 {
		t.Errorf("Expected 15, got %d", sumArr)
	}

	strs := []string{"abc", "def", "ghi"}
	concat := func(x, y string) string {
		return x + y
	}
	concatArr := Reduce(strs, concat)
	if concatArr != "abcdefghi" {
		t.Errorf("Expected abcdefghi, got %s", concatArr)
	}
}
```