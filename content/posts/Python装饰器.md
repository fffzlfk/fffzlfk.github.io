---
title: "Python装饰器"
date: "2021-02-02T14:23:44+08:00"
draft: false
toc: true
images:
tags:
  - Python
summary: "Python decorator"
---

## 引入

当我们想在一个函数执行前后干点什么事情的时候，可以这么写：

```python
def foo():
	print('foo()!!!')

def bar():
	print('Before')
	foo()
	print('After')

bar()
```

## 装饰器实现

但这样看着总是很别扭，我们可以写成这样：

```python
def foo():
	print('foo()!!!')

def bar(func):
	def inner():
		print(f'Before {func.__name__}')
		func()
		print(f'After {func.__name__}')
	return inner

bar(foo)()
```

>在Python中function是第一公民，所以可以它作为参数


这样我们就实现了一个简单的装饰器，我们可以发现这样在调用的时候比较麻烦，Python还提供了这样了一个语法糖：


```python
def bar(func):
	def inner():
		print(f'In Bar Before {func.__name__}')
		func()
		print(f'In Bar After {func.__name__}')
	return inner

@bar
def foo():
	print('foo()!!!')

foo()
```

除此之外，Python装饰器可以有多个，相当与一层层的包裹：

```python
def bar(func):
	def inner():
		print(f'In Bar Before {func.__name__}')
		func()
		print(f'In Bar After {func.__name__}')
	return inner

def baz(func):
	def inner():
		print(f'In Baz Before {func.__name__}')
		func()
		print(f'In Baz After {func.__name__}')
	return inner


@bar
@baz
def foo():
	print('foo()!!!')

foo()
```

上述代码的输出为：
```
In Bar Before inner
In Baz Before foo
foo()!!!
In Baz After foo
In Bar After inner
```

## 一个应用场景

我们有一个这样的函数：

```python
import time

def slow_method():
	time.sleep(2)
	print('Done!')
```

我们想知道这个函数的运行时间，我们可以用装饰器来实现:

```python
import time

def timeit(func):
	def inner():
		s = time.time()
		func()
		e = time.time()
		print(f'{func.__name__} Finished in {e - s}s.')
	return inner

@timeit
def slow_method():
	time.sleep(2)
	print('Done!')

slow_method()
```

运行结果：
```
Done!
slow_method Finished in 2.0159008502960205s.
```

这样我们就实现了我们的需求。

当被装饰的函数有参数时，同样可以：

```python
import time

def timeit(func):
	def inner(*args):
		s = time.time()
		func(*args)
		e = time.time()
		print(f'{func.__name__} Finished in {e - s}s.')
	return inner

@timeit
def slow_method(a, b):
	time.sleep(2)
	print(f'{a} + {b} = {a+b}')
	print('Done!')

slow_method(1, 2)
```

运行结果

```
1 + 2 = 3
Done!
slow_method Finished in 2.0069477558135986s.
```

## 巧用装饰器来实现记忆化递归

首先我们有一个朴素的Fibonacci函数：

```python
def fib(n):
	if n <= 1: return 1
	return fib(n - 1) + fib(n - 2)
```

很明显我们知道这个函数是十分低效的，我们可以使用记忆化递归来优化这个函数。

```python
cache = {}

def fib(n):
	if n in cache: return cache[n]
	if n <= 1: return 1
	cache[n] = fib(n - 1) + fib(n - 2)
	return cache[n]

print(fib(333))
```

这样我们就能使fib函数快很多。但实现稍微有点麻烦，我们可以用装饰器来简化代码：

```python
class MyCache(object):
	def __init__(self, func):
		self.func = func
		self.cache = {}

	def __call__(self, *args):
		if args not in self.cache:
			self.cache[args] = self.func(*args)
		return self.cache[args]

@MyCache
def fib(n):
	if n <= 1: return 1
	return fib(n - 1) + fib(n - 2)


print(fib(333))
```

这样我们就非常优雅的实现了fib函数。


## Reference

[装饰器 Decorator - Python Weekly EP3](https://www.youtube.com/watch?v=5VCywjS8YEA)
