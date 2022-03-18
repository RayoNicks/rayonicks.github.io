---
layout: post
title: 《C++ Templates》第21章 模板和继承
categories: Reading
tags: C++
---

# 21 模板和继承

## 21.1 空基类优化

只包含类型成员、非虚成员函数和静态数据成员的类型为空类型，但是所占内存大小并不为0：

```cpp
// inherit/empty.cpp
#include <iostream>

class EmptyClass {
};

int main()
{
    std::cout << "sizeof(EmptyClass): " << sizeof(EmptyClass) << '\n';
}
```

在我的机器上输出为：

```log
sizeof(EmptyClass): 1
```

### 21.1.1 空基类的布局

空基类优化（Empty Base Class Optimization，EBCO）是指当同一类型的对象（也包括派生类对象）不会被分配到同一起始地址时可以不为空基类分配额外的空间。原文：

>Even though there are no zero-size types in C++, the C++ standard does specify that when an empty class is used as a base class, no space needs to be allocated for it provided that it does not cause it to be allocated to the same address as another object or subobject of the same type.

看一个例子：

```cpp
// inherit/ebco1.cpp
#include <iostream>
class Empty {
    using Int = int; // type alias members don’t make a class nonempty
};

class EmptyToo : public Empty {
};

class EmptyThree : public EmptyToo {
};
int main()
{
    std::cout << "sizeof(Empty): " << sizeof(Empty) << '\n';
    std::cout << "sizeof(EmptyToo): " << sizeof(EmptyToo) << '\n';
    std::cout << "sizeof(EmptyThree): " << sizeof(EmptyThree) << '\n';
}
```

如果编译器实现了空基类优化，则输出相同（但是不为0）；否则不同，在我的机器上输出为：

```log
sizeof(Empty):      1
sizeof(EmptyToo):   1
sizeof(EmptyThree): 1
```

还有一个例子：

```cpp
// inherit/ebco2.cpp
#include <iostream>

class Empty {
    using Int = int; // type alias members don’t make a class nonempty
};

class EmptyToo : public Empty {
};
class NonEmpty : public Empty, public EmptyToo {
};

int main()
{
    std::cout << "sizeof(Empty): " << sizeof(Empty) << '\n';
    std::cout << "sizeof(EmptyToo): " << sizeof(EmptyToo) << '\n';
    std::cout << "sizeof(NonEmpty): " << sizeof(NonEmpty) << '\n';
}
```

在我的机器上输出为：

```log
sizeof(Empty):      1
sizeof(EmptyToo):   1
sizeof(EmptyThree): 2
```

### 21.1.2 将成员作为基类

书中给了一种将模板实参和数据成员绑定在一起的方法，但是没想明白代码的使用场景：

```cpp
template<typename CustomClass>
class Optimizable {
    private:
        BaseMemberPair<CustomClass, void*> info_and_storage;
        // ...
};
```

```cpp
// inherit/basememberpair.hpp
#ifndef BASE_MEMBER_PAIR_HPP
#define BASE_MEMBER_PAIR_HPP

template<typename Base, typename Member>
class BaseMemberPair : private Base {
    private:
        Member mem;
    public:
        // constructor
        BaseMemberPair (Base const & b, Member const & m)
            : Base(b), mem(m) {
        }

        // access base class data via first()
        Base const& base() const {
            return static_cast<Base const&>(*this);
        }
        Base& base() {
            return static_cast<Base&>(*this);
        }

        // access member data via second()
        Member const& member() const {
            return this->mem;
        }
        Member& member() {
            return this->mem;
        }
};
#endif // BASE_MEMBER_PAIR_HPP
```

## 21.2 奇异递归模板模式

奇异递归模板模式（Curiously Recurring Template Pattern，CRTP）是指将派生类作为模板参数传递到基类中来为派生类提供一些功能，这样可以在避免使用虚函数的情况下实现多态。例如利用这种方式可以实现对象计数：

```cpp
// inherit/objectcounter.hpp
#include <cstddef>

template<typename CountedType>
class ObjectCounter {
    private:
        inline static std::size_t count = 0; // number of existing objects
    protected:
        // default constructor
        ObjectCounter() {
            ++count;
        }

        // copy constructor
        ObjectCounter (ObjectCounter<CountedType> const&) {
            ++count;
        }

        // move constructor
        ObjectCounter (ObjectCounter<CountedType> &&) {
            ++count;
        }

        // destructor
        ~ObjectCounter() {
            --count;
        }
    public:
        // return number of existing objects:
        static std::size_t live() {
            return count;
        }
};
```

