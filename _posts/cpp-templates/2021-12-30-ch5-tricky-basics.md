---
layout: post
title: 《C++ Templates》第5章 模板中的复杂问题
categories: Reading
tags: C++
---

# 5 模板中的复杂问题

## 5.1 typename关键字 {#typename}

`typename`用来表明后面的标识符是一个类型：

```cpp
template<typename T>
class MyClass {
    public:
        // ...
        void foo() {
            typename T::SubType* ptr;
        }
};
```

如果直接写为`T::SubType* ptr`，则编译器会认为表达式的含义为`T`类型的静态成员`SubType`和`ptr`相乘。

`typename`的这种使用方式更多的用于使用容器中的迭代器打印容器：

```cpp
// basics/printcoll.hpp
#include <iostream>

// print elements of an STL container
template<typename T>
void printcoll (T const& coll)
{
    typename T::const_iterator pos;                 // iterator to iterate over coll
    typename T::const_iterator end(coll.end());     // end position
    for (pos=coll.begin(); pos!=end; ++pos) {
        std::cout << *pos << ' ';
    }
    std::cout << '\n';
}
```

## 5.2 零值初始化 {#Zero-Initialization}

如果不对内置类型进行初始化，则其初值为内存中的残留值。C++提供使用大括号进行**值初始化（value initialization）**的方式来统一对类类型和内置类型进行初始化：

```cpp
template<typename T>
void foo()
{
    T x{}; // x is zero (or false) if T is a built-in type
}
```

在C++11之前，语句`T x = T()`保证了可以默认初始化对象和用零值初始化内置类型，且在C++17之前，还要求类的拷贝构造函数不能是`explicit`的。C++17中的强制拷贝省略（mandatory copy elision）取消了这个限制，但还是推荐大括号的写法，因为即使没有默认构造函数，还可以通过初始化列表进行初始化。原文：

>Prior to C++17, this mechanism (which is still supported) only worked if the constructor selected for
the copy-initialization is not explicit. In C++17, mandatory copy elision avoids that limitation and
either syntax can work, but the braced initialized notation can use an initializer-list constructor1 if no
default constructor is available.

大括号初始化的方式也适用于成员变量初始化、类内初值和函数的默认参数。

## 5.3 this指针 {#this}

如果基类中含有类模板参数，使用成员变量或者成员函数时最好带上`this`：

```cpp
template<typename T>
    class Base {
        public:
            void bar();
    };

template<typename T>
class Derived : Base<T> {
    public:
        void foo() {
            bar(); // calls external bar() or error
        }
};
```

## 5.4 C风格数组和字符串的模板 {#Templates-for-Raw-Arrays-and-String-Literals}

当模板调用参数为传引用类型时，由于不会进行任何转换，所以数组的推导类型也会包括数组的长度信息，因此如果要传递两个长度不同的数组，模板要写成下面的样子：

```cpp
// basics/lessarray.hpp
template<typename T, int N, int M>
bool less (T(&a)[N], T(&b)[M])
{
    for (int i = 0; i<N && i<M; ++i) {
        if (a[i]<b[i]) return true;
        if (b[i]<a[i]) return false;
    }
    return N < M;
}
```

对于C风格字符串，也可以写成下面的样子：

```cpp
// basics/lessstring.hpp
template<int N, int M>
bool less (char const(&a)[N], char const(&b)[M])
{
    for (int i = 0; i<N && i<M; ++i) {
        if (a[i]<b[i]) return true;
        if (b[i]<a[i]) return false;
    }
    return N < M;
}
```

如果不知道数组的长度，那么只能通过重载或者偏特化的方式定义模板：

```cpp
// basics/arrays.hpp
#include <iostream>

template<typename T>
struct MyClass;             // primary template

template<typename T, std::size_t SZ>
struct MyClass<T[SZ]>       // partial specialization for arrays of known bounds
{
    static void print() { std::cout << "print() for T[" << SZ << "]\n"; }
};

template<typename T, std::size_t SZ>
struct MyClass<T(&)[SZ]>    // partial spec. for references to arrays of known bounds
{
    static void print() { std::cout << "print() for T(&)[" << SZ << "]\n"; }
};

template<typename T>
struct MyClass<T[]>         // partial specialization for arrays of unknown bounds
{
    static void print() { std::cout << "print() for T[]\n"; }
};

template<typename T>
struct MyClass<T(&)[]>      // partial spec. for references to arrays of unknown bounds
{
    static void print() { std::cout << "print() for T(&)[]\n"; }
};

template<typename T>
struct MyClass<T*>          // partial specialization for pointers
{
    static void print() { std::cout << "print() for T*\n"; }
};
```

下面是使用方法：

