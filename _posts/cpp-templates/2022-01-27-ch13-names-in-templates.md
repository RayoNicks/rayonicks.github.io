---
layout: post
title: 《C++ Templates》第13章 模板中的名称解析
categories: Reading
tags: C++
---

# 13 模板中的名称解析

## 13.1 C++名称分类

C++中有两种名称分类的方式：

1. 受限名称和非受限名称：如果一个名称包含作用域运算符`::`或者成员访问运算符`.`和`->`，则就是受限名称
2. 依赖型名称和非依赖型名称：如果一个名称中包含模板参数，则就是依赖型名称

## 13.2 名称查找

受限名称会在指定的作用域中进行查找：

```cpp
int x;

class B {
    public:
        int i;
};

class D : public B {
};

void f(D* pd)
{
    pd->i = 3;  // finds B::i
    D::x = 2;   // ERROR: does not find ::x in the enclosing scope
}
```

非受限名称会在最近的作用域中进行查找：

```cpp
extern int count;               // #1

int lookup_example(int count)   // #2
{
    if (count < 0) {
        int count = 1;          // #3
        lookup_example(count);  // unqualified count refers to #3
    }
    return count + ::count;     // the first (unqualified) count refers to #2 ;
}                               // the second (qualified) count refers to #1
```

### 13.2.1 依赖型名称查找

只应用上述规则会出现名称无法查找的情况，例如下面的例子：

```cpp
template<typename T>
T max (T a, T b)
{
    return b < a ? a : b;
}

namespace BigMath {
    class BigNumber {
        // ...
    };
    bool operator < (BigNumber const&, BigNumber const&);
    // ...
}

using BigMath::BigNumber;

void g (BigNumber const& a, BigNumber const& b)
{
    // ...
    BigNumber x = ::max(a,b);
    // ...
}
```

因为`<`运算符不是`BigNumber`的成员函数，而是定义在命名空间`BigMath`中的函数，所以`max()`无法解析它，这就需要应用依赖于实参的名称查找（Argument-Dependent Lookup，ADL）规则。当一个非受限名称可能是非成员函数或者运算符时，编译器会在关联的命名空间和类中尝试查找该名称，就像是这个非受限名称被这些命名空间和类按顺序修饰变成了受限名称一样：

```cpp
// details/adl.cpp
#include <iostream>

namespace X {
    template<typename T> void f(T);
}

namespace N {
    using namespace X;
    enum E { e1 };
    void f(E) {
        std::cout << "N::f(N::E) called\n";
    }
}

void f(int)
{
    std::cout << "::f(int) called\n";
}

int main()
{
    ::f(N::e1);     // qualified function name: no ADL
    f(N::e1);       // ordinary lookup finds ::f() and ADL finds N::f(),
}                   // the latter is preferred
```

### 13.2.2 依赖型友元查找

友元函数声明可以是该函数的第一次声明：

```cpp
template<typename T>
class C {
    // ...
    friend void f();
    friend void f(C<T> const&);
    // ...
};

void g (C<int>* p)
{
    f();    // is f() visible here?
    f(*p);  // is f(C<int> const&) visible here?
}
```

在上面的例子中：

1. 编译器无法查找`f()`，因为函数没有参数，也没有返回值
2. 编译器可以查找`f(*p)`，因为关联的类是`C<int>`，关联的命名空间是全局命名空间

### 13.3.3 隐式注入的类名称

类的名称以非受限的形式默认注入到类作用域中：

```cpp
// details/inject.cpp
#include <iostream>

int C;

class C {
    private:
        int i[2];
    public:
        static int f() {
            return sizeof(C);
        }
};

int f()
{
    return sizeof(C);
}

int main()
{
    std::cout << "C::f() = " << C::f() << ','
                << " ::f() = " << ::f() << '\n';
}
```

对于类模板，同样会将类名称默认注入到作用域中：

```cpp
template<template<typename> class TT> class X {
};

template<typename T> class C {
    C* a;           // OK: same as “C<T>* a;”
    C<void>& b;     // OK
    X<C> c;         // OK: C without a template argument list denotes the template C
    X<::C> d;       // OK: ::C is not the injected class name and therefore always denotes the template
};
```

### 13.2.4 类模板名称的注入

对于普通类，默认注入的就是类名称，但是对于类模板，每次实例化都会产生不同的类型，所以类模板名称分为当前实例化类型（current instantiation）和未知特化类型（unknown specialization）：

```cpp
template<typename T> class Node {
    using Type = T;
    Node* next;             // Node refers to a current instantiation
    Node<Type>* previous;   // Node<Type> refers to a current instantiation
    Node<T*>* parent;       // Node<T*> refers to an unknown specialization
};
```

当类模板中存在嵌入类时，二者更加难以区分：