```cpp
// inherit/countertest.cpp
#include "objectcounter.hpp"
#include <iostream>

template<typename CharT>
class MyString : public ObjectCounter<MyString<CharT>> {
    // ...
};

int main()
{
    MyString<char> s1, s2;
    MyString<wchar_t> ws;
    std::cout << "num of MyString<char>: "
        << MyString<char>::live() << '\n';
    std::cout << "num of MyString<wchar_t>: "
        << ws.live() << '\n';
}
```

### 21.2.1 巴顿-诺克曼技巧

早期C++无法对函数模板进行重载，巴顿-诺克曼技巧通过将函数模板实现为友元来用来解决这个问题：

```cpp
template<typename T>
class Array {
        static bool areEqual(Array<T> const& a, Array<T> const& b);
    public:
        // ...
        friend bool operator== (Array<T> const& a, Array<T> const& b) {
            return areEqual(a, b);
        }
};
```

例如实例化`Array<float>`时，实例化后的友元函数存在于全局命名空间中，可以访问`Array<float>`的`protected`和`private`成员，同时只是一个普通的函数，可以被重载。在C++支持函数模板重载后，这个“注入到全局命名空间”的技巧不适用了。现代C++中的名称查找是基于ADL规则的：

```cpp
// inherit/wrapper.cpp
class S {
};

template<typename T>
class Wrapper {
    private:
        T object;
    public:
        Wrapper(T obj) : object(obj) { // implicit conversion from T to Wrapper<T>
        }
        friend void foo(Wrapper<T> const&) {
        }
};

int main()
{
    S s;
    Wrapper<S> w(s);
    foo(w);     // OK: Wrapper<S> is a class associated with w
    foo(s);     // ERROR: Wrapper<S> is not associated with s
}
```

`w`关联的类是`Wrapper<S>`，其中定义了`foo()`，而`s`关联的类是`S`，没有定义`foo()`。

### 21.2.2 通过CRTP实现关系运算符

实现关系运算符时，一般会只实现`==`和`<`，`!=`、`<=`、`>`和`>=`都通过组合来实现。如果很多类型都有这种需求，则可以泛化为模板：

```cpp
template<typename T>
bool operator!= (T const& x1, T const& x2) {
    return !(x1 == x2);
}
```

但是这个模板太通用了，可以挪到基类中然后通过CRTP为特定的派生类所用：

```cpp
// inherit/equalitycomparable.cpp
template<typename Derived>
class EqualityComparable
{
    public:
        friend bool operator!= (Derived const& x1, Derived const& x2) {
            return !(x1 == x2);
        }
};

class X : public EqualityComparable<X>
{
    public:
        friend bool operator== (X const& x1, X const& x2) {
            // implement logic for comparing two objects of type X
        }
};

int main()
{
    X x1, x2;
    if (x1 != x2) { }
}
```

### 21.2.3 外观模式

外观模式中的基类定义了大量或者全部的接口，这些接口借助派生类的接口实现。外观模式可以通过CRTP实现，本节中给出了一个迭代器外观类的例子：

```cpp
// inherit/iteratorfacadeskel.hpp
template<typename Derived, typename Value, typename Category,
            typename Reference = Value&, typename Distance = std::ptrdiff_t>
class IteratorFacade
{
    public:
        using value_type = typename std::remove_const<Value>::type;502 Chapter 21: Templates and Inheritance
        using reference = Reference;
        using pointer = Value*;
        using difference_type = Distance;
        using iterator_category = Category;

        // input iterator interface:
        reference operator *() const { ... }
        pointer operator ->() const { ... }
        Derived& operator ++() { ... }
        Derived operator ++(int) { ... }
        friend bool operator== (IteratorFacade const& lhs,
        IteratorFacade const& rhs) { ... }
        // ...

        // bidirectional iterator interface:
        Derived& operator --() { ... }
        Derived operator --(int) { ... }

        // random access iterator interface:
        reference operator [](difference_type n) const { ... }
        Derived& operator +=(difference_type n) { ... }
        // ...
        friend difference_type operator -(IteratorFacade const& lhs, IteratorFacade const& rhs) { ... }
        friend bool operator <(IteratorFacade const& lhs, IteratorFacade const& rhs) { ... }
        // ...
};
```

用户代码调用`IteratorFacade<>`中的各种运算符，而这些运算符调用派生类的接口实现：

- 所有迭代器应该都支持`dereference()`、`increment()`和`equals()`
- 双向迭代器应该支持`decrement()`
- 随机访问迭代器应该支持`advance()`和`measureDistance()`

