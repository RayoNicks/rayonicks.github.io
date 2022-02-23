---
layout: post
title: 《C++ Templates》第18章 模板和多态
categories: Reading
tags: C++
---

# 18 模板和多态

## 18.1 动态多态

动态多态是使用同一种方法在运行时调用不同的代码，书中给出的是最为熟知的画图形的例子：

```cpp
// poly/dynahier.hpp
#include "coord.hpp"
// common abstract base class GeoObj for geometric objects
class GeoObj {
    public:
        // draw geometric object:
        virtual void draw() const = 0;
        // return center of gravity of geometric object:
        virtual Coord center_of_gravity() const = 0;
        // ...
        virtual ~GeoObj() = default;
};\

// concrete geometric object class Circle
// - derived from GeoObj
class Circle : public GeoObj {
    public:
        virtual void draw() const override;
        virtual Coord center_of_gravity() const override;
        // ...
};

// concrete geometric object class Line
// - derived from GeoObj
class Line : public GeoObj {
    public:
        virtual void draw() const override;
        virtual Coord center_of_gravity() const override;
        // ...
};
// ...
```

```cpp
// poly/dynapoly.cpp
#include "dynahier.hpp"
#include <vector>

// draw any GeoObj
void myDraw (GeoObj const& obj)
{
    obj.draw(); // call draw() according to type of object
}

// compute distance of center of gravity between two GeoObjs
Coord distance (GeoObj const& x1, GeoObj const& x2)
{
    Coord c = x1.center_of_gravity() - x2.center_of_gravity();
    return c.abs(); // return coordinates as absolute values
}

// draw heterogeneous collection of GeoObjs
void drawElems (std::vector<GeoObj*> const& elems)
{
    for (std::size_type i=0; i<elems.size(); ++i) {
        elems[i]->draw(); // call draw() according to type of element
    }
}

int main()
{
    Line l;
    Circle c, c1, c2;

    myDraw(l);  // myDraw(GeoObj&) => Line::draw()
    myDraw(c);  // myDraw(GeoObj&) => Circle::draw()

    distance(c1,c2);    // distance(GeoObj&,GeoObj&)
    distance(l,c);      // distance(GeoObj&,GeoObj&)

    std::vector<GeoObj*> coll;  // heterogeneous collection
    coll.push_back(&l);         // insert line
    coll.push_back(&c);         // insert circle
    drawElems(coll);            // draw different kinds of GeoObjs
    }
```

## 18.2 静态多态

使用模板可以得到上例的静态多态版本：

```cpp
// poly/statichier.hpp
#include "coord.hpp"

// concrete geometric object class Circle
// - not derived from any class
class Circle {
    public:
        void draw() const;
        Coord center_of_gravity() const;
        // ...
};

// concrete geometric object class Line
// - not derived from any class
class Line {
    public:
        void draw() const;
        Coord center_of_gravity() const;
        // ...
};
// ...
```

```cpp
// poly/staticpoly.cpp
#include "statichier.hpp"
#include <vector>

// draw any GeoObj
template<typename GeoObj>
void myDraw (GeoObj const& obj)
{
    obj.draw(); // call draw() according to type of object
}

// compute distance of center of gravity between two GeoObjs
template<typename GeoObj1, typename GeoObj2>
Coord distance (GeoObj1 const& x1, GeoObj2 const& x2)
{
    Coord c = x1.center_of_gravity() - x2.center_of_gravity();
    return c.abs(); // return coordinates as absolute values
}

// draw homogeneous collection of GeoObjs
template<typename GeoObj>
void drawElems (std::vector<GeoObj> const& elems)
{
    for (unsigned i=0; i<elems.size(); ++i) {
        elems[i].draw(); // call draw() according to type of element
    }
}

int main()
{
    Line l;
    Circle c, c1, c2;

    myDraw(l);  // myDraw<Line>(GeoObj&) => Line::draw()
    myDraw(c);  // myDraw<Circle>(GeoObj&) => Circle::draw()

    distance(c1,c2);    // distance<Circle,Circle>(GeoObj1&,GeoObj2&)
    distance(l,c);      // distance<Line,Circle>(GeoObj1&,GeoObj2&)

    // std::vector<GeoObj*> coll;   // ERROR: no heterogeneous collection possible
    std::vector<Line> coll;         // OK: homogeneous collection possible
    coll.push_back(l);              // insert line
    drawElems(coll);                // draw all lines
}
```

