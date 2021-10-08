---
title: More Effective C++ Operators
description: Do your work
date: 2021-10-08
slug: cpp/more-effective/operators
image: img/2021/10/RedRoofTile.jpg
categories:
  - c++
---

## Be wary of user-defined conversion functions

C++ 允许编译器在不同类型之间执行隐式转换。

两种函数允许编译器执行转换: 单自变量 constructors 和隐式类型转换操作符，此类 constructor 可能声明拥有单一参数，也可能声明拥有多个参数，并且除了第一参数以外都有默认值。

```c++
class Name {
public:
    Name(const string& s); // 可以把 string 转换成 Name
};

class Rational {
public:
    Rational(int numerator = 0, int denominator = 1); // 可以把 int 转换成 Rational
};
```

隐式类型转换操作符是一个拥有特殊名称的成员函数: 关键词 operator 后面加上一个类型名称。

```c++
class Rational {
public:
    operator double() const; // 将 Rational 转换成 double
};
Rational r(1,2);
double d = 0.5 * r; // 将 r 转换成 double 再进行乘法运算
```

为了解决上述隐式转换的问题: 以功能对等的另一个函数取代类型转换操作符。

```c++
class Rational {
public:
    double asDouble() const;
};

Rational r(1, 2);
cout << r; // error, Rational 没有 operator <<
cout << r.asDouble(); // ok, 以 double 形式输出 r
```

解决单自变量 constructors 的隐式转换带来的问题: 使用关键字 `explicit`。只要将 constructor 声明为 explicit，编译器便不能因隐式类型转换的需要而进行调用。

```c++
template <class T>
class Array {
public:
    Array(int lowBound, int highBound);
    Array(int size); // 单自变量constructor => replace explicit Array(int size);
    T& operator[](int index);
};

bool operator==(const Array<int>& lhs, const Array<int>& rhs);

Array<int> a(10);
Array<int> b(10);
for (int i = 0; i < 10; ++i) {
    if (a == b[i]) { // a 应该是 a[i]
    // 上述代码相当于 if (a == static_cast<Array<int>>(b[i]))
      cout << "index " << i << " is equal" << endl;
    } else {
      cout << "index " << i << " is not equal" << endl;
    }
}
```

## Distinguish between prefix and postfix forms of increment and decrement operators

- prefix: increment and fetch
- postfix: fetch and increment

```c++
class UPInt {
public:
    UPInt& operator++(); // prefix
    const UPInt operator++(int); // postfix

    UPInt& operator--();
    const UPInt operator--(int);

    UPInt& operator+=(int);
};

UPInt& UPInt::operator++() {
  *this += 1;
  return *this;
}

const UPInt UPInt::operator++(int) {
  UPInt oldValue = *this;
  ++(*this);
  return oldValue;
}

UPInt i;
i++++; // 同下
i.operator++(0).operator++(0); // 由于返回值是 const, 此操作不合法
```

设计 `classes` 的一条无上宝典就是: 一旦有疑虑，查看 `ints` 行为如何并遵循之。

## Never overload `&&`, `||` or `.`

C++ 对于 "true or false" 采用的评估方式是 "fail fast"。一旦重载了 `operator ||` 或 `operator &&`，函数调用语义会替代 "fail fast"。

1. 当函数调用动作被执行，所有参数值都必须评估完成
2. C++ 语言规范并未明确定义函数调用动作中各参数的评估顺序

表达式如果包含逗号，那么逗号左侧会被先评估，然后逗号的右侧再被评估；最后，整个逗号表达式的结果以逗号右侧的值为代表。

不能重载的操作符: `.`, `.*`, `::`, `?:`, `new`, `delete`, `sizeof`, `typeid`, `static_cast`, `dynamic_cast`, `const_cast`, `reinterpret_cast`

## Understand the different meanings of new and delete

`new operator` 是语言内建的，不能改变意义

1. 分配足够的内存，用来放置某类型的对象
2. 调用一个 constructor，为刚才分配的内存中的那个对象设定初值

`new operator` 调用某个函数，执行必要的内存分配动作，我们可以重写或重载那个函数，改变其行为，其函数名为 `operator new`。

```c++
string *ps = new string("a"); // new operator

void* operator new(size_t size);

void* raw_memory = operator new(sizeof(string));
call string::string("abc") on *raw_memory;
string* ps = static_cast<string*>(raw_memory);
```

- 如果希望将对象产生于 heap，可以使用 `new operator`
- 如果只是打算分配内存，可以使用 `operator new`
