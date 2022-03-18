---
layout: post
title: 《C++ Templates》第1章 函数模板
categories: Reading
tags: C++
---

# 1 函数模板

## 1.1 函数模板速览

### 1.1.1 定义模板

函数模板是一个函数，该函数可以操作不同类型的数据，也就是返回类型和函数参数类型是暂时不确定的，例如求最大值的函数模板，只要类型`T`支持`<`运算符，则`max()`就能正常工作：

```cpp
// basics/max1.hpp
template<typename T>
T max (T a, T b)
{
    // if b < a then yield a else yield b
    return b < a ? a : b;
}
```

这一节中有这样一个注释，没看懂是什么意思，原文：

>Note that the max() template according to [Stepanov Notes] intentionally returns “b < a ? a : b” instead of “a < b ? b : a” to ensure that the function behaves correctly even if the two values are equivalent but not equal.

### 1.1.2 使用模板

```cpp
// basics/max1.cpp
#include "max1.hpp"
#include <iostream>
#include <string>

int main()
{
    int i = 42;
    std::cout << "max(7,i): " << ::max(7,i) << '\n';
    double f1 = 3.4;
    double f2 = -6.7;
    std::cout << "max(f1,f2): " << ::max(f1,f2) << '\n';
    std::string s1 = "mathematics";
    std::string s2 = "math";
    std::cout << "max(s1,s2): " << ::max(s1,s2) << '\n';
}
```

函数模板并不是让一个函数可以处理不同类型的参数，而是为每种类型的参数单独生成了一份函数定义。用具体类型替换模板参数的过程称为**实例化（instantiation）**，该过程产生一个**实例（instance）**。

### 1.1.3 两阶段翻译

1. 在定义阶段检查语法错误、和参数类型无关的类型错误和断言
2. 在实例化阶段检查所有的错误

## 1.2 模板参数推导

1. 模板调用参数为传引用类型时，不会进行任何转换，相同参数类型的推导结果必须一致
2. 模板调用参数为值类型时，修饰符`const`和`volatile`将会被丢弃，引用转换为引用的类型、数组和函数转化为指针类型，相同模板参数类型转换后的推导结果必须一致

## 1.3 多模板参数

可以在模板定义中声明多个模板参数，例如可以为`max()`函数定义两个不用的模板参数：

```cpp
template<typename T1, typename T2>
T1 max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

此时要保证返回类型和参数`a`的类型是一致的，但是如果返回`b`的话，则就会发生类型转换。C++提供了三种方法来解决这个问题。

### 1.3.1 为返回类型定义新的模板参数

定义新类型`RT`来表示返回类型，但是编译器是不能根据参数推导`RT`的，所以需要在在实例化过程中显示指出返回类型。

### 1.3.2 让编译器推导返回类型

- 通过`auto`推导：

```cpp
// basics/maxauto.hpp
template<typename T1, typename T2>
auto max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

- 通过`decltype`和尾置返回类型推导：

```cpp
// basics/maxdecltype.hpp
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(b<a?a:b)
{
    return b < a ? a : b;
}
```

如果模板参数被推导为引用类型，那么可以使用`std::decay`来去掉引用：

```cpp
// basics/maxdecltypedecay.hpp
#include <type_traits>

template<typename T1, typename T2>
auto max (T1 a, T2 b) -> typename std::decay<decltype(true?a:b)>::type
{
    return b < a ? a : b;
}
```

### 1.3.3 使用公共类型

```cpp
// basics/maxcommon.hpp
#include <type_traits>

template<typename T1, typename T2>
std::common_type_t<T1,T2> max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

## 1.4 默认模板参数

除了使用`auto`外，还可以定义默认模板参数，然后使用`std::decay_t`或者`std::common_type_t`为其赋默认类型：

```cpp
// basics/maxdefault1.hpp
#include <type_traits>

template<typename T1, typename T2,
            typename RT = std::decay_t<decltype(true ? T1() : T2())>>
RT max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

```cpp
// basics/maxdefault3.hpp
#include <type_traits>

template<typename T1, typename T2,
            typename RT = std::common_type_t<T1,T2>>
RT max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

## 1.5 函数模板重载

函数定义重载函数模板，由于普通函数更为精确，编译器在解析的过程中会优先匹配：

```cpp
// basics/max2.cpp
int max (int a, int b)
{
    return b < a ? a : b;
}

// maximum of two values of any type:
template<typename T>
T max (T a, T b)
{
    return b < a ? a : b;
}

int main()
{
    ::max(7, 42);           // calls the nontemplate for two ints
    ::max(7.0, 42.0);       // calls max<double> (by argument deduction)
    ::max('a', 'b');        // calls max<char> (by argument deduction)
    ::max<>(7, 42);         // calls max<int> (by argument deduction)
    ::max<double>(7, 42);   // calls max<double> (no argument deduction)
    ::max('a', 42.7);       // calls the nontemplate for two ints
}
```

通过模板参数个数进行重载：

```cpp
// basics/maxdefault4.hpp
template<typename T1, typename T2>
auto max (T1 a, T2 b)
{
    return b < a ? a : b;
}

