---
layout: post
title: 《C++ Templates》第2章 类模板
categories: Reading
tags: C++
---

# 2 类模板

## 2.1 栈类模板实现

```cpp
// basics/stack1.hpp
#include <vector>
#include <cassert>

template<typename T>
class Stack {
    private:
        std::vector<T> elems;       // elements
    public:
        void push(T const& elem);   // push element
        void pop();                 // pop element
        T const& top() const;       // return top element
        bool empty() const {        // return whether the stack is empty
            return elems.empty();
        }
};

template<typename T>
void Stack<T>::push (T const& elem)
{
    elems.push_back(elem); // append copy of passed elem
}

template<typename T>
void Stack<T>::pop ()
{
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}

template<typename T>
T const& Stack<T>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

### 2.1.1 类模板声明

在类模板内部，可以使用`T`来声明和定义任何成员变量。类的**类型**为`Stack<T>`，所以当需要使用类的类型时需要写明`Stack<T>`（除非`T`可以被推导）。但是在类的内部也是可以直接使用类**名字**`Stack`的，所以下面两种定义拷贝构造函数和拷贝赋值运算符的方式是一样的，书中推荐第一种：

```cpp
template<typename T>
class Stack {
    // ...
    Stack (Stack const&);               // copy constructor
    Stack& operator= (Stack const&);    // assignment operator
    // ...
};
```

```cpp
template<typename T>
class Stack {
    // ...
    Stack (Stack<T> const&);                // copy constructor
    Stack<T>& operator= (Stack<T> const&);  // assignment operator
    // ...
};
```

但是在类的外部必须要使用完整的类型名，除非需要的是类的名字而不是类的类型：

```cpp
template<typename T>
bool operator== (Stack<T> const& lhs, Stack<T> const& rhs);
```

### 2.1.2 成员函数定义

在定义类的成员函数时，需要指出这是一个模板，所以前面要加上`template<typename T>`。

## 2.2 使用类模板

```cpp
// basics/stack1test.cpp
#include "stack1.hpp"
#include <iostream>
#include <string>

int main()
{
    Stack<int> intStack;                // stack of ints
    Stack<std::string> stringStack;     // stack of strings

    // manipulate int stack
    intStack.push(7);
    std::cout << intStack.top() << '\n';

    // manipulate string stack
    stringStack.push("hello");
    std::cout << stringStack.top() << '\n';
    stringStack.pop();
}
```

只有真正被调用的函数才会被实例化，例如在上面的例子中，编译器分别实例化了接收`int`和`std::string`类型的`push()`和`top()`，但是也只实例化了元素类型为`std::string`的`pop()`。如果类模板中有静态成员，那么当类被使用时，静态成员就会被实例化。

被实例化的类模板类型可以被`const`和`volatile`关键字修饰，也可以定义该类型的数组和该类型的引用，还可以用`typedef`和`using`来声明新的类型和作为其它模板的参数。

## 2.3 部分使用类模板

类模板通常需要对模板参数类型进行多种操作，这并不意味着模板参数类型应该支持类模板中所有涉及到的在该类型上的操作，而只需要支持必要的操作就可以了。原文：

>A class template usually applies multiple operations on the template arguments it is instantiated for (including construction and destruction). This might lead to the impression that these template arguments have to provide all operations necessary for all member functions of a class template. But this is not the case: Template arguments only have to provide all necessary operations that are needed (instead of that could be needed).

例如为`Stack<>`定义一个`printOn()`方法，同时实例化模板参数类型为`std::pair<int,int>`的类模板，那么只要不调用`printOn()`函数，就不需要为`std::pair<int,int>`重载`<<`运算符。

```cpp
template<typename T>
class Stack {
    // ...
    void printOn() (std::ostream& strm) const {
        for (T const& elem : elems) {
            strm << elem << ' '; // call << for each element
        }
    }
};

