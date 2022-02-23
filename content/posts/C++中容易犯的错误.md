---
title: "C++中容易犯的错误"
date: 2022-02-23T23:31:50+08:00
tags:
    - C++
---

## 不正确地使用`new`和`delete`

无论我们如何努力，要释放所有动态分配的内存是非常困难的。即使我们能做到这一点，也往往不能安全地避免出现异常。让我们看一个简单的例子。

```cpp
void SomeMethod() {
    ClassA *a = new ClassA;
    SomeOtherMethod(); // it can throws an execption
    delete a;
}
```

如果`SomeOtherMethod`抛出了异常，那么`a`对象永远不会被删除。下面的例子展示了一个更加安全同时又更简洁的实现，使用了在`C++11`提出的`std::unique_ptr`。

```cpp
void SomeMethod() {
    std::unique_ptr<ClassA> a(new ClassA);
    SomeOtherMethod();
}
```

无论发生什么，当`a`退出作用域的时候，它会被释放。

然而，这仅仅是`C++`中这种错误最简单的例子，还有很多例子表明`delete`应该在其他地方调用，也许是在外层函数或者另一个线程中。这就是为什么应该避免使用`new`和`delete`，而应该使用适当的智能指针。

## 被忘记的虚析构函数

这是最常见的错误之一，如果派生类中有动态内存分配，将会导致派生类的内存泄漏。这里有一些例子，当一个类不打算用于继承，并且它的大小和性能是至关重要的。虚析构函数或任何其他虚函数在类在类中引入了额外的数据，即指向虚函数表的指针，这使得类的任何实例的大小变大。

然而，在大多数情况下，类可以被继承，即使它的初衷并非如此。因此，在声明一个类的时候，添加一个虚析构函数是一个非常好的做法。否则，如果一个类由于性能的原因必须不包含虚函数，那么在类的声明文件里面加上一个注释，说明这个类不应该被继承，是一个很好的做法。避免这个问题的最佳选择之一是使用一个支持在创建类时创建虚析构函数的`IDE`。

关于这个问题，还有一点是来自标准库的类或模板。它们不是用来继承的，也没有一个虚析构函数。例如，如果我们创建了一个公开继承自`std::string`的新的增强字符串类，就可能有人错误地使用它与`std::string`的指针或引用，从而导致内存泄漏。

```cpp
class MyString : public std::string {
    ~MyString() {}
};

int main() {
    std::string *s = new MyString();
	delete s; // May not invoke the destructor defined in MyString
}
```

为了避免这样的问题，重用标准库中的类或模板的一个更安全的方法是使用私有继承[^1]或组合。

[^1]: 用《Effective C++》中的话说就是private继承是实现继承。就是`class Derivative` 想要用`class Base`的函数接口，但是又不想让别人通过使用`Derivative`的对象来使用`Base`的函数接口。这时可以用`private`继承。或者`Base`和`Derivative`根本没有任何逻辑上的联系，单纯的`D`想要复用一下`B`的代码，这时就用`private` 继承。

## 用`delete`或智能指针删除一个数组

创建动态大小的临时数组往往是必要的。当它们不再需要时，释放分配的内存是很重要的。这里的问题是，`C++`需要带有`[]`括号的特殊删除操作符，这一点很容易被遗忘。`delete[]`操作符不仅会删除分配给数组的内存，而且会首先调用数组中所有对象的析构函数。对原始类型使用不带`[]`括号的删除操作符也是不正确的，尽管这些类型没有析构函数，每个编译器都不能保证一个数组的指针会指向数组的第一个元素，所以使用不带`[]`括号的`delete`也会导致未定义的行为。

在数组中使用智能指针，如`unique_ptr<T>`, `shared_ptr`，也是不正确的。当这样的智能指针从作用域中退出时，它将调用不带`[]`括号的删除操作符，这将导致上面描述的同样问题。如果需要对数组使用智能指针，可以使用`unique_ptr<T[]>`的特殊化。

如果不需要引用计数的功能，主要是数组的情况，最优雅的方法是使用`STL`向量来代替。它们不只是负责释放内存，而且还提供额外的功能。

## 返回一个局部对象的引用

这主要是一个初学者的错误，但它值得一提，因为有很多遗留的代码都存在这个问题。让我们看看下面的代码，一个程序员想通过避免不必要的复制来进行某种优化。

```cpp
Complex& SumComplex(const Complex& a, const Complex& b) {
    Complex result;
    …..
    return result;
}

Complex& sum = SumComplex(a, b);
```

对象 `sum`现在将指向局部对象`result`。但是，在执行`SumComplex`函数后，对象`result`位于哪里呢？不知道。它位于堆栈中，但在函数返回后，堆栈被收缩，函数中的所有本地对象都被析构了。这最终会导致一个未定义的行为，即使是原始类型。为了避免性能问题，有时可以使用返回值优化。

```cpp
Complex SumComplex(const Complex& a, const Complex& b) {
     return Complex(a.real + b.real, a.imaginar + b.imaginar);
}

Complex sum = SumComplex(a, b);
```

对于今天的大多数编译器来说，如果一个返回行包含一个对象的构造函数，代码将被优化以避免所有不必要的复制--构造函数将直接在`sum`对象上执行。

## 使用对已删除资源的引用