```cpp
// basics/arrays.cpp
#include "arrays.hpp"

template<typename T1, typename T2, typename T3>
void foo(int a1[7], int a2[],   // pointers by language rules
            int (&a3)[42],      // reference to array of known bound
            int (&x0)[],        // reference to array of unknown bound
            T1 x1,              // passing by value decays
            T2& x2, T3&& x3)    // passing by reference
{
    MyClass<decltype(a1)>::print();     // uses MyClass<T*>
    MyClass<decltype(a2)>::print();     // uses MyClass<T*>
    MyClass<decltype(a3)>::print();     // uses MyClass<T(&)[SZ]>
    MyClass<decltype(x0)>::print();     // uses MyClass<T(&)[]>
    MyClass<decltype(x1)>::print();     // uses MyClass<T*>
    MyClass<decltype(x2)>::print();     // uses MyClass<T(&)[]>
    MyClass<decltype(x3)>::print();     // uses MyClass<T(&)[]>
}

int main()
{
    int a[42];
    MyClass<decltype(a)>::print();  // uses MyClass<T[SZ]>
    extern int x[];                 // forward declare array
    MyClass<decltype(x)>::print();  // uses MyClass<T[]>
    foo(a, a, a, x, x, x, x);
}

int x[] = {0, 8, 15};               // define forward-declared array
```

- 虽然形参`a1`中包含了长度`7`，但是编译器仍然将其处理为指针（`7`被忽略了，这应该是为了保证和C语言的兼容）。
- `T1`的推导结果为`int *`
- `T2`和`T3`的推导结果为`int(&)[]`

## 5.5 成员函数模板 {#Member-Templates}

成员函数也可以是模板，下面的例子为`Stack<>`定义了一个拷贝赋值运算符模板：

```cpp
// basics/stack5decl.hpp
template<typename T>
class Stack {
    private:
        std::deque<T> elems;        // elements
    public:
        void push(T const&);        // push element
        void pop();                 // pop element
        T const& top() const;       // return top element
        bool empty() const {        // return whether the stack is empty
            return elems.empty();
        }

        // assign stack of elements of type T2
        template<typename T2>
        Stack& operator= (Stack<T2> const&);
};
```

```cpp
// basics/stack5assign.hpp
template<typename T>
    template<typename T2>
Stack<T>& Stack<T>::operator= (Stack<T2> const& op2)
{
    Stack<T2> tmp(op2);             // create a copy of the assigned stack

    elems.clear();                  // remove existing elements
    while (!tmp.empty()) {          // copy all elements
        elems.push_front(tmp.top());
        tmp.pop();
    }
    return *this;
}
```

`Stack<T>`不能直接访问`Stack<T2>`的私有对象，只能通过`top()`和`pop()`访问`Stack<T2>`的元素。方便起见，还可以添加一个友元定义，使其可以访问`Stack<T2>::elems`：

```cpp
// basics/stack6decl.hpp
template<typename T>
class Stack {
    private:
        std::deque<T> elems;    // elements
    public:
        void push(T const&);    // push element
        void pop();             // pop element
        T const& top() const;   // return top element
        bool empty() const {    // return whether the stack is empty
            return elems.empty();
        }

        // assign stack of elements of type T2
        template<typename T2>
        Stack& operator= (Stack<T2> const&);
        // to get access to private members of Stack<T2> for any type T2:
        template<typename> friend class Stack;
};
```

```cpp
// basics/stack6assign.hpp
template<typename T>
    template<typename T2>
Stack<T>& Stack<T>::operator= (Stack<T2> const& op2)
{
    elems.clear();                  // remove existing elements
    elems.insert(elems.begin(),     // insert at the beginning
                op2.elems.begin(),  // all elements from op2
                op2.elems.end());
    return *this;
}
```

下面的例子将`Stack<>`中的内置容器类型也变成了一个模板参数：

```cpp
// basics/stack7decl.hpp
template<typename T, typename Cont = std::deque<T>>
class Stack {
    private:
        Cont elems; // elements
    public:
        void push(T const&); // push element
        void pop(); // pop element
        T const& top() const; // return top element
        bool empty() const { // return whether the stack is empty
            return elems.empty();
        }
        // assign stack of elements of type T2
        template<typename T2, typename Cont2>
        Stack& operator= (Stack<T2,Cont2> const&);
        // to get access to private members of Stack<T2> for any type T2:
        template<typename, typename> friend class Stack;
};
```

```cpp
// basics/stack7assign.hpp
template<typename T, typename Cont>
    template<typename T2, typename Cont2>
Stack<T,Cont>& Stack<T,Cont>::operator= (Stack<T2,Cont2> const& op2)
{
    elems.clear();                  // remove existing elements
    elems.insert(elems.begin(),     // insert at the beginning
                op2.elems.begin(),  // all elements from op2
                op2.elems.end());
    return *this;
}
```

