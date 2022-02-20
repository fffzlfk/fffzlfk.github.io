---
title: "Go设计模式"
date: "2021-07-18T22:52:35+08:00"
toc: true
draft: false
summary: Go Design Scheme
tags:
    - Go
---

## 创建型

### 单例模式

#### 饿汉式

```go
package singleton

type Singleton struct{}

var singleton *Singleton

func init() {
	singleton = &Singleton{}
}

// 饿汉式
func GetInstance() *Singleton {
	return singleton
}
```

#### 懒汉式

```go
package singleton

import "sync"

var (
	LazySingleton *Singleton
	once          = &sync.Once{}
)

// 懒汉式
func GetLazyInstance() *Singleton {
	if LazySingleton == nil {
		once.Do(func() {
			LazySingleton = new(Singleton)
		})
	}
	return singleton
}
```

#### 测试结果

```bash
~/go_files/temp/scheme/singleton » go test -benchmem -bench="." -v              fffzlfk@DESKTOP-U99TE3D
=== RUN   TestGetInstance
--- PASS: TestGetInstance (0.00s)
=== RUN   TestGetLazyInstance
--- PASS: TestGetLazyInstance (0.00s)
goos: linux
goarch: amd64
pkg: scheme/singleton
cpu: 11th Gen Intel(R) Core(TM) i5-1135G7 @ 2.40GHz
BenchmarkGetInstanceParallel
BenchmarkGetInstanceParallel-8          1000000000               0.2773 ns/op          0 B/op          0 allocs/op
BenchmarkGetLazyInstanceParallel
BenchmarkGetLazyInstanceParallel-8      1000000000               0.8395 ns/op          0 B/op          0 allocs/op
PASS
ok      scheme/singleton        1.244s
```

可以看到饿汉式性能好一点
