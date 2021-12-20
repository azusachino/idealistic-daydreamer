---
title: 继承与面向对象设计
description: Is-a & Has-a
date: 2021-09-02
slug: effective-cpp/oop
image: img/2021/09/DjurdjevicaBridge.jpg
categories:
  - learning
tags:
  - c++
---

## Make sure public inheritance models "is-a"

public inheritance 意味着 "is-a" 的关系。

```c++
class Bird {};

class FlyingBird : public Bird {
public:
    virtual void fly();
};

// Penguin is a bird, but cannot fly
class Penguin: public Bird {};
```

代码通过编译并不代表就可以正确运作

- "public 继承" 意味着 "is-a"。适用于 base classes 身上的每一件事情一定也适用于 derived classes 身上，因为每个 derived class 对象都是一个 base class 对象。

## Avoid hiding inherited names

内层作用域的名称会遮掩外围作用域的名称。

```c++
// a normal case
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf2();
    void mf3();
};

class Derived: public Base {
public:
    virtual void mf1();
    void mf4();
}

void Derived::mf4() {
    mf2(); // find the function in Base class
}
```

作用域的查询路径: Derived class -> Base class -> Base class's namespace -> global

```c++
// override case
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    virtual void mf2();
    void mf3();
    void mf3(double);
};

class Derived: public Base {
public:
    virtual void mf1();
    void mf3();
    void mf4();
}

Derived d;
int x;

d.mf1(); // ok, call Derived::mf1()
d.mf1(x); // wrong, Derived::mf1() override Base::mf1()
d.mf2(); // ok, Base::mf2()
d.mf3(); // ok, Derived::mf3()
d.mf3(x); // wrong, Derived::mf3() override Base::mf3()
```

- derived classes 内的名称会遮掩 base classes 内的名称
- 可以使用 using 声明式或 forwarding 函数

## Differentiate between inheritance of interface and inheritance of implementation

- 成员函数的接口总会被继承
  - pure virtual 函数：必须被任何"继承了它们"的具象 class 重新声明，它们在抽象 class 中通常没有定义。
- 声明一个 pure virtual 函数的目的是为了让 derived classes 只继承函数接口
- 声明简朴的 impure virtual 函数的目的是让 derived classes 继承函数的接口和缺省实现
- 声明 non-virtual 函数的目的是为了令 derived classes 继承函数的接口及一份强制性实现
  - non-virtual 函数代表的意义是不变性凌驾特异性，绝不该在 derived classes 中被重新定义

```c++
class Shape {
public:
    virtual void draw() const = 0;
    virtual void error(const std::string& name);
    int ObjectID() const;
};

class Rectangle: public Shape{};
class Circle: public Shape{};

Shape* ps = new Shape; // wrong, Shape is abstract
Shape* ps1 = new Rectangle; // OK
ps1->draw(); // call Rectagle::draw
Shape* ps2 = new Circle;
ps2->draw(); // call Circle::draw
ps1->Shape::draw(); // call Shape::draw
ps2->Shape::draw(); // call Shape::draw
```

- 接口继承和实现继承不同。在 public 继承下，derived classes 总是继承 base classes 的接口
- pure virtual 函数只具有指定接口继承
- 简朴的 impure virtual 函数具体指定接口函数继承及缺省实现继承
- non-virtual 函数具体指定接口继承及强制性实现继承

## Consider alternatives to virtual functions

基于 Non-Virtual Interface 手法实现 `Template Method` 模式

```c++
class GameCharacter {
public:
    int healthValue() const {
        int retVal = doHealthValue();

        return retVal;
    }
private:
    virtual int doHealthValue() const {
        ...
    }
};
```

基于 Function Pointers 实现 `Strategy` 模式

```c++
class GameCharacter; // forward declaration

int defaultHealthCalc(const GameCharacter& gc);

// Strategy
class GameCharacter {
public:
    typedef int (*HealthCalcFunc) (const GameCharacter&);

    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc) : healthFunc(hcf) {}

    int healthValue() {
        return healthFunc(*this);
    }
private:
    HealthCalcFunc healthFunc;
};

class EvilBadGuy : public GameCharacter {
public:
    explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc) : GameCharacter(hcf) {}
};

int loseHealthQuickly(const GameCharacter&);
int loseHealthSlowly(const GameCharacter&);

// 同一人物类型可以有不同的健康计算函数
EvilBadGuy softEnemy(loseHealthQuickly);
EvilBadGuy toughEnemy(loseHealthSlowly);
```

## Nerver redefine an inherited non-virtual function

- non-virtual 函数是静态绑定
- virtual 函数是动态绑定

## Nerver redefine a function's inherited default parameter value

```c++
class Shape {
public:
    enum ShapeColor {Red, Green, Blue};
    virtual void draw(ShapeColor color = Red) const = 0;
};

class Rectangle : public Shape {
public:
    // apply with different default parameter
    virtual void draw(ShapeColor color = Green) const;
};

class Circle : public Shape {
public:
    virtual void draw(ShapeColor color) const;
    // 静态绑定下，该函数不从其base class继承缺省参数值
    // 若以指针(或 reference)调用该函数，可以不指定参数值。动态绑定会继承缺省值
};

Shape* ps; // static Shape
Shape* pc = new Circle; // static Shape
Shape* pr = new Rectangle; // static Shape

ps = pc; // ps dynamic Circle
ps = pr; // ps dynamic Rectangle

pc->draw(Shape::Red); // call Circle::draw(Shape::Red)
pr->draw(Shape::Red); // call Rectangle::draw(Shape::Red)

pr->draw();  // call Rectangle::draw(Shape::Red);

// 使用NVI手法: 令base class内的一个public non-virtual函数调用private virtual函数
class Shape {
public:
    enum ShapeColor {Red, Green, Blue};
    // non-virtual, must not override
    void draw(ShapeColor color = Red) {
        doDraw(color);
    }
private:
    virtual void doDraw(ShapeColor color) const = 0;
};
```

## Model "has-a" or "is-implemented-in-terms-of" through composition

```c++
class Address {};
class PhoneNumber {};

// compostion class
class Person {
public:
    virtual std::string some() const = 0;
private:
    std::string name;
    Address addr;
    PhoneNumber phone;
};
```

## Use private inheritance judiciously

如果 classes 之间的继承关系是 private，编译器不会自动将一个 derived class 对象转换为一个 base class 对象

```c++
class Timer {
public:
    explicit Timer(int tickFreq);
    virtual void onTick() const;
};

// Widget is not a timer, but can onTick
class Widget : private Timer {
private:
    virtual void onTick() const;
};

// use composition
class Widget {
private:
    class WidgetTimer: public Timer {
    public:
        virtual void onTick() const;
    };
    WidgetTimer timer;
};
```

## Use multiple inheritance judiciously

- 多重继承比单一继承复杂
- virtual 继承会增加大小、速度、初始化(及赋值)复杂度等成本。如果 virtual base classes 不带任何数据，将是最具实用价值的情况
- 多重继承的正确用途
  - public 继承某个 interface class
  - private 继承某个协助实现的 class
