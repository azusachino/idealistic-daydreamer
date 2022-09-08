---
title: More Effective C++ Basics
description: A new start
date: 2021-09-06
slug: more-effective-cpp/basics
image: img/2021/09/Ruskeala.jpg
categories: [Learning]
tags: [Learning, Cpp]
---

## Distinguish between pointers and references

**没有所谓的 null reference**。一个 `reference` 必须总代表着某个对象。

```c++
char *pc = 0; // null pointer
char& rc = *pc; // reference to null pointer
// 结果不可预期(未定义的行为)

string& rs; // error, non intialized
string s("abcd");
string& rs = s;
```

"没有所谓的 null reference"，这意味着使用 `references` 可能会比使用 `pointers`更有效率。

```c++
void printDouble(const double& d) {
  cout << d;
}

void printDouble(const double* pd) {
  if (pd) { // must check non null
    cout << *pd;
  }
}
```

`pointers` 可以被重新赋值，指向另一个对象； `references` 总是指向最初获得的那个对象。

```c++
string s1("nancy");
string s2("clancy");

string& rs = s1;
string *ps = &s1;
rs = s2; // s1 becomes 'clancy'
ps = &s2; // ps points to s2
```

当我们知道需要指向某个东西，而且绝不会改变其指向到其他东西，或者当实现一个操作符而语法需求无法由 pointers 达成时，选用 references。其他任何时候，采用 pointers。

## Prefer C++ style casts

```c++
(type) expression;
static_cast<type> expression;

int first_number, second_number;
double result = static_cast<double>(first_number)/second_number;

update(SpecialWidget *psw);
SpecialWidget sw;
const SpecialWidget &csw = sw;

// 改变表达式中的常量性
update(const_cast<SpecialWidget*>(csw));

// 用来执行继承体系中"安全的向下转型或跨系转型动作"
update(dynamic_cast<SpecialWidget*>(pw));
updateViaRef(dynamic_cast<SpecialWidget&>(*pw));

// 转换"函数指针类型"，与编译平台息息相关
typedef void (*FuncPtr)();
FuncPtr funcPtrArray[10];
int doSomething();
funcPtrArray[0] = doSometing; // error, wrong type
funcPtrArray[0] = reinterpret_cast<FuncPtr>(&doSomething);
```

## Never treat arrays polymorphically

```c++
class BST {};
class BalancedBST: public BST {};

void printArray(ostream& os, const BST array[], int numElements) {
  for (int i = 0; i < numElements; ++i) {
    s << array[i]; // assume BST have defined operator <<
  }
}

// normal case
BST BSTArray[10];
printArray(cout, BSTArray, 10); // runs well

BalancedBST bBSTArray[10];
printArray(cout, bBSTArray, 10); // error
```

`array[i]` 实际上是"指针算术表达式"的简写，所代表的是`*(array+i)`。 所以`array`和`array+i`之间的距离一定是`i+sizeof(BST)`。我们可以合理预期一个 BalancedBST 对象比一个 BST 对象大，运行上述函数会发生不可预期的结果。

```c++
void deleteArray(ostream& os, BST array[]) {
  os << "Deleting array at address "
     << static_cast<void*>(array) << '\n';
    delete [] array;
}

BalancedBST *bArray = new BalancedBST[50];
deleteArray(cout, bArray);

// 当编译器遇到 delete [] array 时，会产生下述类似的代码
for (int i = nums-1; i >= 0; --i) {
  array[i].BST::~BST(); // 调用array[i]的destcructor
}
```

C++语言规范中规定，通过 base class 指针删除一个由 derived classes objects 构成的数组，其结果未定义。

## Avoid gratuitous default constructors

凡可以"合理地从无到有生成对象"的 classes，都应该内含 default constructor，而"必须有某些外来信息才能生成对象"的 classes，则不必拥有 default constructor。

```c++
class EquipmentPiece {
public:
  EquipmentPiece(int ID);
};

EquipmentPiece bestPieces[10]; // error, can't call EquipmentPiece ctors
EquipmentPiece *bestPieces = new EquipmentPiece[10];

// 使用 non-heap数组
int id1, id2, id3;
EquipmentPiece bestPieces[3] = {
  EquipmentPiece(id1),
  EquipmentPiece(id2),
  EquipmentPiece(id3)
};

// 指针数组
typedef EquipmentPiece* PEP;
PEP bestPieces[10]; // ok, no need to call ctor
PEP *bestPieces = new PEP[10]; // ok
for (int i = 0; i < 10; ++i) {
  bestPieces[i] = new EquipmentPiece(i);
}
// 1. 必须将此数组所指的所有对象删除
// 2. 需要额外的内存空间用于存储指针
```

先为此数组分配 raw memory，然后使用 placement new，可以避免"过度使用内存"。

```c++
void *rawMemory = operator new[](10 * sizeof(EquipmentPiece));

EquipmentPiece *bestPieces = static_cast<EquipmentPiece>(rawMemory);

for(int i = 0; i < 10; ++i) {
  new (&bestPieces[i]) EquipmentPiece(id);
}

// 手动调用 destructors 和 operator delete []
for (int i = 9; i >= 0; --i) {
  // 以构造顺序的相反顺序析构
  bestPieces[i].~EquipmentPiece();
}
operator delete[] (rawMemory);
```

如果 classes 缺少 default constructors，它们将不适用于许多 template-based container classes。对那些 templates 而言，被实例化的"目标类型"必须得有一个 default constructor。
