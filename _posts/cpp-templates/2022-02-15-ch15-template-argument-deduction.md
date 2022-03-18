---
layout: post
title: 《C++ Templates》第15章 模板参数推导
categories: Reading
tags: C++
---

# 15 模板参数推导

## 15.1 模板参数推导过程

模板参数推导过程就是用实参类型`A`去确定形参类型`P`的过程。原文：

>We describe it in terms of matching a type A (derived from the call argument type) to a parameterized type P (derived from the call parameter declaration).

如果相同形参类型的推导结果不一致，则推导失败。如果形参是传引用的，则实参被推导为引用类型；如果形参是传值的，则实参会发生类型退化（数组和函数转换为指针类型，`const`和`volatile`被丢弃）。

## 15.2 可推导的上下文 {#Deduced-Context}

除了最简单的形参类型`T`之外，复杂的形参类型也可以进行推导：

```cpp
template<typename T>
void f1(T*);

template<typename E, int N>
void f2(E(&)[N]);

template<typename T1, typename T2, typename T3>
void f3(T1 (T2::*)(T3*));

class S {
    public:
        void f(double*);
};

void g (int*** ppp)
{
    bool b[42];
    f1(ppp);    // deduces T to be int**
    f2(b);      // deduces E to be bool and N to be 42
    f3(&S::f);  // deduces T1 = void, T2 = S, and T3 = double
}
```

复杂类型可以通过递归的方式转换为多个可推导的上下文（deduced contexts，包括指针、引用、数组、函数、成员指针、模板标识等）从而进行推导。原文：

>Complex type declarations are built from more elementary constructs (pointer, reference, array, and function declarators; pointer-to-member declarators; template-ids; and so forth), and the matching process proceeds from the top-level construct and recurses through the composing elements. It is fair to say that most type declaration constructs can be matched in this way, and these are called deduced contexts.

不可推导的上下文包括：

- 受限名称，例如不能通过`Q<T>::X`推导`T`
- 包含非类型模板参数的表达式，例如不能通过`S<I+1>`推导`I`，也不能通过`int(&)[sizeof(S<T>)]`推导`T`

一个有意思的例子如下：

```cpp
// details/fppm.cpp
template<int N>
class X {
    public:
        using I = int;
        void f(int) {
        }
};

template<int N>
void fppm(void (X<N>::*p)(typename X<N>::I));

int main()
{
    fppm(&X<33>::f); // fine: N deduced to be 33
}
```

虽然`X<N>::I`不是可推导的上下文，但是`X<N>::*p`是可推导的上下文（`N`被推导为`33`），从而`void (X<N>::*p)(typename X<N>::I)`也可以被推导。

## 15.3 模板参数推导的特殊情况

第一种特殊情况是获取函数模板实例化后的地址：

```cpp
template<typename T>
void f(T, T);

void (*pf)(char, char) = &f;
```

形参类型为`void(T, T)`，实参类型为`void(char, char)`，因此`T`可以被推导为`char`，同时`pf`为`f<char>`的地址。

第二种特殊情况是类型转换函数模板：

```cpp
class S {
    public:
        template<typename T> operator T&();
};

void f(int (&)[20]);

void g(S s)
{
    f(s);
}
```

`f(s)`试图将`s`转换为`int (&)[20]`类型，实参类型为`int[20]`，形参类型为`T`，因此`T`被推导为`int[20]`。

## 15.4 std::initializer_list和模板参数推导

如果形参类型为`std::initializer_list<T>`，且实参中所有元素的类型一致，则`T`可以被推导：

```cpp
// deduce/initlist.cpp
#include <initializer_list>

template<typename T> void f(std::initializer_list<T>);
int main()
{
    f({2, 3, 5, 7, 9});                 // OK: T is deduced to int
    f({'a', 'e', 'i', 'o', 'u', 42});   // ERROR: T deduced to both char and int
}
```

## 15.5 模板参数包的推导

对于可变参数模板，形参类型可能和多个实参类型对应：

```cpp
template<typename First, typename... Rest>
void f(First first, Rest... rest);

void g(int i, double j, int* k)
{
    f(i, j, k); // deduces First to int, Rest to {double, int*}
}
```

当参数包展开时，还要保证参数包展开后相同形参的推导结果一致，见下面两个例子：