Stack<std::pair<int,int>> ps;           // note: std::pair<> has no operator<< defined
ps.push({4, 5});                        // OK
ps.push({6, 7});                        // OK
std::cout << ps.top().first << '\n';    // OK
std::cout << ps.top().second << '\n';   // OK
```

### 2.3.1 Concept

好像在C++20中才有，书上说了一段没什么实质内容的话。

## 2.4 友元

可以通过重载`<<`运算符并将其定义为友元函数来实现打印：

```cpp
template<typename T>
class Stack {
    // ...
    void printOn() (std::ostream& strm) const {
        // ...
    }
    friend std::ostream& operator<< (std::ostream& strm, Stack<T> const& s) {
        s.printOn(strm);
        return strm;
    }
};
```

在上面这种内联的形式中，重载的`<<`运算符并不是函数模板，而是一个实例化的普通函数。原文：

>Note that this means that operator << for class Stack<> is not a function template, but an "ordinary" function instantiated with the class template if needed.

但是一般是不会将重载的`<<`运算符实现为内联函数的，而是声明其为友元，然后类外部进行定义，有下面两种方法：

- 使用一个新的模板参数`U`：

```cpp
template<typename T>
class Stack {
    // ...
    template<typename U>
    friend std::ostream& operator<< (std::ostream&, Stack<U> const&);
};
```

- 先声明类模板和运算符，然后声明友元，注意友元声明中的`<T>`:

```cpp
template<typename T>
class Stack;
template<typename T>
std::ostream& operator<< (std::ostream&, Stack<T> const&);

template<typename T>
class Stack {
    // ...
    friend std::ostream& operator<< <T> (std::ostream&, Stack<T> const&);
};
```

## 2.5 模板特化

模板特化是为类模板提供某种模板参数类型的特殊实现，例如为`std::string`类型的模板参数进行特化：

```cpp
// basics/stack2.hpp
#include "stack1.hpp"
#include <deque>
#include <string>
#include <cassert>

template<>
class Stack<std::string> {
    private:
        std::deque<std::string> elems;      // elements
    public:
        void push(std::string const&);      // push element
        void pop();                         // pop element
        std::string const& top() const;     // return top element
        bool empty() const {                // return whether the stack is empty
            return elems.empty();
        }
};

void Stack<std::string>::push (std::string const& elem)
{
    elems.push_back(elem); // append copy of passed elem
}

void Stack<std::string>::pop ()
{
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}

std::string const& Stack<std::string>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

## 2.6 模板偏特化（Partial Sepcialization）

模板偏特化是为部分模板参数提供特殊实现，例如为`T*`类型的模板参数进行偏特化：

```cpp
// basics/stackpartspec.hpp
#include "stack1.hpp"

// partial specialization of class Stack<> for pointers:
template<typename T>
class Stack<T*> {
    private:
        std::vector<T*> elems;  // elements
    public:
        void push(T*);          // push element
        T* pop();               // pop element
        T* top() const;         // return top element
        bool empty() const {    // return whether the stack is empty
            return elems.empty();
        }
};

template<typename T>
void Stack<T*>::push (T* elem)
{
    elems.push_back(elem); // append copy of passed elem
}

template<typename T>
T* Stack<T*>::pop ()
{
    assert(!elems.empty());
    T* p = elems.back();
    elems.pop_back();       // remove last element
    return p;               // and return it (unlike in the general case)
}

template<typename T>
T* Stack<T*>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

在有多个模板参数的情况下，只对部分参数进行特化也是偏特化：

```cpp
template<typename T1, typename T2>
class MyClass {
};

// partial specialization: both template parameters have same type
template<typename T>
class MyClass<T,T> {
};

// partial specialization: second type is int
template<typename T>
class MyClass<T,int> {
};

// partial specialization: both template parameters are pointer types
template<typename T1, typename T2>
class MyClass<T1*,T2*> {
};

MyClass<int,float> mif;     // uses MyClass<T1,T2>
MyClass<float,float> mff;   // uses MyClass<T,T>
MyClass<float,int> mfi;     // uses MyClass<T,int>
MyClass<int*,float*> mp;    // uses MyClass<T1*,T2*>

MyClass<int,int> m;     // ERROR: matches MyClass<T,T> and MyClass<T,int>
MyClass<int*,int*> m;   // ERROR: matches MyClass<T,T> and MyClass<T1*,T2*>
```

## 2.7 默认模板参数

可以为类模板提供默认的模板参数，例如为`Stack<>`提供默认的内部容器类型：

```cpp
// basics/stack3.hpp
#include <vector>
#include <cassert>

template<typename T, typename Cont = std::vector<T>>
class Stack {
    private:
        Cont elems;                 // elements
    public:
        void push(T const& elem);   // push element
        void pop();                 // pop element
        T const& top() const;       // return top element
        bool empty() const {        // return whether the stack is empty
            return elems.empty();
        }
};

