---
layout: post
title: 《C++ Templates》第8章 编译时编程
categories: Reading
tags: C++
---

# 8 编译时编程

## 8.1 模板元编程

模板是编译时实例化的，所以可以在编译时将模板递归的展开，从而生成程序，例如下面的素性判断程序：

```cpp
// basics/isprime.hpp
template<unsigned p, unsigned d>    // p: number to check, d: current divisor
struct DoIsPrime {
    static constexpr bool value = (p%d != 0) && DoIsPrime<p,d-1>::value;
};

template<unsigned p>                // end recursion if divisor is 2
struct DoIsPrime<p,2> {
    static constexpr bool value = (p%2 != 0);
};

template<unsigned p>                // primary template
struct IsPrime {
    // start recursion with divisor from p/2:
    static constexpr bool value = DoIsPrime<p,p/2>::value;
};

// special cases (to avoid endless recursion with template instantiation):
template<>
struct IsPrime<0> { static constexpr bool value = false; };
template<>
struct IsPrime<1> { static constexpr bool value = false; };
template<>
struct IsPrime<2> { static constexpr bool value = true; };
template<>
struct IsPrime<3> { static constexpr bool value = true; };
```

对于`IsPrime<9>::value`，模板实例化的过程如下：

- `IsPrime<9>::value`
- `DoIsPrime<9,4>::value`
- `9%4!=0 && DoIsPrime<9,3>::value`
- `9%4!=0 && 9%3!=0 && DoIsPrime<9,2>::value`
- `9%4!=0 && 9%3!=0 && 9%2!=0`

由于该值可以在编译期间被计算，所以`IsPrime<9>::value`的最终结果为`false`。

## 8.2 constexpr

被`constexpr`修饰的函数可以在编译期间求值，这要保证函数中的每一步都能够在编译期间进行求值，依然以素性判断程序为例：

```cpp
// basics/isprime11.hpp
constexpr bool
doIsPrime (unsigned p, unsigned d)  // p: number to check, d: current divisor
{
    return d!=2 ? (p%d!=0) && doIsPrime(p,d-1)  // check this and smaller divisors
                : (p%2!=0);                     // end recursion if divisor is 2
}
constexpr bool isPrime (unsigned p)
{
    return p < 4 ? !(p<2)           // handle special cases
            : doIsPrime(p,p/2);     // start recursion with divisor from p/2
}
```

从C++14开始，被`constexpr`的函数中也可以使用较为复杂的程序控制结构：

```cpp
// basics/isprime14.hpp
constexpr bool isPrime (unsigned int p)
{
    for (unsigned int d=2; d<=p/2; ++d) {
        if (p % d == 0) {
            return false;   // found divisor without remainder
        }
    }
    return p > 1;           // no divisor without remainder found
}
```

## 8.3 通过偏特化实现执行路径选择

下面的代码根据`SZ`的素性使用了不同的结构体：

```cpp
// primary helper template:
template<int SZ, bool = isPrime(SZ)>
struct Helper;

// implementation if SZ is not a prime number:
template<int SZ>
struct Helper<SZ, false>
{
    // ...
};

// implementation if SZ is a prime number:
template<int SZ>
struct Helper<SZ, true>
{
    // ...
};

template<typename T, std::size_t SZ>
long foo (std::array<T,SZ> const& coll)
{
    Helper<SZ> h; // implementation depends on whether array has prime number as size
    // ...
}
```

注意只能对类模板进行偏特化，如果要在编译期间对函数模板中的执行路径进行选择，可以使用：

- 带静态函数的类
- `std::enable_if`
- SFINAE特性
- 编译时`if`

## 8.4 SFINAE

调用重载函数时，编译器会考查每个重载函数的匹配度，然后最终决定要调用哪个函数。当重载函数中包括函数模板时，编译器会推导函数模板中模板参数的类型，然后将其视为一个普通函数来考查匹配度。但是在类型推导的过程中可能会产生毫无意义的结果，与其将其视为错误，语言更倾向于忽略它——这就是**代换失败不是错误（Substitution Failure Is Not An Error，SFINAE，读作sfee-nay）**原则。

这里提到的代换和第2章中提到的部分实例化类模板不是一个概念。编译器可能会通过代换生成一些无用的函数签名，这些模板最终不会变成重载函数的定义（这是我猜的）。

如果有下面两个函数模板定义：

```cpp
// basics/len1.hpp
// number of elements in a raw array:
template<typename T, unsigned N>
std::size_t len (T(&)[N])
{
    return N;
}

// number of elements for a type having size_type:
template<typename T>
typename T::size_type len (T const& t)
{
    return t.size();
}
```

