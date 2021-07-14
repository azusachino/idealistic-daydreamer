---
title: 初步学习Effective C++
description: 半路出家的，读一遍完全不够啊
date: 2021-07-13
slug: cpp/effective/accustom
image: img/2021/07/miku_-^-.jpg
categories:
  - c++
---

首先: **让自己习惯 C++**

## 1. 视 C++为一个语言联邦

现如今的 C++是个多重泛型编程语言，同时支持过程形式，面向对象形式，函数形式，泛型形式，元编程形式；主要成分如下：

- C: blocks, statements, preprocessor, built-in data types, arrays, pointers
- Object-oriented C++: classes, encapsulation, inheritance, polymorphism, virtual
- Template C++
- STL: containers, iterator, algorithms, function objects

## 2. 尽量以 consts, enums, inlines 替换#defines

`#define`不被视为语言的一部分。

- 对于单纯变量，最好以 const 对象或 enums 对象替换`#define`
- 对于形式函数的宏，最好改用 inline 函数替换`#define`

宏相关的内容，我大都不太了解。。。

## 3. 尽可能使用 const

**const 允许指定一个语义约束，然后编译器会强制实施这项约束。**可以使用 const 在 classes 外部修饰 global 或 namespace 作用域中的常量，或修饰文件、函数、区块作用域中被声明为 static 的对象。

```c++
// 如果出现在*左边，表示被指物是常量；如果出现在*右边，表示指针是常量。
char greeting[] = "Hello";
char* p = greeting;
const char* p = greeting;
char* const p = greeting;
const char* const p = greeting;
```

const 成员函数

```c++
class TextBlock {
public:
    const char& operator[](std::size_t pos) const { return text[pos]; }
    char&       operator[](std::size_t pos) { return text[pos]; }

private:
    std::string text;
};
TextBlock tb("Hello");
std::cout << tb[0]; // call non-const

const TextBlock ctb("World");
std::cout << ctb[0]; // call const
```

## 4. 确定对象被使用前已被初始化

**读取未初始化的值会导致不明确的行为**，而这也是我们写 C++的时候极力想避免的。

```c++
// 不同编译环境下，x不一定保证被初始化(0)
int x;
```

最佳的处理办法是：永远在使用对象之前将它初始化。

```c++
// 内置类型的初始化
int x = 0;
const char* text = "A C-style string";
double d;
std::cin >> d;
```

对于非内置类型以外，初始化的责任由构造函数负责，也就是说需要确保每一个构造函数都将对象的每一个成员初始化。

**C++规定对象的成员变量的初始化动作发生在进入构造函数本体之前。**

```c++
#include <list>
#include <string>

class PhoneNumber {};

class AddressEntry {
public:
    AddressEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones);

private:
    std::string            name;
    std::string            address;
    std::list<PhoneNumber> phones;
    int                    numTimesConsulted;
};

// assignment
AddressEntry::AddressEntry(const std::string& name, const std::string& address,
                           const std::list<PhoneNumber>& phones) {
    this->name              = name;
    this->address           = address;
    this->phones            = phones;
    this->numTimesConsulted = 0;
}

// initialization (member initialization list) 效率更高
AddressEntry::AddressEntry(const std::string& name, const std::string& address,
                           const std::list<PhoneNumber>& phones)
    : name(name), address(address), phones(phones), numTimesConsulted(0) {}
```

**non-local static 对象**: 其寿命从被构造出来知道程序结束为止，不属于 stack 和 head-based 对象。包括 global 对象，定义于 namespace 作用域内的对象，在 classes 内，在函数内，在 file 作用域内被声明为 static 的对象。

**C++对"定义于不同编译单元内的 non-local static 对象"的初始化次序并无明确定义。**

```c++
// 文件系统API定义
class FileSystem {
public:
    std::size_t numDisks() const;
};

extern FileSystem tfs;  // 给外界使用的对象

// 用户使用
class Directory {
public:
    Directory();
};
// 两个源文件中的对象，不确定哪一个先行初始化
Director::Director(){
    std::size_t disks = tfs.numDisks(); // 使用tfs对象
}
```

使用单例模式，解决上述问题。

```c++
// 文件系统API定义
class FileSystem {
public:
    FileSystem& tfs() {
        static FileSystem fs;
        return fs;
    }
    std::size_t numDisks() const;
};

// 用户使用
class Directory {
public:
    Directory();
};
// 两个源文件中的对象，不确定哪一个先行初始化
Director::Director() {
    std::size_t disks = tfs().numDisks(); // 使用tfs对象
}
Directory& tempDir() {
    static Directory dir;
    return td;
}
```
