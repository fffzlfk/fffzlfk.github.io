---
title: "从模板元编程到constexpr(C++)"
date: "2021-11-28T19:43:10+08:00"
toc: true
draft: false
summary: C++元编程
tags:
    - C++
---

## 模板元编程

```cpp
#include <cstdint>

template <uint64_t N> struct Fact {
    enum { Value = N * Fact<N - 1>::Value };
};

template <> struct Fact<1> {
    enum { Value = 1 };
};

template <uint64_t N> struct Fib {
    enum { Value = Fib<N - 1>::Value + Fib<N - 2>::Value };
};

template <> struct Fib<1> {
    enum { Value = 1 };
};

template <> struct Fib<2> {
    enum { Value = 2 };
};

template <uint64_t base, uint64_t exp> struct Pow {
    enum { Value = base * Pow<base, exp - 1>::Value };
};

template <uint64_t base> struct Pow<base, 1> {
    enum { Value = base };
};

int main() {
    auto val1 = Fact<10>::Value;
    auto val2 = Fib<20>::Value;
    auto val3 = Pow<2, 10>::Value;
    return 0;
}
```

如上代码片段我们是用模板实现了在编译期的计算（`C++11`起支持）。

## constexpr

从`C++14`开始，支持使用`consrexpr`来修饰函数，这样如果我们给这个函数传入常量，那么将会在编译器计算；如果传入变量，只能在运行期计算。这一点可以从生成的汇编代码看出。

```cpp
#include <cstdint>

constexpr auto fact(uint64_t n) {
    if (n == 1) {
        return n;
    }
    return fact(n-1) * n;
}

int main() {
    constexpr auto val = fact(10);
    return 0;
}
```

```asm
fact(int):
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], edi
        cmp     DWORD PTR [rbp-4], 1
        jne     .L2
        mov     eax, DWORD PTR [rbp-4]
        jmp     .L3
.L2:
        mov     eax, DWORD PTR [rbp-4]
        sub     eax, 1
        mov     edi, eax
        call    fact(int)
        imul    eax, DWORD PTR [rbp-4]
.L3:
        leave
        ret
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 32
        mov     DWORD PTR [rbp-20], edi
        mov     QWORD PTR [rbp-32], rsi
        mov     DWORD PTR [rbp-4], 3628800
        mov     eax, DWORD PTR [rbp-20]
        mov     edi, eax
        call    fact(int)
        mov     DWORD PTR [rbp-8], eax
        mov     eax, 0
        leave
        ret
```

## constexpr与模板元编程的比较

```cpp
#include <cstdint>

constexpr auto fib(uint64_t n) {
    if (n <= 1) {
        return 1;
    }
    return fib(n-1) + fib(n-2);
}

int main(int argc, char *argv[]) {
    constexpr auto val = fib(40);
    return 0;
}
```

如果我们采用`constexpr`这种写法的话，编译器会报错，这是因为这个函数重复计算比较多。

而我们使用下面模板元编程这种写法的话就可以，这是因为编译器会对其进行记忆化，省去了重复的计算。

未来编译器可能会支持`constexpr`函数的记忆化。

```cpp
template<uint64_t N> struct Fib {
    enum { Value = Fib<N-1>::Value + Fib<N-2>::Value };
};

template<> struct Fib<1> {
    enum { Value = 1 };
};

template<> struct Fib<0> {
    enum { Value = 1};
};

int main(int argc, char *argv[]) {
    auto val = Fib<40>::Value;
    return 0;
}
```

## constexpr if

`C++17`开始支持`constexpr if`，这意味着我们可以简化模板元编程的写法，省去了实例化初始值。这样的话即使比较复杂的计算也能够支持，因为模板元编程有记忆化。

```cpp
#include <cstdint>

template<uint64_t N> struct Fib {
    static constexpr uint64_t Value = []{
        if constexpr (N <= 1) {
            return 1;
        } else {
            return Fib<N-1>::Value + Fib<N-2>::Value;
        }
    }();
};

int main(int argc, char *argv[]) {
    constexpr auto val = Fib<40>::Value;
    return 0;
}
```