```cpp
template<typename T, typename U> class pair { };

template<typename T, typename... Rest>
    void h1(pair<T, Rest> const&...);
template<typename... Ts, typename... Rest>
    void h2(pair<Ts, Rest> const&...);

void foo(pair<int, float> pif, pair<int, double> pid, pair<double, double> pdd)
{
    h1(pif, pid); // OK: deduces T to int, Rest to {float, double}
    h2(pif, pid); // OK: deduces Ts to {int, int}, Rest to {float, double}
    h1(pif, pdd); // ERROR: T deduced to int from the 1st arg, but to double from the 2nd
    h2(pif, pdd); // OK: deduces Ts to {int, double}, Rest to {float, double}
}
```

```cpp
template<typename... Types> class Tuple { };

template<typename... Types>
bool f1(Tuple<Types...>, Tuple<Types...>);

template<typename... Types1, typename... Types2>
bool f2(Tuple<Types1...>, Tuple<Types2...>);

void bar(Tuple<short, int, long> sv,
Tuple<unsigned short, unsigned, unsigned long> uv)
{
    f1(sv, sv);     // OK: Types is deduced to {short, int, long}
    f2(sv, sv);     // OK: Types1 is deduced to {short, int, long}, Types2 is deduced to {short, int, long}
    f1(sv, uv);     // ERROR: Types is deduced to {short, int, long} from the 1st arg, but to funsigned short, unsigned, unsigned long} from the 2nd
    f2(sv, uv);     // OK: Types1 is deduced to {short, int, long}, Types2 is deduced to funsigned short, unsigned, unsigned long}
}
```

### 15.5.1 字面值运算符模板

贴个代码自己体会一下吧：

```cpp
template<char... cs>
int operator"" _B7()
{
    std::array<char,sizeof...(cs)> chars{cs...};    // initialize array of passed chars
    for (char c : chars) {                          // and use it (print it here)
        std::cout << "'" << c << "' ";
    }
    std::cout << '\n';
    return ...;
}

auto b = 01.3_B7;       // OK: deduces <'0', '1', '.', '3'>
auto c = 0xFF00_B7;     // OK: deduces <'0', 'x', 'F', 'F', '0', '0'>
auto d = 0815_B7;       // ERROR: 8 is no valid octal literal
auto e = hello_B7;      // ERROR: identifier hello_B7 is not defined
auto f = "hello"_B7;    // ERROR: literal operator _B7 does not match
```

## 15.6 右值引用

### 15.6.1 引用折叠规则

不允许直接定义引用的引用，但是在模板代换和使用类型别名时可能会出现多重引用，这时会应用引用折叠规则，具体为：内层引用的`const`和`volatile`会被丢弃，只有在内外层都为右值引用的情况下，最终结果才为右值引用：

```cpp
using RCI = int const&;
RCI volatile&& r = 42;  // OK: r has type int const&
using RRI = int&&;
RRI const&& rr = 42;    // OK: rr has type int&&
```

### 15.6.2 转发引用

当模板参数为传转发引用类型时，推导规则还会考虑实参是左值还是右值，如果实参是左值，推导结果为引用类型，否则为原始类型：

```cpp
template<typename T> void f(T&& p); // p is a forwarding reference

void g()
{
    int i;
    int const j = 0;
    f(i);   // argument is an lvalue; deduces T to int& and parameter p has type int&
    f(j);   // argument is an lvalue; deduces T to int const& parameter p has type int const&
    f(2);   // argument is an rvalue; deduces T to int parameter p has type int&&
}
```

`i`是左值，所以`T`被推导为`int&`，根据引用折叠规则，最终`p`的类型为`int&`，而`2`是右值，所以`T`被推导为`int`，最终`p`的类型为`int`。综上，引用折叠规则也适用于转发引用。

### 15.6.3 完美转发的实现

通过转发引用和引用折叠规则，可以在模板中实现传递参数类型和参数的左值右值属性：

```cpp
class C {
    // ...
};

void g(C&);
void g(C const&);
void g(C&&);

template<typename T>
void forwardToG(T&& x)
{
    g(static_cast<T&&>(x));     // forward x to g()
}

void foo()
{
    C v;
    C const c;
    forwardToG(v);              // eventually calls g(C&)
    forwardToG(c);              // eventually calls g(C const&)
    forwardToG(C());            // eventually calls g(C&&)
    forwardToG(std::move(v));   // eventually calls g(C&&)
}
```

