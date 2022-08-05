---
title: "Java泛型"
date: 2022-08-04T22:33:37+08:00
tags:
 - Java
---

## 擦拭法

### 编译器把类型`<T>`视为Object

```java
public class Pair<T> {
    private final T first;
    private final T last;

    public Pair(T first, T last) {
        this.first = first;
        this.last = last;
    }

    public T getFirst() {
        return first;
    }

    public T getLast() {
        return last;
    }
}
```

Java中泛型是在编译阶段的，编译器将上述代码经过擦除法，JVM实际看到的代码如下所示：

```java
public class Pair {
    private final Object first;
    private final Object last;

    public Pair(Object first, Object last) {
        this.first = first;
        this.last = last;
    }

    public Object getFirst() {
        return first;
    }

    public Object getLast() {
        return last;
    }
}
```


### 编译器根据`<T>`实现安全的强制转型

```java
Pair<String> p = new Pair<>("Hello", "world");
String first = p.getFirst();
String last = p.getLast();
```

编译器根据T的类型安全强转：

```java
Pair p = new Pair("Hello", "world");
String first = (String) p.getFirst();
String last = (String) p.getLast();
```

### 擦拭法的局限性

- `<T>`不能是基本类型，因为擦除法要将其替换为`Object`
- 无法获取携带泛型的`Class`，因为泛型是在编译期，所以获取到的都为`Pair<Object>`
例如：
```
@Test
void testGetClass() {
    Pair<String> p1 = new Pair<>("Hello", "World");
    Pair<Integer> p2 = new Pair<>(114, 514);
    Class c1 = p1.getClass();
    Class c2 = p2.getClass();

    assertEquals(c1, c2);
    assertEquals(c1, Pair.class);
}
```
- 不恰当的覆写方法

```java
class Pair<T> {
    public boolean equals(T t) {}
}
```

这是因为，定义的`equals(T t)`方法实际上会被擦拭成`equals(Object t)`，而这个方法是继承自`Object`的，编译器会阻止一个实际上会变成覆写的泛型方法定义。

### 泛型继承

类可以继承一个泛型类。在继承了泛型类型的情况下，子类可以获取父类的泛型类型。例如：`IntPair`可以获取到父类的泛型类型`Integer`。

```java
public class IntPair extends Pair<Integer> {

    public IntPair(Integer first, Integer last) {
        super(first, last);
    }
}

class IntPairTest {
    @Test
    void getSuperParameterizedType() {
        Class<IntPair> clazz = IntPair.class;
        Type t = clazz.getGenericSuperclass();
        if (t instanceof ParameterizedType) {
            ParameterizedType pt = (ParameterizedType) t;
            Type[] types = pt.getActualTypeArguments(); // may have many types
            Type firstType = types[0];
            Class<?> typeClass = (Class<?>) firstType;
            assertEquals(typeClass, Integer.class);
        }
    }
}
```

## `extends`通配符

```java
public class PairHelper {
    static int add(Pair<Number> p) {
        Number first = p.getFirst();
        Number last = p.getLast();
        return first.intValue() + last.intValue();
    }
}
```

