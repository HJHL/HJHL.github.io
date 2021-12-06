---
title: C++ 中的类型转换
date: 2021-12-05 11:22:46
tags:
- C++
---

类型转换在 C/C++ 中是普遍存在的操作，不同于 C 只有一种类型转换方式——强制类型转换（此处暂不谈隐式类型转换），C++ 中提供多种类型转换方式，用于支撑它的**面向对象**等诸多特性。

<!-- more -->

## 基础概念

### 什么是类型转换

一个实体，由它当前的类型，转换到另一种类型，这个过程叫做类型转换。



* 注意**实体与类型的区别**：类型是抽象的，实体的具体的，它们是一对多的一种关系。例如 `int` 它是个类型，`int x = 1024` 中，`x` 是我们这里说的实体。
* 当代计算机内，数据的存储是一串 `0`、`1` 组成的序列，序列本身毫无意义，类型是对它的一种**规则描述**，赋予序列一个特定的意义/解释。

### 类型转换的意义

* 类型转换是普遍存在的

大多数编程语言中，少不了类型转换。



比如使用 C/C++，计算采购一种物品需要多少钱，物品的数量往往是**整数**，但单价很可能会是个**浮点数**。数量和单价相乘，得到总价。这里面的计算细节：先将表示数量的整数，转换为浮点数，然后与浮点数与浮点数相乘，得到的结果也是个浮点数。



## C 中的类型转换

### 隐式类型转换

```c
float price = 12.56;
int num = 1024;
float sum = price * num; // num 参与计算时，发生了由 int -> float 的隐式类型转换
```

### 显式类型转换

```c
int num = 1024;
float fn = (float) num; // 显式类型转换
```



## C++ 中的类型转换

C++ 兼容上面提到 C 中的两种类型转换，但**不推荐使用强制类型转换**，它引入了**四种**类型转换方式，它们遵循同一种使用方式：`cast_key_word<new_type>(expression)`。	

### `const_cast`

首先介绍下 `const_cast`。它的特殊之处在于：**满足条件，可以加上或者移除变量的 `CV` 类型限定符**。这点是其他三个类型转换不能完成的。

> CV 类型限定符，cv (const and volatile) type qualifiers，用作修饰变量，以便影响编译器的行为。例如 `const int x = 1023；`，如果后面的代码修改了变量 `x`，则会在**编译期**抛出异常，这样的好处是减少运行时出现难以理解 Bug 的可能。

`const_cast` **对函数指针无用**。通过 `const_cast` 为变量添加 `const` 很容易（甚至可以不需要它），但通过 `const_cast` 移除 `CV` 类型限定符需要 `new_type` 和 `expression` 之间符合以下条件之一：

1. 是相同类型，或者类型可以相互转换的对象的指针（也可以是多级指针）。

```cpp
class Base;

const Base* constBase = new Base();
Base* nonConstBase = const_cast<Base *>(constBase); // Ok，因为 Base* 和变量 constBase 的类型相同
```



2. 任何类型的左值，可以转换为该种类型的左值或右值引用；一个类型的右值或 `xvalue`，可以转换成同类型的右值。

```cpp
const int x = 0;
const_cast<int &>(x) = 10;
```

3. 以上规则适用于指向类**数据成员**的多级指针。

```cpp
class Base {
public:
  int m_i;
  Base() : m_i(1) {}
};

int main() {
  const Base b;
  const_cast<int&>(b.m_i) = 10;
}
```

4. 空指针可以转换为任何类型的空指针。

这很容易理解。



示例：

```cpp
class Base {
public:
  int m_i;
  // 默认构造函数
  Base(): m_i(0) {}

  void f(int x) const { // const 表示在 Base 的实例上调用 f() 后，“不会修改”对象。
    // m_i = v; // 因为 f 被 const 修饰，所以这个会导致编译报错
    const_cast<Base *>(this)->m_i = x; // Ok，因为去掉了 const 限定符
  }
};

int main() {
  const int x = 10;
  // const_cast<int>(x) = 11; // 错误。

  int *px = const_cast<int *>(&x);
  *px = 4; // ！！未定义的行为。
  
  const Base b;
  b.f(4); // ！！未定义的行为。
}
```



### `static_cast`

`static_cast` 用于隐式类型转换，或者**用户自定义的规则转换（user-defined conversions）**。

可能有朋友看到过一些文章，说 `static_cast` 是 兼容 C 的一个类型转换方式。但千万不能看轻了它，`static_cast` 还有一个非常重要的功能，就是支持用户自定义类型转换规则。

```cpp
class Point3D;

class Point2D {
public:
  int x;
  int y;
  Point2D(): x(0), y(0) {}
  
  Point2D(int _x, int _y): x(_x), y(_y) {}
};

class Point3D {
public:
  int x;
  int y;
  int z;
  Point3D(): x(0), y(0), z(0) {}
  
  Point3D(int _x, int _y, int _z): x(_x), y(_y), z(_z) {}
  
  explicit operator Point2D() const {
    return Point2D(x, y);
  }
};

int main() {
  Point3D p3d(2,3, 1);
  Point2D p2d = static_cast<Point2D>(p3d); // Ok
}
```



### `dynamic_cast`

`dynamic_cast` 是唯一的**运行时**转换方式，用于**类**之间的**安全**转换。

`dynamic_cast` 存在转换失败的可能。

* 当 `new_type` 为指针类型时，若转换失败，则**返回空指针**。
* 当 `new_type` 为引用类型时，若转换失败，则将会**抛出异常**。

### `reinterpret_cast`

`reinterpret_cast `的特点十分鲜明——它的转换仅是**对比特位的拷贝**！

## 总结

![C++ 四种类型转换方式总结](/images/cpp-cast.png)

## 参考资料

1. [Value categories](https://en.cppreference.com/w/cpp/language/value_category)
2. [cv (`const` and `volatile`) type qualifiers](https://en.cppreference.com/w/cpp/language/cv)
3. [const_cast conversion](https://en.cppreference.com/w/cpp/language/const_cast)
4. [static_cast](https://en.cppreference.com/w/cpp/language/static_cast)
5. [dynamic_cast](https://en.cppreference.com/w/cpp/language/dynamic_cast)
6. [reinterpret_cast](https://en.cppreference.com/w/cpp/language/reinterpret_cast)
