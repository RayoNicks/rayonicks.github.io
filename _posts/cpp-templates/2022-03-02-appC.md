---
layout: post
title: 《C++ Templates》附录C 重载解析规则
categories: Reading
tags: C++
---

# C 重载解析规则

这一部分应该是在第16章之后的，不过忘记了。

## C.1 重载解析规则的应用范围

重载解析规则应用在调用具名函数时，大致分为以下几个步骤：

1. 根据名称形成重载函数集合
2. 实例化函数模板
3. 剔除不匹配的函数形成可行函数集合
4. 确定是否存在最匹配的调用目标
5. 判断最匹配的函数是否被删除

## C.2 简化的重载解析规则

如果一个函数更为匹配，那么每一个参数都至少应该和另外一个函数一样或者更为匹配。原文：

>For one candidate to be considered better than another, the better candidate cannot have any of its parameters be a worse match than the corresponding parameter in the other candidate.

实参和形参的匹配程度优先级如下：

1. 完美匹配，形参和实参类型一样，或者形参类型是实参类型的引用（也可以包含`const`或者`volatile`）
2. 经过简单转换后的匹配，例如数组退化为指针，指针变为`const`指针
3. 经过类型提升转换后的匹配，即位宽较小的类型提升为位宽较大的类型
4. 经过标准转换后的匹配，例如整型转化为浮点型，派生类转换为公有继承的基类
5. 经用户定义转换后的匹配，也包括隐式转换，例如通过构造函数转换
6. 和变参函数的匹配，任意参数都可以和变参函数匹配（除了具有非平凡的构造函数的类类型）

```cpp
int f1(int);        // #1
int f1(double);     // #2
f1(4);              // calls #1 : perfect match (#2 requires a standard conversion)

int f2(int);        // #3
int f2(char);       // #4
f2(true);           // calls #3 : match with promotion (#4 requires stronger standard conversion)

class X {
    public:
        X(int);
};
int f3(X);          // #5
int f3(...);        // #6
f3(7);              // calls #5 : match with user-defined conversion (#6 requires a match with ellipsis)
```

此外这些规则是在模板参数推导之后起作用的：

```cpp
template<typename T> void strange(T&&, T&&);
template<typename T> void bizarre(T&&, double&&);


int main()
{
    strange(1.2, 3.4);  // OK: with T deduced to double
    double val = 1.2;
    strange(val, val);  // OK: with T deduced to double&
    strange(val, 3.4);  // ERROR: conflicting deductions
    bizarre(val, val);  // ERROR: lvalue val doesn’t match double&&
}
```

### C.2.1 成员函数中的隐含参数 {#The-Implied-Argument-for-Member-Functions}

调用非静态成员函数时，参数种包含隐含参数`*this`，为左值引用类型（如果是`const`成员函数则为常量左值引用类型），`*this`的解析过程和其它参数类似，因此可能会导致模糊调用：

```cpp
#include <cstddef>
class BadString {
    public:
        BadString(char const*);
        // ...

        // character access through subscripting:
        char& operator[] (std::size_t);     // #1
        char const& operator[] (std::size_t) const;

        // implicit conversion to null-terminated byte string:
        operator char* ();                  // #2
        operator char const* ();
        // ...
};

int main()
{
    BadString str("correkt");
    str[5] = 'c'; // possibly an overload resolution ambiguity!
}
```

`str[5]`包含`*this`和`5`两个参数，如果解析为`BadString::operator[]`，由于`std::size_t`一般为无符号类型，所以`5`需要进行类型提升转换；而如果解析为语言内置的下标运算符，由于其参数类型一般为`ptrdiff_t`（也就是`int`），则只需要隐式将`*this`转换为`char*`即可——二者对比没有更为匹配的重载函数。

如果可行函数集合中既包括静态成员函数，也包括非静态成员函数，则重载解析规则会忽略`this`参数。

默认情况下`this`为引用类型，但是从C++11开始可以显示声明为引用类型，同时允许右值对象调用隐式声明为左值引用类型的成员函数，例如：

```cpp
struct S {
    void f1();      // implicit *this parameter is an lvalue reference (see below)
    void f2() &&;   // implicit *this parameter is an rvalue reference
    void f3() &;    // implicit *this parameter is an lvalue reference
};

int main()
{
    S().f1();   // OK: old rule allows rvalue S() to match implied lvalue reference type S& of *this
    S().f2();   // OK: rvalue S() matches rvalue reference type of *this
    S().f3();   // ERROR: rvalue S() cannot match explicit lvalue reference type of *this
}
```

### C.2.2 完美匹配导致的模糊调用

从C++11开始，某一类型`X`可以匹配的类型包括`X`、`X&`、`const X&`和`X&&`，如果重载了以`X`和`X&`为参数的函数，则有可能出现模糊调用：

```cpp
void report(int);           // #1
void report(int&);          // #2
void report(int const&);    // #3

int main()
{
    for (int k = 0; k<10; ++k) {
        report(k);  // ambiguous: #1 and #2 match equally well
    }
    report(42);     // ambiguous: #1 and #3 match equally well
}
```

## C.3 重载细节

### C.3.1 非模板和特化模板的匹配

任何情况下，非模板函数的匹配度都比模板实例化后的函数匹配度高：

```cpp
template<typename T> int f(T);  // #1
void f(int);                    // #2

int main()
{
    return f(7); // ERROR: selects #2 , which doesn’t return a value
}
```

### C.3.2 隐式转换序列