![](https://i.imgur.com/rERvLBT.png)

如上，`add`函数接收的参数类型为`Pair<Number>`，所以我们传入`Pair<Number>`是没有问题的；但传入`Pair<Integer>`时，会得到一个编译错误，这是因为虽然`Integer`是`Number`的子类，但是`Pair<Integer>`却不是`Pair<Number>`的子类。

为了解决这个问题，可以使用`Pair<? extends Number>`，这样就可以接收泛型类型为`Number`子类的`Pair`类型了：

```java
static int add(Pair<? extends Number> p) {
    Number first = p.getFirst();
    Number last = p.getLast();
    return first.intValue() + last.intValue();
}

@Test
void addNumberBased() {
    assertEquals(579, add(new Pair<Integer>(123, 456)));
}
```

### 读取

- 能够从`List<? extends Number>`中获取到`Number`类型，因为其包含的元素是`Number`类型或者`Number`的自类型。
- 不能从`List<? extends Number>`中获取到`Integer`类型，因为其保存的元素可能是`Double`。
- 不能从`List<? extends Number>`中获取到`Double`类型，因为其保存的元素可能是`Integer`。

### 写入

- 不能添加`Number`到`List<? extends Number>`中，因为其保存的元素可能是`Integer`或`Double`。
- 不能添加`Integer`到`List<? extends Number>`中，因为其保存的元素可能是`Double`。
- 不能添加`Number`到`List<? extends Number>`中，因为其保存的元素可能是`Integer`。

```java
public class ExtendsTest {
    List<? extends Number> nums1 = new ArrayList<Number>();
    List<? extends Number> nums2 = new ArrayList<Integer>();
    List<? extends Number> nums3 = new ArrayList<Double>();

    void get() {
        Number x = nums1.get(0);
        // Integer y = nums2.get(0); // compile error
        // Double z = nums3.get(0);  // compile error
    }

    void set() {
        /*
        nums1.add(new Number() {
            @Override
            public int intValue() {
                return 0;
            }

            @Override
            public long longValue() {
                return 0;
            }

            @Override
            public float floatValue() {
                return 0;
            }

            @Override
            public double doubleValue() {
                return 0;
            }
        });
         */ // compile error
        // nums2.add(new Integer(1));    // compile error
        // nums3.add(new Double(1.2)); // compile error
    }
}
```

## `super`通配符

`<? super T>` 描述了通配符下界, 即具体的泛型参数需要满足条件: 泛型参数必须是 `T` 类型或它的父类。

### 读取

- 我们不能保证可以从`List<? super Integer>`类型对象中读取到 `Integer` 类型的数据, 因为其可能是 `List<Number>` 类型的。
- 我们不能保证可以从`List<? super Integer>`类型对象中读取到 `Number` 类型的数据, 因为其可能是 `List<Object>` 类型的。
- 唯一能够保证的是, 我们可以从`List<? super Integer>`类型对象中获取到一个 `Object` 对象的实例。

### 写入

对于上面的例子中的 `List<? super Integer>` array 对象：
- 我们可以添加 `Integer` 对象到 `array` 中, 也可以添加 `Integer` 的子类对象到 `array` 中，因为`Integer`及其父类都可以接受`Integer`的子类对象。
- 我们不能添加 `Double/Number/Object` 等不是 `Integer` 的子类的对象到 `array` 中。

![](https://i.imgur.com/ToEcttT.png)

## `extends` vs `super`

### 对于`List<? super Integer> l1`：

- 正确的理解：`? super Integer` 限定的是泛型参数。令 `l1` 的泛型参数是 `T`, 则 `T` 是 `Integer` 或 `Integer` 的父类, 因此 `Integer` 或 `Integer` 的子类的对象就可以添加到 `l1` 中.
- 错误的理解：~~`? super Integer`限定的是插入的元素的类型, 因此只要是 `Integer` 或 `Integer` 的父类的对象都可以插入 `l1` 中。~~

### 对于`List<? extends Integer> l2:`

- 正确的理解：`? extends Integer` 限定的是泛型参数. 令 `l2` 的泛型参数是 `T`, 则 `T` 是 `Integer` 或 `Integer` 的子类, 进而我们就不能找到一个类 `X`, 使得 `X` 是泛型参数 `T` 的子类, 因此我们就不可以向 `l2` 中添加元素. 不过由于我们知道了泛型参数 `T` 是 `Integer` 或 `Integer` 的子类这一点, 因此我们就可以从 `l2` 中读取到元素(取到的元素类型是 `Integer` 或 `Integer` 的子类), 并可以存放到 `Integer` 中。
- 错误的理解：~~`? extends Integer` 限定的是插入元素的类型, 因此只要是 `Integer` 或 `Integer` 的子类的对象都可以插入 `l2` 中~~

## 应用：PECS

**PECS 原则: Producer Extends, Consumer Super**

- Producer extends: 如果我们需要一个 `List` 提供类型为 `T` 的数据(即希望从 `List` 中读取 `T` 类型的数据), 那么我们需要使用 `? extends T`, 例如 `List<? extends Integer>`。 但是我们不能向这个 `List` 添加数据。
- Consumer Super: 如果我们需要一个 `List` 来消费 `T` 类型的数据(即希望将 `T` 类型的数据写入 `List` 中), 那么我们需要使用 `? super T`, 例如 `List<? super Integer>`。但是这个 `List` 不能保证从它读取的数据的类型。
- 如果我们既希望读取, 也希望写入, 那么我们就必须明确地声明泛型参数的类型, 例如 List<Integer>。

```java
public class Collections {
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (int i = 0; i < src.size(); i++) {
            dest.set(i, src.get(i));
        }
    }
}
```

## Source code

- <https://github.com/fffzlfk/generics_tour>

## References

- <https://www.liaoxuefeng.com/wiki/1252599548343744/1255945193293888>
- <https://segmentfault.com/a/1190000008423240>