```cpp
template<typename T> class C {
    using Type = T;

    struct I {
        C* c;           // C refers to a current instantiation
        C<Type>* c2;    // C<Type> refers to a current instantiation
        I* i;           // I refers to a current instantiation
    };

    struct J {
        C* c;           // C refers to a current instantiation
        C<Type>* c2;    // C<Type> refers to a current instantiation
        I* i;           // I refers to an unknown specialization, because I does not enclose J
        J* j;           // J refers to a current instantiation
    };
};
```

这里的规则就是如果所引用的名称可以被显示的特化，则该名称就是未知特化类型。原文：

>This has implications for name lookup when parsing templates, but it also leads to an alternative, more game-like way to determine whether a type X within the definition of a class template refers to a current instantiation or an unknown specialization: If another programmer can write an explicit specialization such that X refers to that specialization, then X refers to an unknown specialization.

## 13.3 解析类模板

### 13.3.1 非模板中的上下文相关文法

C++的复杂之处在于源程序的解析是上下文相关的，例如在C++11之前，`>>`总是会被解释为右移运算符，除非在其中加一个空格：

```cpp
// names/anglebrackethack.cpp
#include <iostream>

template<int I> struct X {
    static int const c = 2;
};

template<> struct X<0> {
    typedef int c;
};

template<typename T> struct Y {
    static int const c = 3;
};

static int const c = 4;

int main()
{
    std::cout << (Y<X<1> >::c >::c>::c) << ' ';
    std::cout << (Y<X< 1>>::c >::c>::c) << '\n';
}
```

在C++11之前，第一行会被解释为取出`Y::c`并且和`::c`进行两次的比较，最终输出`0`；第二行会被解释为`Y<X<16>::c>::c`，最终输出`3`。不过从C++11开始，两行的输出都为`0`。

### 13.3.2 依赖型类型名称

当类型名称中包含模板参数时，为了避免歧义，需要使用`typename`显示指出后面的是类型名称。必须要使用`typename`作为前导的情况要满足下面的四个条件：

1. 名称中包含`::`运算符
2. 类型名称没有被`struct`、`class`、`union`和`enum`修饰
3. 不是用来表示基类和表示初始化列表的情况
4. 名称中包含模板参数
5. 该类型名称是未知特化类型

### 13.3.3 依赖型模板名称

有时需要用`template`关键字显示指出后面的名称是模板：

```cpp
template<typename T>
class Shell {
    public:
        template<int N>
        class In {
            public:
                template<int M>
                class Deep {
                    public:
                        virtual void f();
                };
        };
};

template<typename T, int N>
class Weird {
    public:
        void case1 (typename Shell<T>::template In<N>::template Deep<N>* p) {
            p->template Deep<N>::f();   // inhibit virtual call
        }

        void case2 ( typename Shell<T>::template In<N>::template Deep<N>& p) {
            p.template Deep<N>::f();    // inhibit virtual call
        }
};
```

### 13.3.4 依赖型using声明

可以使用`using`引入基类中的依赖型类型，但是引入基类中的模板需要变通一下：

```cpp
template<typename T>
class BXT {
    public:
        using Mystery = T;
        template<typename U>
        struct Magic;
};

template<typename T>
class DXTT : private BXT<T> {
    public:
        using typename BXT<T>::Mystery;
        Mystery* p; // would be a syntax error without the earlier typename
};

template<typename T>
class DXTM : private BXT<T> {
    public:
        template<typename U>
        using Magic = typename BXT<T>::template Magic<T>;   // Alias template
        Magic<T>* plink;                                    // OK
};
```

### 13.3.5 ADL和显示模板实参

只有当非受限名称是函数时，才会应用ADL规则：

```cpp
namespace N {
    class X {
        // ...
    };

    template<int I> void select(X*);
}

void g (N::X* xp)
{
    select<3>(xp); // ERROR: no ADL!
}
```

在这个例子中，如果`select`是一个函数模板，那么`<3>`就应该是显示指定的模板实参；如果`<3>`是一个显示的模板实参，那么`select`就应该是一个函数模板，这就出现了先有鸡还是先有蛋的问题。原文：

>Because a compiler cannot decide that xp is a function call argument until it has decided that <3> is a template argument list. Furthermore, a compiler cannot decide that <3> is a template argument list until it has found select() to be a template.

为了解决这个问题，需要显示的引入一个模板声明来指出`select`是一个函数模板：

```cpp
template<typename T> void select();
```

### 13.3.6 依赖型表达式

依赖型表达式分为实例化依赖型表达式、值依赖型表达式和类型依赖型表达式。

### 13.3.7 编译器错误

C++标准允许在解析模板时就报错，而不必等到两阶段实例化时，例如下面的代码在解析时就能发现错误：

```cpp
void f() { }

template<int x> void nondependentCall()
{
    f(x); // x is value-dependent, so f() is nondependent; this call will never succeed
}
```

## 13.4 继承和类模板

### 13.4.1 非依赖型基类

不包含模板参数的基类就是非依赖型基类。

### 13.4.2 依赖型基类

感觉很复杂，最有用的一条应该对于受限名称，先在当前实例化类和非依赖型基类中查找，最后在依赖型基类中查找。

## 13.5 后记
