---
title: 资源管理
description: 小心对待一切资源
date: 2021-08-17
slug: effective-cpp/resource-management
image: img/2021/08/FlintstoneHouse.jpg
categories: [Learning]
tags: [Learning, Cpp]
---

## Use object to manage resources

1. 获得资源后立刻放入管理对象
2. 管理对象运用析构函数确保资源被释放

```c++
void f() {
    Investment* pInv = createInvestment();
    // other operations

    // 可能因为一系列原因导致delete无法执行
    delete pInv; // release pointer
}

// manage resource by auto_ptr
void f2() {
    std::auto_ptr<Investment> pInv(createInvestment());
    // auto_ptr will automatically deconstruct and delete pInv
}
```

由于`auto_ptr`销毁时会自动删除所指之物，所以一定不能让多个`auto_ptr`指向同一对象。

> 若通过复制构造函数或复制操作符复制`auto_ptr`，它们会变成 null，而复制所得的指针将取得资源的唯一拥有权

```c++
std::auto_ptr<Investment> pInv1(createInvestment());
std::auto_ptr<Investment> pInv2(pInv1); // pInv1 becomes null
pInv1 = pInv2; // pInv2 becomes null
```

`auto_ptr`的替代方案是"引用计数型智慧指针"

```c++
void f() {
    std::tr1::shared_ptr<Investment> pInv1(createInvestment());
    std::tr1::shared_ptr<Investment> pInv2(pInv1);
    pInv1 = pInv2;
}
```

> `auto_ptr` & `tr1::shared_ptr`两者都在其析构函数中做`delete`而不是`delete[]`操作。

## Think carefully about copying behavior in resource-managing classes

1. 禁止复制
2. 对底层资源使用"引用计数法" `shared_ptr`
3. 复制底部资源
4. 转移底部资源的拥有权 `auto_ptr`

## Provide access to raw resources in resource-managing classes

```c++
int daysHeld(const Investment *pi);

int days = daysHeld(pInv.get()); // return raw resource
```

提供显式转换函数

```c++
FontHandle getFont();
void releaseFont(FontHandle fh);

class Font {
public:
    explicit Font(FontHandle fh) : f(fh) {
    }
    ~Font() {
        releaseFont(f);
    }
private:
    FontHandle f;
}
```

提供隐式转换函数

```c++
class Font {
public:
    operator FontHandle() const {
        return f;
    }
private:
    FontHandle f;
}
```

## Use the same form in corresponding uses of new and delete

```c++
std::string* stringPtr1 = new std::string;
std::string* stringPtr2 = new std::string[100];

delete stringPtr1;
delete [] stringPtr2;
```

## Store new objects in smart pointer in standalone statements

```c++
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);

processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
// 1. 执行 "new Widget"
// 2. 调用 priority => 如果抛出异常，new Widget返回的指针将会遗失
// 3. 调用 tr1::shared_ptr 构造函数

std::tr1::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```