`IteratorFacade<>`通过下面的方式调用派生类中的实现：

```cpp
Derived& asDerived() { return *static_cast<Derived*>(this); }

Derived const& asDerived() const {
    return *static_cast<Derived const*>(this);
}

reference operator*() const {
    return asDerived().dereference();
}

Derived& operator++() {
    asDerived().increment();
    return asDerived();
}

Derived operator++(int) {
    Derived result(asDerived());
    asDerived().increment();
    return result;
}

friend bool operator== (IteratorFacade const& lhs, IteratorFacade const& rhs) {
    return lhs.asDerived().equals(rhs.asDerived());
}
```

#### 链表迭代器

基于`IteratorFacade<>`的链表迭代器如下所示：

```cpp
// inherit/listnode.hpp
template<typename T>
class ListNode
{
    public:
        T value;
        ListNode<T>* next = nullptr;
        ~ListNode() { delete next; }
};
```

```cpp
// inherit/listnodeiterator0.hpp
template<typename T>
class ListNodeIterator
    : public IteratorFacade<ListNodeIterator<T>, T, std::forward_iterator_tag>
{
        ListNode<T>* current = nullptr;
    public:
        T& dereference() const {
            return current->value;
        }
        void increment() {
            current = current->next;
        }
        bool equals(ListNodeIterator const& other) const {
            return current == other.current;
        }
        ListNodeIterator(ListNode<T>* current = nullptr) : current(current) { }
};
```

#### 接口隐藏

在基类中调用派生类对象的接口要求派生类中的实现是`public`的，为此可以定义访问类`IteratorFacadeAccess`，然后在`ListNodeIterator`中将`IteratorFacadeAccess`定义为友元来解决这个问题：

```cpp
// inherit/iteratorfacadeaccessskel.hpp
// ‘friend’ this class to allow IteratorFacade access to core iterator operations:
class IteratorFacadeAccess
{
    // only IteratorFacade can use these definitions
    template<typename Derived, typename Value, typename Category,
                typename Reference, typename Distance>
        friend class IteratorFacade;

    // required of all iterators:
    template<typename Reference, typename Iterator>
    static Reference dereference(Iterator const& i) {
        return i.dereference();
    }
    // ...
    // required of bidirectional iterators:
    template<typename Iterator>
    static void decrement(Iterator& i) {
        return i.decrement();
    }

    // required of random-access iterators:
    template<typename Iterator, typename Distance>
    static void advance(Iterator& i, Distance n) {
        return i.advance(n);
    }
    // ...
};
```

#### 迭代器适配器

有了`IteratorFacade<>`后可以很容易的将构建迭代器适配器，例如通过对象的迭代器构建成员的迭代器：

```cpp
// inherit/person.hpp
struct Person {
    std::string firstName;
    std::string lastName;
    friend std::ostream& operator<<(std::ostream& strm, Person const& p) {
        return strm << p.lastName << ", " << p.firstName;
    }
};
```

```cpp
// inherit/projectioniteratorskel.hpp
template<typename Iterator, typename T>
class ProjectionIterator
    : public IteratorFacade<ProjectionIterator<Iterator, T>,
                            T,
                            typename std::iterator_traits<Iterator>::iterator_category,
                            T&,
                            typename std::iterator_traits<Iterator>::difference_type>
{
        using Base = typename std::iterator_traits<Iterator>::value_type;
        using Distance = typename std::iterator_traits<Iterator>::difference_type;

        Iterator iter;
        T Base::* member;

        friend class IteratorFacadeAccess;
        // ... // implement core iterator operations for IteratorFacade
    public:
        ProjectionIterator(Iterator iter, T Base::* member)
        : iter(iter), member(member) { }
};

template<typename Iterator, typename Base, typename T>
auto project(Iterator iter, T Base::* member) {
    return ProjectionIterator<Iterator, T>(iter, member);
}
```

```cpp
// inherit/projectioniterator.cpp
#include <vector>
#include <algorithm>
#include <iterator>

int main()
{
    std::vector<Person> authors = { {"David", "Vandevoorde"},
                                    {"Nicolai", "Josuttis"},
                                    {"Douglas", "Gregor"} };

    std::copy(project(authors.begin(), &Person::firstName),
                project(authors.end(), &Person::firstName),
                std::ostream_iterator<std::string>(std::cout, "\n"));
}
```

#### 完整代码

上面的代码很乱，完整代码应该为：

