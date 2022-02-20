---
title: C++(临时对象的分析)
date: "2020-10-10"
draft: false
hideLastModified: true
# cover: "img/tmp.jpg"
# useRelativeCover: true
toc: true
keepImageRatio: true
tags:
  - C++
summary: "关闭编译器优化"
showInMenu: false
---

## 复制构造函数

:laptop:笔者在学习OOP时，看到一个讲C++复制构造函数在什么情况下会执行的代码片段，如下：
```cpp
class Student {
   public:
    Student(){};
    Student returnS(Student s) { return s; }
    Student(const Student &e) { cout << "Copy Constructure\n"; }
    ~Student() { cout << "Destructure\n"; }
};
int main() {
    Student stu1;
    stu1.returnS(stu1);
    return 0;
}
```
于是乎，笔者在电脑上敲了一边，确实按照预期复制构造函数执行了两次，第一次在构造形参对象时执行，第二次在返回值复制到主函数产生临时对象时执行。


接着笔者将主函数修改为
```cpp
int main() {
    Student stu1;
    Student stu2 = stu1.returnS(stu1);
    return 0;
}
```
心想：不出意外在构造stu2时，会再执行一次复制构造函数，然而当笔者看到运行结果后，发现并非如此，复制构造函数还是执行了两次，于是笔者陷入了大思考。  
想到可能是聪明的编译器是不是帮我优化掉了临时对象，直接复制构造了stu2。导致了上面的代码经过优化之后和这样写其实是一样的：
```cpp
int main() {
    Student stu1;
    Student &&stu2 = stu1.returnS(stu1);
    return 0;
}
```

[**右值引用**](https://zh.wikipedia.org/wiki/%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8)  
C++中，引用（reference）是指绑定到内存中的相应对象上。左值引用是绑定到左值对象上；右值引用是绑定到临时对象上。这里的左值对象是指可以通过取地址&运算符得到该对象的内存地址；而临时对象是不能用取地址&运算符获取到对象的内存地址。


于是经过几番搜索，找到了这个东西：  

**-fno-elide-constructors**

The C++ standard allows an implementation to omit creating a temporary that is only used to initialize another object of the same type. Specifying this option disables that optimization, and forces G++ to call the copy constructor in all cases.

果然不出所料，当笔者加上这个参数之后
```bash
g++ test_ref.cpp -o test -fno-elide-constructors
```
一切都对劲了起来:smile:！

## 移动构造函数

使用[移动构造函数](https://en.cppreference.com/w/cpp/language/move_constructor)可以提高内存资源的利用效率，从而改进程序的执行性能。

示例程序

```cpp
#include <bits/stdc++.h>
using namespace std;

class A {
public:
    int *i;
    A(int _i) : i(new int(_i)) {
        cout << "A(int) " << i << '\n';
    }
    ~A() {
        cout << "delete pointer: " << i << '\n';
        delete i;
    }
    // 复制构造函数
    A(const A &a) : i(new int(*a.i)) {
        cout << "A(const A&) " << i << '\n';
    }
    // 移动构造函数
    A(A &&a) : i(new int(*a.i)) {
        cout << "A(const A&&) " << i << '\n';
        a.i = nullptr;
    }
};

A getA(A para) {
    A tmp(2);
    cout << hex << tmp.i << '\n';
    return tmp;
}

int main() {
    A a(1);
    A b = getA(a);
    cout << a.i << '\n';
    return 0;
}
```

运行结果

```
A(int) 0x8016eb0            构造a对象
A(const A&) 0x80172e0       参数复制构造para
A(int) 0x8017300            构造tmp对象
0x8017300
A(const A&&) 0x8017320      调用移动构造函数(临时对象构造返回值)
delete pointer: 0           析构临时对象
A(const A&&) 0x8017340      调用移动构造函数(返回值移动到b对象)
delete pointer: 0           析构临时对象(指返回值)
delete pointer: 0x80172e0   析构函数para
0x8016eb0               
delete pointer: 0x8017340   析构b对象
delete pointer: 0x8016eb0   析构a对象
```