`std::forward`可以通过上面的方式实现。

需要注意的是，完美转发规则并不转发参数的常量属性，所以会存在下面这种奇怪的现象：

```cpp
void g(int*);
void g(...);

template<typename T> void forwardToG(T&& x)
{
    g(std::forward<T>(x));  // forward x to g()
}

void foo()
{
    g(0);                   // calls g(int*)
    forwardToG(0);          // eventually calls g(...)
}
```

### 15.6.4 一些奇怪的推导规则

在下面的情况下，`&&`不会被解释为转发引用：

```cpp
template<typename T>
class X
{
    public:
        X(X&&);                                 // X is not a template parameter
        X(T&&);                                 // this constructor is not a function template

        template<typename Other> X(X<U>&&);     // X<U> is not a template parameter
        template<typename U> X(U, T&&);         // T is a template parameter from an outer template
};
```

## 15.7 SFINAE

简单来说，在代换函数模板**声明**中出现的错误不会被视为错误：

```cpp
template<typename T, unsigned N>
T* begin(T (&array)[N])
{
    return array;
}

template<typename Container>
typename Container::iterator begin(Container& c)
{
    return c.begin();
}

int main()
{
    std::vector<int> v;
    int a[10];
    ::begin(v);     // OK: only container begin() matches, because the first deduction fails
    ::begin(a);     // OK: only array begin() matches, because the second substitution fails
}
```

### 15.7.1 代换相关上下文

SFINAE规则起作用的范围就是代换相关上下文（immediate context），除了下面的这些代换位置，其余都可以视为代换相关上下文，也就是SFINAE规则起作用的范围：

- 类模板的定义
- 函数模板的定义
- 变量模板的初始化部分
- 模板的默认模板实参
- 成员变量的默认初始化部分
- 异常部分
- 合成类模板的默认函数

根据上面定义的作用范围，下面的代码会报错：

```cpp
template<typename T>
class Array {
    public:
        using iterator = T*;
};

template<typename T>
void f(Array<T>::iterator first, Array<T>::iterator last);

template<typename T>
void f(T*, T*);

int main()
{
    f<int&>(0, 0); // ERROR: substituting int& for T in the first function template instantiates Array<int&>, which then fails
}
```

## 15.8 推导过程的一些限制

### 15.8.1 允许的实参类型转换

下列情况中，形参类型不一定要和实参类型完全匹配：

1. 如果形参是传引用类型的，则形参在被`const`和`volatile`修饰的同时实参可以不必
2. 如果实参是指针或者成员指针类型，则可以被隐式转换为被`const`和`volatile`修饰
3. 形参类型可以是基类类型，而实参类型是派生类类型

### 15.8.2 类模板实参

在C++17之前，推导规则仅限于函数模板和成员函数模板，对于类模板必须显示指定实参：

```cpp
template<typename T>
class S {
    public:
        S(T b) : a(b) {
        }
    private:
        T a;
};

S x(12); // ERROR before C++17: the class template parameter T was not deduced from the constructor call argument 12
```

### 15.8.3 默认模板实参

默认模板实参不能用来推导模板形参：

```cpp
template<typename T>
void f (T x = 42)
{
}

int main()
{
    f<int>();   // OK: T = int
    f();        // ERROR: cannot deduce T from default call argument
}
```

### 15.8.4 函数模板的异常声明

在现代C++中已经废弃了。

## 15.9 显示指定函数模板实参

显示指定模板实参类型后，就省略了推导参数类型的过程，同时会将实参进行隐式类型转换。

## 15.10 从初始化和表达式进行推导

### 15.10.1 从auto进行推导

从C++11开始，使用`auto`可以让编译器自动推导类型，其原理和模板实参推导是类似的，例如对于下面的代码：

```cpp
template<typename Container>
void useContainer(Container const& container)
{
    auto pos = container.begin();
    while (pos != container.end()) {
        auto& element = *pos++;
        // ... // operate on the element
    }
}
```

其中的第一个`auto`等价于：

