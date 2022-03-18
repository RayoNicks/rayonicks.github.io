---
layout: post
title: 《C++ Templates》第23章 元编程
categories: Reading
tags: C++
---

# 23 元编程

元编程是对程序进行编程，也就是在编程系统对源代码进行构建的过程中产生出新代码、实现新功能的技术。原文：

>Metaprogramming consists of “programming a program.” In other words, we lay out code that the programming system executes to generate new code that implements the functionality we really want.

## 23.1 现代C++中的元编程

### 23.1.1 值元编程

值元编程（value metaprogramming）是在编译期间对变量进行求值的技术。C++11引入的`constexpr`解决了大量需要在编译期间求值的问题，C++14开始支持复杂的控制结构，下面是一个在编译时计算平方根的例子：

```cpp
// meta/sqrtconstexpr.hpp
template<typename T>
constexpr T sqrt(T x)
{
    // handle cases where x and its square root are equal as a special case to simplify
    // the iteration criterion for larger x:
    if (x <= 1) {
        return x;
    }

    // repeatedly determine in which half of a [lo, hi] interval the square root of x is located,
    // until the interval is reduced to just one value:
    T lo = 0, hi = x;
    for (;;) {
        auto mid = (hi+lo)/2, midSquared = mid*mid;
        if (lo+1 >= hi || midSquared == x) {
            // mid must be the square root:
            return mid;
        }
        // continue with the higher/lower half-interval:
        if (midSquared < x) {
            lo = mid;
        }
        else {
            hi = mid;
        }
    }
}
```

### 23.1.2 类型元编程

类型元编程（type metaprogramming）是在编译期间以类型作为参数产生新类型的技术，下面的是一个利用模板递归实例化技术提取变量类型的例子：

```cpp
// meta/removeallextents.hpp
// primary template: in general we yield the given type:
template<typename T>
struct RemoveAllExtentsT {
    using Type = T;
};

// partial specializations for array types (with and without bounds):
template<typename T, std::size_t SZ>
    struct RemoveAllExtentsT<T[SZ]> {
    using Type = typename RemoveAllExtentsT<T>::Type;
};

template<typename T>
struct RemoveAllExtentsT<T[]> {
    using Type = typename RemoveAllExtentsT<T>::Type;
};

template<typename T>
using RemoveAllExtents = typename RemoveAllExtentsT<T>::Type;
```

### 23.1.3 混合元编程

混合元编程（hybrid metaprogramming）是在编译期间生成代码并在运行期间进行求值的技术。例如要计算向量点积，通常会将代码写为下面的样子：

```cpp
template<typename T, std::size_t N>
auto dotProduct(std::array<T, N> const& x, std::array<T, N> const& y)
{
    T result{};
    for (std::size_t k = 0; k<N; ++k) {
        result += x[k]*y[k];
    }
    return result;
}
```

循环中包含跳转，所以编译器可能会进行循环展开优化，从而形成一系列的乘法求和运算，这种优化也可以通过元编程来实现：

```cpp
template<typename T, std::size_t N>
struct DotProductT {
    static inline T result(T* a, T* b) {
        return *a * *b + DotProduct<T, N-1>::result(a+1,b+1);
    }
};

// partial specialization as end criteria
template<typename T>
struct DotProductT<T, 0> {
    static inline T result(T*, T*) {
        return T{};
    }
};
template<typename T, std::size_t N>
auto dotProduct(std::array<T, N> const& x, std::array<T, N> const& y)
{
    return DotProductT<T, N>::result(x.begin(), y.begin());
}
```

### 23.1.4 对不同单位下的值运算进行混合元编程

混合元编程的另外一个应用是对具有不同单位的值求和，具体为在编译期间确定求和结果的类型，并在运行期间进行计算，例如：

```cpp
// meta/ratio.hpp
template<unsigned N, unsigned D = 1>
struct Ratio {
    static constexpr unsigned num = N; // numerator
    static constexpr unsigned den = D; // denominator
    using Type = Ratio<num, den>;
};
```

```cpp
// meta/ratioadd.hpp
// implementation of adding two ratios:
template<typename R1, typename R2>
struct RatioAddImpl
{
    private:
        static constexpr unsigned den = R1::den * R2::den;
        static constexpr unsigned num = R1::num * R2::den + R2::num * R1::den;
    public:
        typedef Ratio<num, den> Type;
};

// using declaration for convenient usage:
template<typename R1, typename R2>
using RatioAdd = typename RatioAddImpl<R1, R2>::Type;
```