隐式转换可以由一系列转换组成，但是最多只能包含一个用户定义的转换。例如下面的代码包含三个隐式转换，但是只有一个用户定义的隐式转换：

```cpp
class Base {
    public:
        operator short() const;
};

class Derived : public Base {
};

void count(int);

void process(Derived const& object)
{
    count(object); // matches with user-defined conversion
}
```

### C.3.3 指针转换

关于指针的标准转换包括：

1. 指针到`bool`类型的转换
2. 任意指针类型到`void*`类型的转换
3. 派生类指针到基类指针的转换
4. 基类成员指针到派生类成员指针的转换

第3种和第4种转换的优先级比第2种转换的优先级高，第2种转换比第1种转换的优先级高：

```cpp
void check(void*);  // #1
void check(bool);   // #2

void rearrange (Matrix* m)
{
    check(m); // calls #1
    // ...
}
```

具有继承关系的类之间的转换总是在具有更近继承关系的类之间进行：

```cpp
class Interface {
    // ...
};

class CommonProcesses : public Interface {
    // ...
};

class Machine : public CommonProcesses {
    // ...
};

char* serialize(Interface*);            // #1
char* serialize(CommonProcesses*);      // #2

void dump (Machine* machine)
{
    char* buffer = serialize(machine);  // calls #2
    // ...
}
```

### C.3.4 初始化列表的转换

`{}`中的参数可以转换为`std::initializer_list`、通过`std::initializer_list`构造的对象、通过多个参数构造的对象和聚合类型的对象：

```cpp
// overload/initlist.cpp
#include <initializer_list>
#include <string>
#include <vector>
#include <complex>
#include <iostream>

void f(std::initializer_list<int>) {
    std::cout << "#1\n";
}

void f(std::initializer_list<std::string>) {
    std::cout << "#2\n";
}

void g(std::vector<int> const& vec) {
    std::cout << "#3\n";
}

void h(std::complex<double> const& cmplx) {
    std::cout << "#4\n";
}

struct Point {
    int x, y;
};

void i(Point const& pt) {
    std::cout << "#5\n";
}

int main()
{
    f({1, 2, 3});                           // prints #1
    f({"hello", "initializer", "list"});    // prints #2
    g({1, 1, 2, 3, 5});                     // prints #3
    h({1.5, 2.5});                          // prints #4
    i({1, 2});                              // prints #5
}
```

注意，类型提升转换的优先级高于标准转换的优先级：

```cpp
// overload/initlistovl.cpp
#include <initializer_list>
#include <iostream>

void ovl(std::initializer_list<char>) {     // #1
    std::cout << "#1\n";
}
void ovl(std::initializer_list<int>) {      // #2
    std::cout << "#2\n";
}

int main()
{
    ovl({'h', 'e', 'l', 'l', 'o', '\0'});   // prints #1
    ovl({'h', 'e', 'l', 'l', 'o', 0});      // prints #2
}
```

当用初始化列表构造对象时，重载解析分为两个阶段（例外的情况是初始化列表为空时会将跳过第1阶段）：

1. 只考虑以`std::initializer_list`为唯一非默认模板参数的构造函数
2. 如果第1阶段没有匹配的函数，则尝试将初始化列表分割为多个参数来构造对象

下面是一个例子：

```cpp
// overload/initlistctor.cpp
#include <initializer_list>
#include <string>
#include <iostream>

template<typename T>
struct Array {
    Array(std::initializer_list<T>) {
        std::cout << "#1\n";
    }
    Array(unsigned n, T const&) {
        std::cout << "#2\n";
    }
};

void arr1(Array<int>) {
}

void arr2(Array<std::string>) {
}

int main()
{
    arr1({1, 2, 3, 4, 5});                      // prints #1
    arr1({1, 2});                               // prints #1
    arr1({10u, 5});                             // prints #1
    arr2({"hello", "initializer", "list"});     // prints #1
    arr2({10, "hello"});                        // prints #2
}
```

### C.3.5 函数和代理函数

当一个类重载了函数调用运算符，或者可以隐式转换为函数指针或者函数引用类型时，可能导致模糊调用：

```cpp
using FuncType = void (double, int);

class IndirectFunctor {
    public:
        // ...
        void operator()(double, double) const;
        operator FuncType*() const;
};

void activate(IndirectFunctor const& funcObj)
{
    funcObj(3, 5); // ERROR: ambiguous
}
```

这个例子和[C.2.1](#The-Implied-Argument-for-Member-Functions)中的原理是类似的。`funcObj(3, 5)`包含三个参数：`funcObj`、`3`和`5`。如果解析为函数调用运算符，则后两个参数都需要进行标准转换；而如果解析为`FuncType`类型的函数指针，则需要将`funcObj`隐式转换为函数指针类型——二者对比没有更为匹配的重载函数。

### C.3.6 其它重载情景

- 获取函数指针

```cpp
int numElems(Matrix const&);    // #1
int numElems(Vector const&);    // #2
// ...
int (*funcPtr)(Vector const&) = numElems; // selects #2
```

- 匹配构造函数

```cpp
#include <string>
class BigNum {
    public:
        BigNum(long n);                 // #1
        BigNum(double n);               // #2
        BigNum(std::string const&);     // #3
        // ...
        operator double();              // #4
        operator long();                // #5
        // ...
};

void initDemo()
{
    BigNum bn1(100103);                 // selects #1
    BigNum bn2("7057103224.095764");    // selects #3
    int in = bn1;                       // selects #5
}
```