这些`C++`问题比你想象的要经常发生，而且通常出现在多线程的应用程序中。让我们考虑一下下面的代码。

- Thread 1:
```cpp
Connection& connection = connections.GetConnection(connectionId);
// ...
```

- Thread 2:
```cpp
connections.DeleteConnection(connectionId);
// …
```

- Thread 1:
```cpp
connection.send(data);
```

在这个例子中，如果两个线程使用相同的连接ID，这将导致未定义的行为。违反访问权限的错误往往是很难发现的。

在这种情况下，当一个以上的线程访问同一资源时，保留资源的指针或引用是非常危险的，因为其他线程可以删除它。使用带有引用计数的智能指针要安全得多，例如`std::shared_ptr`。它使用原子操作来增加/减少一个引用计数器，所以它是线程安全的。

## 允许异常离开析构函数

并不经常需要从一个析构函数中抛出一个异常。即使如此，也有更好的方法来做到这一点。然而，异常大多不是明确地从析构器中抛出的。可能发生的情况是，一个简单的记录对象销毁的命令就会导致异常的抛出。让我们考虑以下代码。

```cpp
class A {
public:
   A() {}
   ~A() {
      writeToLog(); // could cause an exception to be thrown
   }
};

// …

try {
   A a1;		
   A a2;		
} catch (std::exception& e) {
   std::cout << "exception caught";
}
```

在上面的代码中，如果异常发生了两次，比如在销毁两个对象的过程中，`catch`语句就不会被执行。因为有两个并行的异常，无论它们是同一类型还是不同类型，`C++`运行环境都不知道如何处理，并调用一个终止函数，导致程序执行的终止。为了避免这一点`C++11`开始，`destructor`默认是`noexcept`。

## 使用无效的迭代器和引用

关于这个问题，可以写一整本书。每个`ST`L容器都有一些特定的条件，在这些条件下它会使迭代器和引用失效。在使用任何操作时，都要注意这些细节。就像之前的`C++`问题一样，这个问题在多线程环境中也会经常发生，所以需要使用同步机制来避免它。让我们看看下面的顺序代码作为一个例子。

```cpp
vector<string> v;
v.push_back(“string1”);
string& s1 = v[0];     // assign a reference to the 1st element
vector<string>::iterator iter = v.begin();    // assign an iterator to the 1st element
v.push_back(“string2”);
cout << s1;     // access to a reference of the 1st element
cout << *iter;  // access to an iterator of the 1st element
```

从逻辑的角度来看，这段代码似乎完全没有问题。然而，在向量中添加第二个元素可能会导致向量内存的重新分配，这将使迭代器和引用都无效，并导致在最后两行试图访问它们时出现访问违规错误。

## 通过值传递对象

你可能知道，由于对性能的影响，按值传递对象是个坏主意。许多人为了避免输入额外的字符而让它保持这样的状态，或者可能想到以后再返回去做优化。这通常是不可能的，结果是导致了性能较差的代码和容易出现意外行为的代码。

```cpp
class A {
  public:
    virtual std::string GetName() const { return "A"; }
    ...
};

class B : public A {
  public:
    virtual std::string GetName() const { return "B"; }
    ...
};

void func1(A a) {
    std::string name = a.GetName();
    ...
}

B b;
func1(b);
```

这段代码调用`func1`函数将创建一个对象`b`的部分副本，即它将只复制类`A`的部分对象`b`到对象`a`（"切片问题"）。所以在函数中，它也会调用类`A`的方法，而不是类`B`的方法，这很可能不是调用该函数的人所期望的。

类似的问题也发生在试图捕获异常的时候，比如说：

```cpp
class ExceptionA : public std::exception;
class ExceptionB : public ExceptionA;

try {
    func2(); // can throw an ExceptionB exception
} catch (ExceptionA ex) {
    writeToLog(ex.GetDescription());
    throw;
}
```

当一个`ExceptionB`类型的异常从函数`func2`抛出时，它将被`catch`块捕获，但由于切片问题，只有`ExceptionA`类的一部分会被复制，不正确的方法会被调用，而且重新抛出也会向外部的`try-catch`块抛出一个不正确的异常。

总而言之，总是通过引用来传递对象，而不是通过值。

## 类构造函数隐式调用

虽然有时候定义的转换也非常有用，但它们会导致难以预测的转换，而且很难定位。比方说，有人创建了一个有字符串类的库：

```cpp
class String {
public:
    String(int n);
    String(const char *s);
    ...
}
```

第一个方法的目的是创建一个长度为 $n$ 的字符串，第二个方法的目的是创建一个包含给定字符的字符串。但是当你有这样的东西时，问题就出现了。

```cpp
String s1 = 123;
String s2 = 'abc';
```
在上面的例子中，`s1`将成为一个大小为`123`的字符串，而不是一个包含`"123"`字符的字符串。第二个例子包含单引号而不是双引号（这可能是意外发生的），这也会导致调用第一个构造函数并创建一个尺寸非常大的字符串。这些都是非常简单的例子，还有很多更复杂的情况会导致混乱和难以预料的转换，很难发现。对于如何避免这类问题，`C++11`开始有了`explicit`关键字，可以指定构造函数或转换函数为显式, 即它不能用于隐式转换和复制初始化。