```cpp
// meta/duration.hpp
// duration type for values of type T with unit type U:
template<typename T, typename U = Ratio<1>>
class Duration {
    public:
        using ValueType = T;
        using UnitType = typename U::Type;
    private:
        ValueType val;
    public:
        constexpr Duration(ValueType v = 0)
            : val(v) {
        }
        constexpr ValueType value() const {
            return val;
        }
};
```

```cpp
// meta/durationadd.hpp
// adding two durations where unit type might differ:
template<typename T1, typename U1, typename T2, typename U2>
auto constexpr operator+(Duration<T1, U1> const& lhs, Duration<T2, U2> const& rhs)
{
    // resulting type is a unit with 1 a nominator and
    // the resulting denominator of adding both unit type fractions
    using VT = Ratio<1,RatioAdd<U1,U2>::den>;
    // resulting value is the sum of both values
    // converted to the resulting unit type:
    auto val = lhs.value() * VT::den / U1::den * U1::num + rhs.value() * VT::den / U2::den * U2::num;
    return Duration<decltype(val), VT>(val);
}
```

看到这里我一下子就想起来了标准库中关于时间的模板`std::chrono`。`Ratio<>`是一个单位类型，`RatioAddImpl<>`对两个具有不同单位的单位元进行求和得到新的单位元类型。`Duration<T, U>`表示`U`单位下的数值，该数值用内置类型`T`表示，最终的加法是对两个不同单位下的数值进行求和。下面是测试代码：

```cpp
int x = 42;
int y = 77;
auto a = Duration<int, Ratio<1,1000>>(x);       // x milliseconds
auto b = Duration<int, Ratio<2,3>>(y);          // y 2/3 seconds
auto c = a + b;                                 // computes resulting unit type 1/3000 seconds
                                                // and generates run-time code for c = a*3 + b*2000
auto d = Duration<double, Ratio<1,3>>(7.5);     // 7.5 1/3 seconds
auto e = Duration<int, Ratio<1>>(4);            // 4 seconds
auto f = d + e;                                 // computes resulting unit type 1/3 seconds
                                                // and generates code for f = d + e*3
```

## 23.2 元编程中的反射

在没有`constexpr`之前，通过元编程计算平方根的方法为：

```cpp
// meta/sqrt1.hpp
// primary template to compute sqrt(N)
template<int N, int LO=1, int HI=N>
struct Sqrt {
    // compute the midpoint, rounded up
    static constexpr auto mid = (LO+HI+1)/2;
    // search a not too large value in a halved interval
    static constexpr auto value = (N<mid*mid) ? Sqrt<N,LO,mid-1>::value : Sqrt<N,mid,HI>::value;
};

// partial specialization for the case when LO equals HI
template<int N, int M>
struct Sqrt<N,M,M> {
    static constexpr auto value = M;
};
```

通常元编程可以包含下面三部分内容：

- 计算（computation），通过递归实例化或者常量表达式在编译期间进行求值
- 反射（reflection），通过代码在编译期间侦测程序的特性
- 生成（generation），在编译器期间生成额外的代码

## 23.3 递归实例化的开销

### 23.3.1 跟踪递归实例化的过程

当使用如下代码实例化上一节的计算平方根的代码时：

```cpp
// meta/sqrt1.cpp
#include <iostream>
#include "sqrt1.hpp"

int main()
{
    std::cout << "Sqrt<16>::value = " << Sqrt<16>::value << '\n';
    std::cout << "Sqrt<25>::value = " << Sqrt<25>::value << '\n';
    std::cout << "Sqrt<42>::value = " << Sqrt<42>::value << '\n';
    std::cout << "Sqrt<1>::value = " << Sqrt<1>::value << '\n';
}
```

想当然的会认为`Sqrt<16>::value`的递归过程为：

1. `Sqrt<16,1,16>::value`
2. `Sqrt<16,1,8>::value`
3. `Sqrt<16,1,4>::value`
4. `Sqrt<16,3,4>::value`
5. `Sqrt<16,4,4>::value`

但是实际实例化的过程并没有这么简单，这是因为`Sqrt::value`包含两个分支，编译期间无法根据条件优化掉无效的分支，最终的结果是一个超长的条件判断表达式。虽然是通过二分法计算平方根，但是实际实例化的类模板为`O(2logN)`个。不仅如此，当使用`::`运算符引用`value`时，还需要实例化整个类。不过好在可以借助`IfThenElse<>`实现分支优化：