```cpp
template<typename T> void deducePos(T pos);
deducePos(container.begin());
```

第二个`auto`等价于：

```cpp
template<typename T> deduceElement(T& element);
deduceElement(*pos++);
```

由于`auto`和模板参数推导的原理一致，所以右值引用将解释为转发引用：

```cpp
int x;
auto&& rr = 42;     // OK: rvalue reference binds to an rvalue (auto = int)
auto&& lr = x;      // Also OK: auto = int& and reference collapsing makes lr an lvalue reference
```

此外，`auto`只推导主类型，因此`const`、指针等需要显示添加：

```cpp
template<typename T> struct X { T const m; };
auto const N = 400u;                            // OK: constant of type unsigned int
auto* gp = (void*)nullptr;                      // OK: gp has type void*
auto const S::*pm = &X<int>::m;                 // OK: pm has type int const X<int>::*
X<auto> xa = X<int>();                          // ERROR: auto in template argument
int const auto::*pm2 = &X<int>::m;              // ERROR: auto is part of the “declarator”
```

最后，`auto`也可以用于推导函数返回类型非类型模板参数。

### 15.10.2 从decltype进行推导

`decltype`可以推导变量、函数等的类型，也可以推导表达式的返回类型，此时还会带上表达式最终求值后的值类型：

```cpp
void g (std::string&& s)
{
    // check the type of s:
    std::is_lvalue_reference<decltype(s)>::value;       // false
    std::is_rvalue_reference<decltype(s)>::value;       // true (s as declared)
    std::is_same<decltype(s),std::string&>::value;      // false
    std::is_same<decltype(s),std::string&&>::value;     // true

    // check the value category of s used as expression:
    std::is_lvalue_reference<decltype((s))>::value;     // true (s is an lvalue)
    std::is_rvalue_reference<decltype((s))>::value;     // false
    std::is_same<decltype((s)),std::string&>::value;    // true (T& signals an lvalue)
    std::is_same<decltype((s)),std::string&&>::value;   // false
}
```

当推导表达式的类型时：

- 如果表达式是左值，则`decltype(e)`结果为`T&`
- 如果表达式是右值，则`decltype(e)`结果为`T&&`
- 如果表达式时过期值，则`decltype(e)`结果为`T`

### 15.10.3 从decltype(auto)推导

从C++14开始，可以使用`decltype(auto)`的写法，用来统一`decltype`上述的区别：

```cpp
auto element = *pos;
auto& element = *pos;
decltype(*pos) element = *pos;
decltype(auto) element = *pos;
```

第一行的写法将产生拷贝，第二行的写法仅在`*`运算符支持返回引用的情况下才能编译通过，第三行的写法综合了上面两种。为了避免写两遍表达式，从C++14开始，可以写为最后一行的形式。

### 15.10.4 auto推导的特殊情况

在同一行使用`auto`定义多个变量时，仅当变量类型一致时才是合法的：

```cpp
char c;
auto *cp = &c, d = c;   // OK
auto e = c, f = c+1;    // ERROR: deduction mismatch char vs. int
```

用`auto`作为递归函数的返回类型时，递归结束的情况应该写在前面，以让编译器可以推导递归函数的返回类型：

```cpp
auto f(int n)
{
    if (n <= 1) {
        return 1;           // return type is deduced to be int
    } else {
        return n*f(n-1);    // OK: type of f(n-1) is int and so is type of n*f(n-1)
    }
}
```

当用`auto`作为函数模板的返回类型时，函数体必须被实例化才能推导函数的返回类型：

```cpp
// deduce/resulttypetmpl.cpp
template<typename T, typename U>
auto addA(T t, U u) -> decltype(t+u)
{
    return t + u;
}

void addA(...);

template<typename T, typename U>
auto addB(T t, U u) -> decltype(auto)
{
    return t + u;
}

void addB(...);

struct X {
};

using AddResultA = decltype(addA(X(), X()));    // OK: AddResultA is void
using AddResultB = decltype(addB(X(), X()));    // ERROR: instantiation of addB<X> is ill-formed
```

### 15.10.5 结构化绑定（Structured Bindings）

C++17引入了结构化绑定，用于以下三种情形：

- 使用简单类类型初始化多个变量：