```cpp
#include <algorithm>
#include <iterator>
#include <iostream>
#include <vector>

struct Person
{   
    std::string firstName;
    std::string lastName;
};

class IteratorFacadeAccess
{
    template<typename Derived, typename Value, typename Category,
                typename Reference, typename Distance>
    friend class IteratorFacade;

    template<typename Reference, typename Iterator>
    static Reference dereference(Iterator const& i)
    {   
        return i.dereference();
    }

    template<typename Iterator>
    static bool equals(Iterator& il, Iterator& ir)
    {   
        return il.equals(ir);
    }

    template<typename Iterator>
    static void increment(Iterator& i)
    {
        return i.increment();
    }
};

template<typename Derived, typename Value, typename Category,
            typename Reference = Value&, typename Distance = std::ptrdiff_t>
class IteratorFacade
{
    public:
        using value_type = typename std::remove_const<Value>::type;
        using reference = Reference;
        using pointer = Value*;
        using difference_type = Distance;
        using iterator_category = Category;

        Derived& asDerived()
        {
            return *static_cast<Derived*>(this);
        }
        Derived const& asDerived() const
        {
            return *static_cast<Derived const*>(this);
        }

        //input iterator interface
        reference operator*() const
        {
            return IteratorFacadeAccess::dereference<Reference>(asDerived());
        }
        Derived& operator++()
        {
            IteratorFacadeAccess::increment(asDerived());
            return asDerived();
        }
        Derived operator++(int)
        {
            Derived result(asDerived());
            IteratorFacadeAccess::increment(asDerived());
            return result;
        }

        friend bool operator==(IteratorFacade const& lhs, IteratorFacade const& rhs)
        {
            return IteratorFacadeAccess::equals(lhs.asDerived(), rhs.asDerived());
        }
        friend bool operator!=(IteratorFacade const& lhs, IteratorFacade const& rhs)
        {
            return !(lhs == rhs);
        }

        friend difference_type operator-(IteratorFacade const& lhs, IteratorFacade const& rhs)
        {
            return &(*lhs) - &(*rhs);
        }
};

template<typename Iterator, typename T>
class ProjectionIterator
    : public IteratorFacade<ProjectionIterator<Iterator, T>,
                            T,
                            typename std::iterator_traits<Iterator>::iterator_category,
                            T&,
                            typename std::iterator_traits<Iterator>::difference_type>
{
        using Base = typename std::iterator_traits<Iterator>::value_type;
        using Distance = typename std::iterator_traits<Iterator>::difference_type;

        Iterator iter;
        T Base::* member;

        friend class IteratorFacadeAccess;

        T& dereference() const
        {
            return (*iter).*member;
        }
        void increment()
        {
            ++iter;
        }
        bool equals(ProjectionIterator const& other) const
        {
            return iter == other.iter;
        }
    public:
        ProjectionIterator(Iterator iter, T Base::* member) : iter(iter), member(member) {}
};

template<typename Iterator, typename Base, typename T>
auto project(Iterator iter, T Base::* member)
{
    return ProjectionIterator<Iterator, T>(iter, member);
}

int main()
{
    std::vector<Person> authors = { {"David", "Vandevoorde"},
                                    {"Nicolai", "Josuttis"},
                                    {"Douglas", "Gregor"} };

    std::copy(project(authors.begin(), &Person::lastName),
                project(authors.end(), &Person::lastName),
                std::ostream_iterator<std::string>(std::cout, "\n"));
}
```

但是输出却多了三行：

```log
Vandevoorde
Josuttis
Gregor



```

这是因为我机器上的`std::copy`通过减法计算了两个迭代器指向的`Person::lastName`的距离，大小是6个`std::string`，而不是3个`Person`，所以会多输出三行。

## 21.3 混入

混入（mixins）是一种通过反转继承关系的方式来对类型进行定制的方法。假如现在有一个`Point`类和`Polygon`类：

```cpp
class Point
{
    public:
        double x, y;
        Point() : x(0.0), y(0.0) { }
        Point(double x, double y) : x(x), y(y) { }
};

class Polygon
{
    private:
        std::vector<Point> points;
    public:
        // ... // public operations
};
```

如果再为每个点增加颜色和标签等属性，同时还要保证可以在`Polygon`中使用扩展的点，则可以从`Point`派生出新的`LabeledPoint`，并将`Polygon`泛化为模板：

```cpp
class LabeledPoint : public Point
{
    public:
        std::string label;
        LabeledPoint() : Point(), label("") { }
        LabeledPoint(double x, double y) : Point(x, y), label("") { }
};

template<typename P>
class Polygon
{
    private:
        // std::vector<P> points;
    public:
        // ... // public operations
};
```