```cpp
// meta/sqrt2.hpp
#include "ifthenelse.hpp"

// primary template for main recursive step
template<int N, int LO=1, int HI=N>
struct Sqrt {
    // compute the midpoint, rounded up
    static constexpr auto mid = (LO+HI+1)/2;

    // search a not too large value in a halved interval
    using SubT = IfThenElse<(N<mid*mid), Sqrt<N,LO,mid-1>, Sqrt<N,mid,HI>>;
    static constexpr auto value = SubT::value;
};

// partial specialization for end of recursion criterion
template<int N, int S>
struct Sqrt<N, S, S> {
s   tatic constexpr auto value = S;
};
```

书中给出的一段解释解答了一个让我疑惑很久的问题。原文：

>At this point it is important to remember that defining a type alias for a class template instance does not cause a C++ compiler to instantiate the body of that instance. Therefore, when we write `using SubT = IfThenElse<(N<mid*mid),Sqrt<N,LO,mid-1>,Sqrt<N,mid,HI>>`, neither `Sqrt<N,LO,mid-1>` nor `Sqrt<N,mid,HI>` is fully instantiated. Whichever of these two types ends up being a synonym for `SubT` is fully instantiated when looking up `SubT::value`.

这也就是说实例化`IfThenElse<>`时只会实例化`IfThenElseT<>`，因为引用了`IfThenElseT<>::Type`。在后面引用`SubT::value`时，才会实例化`Type`的具体类型（也就是`Sqrt<N,LO,mid-1>`或者`Sqrt<N,mid,HI>`）。联想到第14章开头提到过，编译器只在获取类大小或者访问类成员时才实例化类模板，也印证了这个说法。

## 23.4 计算完整性

`Sqrt<>`的例子表明模板元编程是可以实现任意计算的。

## 23.5 递归实例化和递归模板

假设有下面的递归模板：

```cpp
template<typename T, typename U>
struct Doublify {
};

template<int N>
struct Trouble {
    using LongType = Doublify<typename Trouble<N-1>::LongType,
    typename Trouble<N-1>::LongType>;
};
template<>
struct Trouble<0> {
    using LongType = double;
};

Trouble<10>::LongType ouch;
```

随着实例化`Trouble<>`的`N`的增大，`Trouble::LongType`的名字也会越来越长，这对编译器是一个很大的负担，因此编译器都对模板名字进行了压缩。

## 23.6 枚举值和常量值

早期C++只支持枚举类型作为元编程中的常量，例如下面计算3的幂的例子：

```cpp
// meta/pow3enum.hpp
// primary template to compute 3 to the Nth
template<int N>
struct Pow3 {
    enum { value = 3 * Pow3<N-1>::value };
};

// full specialization to end the recursion
template<>
struct Pow3<0> {
    enum { value = 1 };
};
```

然后从C++98开始，可以使用类内静态常量参与元编程计算：

```cpp
// meta/pow3const.hpp
// primary template to compute 3 to the Nth
template<int N>
struct Pow3 {
    static int const value = 3 * Pow3<N-1>::value;
};

// full specialization to end the recursion
template<>
struct Pow3<0> {
    static int const value = 1;
};
```

本来编译器是不需要为`Pow3::value`分配地址的，但是当向`void foo(int const&)`传递`value`时，就需要为`value`分配地址了，`value`不再是编译期间的常量了。

直到C++17允许内联静态数据成员后，才解决了这个问题（但是原理书上没说）。

## 23.7 后记

最早由Erwin Unruh提出的元编程例子：

```cpp
// meta/unruh.cpp
// prime number computation
// (modified from original from 1994 by Erwin Unruh)

template<int p, int i>
struct is_prime {
    enum { pri = (p==2) || ((p%i) && is_prime<(i>2?p:0),i-1>::pri) };
};

template<>
struct is_prime<0,0> {
    enum {pri=1};
};

template<>
struct is_prime<0,1> {
    enum {pri=1};
};

template<int i>
struct D {
    D(void*);
};

template<int i>
struct CondNull {
    static int const value = i;
};

template<>
struct CondNull<0> {
    static void* value;
};

void* CondNull<0>::value = 0;

template<int i>
struct Prime_print { // primary template for loop to print prime numbers
    Prime_print<i-1> a;
    enum { pri = is_prime<i,i-1>::pri };
    void f() {
        D<i> d = CondNull<pri ? 1 : 0>::value; // 1 is an error, 0 is fine
        a.f();
    }
};
template<>
struct Prime_print<1> { // full specialization to end the loop
    enum {pri=0};
    void f() {
        D<1> d = 0;
    };
};

#ifndef LAST
#define LAST 18
#endif

int main()
{
    Prime_print<LAST> a;
    a.f();
}
```

这个程序会在编译错误中打印出所有的小于18的素数。。。
