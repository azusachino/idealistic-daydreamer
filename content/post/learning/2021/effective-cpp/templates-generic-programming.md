---
title: 模板与泛型编程
description: templates & generics
date: 2021-08-31
slug: effective-cpp/templates-generics
image: img/2021/08/SeaSwallow.jpg
categories:
  - learning
tags:
  - c++
---

## Understand implicit interfaces and compile-time polymorphism

面向对象编程世界总是以显式接口和运行时多态解决问题。

```c++
// 显式接口定义
class Widget {
public:
    Widget();
    virtual ~Widget();
    // 函数名称、参数类型、返回类型
    virtual std::size_t size() const;
    virtual void normalize();
    void swap(Widget & other);
};

void doProcess(Widget& w) {
    if (w.size() > 10 && w != someNastyWidget) {
      Widget temp(w);
      temp.normalize();
      temp.swap(w);
    }
}

// 模板定义
template <typename T>
void doProcess(T& w) {
    // w必须提供 size成员函数，支持 operator !=函数
    if (w.size() > 10 && w != someNastyWidget) {
      T temp(w);
      temp.normalize();
      temp.swap(w);
    }
}
```

- classes 和 templates 都支持接口和多态
- 对 classes 而言接口时显式的，以函数签名为中心。多态是通过 virtual 函数发生于运行时
- 对 templates 参数而言，接口是隐式的，基于有效表达式。多态是通过 template 具现化和函数重载解析发生于编译期

## Understand the two meanings of typename

声明 template 参数时，class 和 typename 意义相同

```c++
template<class T> class Widget;
// typename暗示参数并非一定是一个class类型
template<typename T> class Widget;
```

通过`typename C::const_iterator iter(container.begin())`声明这是一个类型

```c++
// C refers to Container
template<typename C>
void print2nd(const C& container) {
  // 如果 C 有个static成员变量而碰巧被命名为const_iterator
  // 或者 x 碰巧是个global变量名称
  if (container.size() >= 2) {
      C::const_iterator* x; // 不再是声明，而是一个相乘动作
      x = iter(container.begin());
      ++iter;
      int value = *iter;
      std::cout << value;
  }
}
```

- 任何需要在 `template` 中指涉一个嵌套从属类型名称的时候，就必须在紧临它的前一个位置加上关键字 `typename`
- `typename`只被用来验明嵌套从属类型名称
- `typename`不可以出现在 base class list 内的嵌套从属类型名称之前，也不可以在 member initialization list 中作为 base class 修饰符

```c++
template<typename T>
class Derived: public Base<T>::Nested { // base class list 不允许
public:
    explicit Derived(int x) :Base<T>::Nested(x) { // member init list 不允许
      typename Base<T>::Nested temp; // ok
    }
}
```

```c++
template<typename IterT>
void workWithIterator(IterT iter) {
  typedef typename std::iterator_traits<IterT>::value_type value_type;
  value_type temp(*iter);
}
```

## Know how to access names in templatized base classes

```c++
class CompanyA {
public:
    void sendClearText(const std::string & msg);
    void sendEncrypted(const std::string & msg);
};

class CompanyB {
public:
    void sendClearText(const std::string & msg);
    void sendEncrypted(const std::string & msg);
};

class MsgInfo {};

template<typename Company>
class MsgSender {
public:
    void sendClear(const MsgInfo& info) {
        std::string msg;
        Company c;
        c.sendClearText(msg);
    }

    void sendSecret(const MsgInfo& info) {
        std::string msg;
        Company c;
        c.sendEncrypted(msg);
    }
};

template<typename Company>
class LoggingMsgSender : public MsgSender<Company> {
public:
    void sendClearMsg(const MsgInfo& info) {
        // do log before
        sendClear(info); // there are no arguments to ‘sendClear’ that depend on a template parameter
        // 如果Company == CompanyZ, sendClear函数不存在
        // do log after
    }
};

// no sendClearText for companyZ
class CompanyZ {
public:
    void sendEncrypted(const std::string & msg);
};

// 特化版的MsgSender，既不是template，也不是class
template<>
class MsgSender<CompanyZ> {
public:
    void sendSecret(const MsgInfo& info) {
      ...
    }
}
```

C++知道 base class templates 有可能被特化，而那个特化版本可能不提供和一般性 template 相同的接口，因此往往拒绝在 templatized base classes 中寻找继承而来的名称。令 C++"不进入 templatized base classes 观察"的行为失效:

- 在 base class 函数调用动作之前加上`this->` => `this->sendClear(info)`
- 使用 using 声明式 => `using MsgSender<Company>::sendClear`
- 明确指出被调用的函数位于 base class 内 => `MsgSender<Company>::sendClear(info)`

## Factor parameter-independent code out of templates

- Templates 生成多个 classes 和多个函数，所以任何 template 代码都不应该于某个造成膨胀的 template 参数产生相依关系
- 因非类型模板参数而造成的代码膨胀，往往可以消除，以函数参数或 class 成员变量替换 template 参数
- 因类型参数而造成的代码膨胀往往可以降低，让带有完全相同二进制表述的具体类型共享实现码

## Use member function templates to accept "all compatible types"

```c++
template <class T>
class shared_ptr {
public:
    shared_ptr(shared_ptr const & r); // copy
    template <class Y>
    shared_ptr(shared_ptr<Y> const & r); // generic copy

    shared_ptr operator=(shared_ptr const& r);
    template <class Y>
    shared_ptr operator=(shared_ptr<Y> const& r);
};
```

## Define non-member functions inside templates when typeconversion are desired

```c++
template <typename T>
class Rational {
public:
    Rational(const T& numerator = 0, const T& denominator = 1);
    const T numerator() const;
    const T denominator() const;
    // template class 内的 friend 声明式可以指涉某个特定函数
    // friend
    // const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs);
};

template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs) {

}

// template实参推导过程中从不将隐式类型转换函数纳入考虑
int main() {
    Rational<int> oneHalf(1,2);
    Rational<int> result = oneHalf * 2; // can't compile
}
```

- 编写 class template 的时候，如果 template 相关的函数支持"所有参数隐式类型转换"，将函数定义为"class template 内部的 friend 函数"

## Use traits classes for information about types

```c++
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d) {
  // 利用iterator_traits获取类型信息
  doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
}

// random access iterator
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::random_access_iterator_tag) {
  iter += d;
}

// bidirectional iterator
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::bidirectional_iterator_tag) {
  if (d >= 0) {
    while (d--) {
      ++iter;
    }
  } else {
    while(d++){
      --iter;
    }
  }
}

// input iterator
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, std::input_iterator_tag) {
  if (d < 0) {
    throw std::out_of_range("Negative distance");
  }
  while(d--) {
    ++iter;
  }
}
```
