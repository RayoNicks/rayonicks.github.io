---
layout: post
title: 《C++ Templates》第28章 模板调试
categories: Reading
tags: C++
---

# 28 模板调试

模板对于开发者和使用者都会带来问题：对于模板的设计者，该如何保证模板针对任意类型的实参都有效？对于模板使用者，实例化出错时该如何进行调试？本章将模板实参的限制分为了两类，第一类是语法限制（syntactic constraint），这一类限制由于缺少特定的构造函数或者成员函数导致，第二类是语义限制，比如在要求模板实参支持`<`运算符的语法限制下，可能就要求模板实参是具有偏序关系的。

## 28.1 浅层初始化

当模板代码嵌套层数过多时就会引发难以查找的错误：

```cpp
template<typename T>
void clear (T& p)
{
    *p = 0; // assumes T is a pointer-like type
}

template<typename T>
void core (T& p)
{
    clear(p);
}

template<typename T>
void middle (typename T::Index p)
{
    core(p);
}

template<typename T>
void shell (T const& env)
{
    typename T::Index i;
    middle<T>(i);
}

class Client
{
    public:
        using Index = int;
};

int main()
{
    Client mainClient;
    shell(mainClient);
}
```

错误原因在于从`middle<>`传入的`int`在`clear<>`中无法解引用。为了防止调用层数过多引起的错误，可以插入一些无用的代码进行浅层实例化：

```cpp
template<typename T>
void ignore(T const&)
{}

template<typename T>
void shell (T const& env)
{
    class ShallowChecks
    {
        void deref(typename T::Index ptr) {
            ignore(*ptr);
        }
    };
    typename T::Index i;
    middle(i);
}
```

## 28.2 静态断言

C++11引入的`static_cast`关键字可以在编译时进行检查，例如可以使用下面的类型特征模板检查类型是否可以解引用：

```cpp
// debugging/hasderef.hpp
#include <utility>      // for declval()
#include <type_traits>  // for true_type and false_type

template<typename T>
class HasDereference {
    private:
        template<typename U> struct Identity;
        template<typename U> static std::true_type
            test(Identity<decltype(*std::declval<U>())>*);
        template<typename U> static std::false_type
            test(...);
    public:
        static constexpr bool value = decltype(test<T>(nullptr))::value;
};
```

## 28.3 测试类型

一般来说文档中会对模板实参提出需要满足的限制，因此可以根据这些限制构造一个测试类型（archetype）来测试。例如对于下面的模板代码：

```cpp
// T must be EqualityComparable, meaning:
// two objects of type T can be compared with == and the result converted to bool
template<typename T>
int find(T const* array, int n, T const& value) {
    int i = 0;
    while(i != n && array[i] != value)
        ++i;
    return i;
}
```

可以构造下面的测试类型来测试：

```cpp
class EqualityComparableArchetype
{
};

class ConvertibleToBoolArchetype
{
    public:
        operator bool() const;
};

ConvertibleToBoolArchetype operator==(
    EqualityComparableArchetype const&, EqualityComparableArchetype const&);

template int find(EqualityComparableArchetype const*, int,
                    EqualityComparableArchetype const&);
```

在实例化的过程中会报错，因为模板中使用的是`!=`运算符，而不是`==`。解决办法是重载`!=`运算符，或者修改实现（但最好不要这么做）。同样的方法也可以测试`==`运算符转换为`bool`类型的要求。

## 28.4 追踪器

前面的方法可以有效解决编译时问题，但是还有运行时的问题需要解决，为此可以设计一个追踪器（tracer）作为模板实参来记录模板代码的执行过程，通常追踪器也是测试类型。

下面是一个测试`std::sort`的追踪器：