和下面的调用代码：

```cpp
int a[10];
std::cout << len(a);        // OK: only len() for array matches
std::cout << len("tmp");    // OK: only len() for array matches

std::vector<int> v;
std::cout << len(v);        // OK: only len() for a type with size_type matches

int* p;
std::cout << len(p);        // ERROR: no matching len() function found

std::allocator<int> x;
std::cout << len(x);        // ERROR: len() function found, but can’t size()
```

虽然两个模板都可以匹配`len(a)`和`len("tmp")`，但是`int [10]`和`char const[4]`类型中没有`size_type`成员，所以会匹配第一个函数模板，同时编译器不会因为第二个函数模板代换错误而报错。但是对于`len(x)`，由于只能匹配第二个，但是却没有`size()`成员，所以会触发编译时错误。

代换中的错误可以迫使编译器去选择其它并不精确的匹配：

```cpp
// basics/len2.hpp
// number of elements in a raw array:
template<typename T, unsigned N>
std::size_t len (T(&)[N])
{
    return N;
}

// number of elements for a type having size_type:
template<typename T>
typename T::size_type len (T const& t)
{
    return t.size();
}

// fallback for all other types:
std::size_t len (...)
{
    return 0;
}
```

此时`len(p)`会匹配最后一个函数模板，但是`len(x)`仍然会匹配第二个函数模板，还是无法避免触发编译时错误。

### 8.4.1 decltype和SFINAE

为了解决上面的问题，可以将模板定义为：

```cpp
template<typename T>
auto len (T const& t) -> decltype( (void)(t.size()), T::size_type() )
{
    return t.size();
}
```

`decltype`中的`,`运算符保证了表达式的返回值为`T::size_type()`，这也是函数模板的返回类型，但是在此之前需要先保证表达式是合法的，也就是`t.size()`不会报错。

## 8.5 编译时if

C++17引入了编译时`if`语句来实现模板的选择，回顾第4章中的`print()`模板，使用编译时`if`的改写版本如下：

```cpp
template<typename T, typename... Types>
void print (T const& firstArg, Types const&... args)
{
    std::cout << firstArg << '\n';
    if constexpr(sizeof...(args) > 0) {
        print(args...); // code only available if sizeof...(args)>0 (since C++17)
    }
}
```

当函数参数包`args`为空时，`sizeof...(args)`为`0`，可以不用继续实例化`print()`，递归就可以结束了。

`if constexpr`是在两阶段翻译的第二阶段中起作用的：

```cpp
template<typename T>
void foo(T t)
{
    if constexpr(std::is_integral_v<T>) {
        if (t > 0) {
            foo(t-1); // OK
        }
    }
    else {
        undeclared(t);  // error if not declared and not discarded (i.e. T is not integral)
        undeclared();   // error if not declared (even if discarded)
        static_assert(false, "no integral"); // always asserts (even if discarded)
        static_assert(!std::is_integral_v<T>, "no integral"); // OK
    }
}
```

在两阶段翻译的第一阶段，要保证能够找到`undeclared(t)`和`undeclared()`的声明，同时`static_assert(false, "no integral")`生效；在两阶段翻译的第二阶段，`if constexpr`中的条件会被计算，如果为假，则要求以`T`类型为参数的`undeclared()`存在。

`if constexpr`也可以用在普通函数中，功能应该是只是迫使编译器在编译时计算条件表达式的结果。

```cpp
int main()
{
    if constexpr(std::numeric_limits<char>::is_signed) {
        foo(42); // OK
    }
    else {
        undeclared(42); // error if undeclared() not declared
        static_assert(false, "unsigned"); // always asserts (even if discarded)
        static_assert(!std::numeric_limits<char>::is_signed, "char is unsigned"); // OK
    }
}
```

## 8.6 总结

1. 通过递归、偏特化等技术可以在编译时将模板展开并在编译时进行求值
2. `constexpr`使得普通函数可以在编译时进行计算
3. 通过偏特化技术，可以实现根据编译时条件选择实例化的类模板
4. 当在代换的过程中不会产生错误代码时，函数模板才可能被实例化
5. 可以使用SFINAE规则来实现特定模板的选择
6. C++17支持编译时`if`

第2条原文：

>With constexpr functions, we can replace most compile-time computations with “ordinary functions” that are callable in compile-time contexts.

第4条原文：

>Templates are used only if needed and substitutions in function template declarations do not result in invalid code. This principle is called SFINAE (substitution failure is not an error).