## 18.3 动态多态和静态多态的对比

- 术语
  - 通过继承实现的多态是绑定的（bounded）和动态的（dynamic）
    - 绑定是指参与多态的类型的接口由基类固定
    - 动态是指在运行时通过函数指针调用正确的函数
  - 通过模板实现的多态是未绑定的（unbounded）和静态的（static）
    - 未绑定是指参与多态的类型的接口是不固定的
    - 静态是指在编译时由编译器解析要调用的函数

- 区别
  - 动态多态
    - 处理派生类集合的代码很优雅
    - 代码体积小
    - 代码可以被编译为二进制，源码可以不公开
  - 静态多态
    - 内置集合类型的处理很方便，但是类类型的通用接口不能通过统一的接口定义
    - 代码运行速度会快一些
    - 没有完整实现接口的类型也可以参与多态

## 18.4 Concept

Concept是指模板实参类型所需要支持的接口操作：

```cpp
// poly/conceptsreq.hpp
#include "coord.hpp"

template<typename T>
concept GeoObj = requires(T x) {
    { x.draw() } -> void;
    { x.center_of_gravity() } -> Coord;
    // ...
};
```

```cpp
// poly/conceptspoly.hpp
#include "conceptsreq.hpp"
#include <vector>

// draw any GeoObj
template<typename T>
requires GeoObj<T>
void myDraw (T const& obj)
{
    obj.draw(); // call draw() according to type of object
}

// compute distance of center of gravity between two GeoObjs
template<typename T1, typename T2>
requires GeoObj<T1> && GeoObj<T2>
Coord distance (T1 const& x1, T2 const& x2)
{
    Coord c = x1.center_of_gravity() - x2.center_of_gravity();
    return c.abs(); // return coordinates as absolute values
}

// draw homogeneous collection of GeoObjs
template<typename T>
requires GeoObj<T>
void drawElems (std::vector<T> const& elems)
{
    for (std::size_type i=0; i<elems.size(); ++i) {
        elems[i].draw(); // call draw() according to type of element
    }
}
```

## 18.5 静态多态下的设计模式

在传统的设计模式中，桥接模式（bridge pattern）将接口与实现分离开。抽象类中包含一个实现类，抽象类通过该实现类提供具体的功能。可以简单理解为抽象类提供了一个功能，但是这个功能需要分若干步骤实现，实现类接口定义了每个步骤。

在动态多态下，抽象类中包含的是实现类的指针，而在静态多态下，抽象类中包含的是实现类的对象。

## 18.6 泛型编程

泛型编程是指用泛化的参数来为算法提供抽象的表达。原文：

>Programming with generic parameters to finding the most abstract representation of efficient algorithms.

在C++中，泛型编程基本等价于模板编程，就像面向对象编程基本等价于使用虚函数一样。C++标准模板库（Standard Template Library，STL）是泛型编程的典型代表。STL中定义了算法和容器，容器是类，但是算法并不是容器类的成员函数，所以它们可以被应用于各种容器，并且用迭代器的类型进行了限制，这也是concept的一种体现。

```cpp
// poly/printmax.cpp
#include <vector>
#include <list>
#include <algorithm>
#include <iostream>
#include "MyClass.hpp"

template<typename T>
void printMax (T const& coll)
{
    // compute position of maximum value
    auto pos = std::max_element(coll.begin(),coll.end());

    // print value of maximum element of coll (if any):
    if (pos != coll.end()) {
        std::cout << *pos << '\n';
    }
    else {
        std::cout << "empty" << '\n';
    }
}

int main()
{
    std::vector<MyClass> c1;
    std::list<MyClass> c2;
    // ...
    printMax(c1);
    printMax(c2);
}
```

## 18.7 后记
