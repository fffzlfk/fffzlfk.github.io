---
title: "C++ 完美转发"
date: "2022-02-05T14:10:47+08:00"
tags:
  - C++
---

## 为什么要有完美转发

下面是一个类工厂函数：

```cpp
template <typename T, typename Arg>
std::shared_ptr<T> factory(Arg arg) {
  return std::shared_ptr<T>( new T(arg));
}
```

参数对象`arg`在上面的例子中是传值方式传递，这带来了生成额外临时对象[^1]的代价，所以我们改成引用传递：

[^1]: 临时对象的生命周期只有一条语句的时间。

```cpp
template <typename T, typename Arg>
std::shared_ptr<T> factory(Arg &arg) {
  return std::shared_ptr<T>( new T(arg));
}
```

但这种实现的问题是不能绑定右值实参。如`factory<X>(42)`将编译报错，进一步的，我们按常量引用来传递：
```cpp
template <typename T, typename Arg>
std::shared_ptr<T> factory(const Arg &arg) {
  return std::shared_ptr<T>( new T(arg));
}
```

这种实现的问题是不能支持移动语义，形参使用右值引用可以解决完美转发问题。


## 引用折叠

在`C++11`之前，我们不能对一个引用类型继续引用，但`C++`由于右值引用的出现而**放宽**[^2]了这一做法，从而产生了引用折叠规则，允许我们对引用进行引用，既能左引用，又能右引用。但是却遵循如下规则：

函数形参类型 | 实参类型 | 推导后函数形参类型
-- | -- | --
`T&` | 左引用 | `T&`
`T&` | 右引用 | `T&`
`T&&` | 左引用 | `T&`
`T&&` | 右引用 | `T&&`

[^2]: 这里使用**放宽**一词是因为：只有在类型别名和模板参数时可以间接定义引用的引用。

## 模板参数类型推导

对函数模板`template<typename T>void foo(T&&);`，应用上述引用折叠规则，可总结出以下结论：
- 如果实参是类型A的左值，则模板参数T的类型为`A&`，形参类型为`A&`；
- 如果实参是类型A的右值，则模板参数T的类型为`A&&`，形参类型为`A&&`。

这同样适用于类模板的成员函数模板的类型推导：
```cpp
template <class T> class vector {
  public:
  // T是类模板参数 ⇒ 该成员函数不需要类型推导;这里的函数参数类型就是T的右值引用
  void push_back(T &&x);   
  
  // 该成员函数是个函数模板，有自己的模板参数，需要类型推导
  template <typename Args> void emplace_back(Args &&args); } 
```

函数模板的形参必须是T&&形式，才需要模板参数类型推导。即使形参声明为const T&&形式，就只能按字面意义使用，不需要模板参数类型推导。

## 完美转发

下面是上述代码完美转发版本的实现：

```cpp
template <typename T, typename Arg>
shared_ptr<T> factory(Args&& arg) {
  return std::shared_ptr<T>( new T(std::forward<Arg>(arg)) );
}
```

其中`std::forward`是定义在标准库`<utility>`中的模板函数：

```cpp
template< class T > T&& forward( typename std::remove_reference<T>::type& t ) {
  return static_cast<T&&>(t);
}
template< class T > T&& forward( typename std::remove_reference<T>::type&& t ) {
  return static_cast<T&&>(t);
}
```

`std::remove_reference`是个类模板，定义在标准库`<type_traits>`中，用于移除类型的引用其中定义的类型type是引用的基类型。

```cpp
template< class T > struct remove_reference      {typedef T type;};
template< class T > struct remove_reference<T&>  {typedef T type;};
template< class T > struct remove_reference<T&&> {typedef T type;};
```

- 实参的数据类型是左值引用类型`S&`时(`T = S&`)，`t`的类型为`S&`，`static_cast<S& &&>(t)`折叠为`static_cast<S&>(t)`
- 实参的数据类型是右值引用类型`S&&`时(`T = S&&`)，`t`的类型为`S&&`，`static_cast<S&& &&>(t)`折叠为`static_cast<S&&>(t)`


## References

<https://zh.wikipedia.org/zh-cn/%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8>