成员函数模板可以被特化或者偏特化：

```cpp
// basics/boolstring.hpp
class BoolString {
    private:
        std::string value;
    public:
        BoolString (std::string const& s)
            : value(s) {
        }
        template<typename T = std::string>
        T get() const {         // get value (converted to T)
            return value;
    }
};
```

```cpp
// basics/boolstringgetbool.hpp
// full specialization for BoolString::getValue<>() for bool
template<>
inline bool BoolString::get<bool>() const {
    return value == "true" || value == "1" || value == "on";
}
```

### 5.5.1 .template

在下面的例子中，如果没有`.template`，则`to_string`后面的`<`会被编译器解析为小于比较运算符（目前好像感受不到作用）：

```cpp
template<unsigned long N>
void printBitset (std::bitset<N> const& bs) {
    std::cout << bs.template to_string<char, std::char_traits<char>,
                                        std::allocator<char>>();
}
```

### 5.5.2 泛型lambda表达式和成员函数模板

泛型lambda表达式只是成员函数模板的一种简便写法，例如：

```cpp
[] (auto x, auto y) {
    return x + y;
}
```

会被编译器解析为：

```cpp
class SomeCompilerSpecificName {
    public:
        SomeCompilerSpecificName(); // constructor only callable by compiler
        template<typename T1, typename T2>
        auto operator() (T1 x, T2 y) const {
            return x + y;
        }
};
```

## 5.6 变量模板 {#Variable-Templates}

C++14开始支持变量模板：

```cpp
template<typename T> constexpr T pi{3.1415926535897932385};
```

注意上面的声明不能出现在函数或者块作用域中。下面是使用方式：

```cpp
std::cout << pi<double> << '\n';
std::cout << pi<float> << '\n';
```

可以在不同的翻译单元中使用实例化的同一变量模板（全局变量）：

```cpp
// header.hpp:
template<typename T> T val{}; // zero initialized value

// translation unit 1:
#include "header.hpp"

int main()
{
    val<long> = 42;
    print();
}

// translation unit 2:
#include "header.hpp"

void print()
{
    std::cout << val<long> << '\n'; // OK: prints 42
}
```

变量模板支持默认模板参数：

```cpp
template<typename T = long double>
constexpr T pi = T{3.1415926535897932385};

std::cout << pi<> << '\n';          // outputs a long double
std::cout << pi<float> << '\n';     // outputs a float
std::cout << pi << '\n';            // ERROR
```

变量模板支持非类型模板参数：

```cpp
#include <iostream>
#include <array>

template<int N>
    std::array<int,N> arr{}; // array with N elements, zero-initialized

template<auto N>
    constexpr decltype(N) dval = N; // type of dval depends on passed value

int main()
{
    std::cout << dval<'c'> << '\n';                 // N has value 'c' of type char
    arr<10>[0] = 42;                                // sets first element of global arr
    for (std::size_t i=0; i<arr<10>.size(); ++i) {  // uses values set in arr
        std::cout << arr<10>[i] << '\n';
    }
}
```

变量模板的典型应用是用来表示类模板成员变量：

```cpp
namespace std {
    template<typename T> class numeric_limits {
        public:
            // ...
            static constexpr bool is_signed = false;
            // ...
    };
}

template<typename T>
constexpr bool isSigned = std::numeric_limits<T>::is_signed;
```

C++17使用带`_v`后缀的变量模板来简化标准库中产生布尔变量的型别模板，例如：

```cpp
namespace std {
    template<typename T> constexpr bool is_const_v = is_const<T>::value;
}
```

## 5.7 模板的模板参数（Template Template Parameter） {#Template-Template-Parameter}

使用栈类模板时，如果要指定内部容器的类型，需要在实例化类模板时写两遍元素的类型：

```cpp
Stack<int, std::vector<int>> vStack; // integer stack that uses a vector
```

如果将第二个参数定义为模板的模板参数，就可以只写一遍元素的类型：

```cpp
Stack<int, std::vector> vStack; // integer stack that uses a vector
```

此时类模板的定义如下：

```cpp
// basics/stack8decl.hpp
template<typename T,
         template<typename Elem> class Cont = std::deque>
class Stack {
    private:
        Cont<T> elems; // elements
    public:
        void push(T const&);    // push element
        void pop();             // pop element
        T const& top() const;   // return top element
        bool empty() const {    // return whether the stack is empty
            return elems.empty();
        }
        // ...
};
```

从C++17开始，也可以写为下面的形式：

```cpp
template<typename T,
         template<typename Elem> typename Cont = std::deque>
class Stack { // ERROR before C++17
    // ...
};
```

因为`Elem`没有使用，所以也可以写为下面的形式：