```cpp
// debugging/tracer.hpp
#include <iostream>

class SortTracer {
    private:
        int value;                              // integer value to be sorted
        int generation;                         // generation of this tracer
        inline static long n_created = 0;       // number of constructor calls
        inline static long n_destroyed = 0;     // number of destructor calls
        inline static long n_assigned = 0;      // number of assignments
        inline static long n_compared = 0;      // number of comparisons
        inline static long n_max_live = 0;      // maximum of existing objects

        // recompute maximum of existing objects
        static void update_max_live() {
            if (n_created-n_destroyed > n_max_live) {
                n_max_live = n_created-n_destroyed;
            }
        }
    public:
        static long creations() {
            return n_created;
        }
        static long destructions() {
            return n_destroyed;
        }
        static long assignments() {
            return n_assigned;
        }
        static long comparisons() {
            return n_compared;
        }
        static long max_live() {
            return n_max_live;
        }
    public:
        // constructor
        SortTracer (int v = 0) : value(v), generation(1) {
            ++n_created;
            update_max_live();
            std::cerr << "SortTracer #" << n_created
                        << ", created generation " << generation
                        << " (total: " << n_created - n_destroyed
                        << ")\n";
        }

        // copy constructor
        SortTracer (SortTracer const& b)
            : value(b.value), generation(b.generation+1) {
                ++n_created;
                update_max_live();
                std::cerr << "SortTracer #" << n_created
                            << ", copied as generation " << generation
                            << " (total: " << n_created - n_destroyed
                            << ")\n";
        }

        // destructor
        ~SortTracer() {
            ++n_destroyed;
            update_max_live();
            std::cerr << "SortTracer generation " << generation
                        << " destroyed (total: "
                        << n_created - n_destroyed << ")\n";
        }

        // assignment
        SortTracer& operator= (SortTracer const& b) {
            ++n_assigned;
            std::cerr << "SortTracer assignment #" << n_assigned
                        << " (generation " << generation
                        << " = " << b.generation
                        << ")\n";
            value = b.value;
            return *this;
        }

        // comparison
        friend bool operator < (SortTracer const& a, SortTracer const& b) {
            ++n_compared;
            std::cerr << "SortTracer comparison #" << n_compared
                        << " (generation " << a.generation
                        << " < " << b.generation
                        << ")\n";
            return a.value < b.value;
        }

        int val() const {
            return value;
        }
};
```

测试代码如下：

```cpp
// debugging/tracertest.cpp
#include <iostream>
#include <algorithm>
#include "tracer.hpp"

int main()
{
    // prepare sample input:
    SortTracer input[] = { 7, 3, 5, 6, 4, 2, 0, 1, 9, 8 };

    // print initial values:
    for (int i=0; i<10; ++i) {
        std::cerr << input[i].val() << ' ';
    }

    std::cerr << '\n';

    // remember initial conditions:
    long created_at_start = SortTracer::creations();
    long max_live_at_start = SortTracer::max_live();
    long assigned_at_start = SortTracer::assignments();
    long compared_at_start = SortTracer::comparisons();

    // execute algorithm:
    std::cerr << "---[ Start std::sort() ]--------------------\n";
    std::sort<>(&input[0], &input[9]+1);
    std::cerr << "---[ End std::sort() ]----------------------\n";

    // verify result:
    for (int i=0; i<10; ++i) {
        std::cerr << input[i].val() << ' ';
    }
    std::cerr << "\n\n";

    // final report:
    std::cerr << "std::sort() of 10 SortTracer’s"
                << " was performed by:\n "
                << SortTracer::creations() - created_at_start
                << " temporary tracers\n "
                << "up to "
                << SortTracer::max_live()
                << " tracers at the same time ("
                << max_live_at_start << " before)\n "
                << SortTracer::assignments() - assigned_at_start
                << " assignments\n "
                << SortTracer::comparisons() - compared_at_start
                << " comparisons\n\n";
}
```

编译命令为`g++ tracertest.cpp -o tracertest -std=c++17`，在我的机器上输出为（只保留了关键部分）：

```log
std::sort() of 10 SortTracer's was performed by:
 9 temporary tracers
 up to 11 tracers at the same time (10 before)
 33 assignments
 27 comparisons
```

## 28.5 Oracles

一种可以检查语义的东西，姑且就叫神谕吧~

## 28.6 后记