template<typename T, typename Cont>
void Stack<T,Cont>::push (T const& elem)
{
    elems.push_back(elem); // append copy of passed elem
}

template<typename T, typename Cont>
void Stack<T,Cont>::pop ()
{
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}

template<typename T, typename Cont>
T const& Stack<T,Cont>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

当然也可以为默认模板参数提供具体的参数：

```cpp
// basics/stack3test.cpp
#include "stack3.hpp"
#include <iostream>
#include <deque>

int main()
{
    // stack of ints:
    Stack<int> intStack;

    // stack of doubles using a std::deque<> to manage the elements
    Stack<double,std::deque<double>> dblStack;

    // manipulate int stack
    intStack.push(7);
    std::cout << intStack.top() << '\n';
    intStack.pop();

    // manipulate double stack
    dblStack.push(42.42);
    std::cout << dblStack.top() << '\n';
    dblStack.pop();
}
```

## 2.8 类型别名

没什么可写的，推荐用`using`而不是`typedef`。

## 2.9 类模板参数推导

从C++17开始，有时可以省略模板参数，而让编译器去推导，例如下面的代码：

```cpp
Stack<int> intStack1;               // stack of strings
Stack<int> intStack2 = intStack1;   // OK in all versions
Stack intStack3 = intStack1;        // OK since C++17
```

假如再为`Stack<>`提供一个以`T`类型的引用为参数的构造函数`Stack (T const& elem)`，那么便可以使用一个初值来实例化：

```cpp
template<typename T>
class Stack {
    private:
        std::vector<T> elems;   // elements
    public:
        Stack () = default;
        Stack (T const& elem)   // initialize stack with one element
            : elems({elem}) {
        }
    // ...
};

Stack intStack = 0;     // Stack<int> deduced since C++17
```

如果这个初值是一个字符串常量（例如`Stack stringStack = "bottom"`），就会有一点复杂。由于`Stack (T const& elem)`是传引用的，所以不会发生任何类型转换，因此类模板实例化后的类型为`Stack<char const[7]>`。但是我们需要的类型是`Stack<char const*>`，所以需要添加一个传值类型的构造函数，并且将其移动到`Stack<T>`中：

```cpp
template<typename T>
class Stack {
    private:
        std::vector<T> elems;   // elements
    public:
        Stack (T elem)          // initialize stack with one element by value
            : elems({std::move(elem)}) {
        }
    // ...
};
```

后面是关于**推导指引（deduction guide）**的，没有示例代码，我根据原文的意思写代码测试了一下，编译命令为`g++ test.cpp -o test -std=c++17`，输出为`true`：

```cpp
#include <vector>
#include <string>
#include <iostream>
#include <iomanip>

template<typename T>
class Stack {
    private:
        std::vector<T> elems;   // elements
    public:
        Stack (T elem)          // initialize stack with one element by value
            : elems({std::move(elem)}) {
        }
};

Stack(char const*) -> Stack<std::string>;

int main()
{
    Stack stringStack{"bottom"};    // Stack<std::string> deduced and valid
    Stack stack2{stringStack};      // Stack<std::string> deduced
    Stack stack3(stringStack);      // Stack<std::string> deduced
    Stack stack4 = {stringStack};   // Stack<std::string> deduced
    std::cout << std::boolalpha << (typeid(stringStack) == typeid(Stack<std::string>)) << std::endl;
    return 0;
}
```

## 2.10 聚合类的模板（Templatized Aggregates）

聚合类也可以作为模板：

```cpp
#include <string>

template<typename T>
struct ValueWithComment {
    T value;
    std::string comment;
};

ValueWithComment(char const*, char const*) -> ValueWithComment<std::string>;

int main()
{
    ValueWithComment<int> vc;
    vc.value = 42;
    vc.comment = "initial value";

    ValueWithComment vc2 = {"hello", "initial value"};
}
```

## 2.11 总结

1. 类模板是一个或者多个类型参数待确定的类
2. 实例化类模板时需要传递具体的类型作为类模板参数
3. 只有被调用的类模板才会被实例化
4. 类模板可以特化
5. 类模板也可以被偏特化
6. C++17支持类模板参数推导
7. 可以定义实例化聚合类的模板
8. 如果模板参数是传值的，那么参数会退化为原始类型
9. 模板只能被声明和定义在命名空间或者类声明的内部

第8条原文：

>Call parameters of a template type decay if declared to be called by value.
