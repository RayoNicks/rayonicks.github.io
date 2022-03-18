---
layout: post
title: 《C++ Templates》第16章 特化和重载
categories: Reading
tags: C++
---

# 16 特化和重载

## 16.1 泛型代码无法解决的问题

考虑下面的例子：

```cpp
template<typename T>
class Array {
    private:
        T* data;
        // ...
    public:
        Array(Array<T> const&);
        Array<T>& operator= (Array<T> const&);
        void exchangeWith (Array<T>* b) {
            T* tmp = data;
            data = b->data;
            b->data = tmp;
        }
        T& operator[] (std::size_t k) {
            return data[k];
        }
        // ...
};

template<typename T> inline
void exchange (T* a, T* b)
{
    T tmp(*a);
    *a = *b;
    *b = tmp;
}
```

上例中的`exchange()`适用于简单类型，对于具有较大拷贝开销的`Array<>`不是最好的实现，成员函数`exchangeWith()`是更好的选择，但是却无法实现一个泛型算法。

### 16.1.1 通过特化提供相同的接口

为了实现泛型算法，需要提供一个特化版本，并在其中调用`exchangeWith()`：

```cpp
template<typename T>
void quickExchange(T* a, T* b) // #1
{
    T tmp(*a);
    *a = *b;
    *b = tmp;
}

template<typename T>
void quickExchange(Array<T>* a, Array<T>* b) // #2
{
    a->exchangeWith(b);
}

void demo(Array<int>* p1, Array<int>* p2)
{
    int x=42, y=-7;
    quickExchange(&x, &y);  // uses #1
    quickExchange(p1, p2);  // uses #2
}
```