```cpp
struct MaybeInt { bool valid; int value; };
MaybeInt g();
auto const&& [b, N] = g(); // binds b and N to the members of the result of g()
```

- 使用数组初始化多个变量：

```cpp
int main() {
    double pt[3];
    auto& [x, y, z] = pt;
    x = 3.0; y = 4.0; z = 0.0;
    plot(pt);
}
```

- 从`std::tuple`、`std::pair`和`std::array`模板初始化变量（这些模板支持`std::tuple_element`系列模板）：

```cpp
#include <tuple>

std::tuple<bool, int> bi{true, 42};
auto [b, i] = bi;
int r = i; // initializes r to 42
```

### 15.10.6 泛型lambda表达式

编译器处理具有具体参数类型的lambda表达式的方法是创建一个闭包类型（closure type），并为该类型重载函数调用运算符`()`：

```cpp
[] (int i) {
    return i < 0;
}
```

上面的代码创建的闭包类型为：

```cpp
class SomeCompilerSpecificNameX
{
    public:
        SomeCompilerSpecificNameX(); // only callable by the compiler
        bool operator() (int i) const
        {
            return i < 0;
        }
};
```

当lambda表达式的参数类型为`auto`时，就变成了泛型lambda表达式，编译器处理的方式依然是创建一个闭包类型，但是是为该类型重载了函数调用运算符`()`成员模板：

```cpp
[] (auto i) {
    return i < 0;
}
```

上面的代码创建的闭包类型为：

```cpp
class SomeCompilerSpecificNameZ
{
    public:
        SomeCompilerSpecificNameZ(); // only callable by compiler
        template<typename T>
        auto operator() (T i) const
        {
            return i < 0;
        }
};
```

## 15.11 别名模板

编译器对别名模板的处理方式是首先进行别名代换，并用代换后的结果进行推导：

```cpp
// deduce/aliastemplate.cpp
template<typename T, typename Cont>
class Stack;

template<typename T>
using DequeStack = Stack<T, std::deque<T>>;

template<typename T, typename Cont>
void f1(Stack<T, Cont>);

template<typename T>
void f2(DequeStack<T>);

template<typename T>
void f3(Stack<T, std::deque<T>); // equivalent to f2

void test(DequeStack<int> intStack)
{
    f1(intStack);   // OK: T deduced to int, Cont deduced to std::deque<int>
    f2(intStack);   // OK: T deduced to int
    f3(intStack);   // OK: T deduced to int
}
```

- 调用`f1()`时，实参类型为`Stack<int, std::deque<int>>`，使用该类型推导`T`和`Cont`
- 因为`DequeStack<T>`就是`Stack<T, std::deque<T>>`，所以`f2()`和`f3()`等价，因此使用`Stack<int, std::deque<int>>`推导`T`

## 15.12 类模板参数推导

从C++17开始，也可以根据类的构造函数对类模板参数进行推导，但是一个原则是要么全部指定类模板参数的类型，要么全部让编译器进行推导：

```cpp
template<typename T1, typename T2, typename T3 = T2>
class C
{
    public:
        // constructor for 0, 1, 2, or 3 arguments:
        C (T1 x = T1{}, T2 y = T2{}, T3 z = T3{});
        // ...
};

C c1(22, 44.3, "hi");   // OK in C++17: T1 is int, T2 is double, T3 is char const*
C c2(22, 44.3);         // OK in C++17: T1 is int, T2 and T3 are double
C c3("hi", "guy");      // OK in C++17: T1, T2, and T3 are char const*
C c4;                   // ERROR: T1 and T2 are undefined
C c5("hi");             // ERROR: T2 is undefined

C<string> c10("hi","my", 42);       // ERROR: only T1 explicitly specified, T2 not deduced
C<> c11(22, 44.3, 42);              // ERROR: neither T1 nor T2 explicitly specified
C<string,string> c12("hi","my");    // OK: T1 and T2 are deduced, T3 has default
```

### 15.12.1 推导指引

C++17引入了推导指引，用来推导类模板参数：

```cpp
template<typename T>
class S {
    private:
        T a;
    public:
        S(T b) : a(b) {
        }
};

template<typename T> S(T) -> S<T>; // deduction guide

S x{12};            // OK since C++17, same as: S<int> x{12};
S y(12);            // OK since C++17, same as: S<int> y(12);
auto z = S{12};     // OK since C++17, same as: auto z = S<int>{12};
```

