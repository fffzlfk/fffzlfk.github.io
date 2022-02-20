---
title: "MapReduce_in_Go"
date: "2021-04-24T21:19:14+08:00"
toc: true
draft: false
summary: Map-Reduce 
tags:
    - Go
    - MapReduce
---

### 简单版Generic Map

```go
func Map(data interface{}, fn interface{}) []interface{} {
	vdata := reflect.ValueOf(data)
	vfn := reflect.ValueOf(fn)
	res := make([]interface{}, vdata.Len())
	for i := 0; i < vdata.Len(); i++ {
		res[i] = vfn.Call([]reflect.Value{vdata.Index(i)})[0].Interface()
	}
	return res
}
```

这里用到了[reflect](https://go-zh.org/pkg/reflect/)

我们可以下面的代码使用这个简易的Map函数

```go
func main() {
	square := func(x int) int {
		return x * x
	}
	nums := []int{1, 2, 3, 4}

	squareArr := Map(nums, square)
	for _, v := range squareArr {
		fmt.Print(v, " ")
	}

	upCase := func(str string) string {
		return strings.ToUpper(str)
	}

	fmt.Println()

	strs := []string{"abc", "def", "ghi"}

	upperStrs := Map(strs, upCase)
	for _, v := range upperStrs {
		fmt.Print(v, " ")
	}
}
```

但我们知道reflect是runtime时的事情，如果类型出错，就会出错，所以我们给出健壮版的实现

### Generic Map

...