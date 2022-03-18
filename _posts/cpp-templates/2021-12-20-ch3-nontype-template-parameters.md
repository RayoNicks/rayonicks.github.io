---
layout: post
title: 《C++ Templates》第3章 非类型模板参数
categories: Reading
tags: C++
---

# 3 非类型模板参数

对于函数模板和类模板，模板参数也可以是普通的数值。

## 3.1 非类型类模板参数

将栈的大小设定为固定值可以避免内存管理的问题，但是模板作者并不知道设定为多大是合适的，所以可以提供一个非类型模板参数来让用户设定栈的大小：

```cpp
// basics/stacknontype.hpp
#include <array>
#include <cassert>

template<typename T, std::size_t Maxsize>
class Stack {
    private:
        std::array<T,Maxsize> elems;    // elements
        std::size_t numElems;           // current number of elements
    public:
        Stack();                    // constructor
        void push(T const& elem);   // push element
        void pop();                 // pop element
        T const& top() const;       // return top element
        bool empty() const {        // return whether the stack is empty
            return numElems == 0;
        }
        std::size_t size() const {  // return current number of elements
            return numElems;
        }
};

template<typename T, std::size_t Maxsize>
Stack<T,Maxsize>::Stack ()
    : numElems(0) // start with no elements
{
    // nothing else to do
}

template<typename T, std::size_t Maxsize>
void Stack<T,Maxsize>::push (T const& elem)
{
    assert(numElems < Maxsize);
    elems[numElems] = elem;     // append element
    ++numElems;                 // increment number of elements
}

template<typename T, std::size_t Maxsize>
void Stack<T,Maxsize>::pop ()
{
    assert(!elems.empty());
    --numElems; // decrement number of elements
}

template<typename T, std::size_t Maxsize>
T const& Stack<T,Maxsize>::top () const
{
    assert(!elems.empty());
    return elems[numElems-1]; // return last element
}
```

下面是使用方法：

```cpp
// basics/stacknontype.cpp
#include "stacknontype.hpp"
#include <iostream>
#include <string>

int main()
{
    Stack<int,20> int20Stack;           // stack of up to 20 ints
    Stack<int,40> int40Stack;           // stack of up to 40 ints
    Stack<std::string,40> stringStack;  // stack of up to 40 strings

    // manipulate stack of up to 20 ints
    int20Stack.push(7);
    std::cout << int20Stack.top() << '\n';
    int20Stack.pop();

    // manipulate stack of up to 40 strings
    stringStack.push("hello");
    std::cout << stringStack.top() << '\n';
    stringStack.pop();
}
```

当然非类型模板参数也是可以有默认值的。

## 3.2 非类型函数模板参数

下面是一个带非类型参数的函数模板：

```cpp
// basics/addvalue.hpp
template<int Val, typename T>
T addValue (T x)
{
    return x + Val;
}
```

然后通过算法`transform`就可以实现将`source`中的每个值加5后存储到`dest`中：

```cpp
std::transform (source.begin(), source.end(),   // start and end of source
                dest.begin(),                   // start of destination
                addValue<5,int>);               // operation
```

和前面的类模板一样，非类型模板参数也是可以有默认值的。

## 3.3 非类型模板参数的限制 {#Restrictions}

非类型模板参数只能是：

1. 常量整数，也包括枚举类型
2. 指向对象、函数和成员的指针
3. 对象和函数的左值引用
4. `std::nullptr_t`

不能是：

1. 浮点类型
2. 类类型，例如`std::string`

*当非类型模板参数是指针或者引用时，指针指向的对象或者引用引用的对象不能是字符串字面值、临时变量、成员变量或者其它子对象。不过随着演变到C++17，这些限制已经慢慢的放开了，所以可以精简为下面的几条：*

- 在C++11中，这个对象必须是可以被外部链接的
- 在C++14中，这个对象必须是外部链接或者内部链接的

因此下面的代码是错误的：

```cpp
template<char const* name>
class MyClass {
};

MyClass<"hello"> x; // ERROR: string literal "hello" not allowed
```