```cpp
template<typename T,
         template<typename> class Cont = std::deque>
class Stack {
    // ...
};
```

成员函数`push()`的定义也要进行相应修改：

```cpp
template<typename T, template<typename> class Cont>
void Stack<T,Cont>::push (T const& elem)
{
    elems.push_back(elem); // append copy of passed elem
}
```

在C++17之前，即使实例化时使用的模板中包含默认参数，也要保证模板的模板参数中的参数数量和实例化时使用的模板需要的参数数量相同，而在C++17中解除了这个限制。原文：

>The problem is that prior to C++17 a template template argument had to be a template with parameters that exactly match the parameters of the template template parameter it substitutes, with some exceptions related to variadic template parameters (see Section 12.3.4 on page 197). Default template arguments of template template arguments were not considered, so that a match couldn't be achieved by leaving out arguments that have default values (in C++17, default arguments are considered).

具体来说，因为`std::deque`包含两个模板参数，虽然第二个参数是`std::allocator`并且有默认值，但是编译器无法进行匹配，所以应该写为下面的形式：

```cpp
// basics/stack9.hpp
#include <deque>
#include <cassert>
#include <memory>

template<typename T,
        template<typename Elem,
                typename = std::allocator<Elem>>
        class Cont = std::deque>
class Stack {
    private:
        Cont<T> elems;          // elements
    public:
        void push(T const&);    // push element
        void pop();             // pop element
        T const& top() const;   // return top element
        bool empty() const {    // return whether the stack is empty
            return elems.empty();
        }

    // assign stack of elements of type T2
    template<typename T2,
            template<typename Elem2,
                    typename = std::allocator<Elem2>
                    >class Cont2>
    Stack<T,Cont>& operator= (Stack<T2,Cont2> const&);

    // to get access to private members of any Stack with elements of type T2:
    template<typename, template<typename, typename>class>
    friend class Stack;
};

template<typename T, template<typename,typename> class Cont>
void Stack<T,Cont>::push (T const& elem)
{
    elems.push_back(elem); // append copy of passed elem
}

template<typename T, template<typename,typename> class Cont>
void Stack<T,Cont>::pop ()
{
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}

template<typename T, template<typename,typename> class Cont>
T const& Stack<T,Cont>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}

template<typename T, template<typename,typename> class Cont>
template<typename T2, template<typename,typename> class Cont2>
Stack<T,Cont>&
Stack<T,Cont>::operator= (Stack<T2,Cont2> const& op2)
{
    elems.clear();                  // remove existing elements
    elems.insert(elems.begin(),     // insert at the beginning
                op2.elems.begin(),  // all elements from op2
                op2.elems.end());
    return *this;
}
```

下面是使用方法：

```cpp
// basics/stack9test.cpp
#include "stack9.hpp"
#include <iostream>
#include <vector>
int main()
{
    Stack<int> iStack; // stack of ints
    Stack<float> fStack; // stack of floats

    // manipulate int stack
    iStack.push(1);
    iStack.push(2);
    std::cout << "iStack.top(): " << iStack.top() << '\n';

    // manipulate float stack:
    fStack.push(3.3);
    std::cout << "fStack.top(): " << fStack.top() << '\n';

    // assign stack of different type and manipulate again
    fStack = iStack;
    fStack.push(4.4);
    std::cout << "fStack.top(): " << fStack.top() << '\n';

    // stack for doubless using a vector as an internal container
    Stack<double, std::vector> vStack;
    vStack.push(5.5);
    vStack.push(6.6);
    std::cout << "vStack.top(): " << vStack.top() << '\n';
    vStack = fStack;
    std::cout << "vStack: ";
    while (! vStack.empty()) {
        std::cout << vStack.top() << ' ';
        vStack.pop();
    }
    std::cout << '\n';
}
```

## 5.8 总结

1. 需要用`typename`关键字来提示编译器接下来的标识符是由模板参数决定的一个类型，参见[5.1](#typename)
2. 访问基类模板中的成员时要带上`this`，参见[5.3](#this)
3. 嵌入类和成员函数也可以是模板，参见[5.5](#Member-Templates)
4. 模板构造函数和模板赋值运算符不能代替普通的构造函数和赋值运算符
5. 使用大括号可以保证所有类型都能被正确初始化，参见[5.2](#Zero-Initialization)
6. C++支持C风格数组和字符串的模板，参见[5.4](#Templates-for-Raw-Arrays-and-String-Literals)
7. 模板调用参数不为引用类型时，数组类型会转换为指针类型
8. C++14开始支持变量模板
9. 类模板可以作为模板参数，也就是模板的模板参数
10. 要保证模板的模板参数中的参数数量和实例化时使用的模板需要的参数数量相同，参见[5.7](#Template-Template-Parameter)