template<typename RT, typename T1, typename T2>
RT max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

1. `::max(4, 7.2)`匹配两参数版本，因为三参数版本中的`RT`没有默认模板参数，无法推导其类型
2. `::max<long double>(7.2, 4)`匹配三参数版本，因为`7.2`默认为`double`类型，而`long double`和`double`不是同一类型
3. `::max<int>(4, 7.2)`无法匹配

函数模板重载更多用于重载C指针和C字符串版本的函数：

```cpp
// basics/max3val.cpp
#include <cstring>
#include <string>

// maximum of two values of any type:
template<typename T>
T max (T a, T b)
{
    return b < a ? a : b;
}

// maximum of two pointers:
template<typename T>
T* max (T* a, T* b)
{
    return *b < *a ? a : b;
}

// maximum of two C-strings:
char const* max (char const* a, char const* b)
{
    return std::strcmp(b,a) < 0 ? a : b;
}

int main ()
{
    int a = 7;
    int b = 42;
    auto m1 = ::max(a,b);       // max() for two values of type int
    std::string s1 = "hey";
    std::string s2 = "you";
    auto m2 = ::max(s1,s2);     // max() for two values of type std::string
    int* p1 = &b;
    int* p2 = &a;
    auto m3 = ::max(p1,p2);     // max() for two pointers
    char const* x = "hello";
    char const* y = "world";
    auto m4 = ::max(x,y);       // max() for two C-strings
}
```

如果函数模板中的参数为传引用类型，同时重载了传C字符串类型的版本，就会出现问题：

```cpp
// basics/max3ref.cpp
#include <cstring>

// maximum of two values of any type (call-by-reference)
template<typename T>
T const& max (T const& a, T const& b)
{
    return b < a ? a : b;
}

// maximum of two C-strings (call-by-value)
char const* max (char const* a, char const* b)
{
    return std::strcmp(b,a) < 0 ? a : b;
}

// maximum of three values of any type (call-by-reference)
template<typename T>
T const& max (T const& a, T const& b, T const& c)
{
    return max (max(a,b), c); // error if max(a,b) uses call-by-value
}

int main ()
{
    auto m1 = ::max(7, 42, 68);     // OK

    char const* s1 = "frederic";
    char const* s2 = "anica";
    char const* s3 = "lucas";
    auto m2 = ::max(s1, s2, s3);    // run-time ERROR
}
```

g++的编译警告为：

```log
max3ref.cpp: In instantiation of ‘const T& max(const T&, const T&, const T&) [with T = const char*]’:
max3ref.cpp:30:29:   required from here
max3ref.cpp:20:26: warning: returning reference to temporary [-Wreturn-local-addr]
   return max (max(a,b), c);       // error if max(a,b) uses call-by-value
                          ^
```

这个例子中三参数版本的`max()`中`T`被推导为`const char*`，所以会调用非模板的`max()`，而该函数返回值会创建一个临时变量，且三参数版本的`max()`又返回了该临时变量的引用，这就导致`main()`中会出现悬垂引用。原文：

>Because for C-strings, max(a,b) creates a new, temporary local value that is returned by reference, but that temporary value expires as soon as the return statement is complete, leaving main() with a dangling reference.

最后一个需要注意的是定义的顺序：

```cpp
// basics/max4.cpp
#include <iostream>

// maximum of two values of any type:
template<typename T>
T max (T a, T b)
{
    std::cout << "max<T>() \n";
    return b < a ? a : b;
}

// maximum of three values of any type:
template<typename T>
T max (T a, T b, T c)
{
    return max (max(a,b), c);   // uses the template version even for ints
}                               // because the following declaration comes
                                // too late:
// maximum of two int values:
int max (int a, int b)
{
    std::cout << "max(int,int) \n";
    return b < a ? a : b;
}

int main()
{
    ::max(47,11,33); // OOPS: uses max<T>() instead of max(int,int)
}
```

## 1.6 三个问题

### 1.6.1 传值还是传引用

一般来说传值，因为：
1. 语法简单
2. 便于优化
3. C++11中引入了移动语义
4. 某些情况下不会发生拷贝和移动

### 1.6.2 内联

一般来说只将特化版本定义为内联，而不将函数模板定义为内联。

### 1.6.3 常量表达式

`constexpr`也可以用于模板，用于提示编译器这个函数的返回值可以在编译期间被计算：

```cpp
// basics/maxconstexpr.hpp

template<typename T1, typename T2>
constexpr auto max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

## 1.7 总结

1. 函数模板定义了一类可以用于不同参数类型的函数
2. 调用通过函数模板定义的函数时，编译器会推导模板参数的类型
3. 调用时可以显示指定前几个模板参数的类型
4. 可以为模板参数提供默认参数，且默认参数可以依赖于前面的参数
5. 函数模板可以被重载
6. 重载时要保证编译器只能匹配一个
7. 重载时尽量保证只有模板参数不同
8. 在调用函数模板时保证编译器可以看见所有的重载版本
