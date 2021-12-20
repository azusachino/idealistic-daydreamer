---
title: 构造/析构/赋值
description: 一切皆有代价
date: 2021-07-27
slug: effective-cpp/construct-destruct-assign
image: img/2021/07/GCVenice.jpg
categories:
  - learning
tags:
  - c++
---

## 了解 C++默默编写并调用哪些函数

在 C++处理完一个 Empty Class 之后，会为它声明(编译器版本的)一个 copy constructor，一个 copy assignment operator 和一个析构函数。如果 Empty Class 中没有声明任何构造函数，编译器还会声明一个 default 构造函数。

```c++
class Empty {};
// equivalent class
class Empty {
public:
    Empty() {}
    Empty(const Empty& rhs) {}
    ~Empty() {}

    Empty& operator=(const Empty& rhs) {}
};
```

```c++
template<class T>
class NamedObject {
public:
    NamedObject(std::string& name, const T& value);
private:
    std::string& nameValue;
    const T objValue;
};

std::string newDog("abc");
std::string oldDog("ccc");
NamedObject<int> p(newDog, 2);
NamedObject<int> s(oldDog, 18);

// C++ 不允许 "让reference改指向不同对象"
p = s; // ? => c== will refuse to compile this line
```

## 明确拒绝不想使用的编译器自动生成的函数

为驳回编译器自动提供的技能，可将相应的成员函数声明为 private 并且不予实现。

```c++
class Uncopyable {
protected:
    Uncopyable() {}
    ~Uncopyable() {}
private:
    Uncopyable(const Uncopyable&);
    Uncopyable& operator=(const Uncopyable&);
}
```

## 为多态基类声明 virtual 析构函数

C++明确指出，当 derived class 对象经由一个 base class 指针被删除，而该 base class 带着一个 non virtual 析构函数，其结果未定义。

```c++
class TimeKeeper {
public:
    TimeKeeper();
    ~TimeKeeper();
    // virtual ~TimeKeeper(); // will do the magic
};

class AtomicClock: public TimeKeeper {};
class AtomicClock: public TimeKeeper {};
class AtomicClock: public TimeKeeper {};

TimeKeeper* getTimeKeeper(); // return a pointer, points to derived object
// 如果 ptk 是 AtomicClock，其中的 AtomicClock 成分可能没被销毁，AtomicClock 的析构函数也未被执行；而 base class 成分通常会被销毁。
TimeKeeper* ptk = getTimeKeeper();
delete ptk;
```

只有当 class 内含至少一个 virtual 函数，才为它声明 virtual 析构函数。

## 别让异常逃离析构函数

C++并不禁止析构函数抛出异常，但是它不鼓励你这样做。

```c++
class Widget {
public:
    ...
    ~Widget() {...} // probably throws a exception
};

void doSomething() {
    std::vector<Widget> v;
    // 在两个异常同时存在的情况下，程序若不是结束执行就是导致不明确行为
    ... // v destroys here
}
```

```c++
class DBConn
{
public:
    void close()
    {
        db.close();
        closed = true;
    }
    ~DBConn()
    {
        if (!closed)
        {
            try
            {
                db.close();
            }
            catch (std::exception &e)
            {
                // ignored exception, do some record
            }
        }
    }

private:
    DBConnection db;
    bool closed;
};
```

## 绝不在构造和析构的过程中调用 virtual 函数

base class 构造期间 virtual 函数绝不会下降到 derived classes 阶层。**在 base class 构造期间，virtual 函数不是 virtual 函数**

```c++
class Transaction
{
public:
    Transaction();
    virtual void logT() const = 0;
};

Transaction::Transaction()
{
    logT(); // call virtual logT in constructor
}

// base class的构造函数执行derived class构造函数
class BuyTransaction : public Transaction
{
public:
    virtual void logT() const;
};

class SellTransaction : public Transaction
{
public:
    virtual void logT() const;
};

BuyTransaction b; // 要求使用对象内部尚未初始化的成分
```

## 让`operator=`返回一个`reference to *this`

为了实现"连锁赋值"，赋值操作符必须返回一个 reference 指向操作符的左侧实参。

```c++
class Widget {
public:
    Widget& operator=(const Widget& rhs) {
        ...
        return *this;
    }
};
```

## 在`operator=`中处理自我赋值

```c++
class Widget {
private:
    Bitmap* pb;
};

// 如果自我赋值，会导致自己持有一个指针指向一个已删除的对象
Widget& Widget::operator=(const Widget& rhs) {
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}

// 利用CAS
Widget& Widget::operator=(const Widget& rhs) {
    Widget temp(rhs); // copy
    swap(temp); // swap temp and this
    return *this;
}
```

## 复制对象时勿忘其每一个成分

- Copying 函数应该确保复制"对象内的所有成员变量"及"所有 base class 成分"
- 不要尝试以某个 copying 函数实现另一个 copying 函数。应该将共同机能放进第三个函数中，并由两个 copying 函数共同调用。

```c++
class Customer {
public:
    Customer(const Customer& rhs);
    Customer& operator=(const Customer& rhs);
private:
    std::string name;
};

Customer::Customer(const Customer& rhs): name(rhs.name) {
    logCall("Customer copy constructor");
}

Customer& Customer::operator=(const Customer& rhs) {
    logCall("Customer copy assignment operator");
    name = rhs.name;
    return *this;
}

class PriorityCustomer : public Customer {
public:
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
private:
    int priority;
};

// no default constructor for class Customer if not adding Customer(rhs)
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs): priority(rhs.priority), Customer(rhs) {
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs) {
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs); // assign base class
    priority = rhs.priority;
    return *this;
}

```
