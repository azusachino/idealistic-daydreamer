---
title: 设计与声明
description: 好的设计、好的开始
date: 2021-08-21
slug: cpp/effective/design-declaration
image: img/2021/08/magical-miku.jpg
categories:
  - c++
---

## Make interfaces easy to ues correctly and hard to use incorrectly

```c++
class Date {
public:
    Date(int month, int day, int year);
};

// v2 with builder
struct Day {
    explicit Day(int d): val(d){}
int val;
};
struct Month {
    explicit Month(int d): val(d){}
int val;
};
struct Year {
    explicit Year(int d): val(d){}
int val;
};
class Date {
public:
    Date(const Month&, const Day&, const Year&);
};

Date d(Month(12), Day(31), Year(2020));
```

- 促进正确使用的办法包括接口的一致性、内置类型的行为兼容
- 阻止误用的办法包括建立新类型、限制类型上的操作、束缚对象值、消除客户的资源管理责任

## Treat class design as type design

- 新 type 的对象应该如何被创建和销毁？
- 对象的初始化和对象的赋值该有什么样的差别？
- 新 type 的对象如果被 passed-by-value，意味着什么？
- 什么是新 type 的“合法值”？
- 你的新 type 需要配合某个继承图系？
- 你的新 type 需要什么样的转换？
- 什么样的操作符和函数对此新 type 是合理的？
- 什么样的标准函数应该驳回？
- 谁该取用新 type 的成员？
- 什么是新 type 的“未申明接口”？
- 你的新 type 有多么一般化？
- 你真的需要一个新 type 吗？

## Prefer pass-by-reference-to-const to pass-by-value

```c++
class Person {
public:
    Person();
    virtual ~Person();
private:
    std::string name;
    std::string addr;
};
class Student : public Person {
public:
    Student();
    ~Student();
private:
    std::string schoolName;
    std::string className;
};

bool validate(Student s);

// 一次Student copy构造函数、一次Person copy构造函数
// 四次string copy构造函数 => 六次析构函数
int main() {
    Student plato;
    bool ok = validate(plato);
}
```

slicing 对象分割问题

```c++
class Window {
public:
    std::string name() const;
    virtual void display() const;
};

class CircleWindow : public Window {
public:
    virtual void display() const;
};

// 在printName函数内，不论传递过来的对象原本是什么类型
// 参数w总是一个Window对象 (调用的总是Window::display)
void printName(Window w) {
    std::cout << w.name();
    w.display();
}

// 引用传递没有上述问题
void printName(const Window& w) {
    std::cout << w.name();
    w.display();
}
```

**对于内置类型，值传递比引用传递的效率更高**。

## Don't try to return a reference when you must return an object

函数创建的对象，要么在 stack，要么在 heap。

- 如果定义一个 local 变量，就是在 stack 空间创建对象
- Heap-based 对象由 new 创建

```c++
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
private:
    int n, d;
    friend const Rational operator*(const Rational& lhs, const Rational& rhs);
};

// 函数返回一个reference指向result，
// 但result是local对象，在函数退出前就被销毁了。
const Rational& operator*(const Rational& lhs, const Rational& rhs) {
    Rational result(lhs.n*rhs.n, lhs.d*rhs.d); // bad code
    return result;
}

const Rational& operator*(const Rational& lhs, const Rational& rhs) {
    Rational* result = new Rational(lhs.n*rhs.n, lhs.d*rhs.d);
    return *result;
}

Rational w, x, y, z;
// 同一个语句内调用了两次operator*，却没有合理的办法调用delete
w = x * y * z; // operator*(operator*(x,y), z)

// use static
const Rational& operator*(const Rational& lhs, const Rational& rhs) {
    static Rational result;
    result = Rational(lhs.n*rhs.n, lhs.d*rhs.d);
    return result;
}

bool operator==(const Rational& lhs, const Rational& rhs);
Rational a, b, c, d;

if ((a*b) == (c*d)) {
    // always return true
}
if (operator==(operator*(a,b), operator*(c,d))) {
    // 中间返回相同的result对象，所以始终返回true
}
```

## Declare data members private

- 客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证
- protected 并不比 public 更具封装性

## Prefer non-member non-friend functions to member functions

```c++
class WebBrowser {
public:
    void clearCache();
    void clearHistory();
    void removeCookies();
};

// member function
class WebBrowser {
public:
    ...
    void clearEverything();
};

// non member function
void clearBrowser(WebBrowser &wb) {
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```

## Declare non-member functions when type conversions should apply to all parameters

```c++
Rational oneHalf(1, 2);

// 隐式类型转换
Rational result = oneHalf * 2; // ok
result = 2 * oneHalf; // error

// non member
const Rational operator*(const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.numerator() * rhs.numerator(), lhs.denominator() * rhs.denominator());
}
```

只有当参数被列于参数列(parameter list)内，这个参数才是隐式类型转换的合格参与者。

## Consider support for a non-throwing swap

实际上，感觉不是很熟悉啊。。。想了解的话，看书吧

- 当 std::swap 对你的类型效率不高时，提供一个 swap 成员函数，并保证它不抛出异常
- 如果提供了一个 member swap，也需要提供一个 non member swap 用来调用前者
- 调用 swap 时应针对 std::swap 使用 using 声明式，然后调用 swap 并且不带任何"命名空间资格修饰"