推导指引可以理解为当用类型`T`初始化类`S`时，模板参数的类型就为`T`。

### 15.12.2 隐式推导指引

在引入了推导指引的情况下，类模板的每一个构造函数都有隐式的推导指引。如果想要禁掉某一隐式的推导指引，可以将相关构造函数写为这样子：

```cpp
template<typename T>
struct ValueArg {
    using Type = T;
};

template<typename T>
class S {
    private:
        T a;
    public:
        using ArgType = typename ValueArg<T>::Type;
        S(ArgType b) : a(b) {
        }
};
```

这将产生类似`template<typename> S(typename ValueArg<T>::Type) -> S<T>`的推导指引，而`ValueArg<T>::Type`并不是可推导的上下文（参见[15.2](#Deduced-Context)）。

### 15.12.3 其它问题

#### 在注入类名称的情况下禁用类模板参数推导

```cpp
template<typename T> struct X {
    template<typename Iter> X(Iter b, Iter e);
    template<typename Iter> auto f(Iter b, Iter e) {
        return X(b, e); // What is this?
    }
};
```

`X(b, e)`中的`X`应该是等价于`X<T>`的，而不是`X<Iter>`，所以在这种情况下禁用了推导。

#### 在转发引用的情况下不考虑值类型

```cpp
template<typename T> struct Y {
    Y(T const&);
    Y(T&&);
};

void g(std::string s) {
    Y y = s;
}
```

上面代码包含的两条隐式推导指引为：

```cpp
template<typename T> Y(T const&) -> Y<T>;   // #1
template<typename T> Y(T&&) -> Y<T>;        // #2
```

对于第一条推导指引，`T`被推导为`std::string`，所以在调用时需要将参数转换为`std::string const`类型。对于第二条推导指引，`T`仍然被推导为`std::string`，但是由于参数类型是转发引用，还会考虑值类型（在这里是左值），根据引用折叠规则得到的参数类型为`std::string&`，明显更为匹配，但是这并不是我们想要的结果，所以这种情况下的推导不考虑值类型。

#### 推导指引中的explicit

```cpp
template<typename T, typename U> struct Z {
    Z(T const&);
    Z(T&&);
};

template<typename T> Z(T const&) -> Z<T, T&>;       // #1
template<typename T> explicit Z(T&&) -> Z<T, T>;    // #2

Z z1 = 1;   // only considers #1 ; same as: Z<int, int&> z1 = 1;
Z z2{2};    // prefers #2 ; same as: Z<int, int> z2{2};
```

当推导指引中带有`explicit`关键字时，只有直接初始化的情况才可能有效。

#### 拷贝构造和初始化列表

```cpp
template<typename ... Ts> struct Tuple {
    Tuple(Ts...);
    Tuple(Tuple<Ts...> const&);
};
```

上面代码包含的推导指引为：

```cpp
template<typename... Ts> Tuple(Ts...) -> Tuple<Ts...>;
template<typename... Ts> Tuple(Tuple<Ts...> const&) -> Tuple<Ts...>;
```

对于下面的初始化过程：

```cpp
auto x = Tuple{1,2};
Tuple a = x;
Tuple b(x);
Tuple c{x, x};
Tuple d{x};
```

1. `x`的类型为`Tuple<int,int>`
2. `a`和`b`同时匹配两个推导指引（类型分别为`Tuple<Tuple<int, int>>`和`Tuple<int, int>`），但是第二个更匹配，所以类型为`Tuple<int, int>`
3. `c`的类型为`Tuple<Tuple<int, int>, Tuple<int, int>>`
4. `d`的类型为`Tuple<int, int>`

#### 显示的推导指引只用来推导

```cpp
template<typename T> struct X {
    // ...
};

template<typename T> struct Y {
    Y(X<T> const&);
    Y(X<T>&&);
};

template<typename T> Y(X<T>) -> Y<T>;
```

上面的推导指引并不和任一构造函数对应，它只是指出当以`X<T>`类型构造`Y`时，类模板参数类型也为`T`。

## 15.13 后记
