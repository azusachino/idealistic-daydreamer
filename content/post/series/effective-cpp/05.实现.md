---
title: 实现
description: 好的代码，效率至上
date: 2021-08-25
slug: effective-cpp/implementations
image: img/2021/08/HowgillFells.jpg
categories: [Learning]
tags: [Learning, Cpp]
---

## Postpone variable definitions as long as possible

只要定义了一个变量而其类型带有一个构造函数或析构函数，当程序的控制流到达这个变量定义式时，便需要承受构造成本；当这个变量离开作用域时，便需要承受析构成本。

```c++
std::string encryptPassword(const std::string password) {
    using namespace std;
    string encryptedPassword;
    // 如果程序直接异常退出，浪费了一次string的构造和析构
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password too short!!!");
    }
    // do the encryption
    return encryptedPassword;
}

// 另一种浪费形式 多执行了一次构造
std::string encryptPassword(const std::string& password) {
    using namespace std;
    string encryptedPassword; // default construct
    encryptedPassword = password; // copy
    encrypt(encryptedPassword);
    return encryptedPassword;
}

// best practice
std::string encryptPassword(const std::string& password) {
    using namespace std;
    // check length
    string encryptedPassword(password); // copy construct
    encrypt(encryptedPassword);
    return encryptedPassword;
}
```

不只应该延后变量的定义，直到非得使用该变量的前一刻为止，甚至应该尝试延后这份定义知道能够给它初值实参为止。

## Minimize casting

- `(T)expression` & `T(expression)`
- `const_cast` 通常被用来将对象的常量性移除，这也是唯一有此能力的 c++ style 转型操作符
- `dynamic_cast` 主要用来执行"安全向下转型"，唯一无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作
- `reinterpret_cast` 意图执行低级转型，实际动作(及结果)可能取决于编译器(不可移植)
- `static_cast` 用于强制隐式转换

```c++
class Window {
public:
    virtual void onResize() {}
};

class SpecialWindow : public Window {
public:
    virtual void onResize() {
        static_cast<Window>(*this).onResize();
        // use the origin one => Window::onResize();
    }
};
```

将`*this`转型为 `Window`，对函数 `onResize` 的调用也因此调用了`Window::onResize`；但它调用的并不是当前对象上的函数，而是稍早转型动作所建立的一个"`*this` 对象之 base 成分"的临时副本上的 `onResize`。

- 尽量避免转型，特别是在注重效率的代码中避免`dynamic_casts`
- 如果转型是必要的，试着将它隐藏于某个函数背后
- 优先实现 C++ style 转型

## Avoid returning "handles" to object intervals

```c++
struct RectData {
    Point ulhc; // upper left-hand corner
    Point lrhc; // lower right-hand corner
};

class Rectangle {
public:
    // !! 调用者可以通过 reference 更改内部数据，会带来不可预期的后果
    Point& upperLeft() const {
        return pData->ulhc;
    }
    Point& lowerRight() const {
        return pData->lrhc;
    }
    // better usage => 添加 const 保证数据是不可变的
    const Point& getPoint() const {
        return pData->ulhc;
    }
private:
    std::tr1::shared_ptr<RectData> pData;
};
```

## Strive for exception-safe code

**异常安全性**:

1. 不泄漏任何资源
2. 不允许数据破坏

**异常安全函数**:

1. 基本承诺 => 如果异常被抛出，程序内的任何事物仍然保持在有效状态下
2. 强烈保证 => 如果异常被抛出，程序状态不改变
3. 不抛掷保证 => 承诺绝不抛出异常

```c++
class PrettyMenu {
public:
    void changeBackground(std::istream& imgSrc);
private:
    Mutex mutex;
    Image* image;
    int imageChanges;
};

// 原始版本
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    lock(&mutex);
    delete image;
    ++imageChanges;
    image = new Image(imgSrc); // 如果构造函数抛出异常的话
    unlock(&mutex);
}

// 使用资源管理器
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    Lock ml(&mutex); // 获得mutex并保证后续释放
    delete image;
    ++imageChanges;
    image = new Image(imgSrc);
}

// 智能指针版本
class PrettyMenu {
    ...
    std::tr1::shared_ptr<Image> image;
}

void PrettyMenu::changeBackground(std::istream& imgSrc) {
    Lock ml(&mutex);
    image.reset(new Image(imgSrc));
    ++imageChanges;
}
```

符合 PIMPL IDIOM 的版本：将所有"隶属于对象的数据"从原对象放进另一个对象中，然后赋予原对象一个指针，指向那个实现对象(implementation object)

```c++
// 副本实现类
struct PMImpl {
    std::tr1::shared_ptr<Image> image;
    int imageChanges;
};

class PrettyMenu {
...
private:
    Mutex mutex;
    std::tr1::shared_ptr<PMIpml> pImpl;
};

// Copy & Swap
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    using std::swap;
    Lock ml(&mutex);
    std::tr1::shared_ptr<PMImpl> pNew(new PMImpl(*pImpl));
    pNew->image.reset(new Image(imgSrc));
    ++pNew->imageChanges;
    swap(pImpl, pNew); // 置换数据，释放mutex
}
```

## Understand the ins and outs of inlining

inline 函数背后的整体观点是，将"对此函数的每一次调用"都用函数本体替换之。 => 可能会导致目标码的大小。

inline 造成的代码膨胀会导致额外的换页行为，降低高速缓存装置的命中率，以及伴随而来的效率损失。

```c++
// 隐式声明inline
class Person {
public:
    int age() {return theAge;} // inline
private:
    int theAge;
};
```

- virtual => 等待，直到运行期在确定调用哪个函数
- inline => 执行前，先将调用动作替换为被调用函数的本体

```c++
inline void f() {}
void (*pf)() = f;

f(); // inlined
pf(); // may not be inlined, as for it's a function pointer
```

- 将大多数 inlining 限制在小型、被频繁调用的函数身上。这可以使后续的调试过程和二进制升级更加容易，也可以使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。
- 不要只因为 function templates 出现在头文件，就将它们声明为 inline

## Minimize compilation dependencies between files

- 如果使用 object reference 或 object pointer 可以完成任务，就不要使用 object
- 如果能够，尽量以 class 声明式替换 class 定义式
- 为声明式和定义式提供不同的头文件

```c++
#include <string>
#include <memory>

class PersonImpl;
class Date;
class Address;

class Person {
public:
    Person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthday() const;
    std::string address() const;

private:
    std::tr1::shared_ptr<PersonImpl> pImpl;
};
```

```c++
#include "Person.h"
#include "PersonImpl.h"

Person::Person(const std::string& name, const Date& birthday, const Address& addr) :
pImpl(new PersonImpl(name, birthday, addr)) {}

std::string Person::name() const {
    return pImpl->name();
}
```

越来越看不明白了，还是没有好好使用 C++，光有理论，毫无意义啊。