虽然`quickExchange(p1, p2)`可以匹配两个版本的`quickExchange()`，但是第二个比第一个更为特化（具体规则见[16.2.3](#Formal-Ordering-Rules)）。

### 16.1.2 特化和语义一致性

在下面的例子中，`quickExchange()`并不能保证语义一致性：

```cpp
struct S {
    int x;
} s1, s2;

void distinguish (Array<int> a1, Array<int> a2)
{
    int* p = &a1[0];
    int* q = &s1.x;
    a1[0] = s1.x = 1;
    a2[0] = s2.x = 2;
    quickExchange(&a1, &a2);    // *p == 1 after this (still)
    quickExchange(&s1, &s2);    // *q == 2 after this
}
```

在第一个`quickExchange()`后，`p`和`a2.data`指向同一缓冲区，如果保证语义一致性，应该这样写（虽然有bug）：

```cpp
template<typename T>
void exchange (Array<T>* a, Array<T>* b)
{
    T* p = &(*a)[0];
    T* q = &(*b)[0];
    for (std::size_t k = a->size(); k-- != 0; ) {
        exchange(p++, q++);
    }
}
```

## 16.2 函数模板的重载 {#Overloading-Function-Templates}

假设有下面两个模板：

```cpp
// details/funcoverload1.hpp
template<typename T>
int f(T)
{
    return 1;
}

template<typename T>
int f(T*)
{
    return 2;
}
```

当用`int*`实例化`f(T)`且用`int`实例化`f(T*)`时，将会得到两个相同的函数，这两个函数也是可以共存的，例如下面的代码：

```cpp
// details/funcoverload1.cpp
#include <iostream>
#include "funcoverload1.hpp"

int main()
{
    std::cout << f<int*>((int*)nullptr);    // calls f<T>(T)
    std::cout << f<int>((int*)nullptr);     // calls f<T>(T*)
}
```

### 16.2.1 函数签名

上例中实例化后的函数之所以可以共存是因为函数签名不同，函数签名包含以下内容：

1. 函数的非受限名称
2. 函数名称所在的类和命名空间，如果是内部链接的名称，还包括翻译单元的名称
3. `const`、`volatile`修饰符
4. 如果是成员函数，还会包含左值、右值引用限定符
5. 函数的形参类型，如果是函数模板，则是代换前的类型
6. 如果是函数模板，还包括返回类型
7. 如果是函数模板，还包括模板形参类型和模板实参类型

这就意味下面的的函数模板及实例化的结果可能可以共存：

```cpp
template<typename T1, typename T2>
void f1(T1, T2);

template<typename T1, typename T2>
void f1(T2, T1);

template<typename T>
long f2(T);

template<typename T>
char f2(T);
```

但是在解析`f1<char, char>('a', 'b')`时，可能找不到最佳匹配的函数。

### 16.2.2 重载的函数模板的匹配顺序 {#Partial-Ordering-of-Overloaded-Function-Templates}

假设将[16.2](#Overloading-Function-Templates)中的`details/funcoverload1.cpp`替换为下面的代码：

```cpp
// details/funcoverload2.cpp
#include <iostream>

template<typename T>
int f(T)
{
    return 1;
}

template<typename T>
int f(T*)
{
    return 2;
}

int main()
{
    std::cout << f(0);              // calls f<T>(T)
    std::cout << f(nullptr);        // calls f<T>(T)
    std::cout << f((int*)nullptr);  // calls f<T>(T*)
}
```

匹配过程为：

- `0`的类型是`int`，显然只能匹配`f<int>(int)`
- `nullptr`的类型是`std::nullptr_t`，也只能匹配`f<std::nullptr_t>(std::nullptr_t)`
- `(int*)nullptr`的类型是`int*`，可以匹配`f<int*>(int*)`，也可以匹配`f<int>(int*)`，而`f<T>(T*)`相对于`f<T>(T)`则是更为特化的版本（具体规则见[16.2.3](#Formal-Ordering-Rules)），所以将匹配`f<T>(T*)`

### 16.2.3 重载的函数模板的匹配规则 {#Formal-Ordering-Rules}

重载的函数模板之间仅可能存在偏序的关系，也就是只有一个可能比另一个更为特化。重载解析的过程如下：

1. 没有使用的具有默认实参的形参和没有使用的可变参数部分将被忽略
2. 根据以下规则形成参数列表：
  - 使用独特的构造类型（a unique invented type）替换模板形参
  - 使用独特的构造类类型（a unique invented class template）替换模板的模板形参
  - 使用独特的构造值（a unique invented value）替换非类型模板形参
3. 如果函数模板2的推导类型和参数列表1完全匹配，但是反过来不能，则得到参数列表1的函数模板更为特化，否则两个函数模板之间不存在更为特化的偏序关系

匹配规则的原文：

>If template argument deduction of the second template against the first synthesized list of argument types succeeds with an exact match, but not vice versa, then the first template is more specialized than the second. Conversely, if template argument deduction of the first template against the second synthesized list of argument types succeeds with an exact match, but not vice versa, then the second template is more specialized than the first. Otherwise (either no deduction succeeds or both succeed), there is no ordering between the two templates.

以前一节（[16.2.2](#Partial-Ordering-of-Overloaded-Function-Templates)）中代码为例：解析`f((int*)nullptr)`时，两个函数模板实例化的结果分别为`f<int*>(int*)`和`f<int>(int*)`，形参是一样的，参数列表分别为`(A1)`和`(A2*)`，用`A2*`代换第一个函数模板中的`T`后可以和参数列表2完全匹配，而用任何类型代换第二个函数模板中的`T*`都将得到指针类型，和参数列表1无法匹配，因此`f<T>(T*)`更为特化。

书中还有一个例子：

```cpp
template<typename T>
void t(T*, T const* = nullptr, ...);

template<typename T>
void t(T const*, T*, T* = nullptr);

void example(int* p)
{
    t(p, p);
}
```

第一个函数模板中的可变形参和第二个函数模板中的具有默认实参的第三个参数没有使用，所以将被忽略。根据模板参数推导规则，参数列表1为`(A1*, A1 const*)`，参数列表2为`(A2 const*, A2*)`，因此没有办法将参数列表1代换为参数列表2，也没有办法将参数列表2代换为参数列表1，所以两个函数模板之间不存在偏序关系，重载解析过程将会失败。

看完整本书后再回头在看这里，觉得书上解释不是特别恰当。解析`t(p, p)`时，两个函数模板实例化后的结果分别为`t<int>(int*, int const*)`和`t<int>(int const*, int*)`，第一个函数的第一个形参比第二个函数的第一个形参更为匹配，而第二个函数的第二个形参却比第一个函数的第二个形参更为匹配，因此这里存在模糊调用。

### 16.2.4 函数模板和普通函数之间重载

重载解析过程将首先匹配普通函数：

```cpp
// details/nontmpl1.cpp
#include <string>
#include <iostream>
template<typename T>
std::string f(T)
{
    return "Template";
}

std::string f(int&)
{
    return "Nontemplate";
}

int main()
{
    int x = 7;
    std::cout << f(x) << '\n'; // prints: Nontemplate
}
```

当需要`const`转换或者引用转换时，如果普通函数不能精确匹配，则重载解析过程会尝试匹配函数模板；如果其余方面都相同，则重载解析过程会优先匹配普通函数，例如下面的代码：

```cpp
// details/nontmpl2.cpp
#include <string>
#include <iostream>

template<typename T>
std::string f(T&)
{
    return "Template";
}

std::string f(int const&)
{
    return "Nontemplate";
}

int main()
{
    int x = 7;
    std::cout << f(x) << '\n';  // prints: Template
    int const c = 7;
    std::cout << f(c) << '\n';  // prints: Nontemplate
}
```

当成员函数模板作为构造函数时，可能会比普通的构造函数匹配度更高：

```cpp
// details/tmplconstr.cpp
#include <string>
#include <iostream>

class C {
    public:
        C() = default;
        C (C const&) {
            std::cout << "copy constructor\n";
        }
        C (C&&) {
            std::cout << "move constructor\n";
        }
        template<typename T>
        C (T&&) {
            std::cout << "template constructor\n";
        }
};

int main()
{
    C x;
    C x2{x};                // prints: template constructor
    C x3{std::move(x)};     // prints: move constructor
    C const c;
    C x4{c};                // prints: copy constructor
    C x5{std::move(c)};     // prints: template constructor
}
```

匹配过程为：

- 初始化`x2`时，`T`的推导类型为`C&`，折叠后模板构造函数的参数类型为`C&`，比拷贝构造函数更为匹配
- 初始化`x3`时，`T`的推导类型为`C&&`，折叠后模板构造函数的参数类型为`C&&`，和移动构造函数一致，优先匹配非模板函数
- 初始化`x4`时，`T`的推导类型为`C const&`，折叠后模板构造函数的参数类型为`C const&`，和拷贝构造函数一致，优先匹配非模板函数
- 初始化`x5`时，`T`的推导类型为`C const&&`，折叠后模板构造函数的参数类型为`C const&&`，比移动构造函数更为匹配

### 16.2.5 可变参数模板的重载

由于可变参数模板的一个形参类型会对应多个实参类型，匹配情况就会有些复杂：

```cpp
// details/variadicoverload.cpp
#include <iostream>

template<typename T>
int f(T*)
{
    return 1;
}

template<typename... Ts>
int f(Ts...)
{
    return 2;
}

template<typename... Ts>
int f(Ts*...)
{
    return 3;
}

int main()
{
    std::cout << f(0, 0.0);                             // calls f<>(Ts...)
    std::cout << f((int*)nullptr, (double*)nullptr);    // calls f<>(Ts*...)
    std::cout << f((int*)nullptr);                      // calls f<>(T*)
}
```

匹配过程为：

- `f(0, 0.0)`有两个参数，且没有指针类型，所以只能匹配`f<>(Ts...)`
- `f((int*)nullptr, (double*)nullptr)`可以同时匹配两个可变参数模板，参数列表分别为`(A1)`和`(A2*)`，显然参数列表1可以代换为参数列表2，但是反过来不能，也就是得到参数列表2的函数模板更为特化
- `f((int*)nullptr)`可以同时匹配三个函数模板，参数列表分别为`(A1*)`、`(A2)`和`(A3*)`，显然参数列表2不够特化，所以接下来要比较参数列表1和参数列表3特化偏序关系。虽然可以双向代换，但是由于`A3`是参数包，单个参数`A1`不能转换为参数包，所以得到参数列表1的函数模板更为特化——这条规则也意味着普通函数模板比可变参数函数模板更为特化

上述规则也适用于需要参数包展开的可变参数模板：

```cpp
// details/tupleoverload.cpp
#include <iostream>

template<typename... Ts> class Tuple
{
};

template<typename T>
int f(Tuple<T*>)
{
    return 1;
}

template<typename... Ts>
int f(Tuple<Ts...>)
{
    return 2;
}

template<typename... Ts>
int f(Tuple<Ts*...>)
{
    return 3;
}

int main()
{
    std::cout << f(Tuple<int, double>());       // calls f<>(Tuple<Ts...>)
    std::cout << f(Tuple<int*, double*>());     // calls f<>(Tuple<Ts*...>)
    std::cout << f(Tuple<int*>());              // calls f<>(Tuple<T*>)
}
```

## 16.3 显示特化

显示特化一般是指将全部的模板形参代换为模板实参后得到的具体实现代码，也称为全特化（full specialization）。类模板、函数模板、变量模板和成员模板可以被全特化。

### 16.3.1 类模板的全特化

全特化类模板以`template<>`开始，且特化的结果可以和原始模板没有关系，例如：

```cpp
template<typename T>
class S {
    public:
        void info() {
            std::cout << "generic (S<T>::info())\n";
        }
};

template<>
class S<void> {
    public:
        void msg() {
            std::cout << "fully specialized (S<void>::msg())\n";
        }
};
```

全特化时要为所有的模板形参提供模板实参（具有默认实参的模板形参除外）：

```cpp
template<typename T>
class Types {
    public:
        using I = int;
};

template<typename T, typename U = typename Types<T>::I>
class S;                            // #1

template<>
class S<void> {                     // #2
    public:
        void f();
};

template<> class S<char, char>;     // #3

template<> class S<char, 0>;        // ERROR: 0 cannot substitute U

int main()
{
    S<int>* pi;         // OK: uses #1 , no definition needed
    S<int> e1;          // ERROR: uses #1, but no definition available
    S<void>* pv;        // OK: uses #2
    S<void,int> sv;     // OK: uses #2, definition available
    S<void,char> e2;    // ERROR: uses #1, but no definition available
    S<char,char> e3;    // ERROR: uses #3, but no definition available
}

template<>
class S<char, char> {   // definition for #3
};
```

上例还表明，全特化可以只是声明。一旦全特化被声明，编译器将不再考虑模板定义，即全特化的实例和普通类是一样的，唯一的不同就是需要对应一个模板定义，因此定义成员函数时不需要前缀`template<>`：

```cpp
template<typename T>
class S;

template<> class S<char**> {
    public:
        void print() const;
};

// the following definition cannot be preceded by template<>
void S<char**>::print() const
{
    std::cout << "pointer to pointer to char\n";
}
```

当编译器已经通过模板定义生成了特化版本后，将不能再声明全特化版本：

```cpp
template<typename T>
class Invalid {
};

Invalid<double> x1;     // causes the instantiation of Invalid<double>

template<>
class Invalid<double>;  // ERROR: Invalid<double> already instantiated
```

### 16.3.2 函数模板的全特化

函数模板的全特化会考虑模板之间的特化偏序关系，例如：

```cpp
template<typename T>
int f(T)    // #1
{
    return 1;
}

template<typename T>
int f(T*)   // #2
{
    return 2;
}

template<> int f(int)   // OK: specialization of #1
{
    return 3;
}

template<> int f(int*)  // OK: specialization of #2
{
    return 4;
}
```

函数模板全特化时不能包含默认实参，因为会被视为一个新的函数定义：

```cpp
template<typename T>
int f(T, T x = 42)
{
    return x;
}

template<> int f(int, int = 35) // ERROR
{
    return 0;
}
```

### 16.3.3 变量模板的全特化

举个例子：

```cpp
template<typename T> constexpr std::size_t SZ = sizeof(T);

template<> constexpr std::size_t SZ<void> = 0;
```

### 16.3.4 成员模板的全特化

本节中首先定义了一个类模板：

```cpp
template<typename T>
class Outer {                       // #1
    public:
        template<typename U>
        class Inner {               // #2
            private:
                static int count;   // #3
        };
        static int code;            // #4
        void print() const {        // #5
            std::cout << "generic";
        }
};

template<typename T>
int Outer<T>::code = 6;             // #6

template<typename T> template<typename U>
int Outer<T>::Inner<U>::count = 7;  // #7
```

并对外层类模板进行了特化：

```cpp
template<>
class Outer<bool> {                 // #8
    public:
        template<typename U>
        class Inner {               // #9
            private:
                static int count;   // #10
        };
        void print() const {        // #11
        }
};
```

可以仅对类的静态数据成员和成员函数进行特化，类中其余部分将仍然从模板定义生成：

```cpp
template<>
int Outer<void>::code = 12;

template<>
void Outer<void>::print() const
{
    std::cout << "Outer<void>";
}
```

如果仅对类的静态数据成员和成员函数进行声明，应该写为：

```cpp
template<>
int Outer<void>::code;

template<>
void Outer<void>::print() const;
```

静态数据成员的全特化声明虽然看起来像通过默认值进行初始化的定义，但是对于模板来说这被解释为声明。

本节开头定义的模板中还包含内嵌类，只可以逐次特化：

```cpp
template<>
    template<typename X>
    class Outer<wchar_t>::Inner {
        public:
            static long count; // member type changed
    };

template<>
    template<typename X>
    long Outer<wchar_t>::Inner<X>::count;

template<>
    template<>
    class Outer<char>::Inner<wchar_t> {
        public:
            enum { count = 1 };
    };

// the following is not valid C++: template<> cannot follow a template parameter list
template<typename X>
template<> class Outer<X>::Inner<void>; // ERROR

template<>
class Outer<bool>::Inner<wchar_t> {
    public:
        enum { count = 2 };
};
```

## 16.4 类模板的偏特化

偏特化（partial specialization）是指代换模板中的一部分参数得到一个新的模板，原始模板称为主模板。偏特化的限制如下：

1. 偏特化的模板应该和主模板对应
2. 偏特化的模板中不能包含默认实参
3. 偏特化的非类型模板实参应该是非依赖型的值或者其它的非类型模板形参
4. 偏特化的模板实参列表应该和主模版的形参列表不同
5. 可变参数应该在偏特化模板的最后

上述限制对应的例子如下：

```cpp
template<typename T, int I = 3>
class S;                            // primary template

template<typename T>
class S<int, T>;                    // ERROR: parameter kind mismatch

template<typename T = int>
class S<T, 10>;                     // ERROR: no default arguments

template<int I>
class S<int, I*2>;                  // ERROR: no nontype expressions

template<typename U, int K>
class S<U, K>;                      // ERROR: no significant difference from primary template

template<typename... Ts>
class Tuple;

template<typename Tail, typename... Ts>
class Tuple<Ts..., Tail>;           // ERROR: pack expansion not at the end

template<typename Tail, typename... Ts>
class Tuple<Tuple<Ts...>, Tail>;    // OK: pack expansion is at the end of a nested template argument list
```

## 16.5 变量模板的偏特化

C++标准中针对变量模板偏特化还有很多问题没有规定，取决于编译器的实现。书中只有两个例子：

```cpp
template<typename T> constexpr std::size_t SZ = sizeof(T);
template<typename T> constexpr std::size_t SZ<T&> = sizeof(void*);

template<typename T> typename T::iterator null_iterator;
template<typename T, std::size_t N> T* null_iterator<T[N]> = null_ptr; // T* doesn’t match T::iterator, and that is fine
```

## 16.6 后记