另一种实现方式是通过混入：

```cpp
template<typename... Mixins>
class Point : public Mixins...
{
    public:
        double x, y;
        Point() : Mixins()..., x(0.0), y(0.0) { }
        Point(double x, double y) : Mixins()..., x(x), y(y) { }
};

class Label
{
    public:
        std::string label;
        Label() : label("") { }
};
using LabeledPoint = Point<Label>;

class Color
{
    public:
        unsigned char red = 0, green = 0, blue = 0;
};
using MyPoint = Point<Label, Color>;
```

### 21.3.1 基于CRTP的混入

混入也可以和CRTP结合来进行定制：

```cpp
template<template<typename>... Mixins>
class Point : public Mixins<Point>...
{
    public:
        double x, y;
        Point() : Mixins<Point>()..., x(0.0), y(0.0) { }
        Point(double x, double y) : Mixins<Point>()..., x(x), y(y) { }
};
```

### 21.3.2 参数化虚函数

混入也可以将成员函数变为虚函数，但是感觉很混乱：

```cpp
// inherit/virtual.cpp
#include <iostream>
class NotVirtual {
};

class Virtual {
    public:
        virtual void foo() {
        }
};

template<typename... Mixins>
class Base : public Mixins... {
    public:
        // the virtuality of foo() depends on its declaration
        // (if any) in the base classes Mixins...
        void foo() {
            std::cout << "Base::foo()" << '\n';
        }
};

template<typename... Mixins>
class Derived : public Base<Mixins...> {
    public:
        void foo() {
            std::cout << "Derived::foo()" << '\n';
        }
};

int main()
{
    Base<NotVirtual>* p1 = new Derived<NotVirtual>;
    p1->foo();  // calls Base::foo()
    Base<Virtual>* p2 = new Derived<Virtual>;
    p2->foo();  // calls Derived::foo()
}
```

## 21.4 模板实参指定初始化

如果模板中有多个参数，且每个参数都有默认值，当前标准并不支持指定初始化靠后的模板参数，所以下面的代码得不到想要的结果：

```cpp
template<typename Policy1 = DefaultPolicy1,
            typename Policy2 = DefaultPolicy2,
            typename Policy3 = DefaultPolicy3,
            typename Policy4 = DefaultPolicy4>
class BreadSlicer {
    // ...
};

BreadSlicer<Policy3 = Custom> bc;
```

书中给了一种解决办法：

```cpp
// PolicySelector<A,B,C,D> creates A,B,C,D as base classes
// Discriminator<> allows having even the same base class more than once
template<typename Base, int D>
class Discriminator : public Base {
};

template<typename Setter1, typename Setter2, typename Setter3, typename Setter4>
class PolicySelector : public Discriminator<Setter1,1>,
                        public Discriminator<Setter2,2>,
                        public Discriminator<Setter3,3>,
                        public Discriminator<Setter4,4> {
};

// name default policies as P1, P2, P3, P4
class DefaultPolicies {
    public:
        using P1 = DefaultPolicy1;
        using P2 = DefaultPolicy2;
        using P3 = DefaultPolicy3;
        using P4 = DefaultPolicy4;
};

// class to define a use of the default policy values
// avoids ambiguities if we derive from DefaultPolicies more than once
class DefaultPolicyArgs : virtual public DefaultPolicies {
};

template<typename Policy>
class Policy1_is : virtual public DefaultPolicies {
    public:
        using P1 = Policy; // overriding type alias
};

template<typename Policy>
class Policy2_is : virtual public DefaultPolicies {
    public:
        using P2 = Policy; // overriding type alias
};

template<typename Policy>
class Policy3_is : virtual public DefaultPolicies {
    public:
        using P3 = Policy; // overriding type alias
};

template<typename Policy>
class Policy4_is : virtual public DefaultPolicies {
    public:
        using P4 = Policy; // overriding type alias
};

template<typename PolicySetter1 = DefaultPolicyArgs,
            typename PolicySetter2 = DefaultPolicyArgs,
            typename PolicySetter3 = DefaultPolicyArgs,
            typename PolicySetter4 = DefaultPolicyArgs>
class BreadSlicer {
    using Policies = PolicySelector<PolicySetter1, PolicySetter2, PolicySetter3, PolicySetter4>;
    // use Policies::P1, Policies::P2, ... to refer to the various policies
    // ...
};

BreadSlicer<Policy3_is<CustomPolicy>> bc;
```

## 21.5 后记