不过有方法可以解决这个问题：

```cpp
extern char const s03[] = "hi";     // external linkage
char const s11[] = "hi";            // internal linkage

int main()
{
    Message<s03> m03;                   // OK (all versions)
    Message<s11> m11;                   // OK since C++11
    static char const s17[] = "hi";     // no linkage
    Message<s17> m17;                   // OK since C++17
}
```

上面斜体原文：

>When passing template arguments to pointers or references, the objects must not be string literals, temporaries, or data members and other subobjects. Because these restrictions were relaxed with each and every C++ version before C++17, additional constraints apply.

非类型模板参数也可以是表达式。

## 3.4 auto类型的非类型模板参数

从C++17开始，非类型模板参数的类型可以为`auto`：

```cpp
// basics/stackauto.hpp
#include <array>
#include <cassert>

template<typename T, auto Maxsize>
class Stack {
    public:
        using size_type = decltype(Maxsize);
    private:
        std::array<T,Maxsize> elems;    // elements
        size_type numElems;             // current number of elements
    public:
        Stack();                        // constructor
        void push(T const& elem);       // push element
        void pop();                     // pop element
        T const& top() const;           // return top element
        bool empty() const {            // return whether the stack is empty
            return numElems == 0;
        }
        size_type size() const {        // return current number of elements
            return numElems;
        }
};

// constructor
template<typename T, auto Maxsize>
Stack<T,Maxsize>::Stack ()
    : numElems(0) // start with no elements
{
    // nothing else to do
}

template<typename T, auto Maxsize>
void Stack<T,Maxsize>::push (T const& elem)
{
    assert(numElems < Maxsize);
    elems[numElems] = elem;     // append element
    ++numElems;                 // increment number of elements
}

template<typename T, auto Maxsize>
void Stack<T,Maxsize>::pop ()
{
    assert(!elems.empty());
    --numElems; // decrement number of elements
}

template<typename T, auto Maxsize>
T const& Stack<T,Maxsize>::top () const
{
    assert(!elems.empty());
    return elems[numElems-1]; // return last element
}
```

下面是使用方法：

```cpp
// basics/stackauto.cpp
#include <iostream>
#include <string>
#include "stackauto.hpp"

int main()
{
    Stack<int,20u> int20Stack;          // stack of up to 20 ints
    Stack<std::string,40> stringStack;  // stack of up to 40 strings

    // manipulate stack of up to 20 ints
    int20Stack.push(7);
    std::cout << int20Stack.top() << '\n';
    auto size1 = int20Stack.size();

    // manipulate stack of up to 40 strings
    stringStack.push("hello");
    std::cout << stringStack.top() << '\n';
    auto size2 = stringStack.size();
    if (!std::is_same_v<decltype(size1), decltype(size2)>) {
        std::cout << "size types differ" << '\n';
    }
}
```

注意`int20Stack`中的`size_type`是`unsigned int`，而`stringStack`中的`size_type`是`int`，所以程序的输出是`size types differ`。

虽然使用了`auto`，但是3.3中的限制是依然存在的，所以`Stack<int,3.14> sd`依然是错误的，但是由于字符串可以被作为常量数组，所以下面的代码是合法的：

```cpp
// basics/message.cpp
#include <iostream>

template<auto T> // take value of any possible nontype parameter (since C++17)
class Message {
    public:
        void print() {
            std::cout << T << '\n';
        }
};

int main()
{
    Message<42> msg1;
    msg1.print();       // initialize with int 42 and print that value
    static char const s[] = "hello";
    Message<s> msg2;    // initialize with char const[6] "hello"
    msg2.print();       // and print that value
}
```

还有`template<decltype(auto) N>`也是合法的，这种声明会让`N`的类型变为引用：

```cpp
template<decltype(auto) N>
class C {
};

int i;
C<(i)> x; // N is int&
```

## 3.5 总结

1. 模板参数也可以是非类型的
2. 浮点类型和类类型是不能作为非类型模板参数的，对于指针和引用的限制在[3.3](#Restrictions)节
3. 可以使用`auto`作为非类型模板参数
