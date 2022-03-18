---
layout: post
title: 《C++ Templates》第4章 可变参数模板
categories: Reading
tags: C++
---

# 4 可变参数模板

## 4.1 可变参数模板

### 4.1.1 例子

```cpp
// basics/varprint1.hpp
#include <iostream>

void print ()
{
}

template<typename T, typename... Types>
void print (T firstArg, Types... args)
{
    std::cout << firstArg << '\n';  // print first argument
    print(args...);                 // call print() for remaining arguments
}
```

`args`称为**函数参数包（function parameter pack）**，`Types`称为**模板参数包（template parameter pack）**。可变参数模板通过递归的方式进行实例化，例如对于下面的代码：

```cpp
std::string s("world");
print (7.5, "hello", s);
```

解析过程为：

1. `print (7.5, "hello", s)`实例化为`print<double, char const*, std::string> (7.5, "hello", s)`，`firstArg`值为`7.5`，`T`为`double`，`args`中包含`"hello"`和`s`，`Types`中包含`char const *`和`std::string`
2. `print(args...)`实例化为`print<char const*, std::string> ("hello", s)`，`firstArg`值为`"hello"`，`T`为`char const *`，`args`中包含`s`，`Types`中包含`std::string`
3. `print(args...)`实例化为`print<std::string> (s)`，`firstArg`值为`"world"`，`args`为空，`Types`也为空
4. 调用重载函数`print()`

### 4.1.2 重载可变参数模板和非可变参数模板

如果两个重载函数模板只有尾部参数包不同，那么编译器优先匹配没有尾部参数包的版本，所以上面的代码也可以这样实现：

```cpp
// basics/varprint2.hpp
#include <iostream>

template<typename T>
void print (T arg)
{
    std::cout << arg << '\n'; // print passed argument
}

template<typename T, typename... Types>
void print (T firstArg, Types... args)
{
    print(firstArg);    // call print() for the first argument
    print(args...);     // call print() for remaining arguments
}
```

### 4.1.3 sizeof...

C++11引入了`sizeof...`运算符（没错就是包含三个点）来计算模板参数包或者函数参数包的大小：

```cpp
template<typename T, typename... Types>
void print (T firstArg, Types... args)
{
    std::cout << sizeof...(Types) << '\n';  // print number of remaining types
    std::cout << sizeof...(args) << '\n';   // print number of remaining args
    // ...
}
```

自然而然会想到使用`sizeof...`来判断参数包中是否还有参数剩余，从而可以避免重载，例如下面的方式：

```cpp
template<typename T, typename... Types>
void print (T firstArg, Types... args)
{
    std::cout << firstArg << '\n';
    if (sizeof...(args) > 0) {  // error if sizeof...(args)==0
        print(args...);         // and no print() for no arguments declared
    }
}
```

但是上面的代码会报错，这是因为实例化的`print()`是否被调用是运行时决定的，而`sizef...`会在编译期间求值，所以尽管`if`中的条件恒为真或者恒为假，编译器还是会递归的实例化所有的`print()`，当找不到无参数版本的`print()`的定义时就会报错。原文：

>However, this approach doesn’t work because in general both branches of all if statements in function templates are instantiated. Whether the instantiated code is useful is a run-time decision, while the instantiation of the call is a compile-time decision. For this reason, if you call the print() function template for one (last) argument, the statement with the call of print(args...) still is instantiated for no argument, and if there is no function print() for no arguments provided, this is an error.

## 4.2 折叠表达式

C++17开始支持折叠表达式，即可以使用二元运算符作用于参数包中所有的参数：

```cpp
template<typename... T>
auto foldSum (T... s) {
    return (... + s); // ((s1 + s2) + s3) ...
}
```

如果参数包为空，一般来说会抛出异常，但是也有例外：

1. `&&`求值为`true`
2. `||`求值为`false`
3. `,`求值为`void()`

下表列出了支持的折叠表达式及相应的展开结果：

|Fold Expression        |Evaluation                             |
|:----------------------|:--------------------------------------|
|( ... op pack )        |(( pack1 op pack2 ) ... op packN )     |
|( pack op ... )        |( pack1 op ( ... ( packN-1 op packN )))|
|( init op ... op pack )|(( init op pack1 ) ... op packN )      |
|( pack op ... op init )|( pack1 op ( ... ( packN op init )))   |

**表中带初值`init`的行的折叠表达式中没有第二个op，但是下面两个示例代码中都有两个op。**

第1个是使用成员指针运算符`->*`取出二叉树中指定节点的例子：

```cpp
// basics/foldtraverse.cpp
// define binary tree structure and traverse helpers:
struct Node {
    int value;
    Node* left;
    Node* right;
    Node(int i=0) : value(i), left(nullptr), right(nullptr) {
    }
    // ...
};
auto left = &Node::left;
auto right = &Node::right;

// traverse tree, using fold expression:
template<typename T, typename... TP>
Node* traverse (T np, TP... paths) {
    return (np ->* ... ->* paths); // np ->* paths1 ->* paths2 ...
}

int main()
{
    // init binary tree structure:
    Node* root = new Node{0};
    root->left = new Node{1};
    root->left->right = new Node{2};
    // ...
    // traverse binary tree:
    Node* node = traverse(root, left, right);
    // ...
}
```

第2个是递归打印的例子：

```cpp
// basics/addspace.hpp
template<typename T>
class AddSpace
{
    private:
        T const& ref; // refer to argument passed in constructor
    public:
        AddSpace(T const& r): ref(r) {
        }
        friend std::ostream& operator<< (std::ostream& os, AddSpace<T> s) {
            return os << s.ref << ' '; // output passed argument and a space
        }
};

template<typename... Args>
void print (Args... args) {
    ( std::cout << ... << AddSpace(args) ) << '\n';
}
```

## 4.3 应用

标准库中使用可变参数模板的例子：

1. 通过智能指针初始化一个对象，例如`std::make_shared<std::complex<float>>(4.2, 7.7)`
2. 线程库，例如`std::thread t (foo, 42, "hello")`
3. 通过调用构造函数向容器中添加元素，例如`emplace_back()`

## 4.4 可变参数类模板和可变参数表达式

### 4.4.1 可变参数表达式

可变参数模板可以实现函数参数包中每个参数和自身相加：

```cpp
template<typename... T>
void printDoubled (T const&... args)
{
    print (args + args...);
}
```

`printDoubled(7.5, std::string("hello"), std::complex<float>(4,2))`的实例化结果为`print(7.5 + 7.5, std::string("hello") + std::string("hello"), std::complex<float>(4,2) + std::complex<float>(4,2)`。

也可以实现将函数参数包中每个参数加1：

```cpp
template<typename... T>
void addOne (T const&... args)
{
    print (args + 1...);    // ERROR: 1... is a literal with too many decimal points
    print (args + 1 ...);   // OK
    print ((args + 1)...);  // OK
}
```

编译时被解析的表达式中可以包含模板参数包，例如判断模板参数包中所有类型是否相同：

```cpp
template<typename T1, typename... TN>
constexpr bool isHomogeneous (T1, TN...)
{
    return (std::is_same<T1,TN>::value && ...); // since C++17
}
```

`isHomogeneous(43, -1, "hello")`的实例化结果为`std::is_same<int,int>::value && std::is_same<int,char const*>::value`。

### 4.4.2 可变参数和下标运算符

下标运算符可以应用于函数参数包：

```cpp
template<typename C, typename... Idx>
void printElems (C const& coll, Idx... idx)
{
    print (coll[idx]...);
}

std::vector<std::string> coll = {"good", "times", "say", "bye"};
printElems(coll,2,0,3); // same effect to call print (coll[2], coll[0], coll[3])
```

非类型模板参数也可以是可变的：

```cpp
template<std::size_t... Idx, typename C>
void printIdx (C const& coll)
{
    print(coll[Idx]...);
}

std::vector<std::string> coll = {"good", "times", "say", "bye"};
printIdx<2,0,3>(coll);
```

### 4.4.3 可变参数类模板

`std::get`是编译时求值的，所以下面的代码是合法的（这个例子不太好解释，意会一下）：

```cpp
template<std::size_t...>
struct Indices {
};

template<typename T, std::size_t... Idx>
void printByIdx(T t, Indices<Idx...>)
{
    print(std::get<Idx>(t)...);
}

std::array<std::string, 5> arr = {"Hello", "my", "new", "!", "World"};
printByIdx(arr, Indices<0, 4, 3>());

auto t = std::make_tuple(12, "monkeys", 2.0);
printByIdx(t, Indices<0, 1, 2>());
```

### 4.4.4 可变参数推导指引

C++标准库设置了如下的推导指引：

```cpp
namespace std {
    template<typename T, typename... U> array(T, U...)
        -> array<enable_if_t<(is_same_v<T, U> && ...), T>,(1 + sizeof...(U))>;
}
```

根据推导指引，对于数组定义`std::array a{42,45,77}`，`enable_if_t<(is_same_v<T, U> && ...), T>`会被展开为`is_same_v<T, U1> && is_same_v<T, U2> && is_same_v<T, U3>`，如果`U1`、`U2`和`U3`的类型不同，推导就会失败，标准库通过这种方式来保证`std::array`中的元素具有同一类型。

### 4.4.5 可变参数基类和using声明

可变参数模板可以实现多重继承：

```cpp
// basics/varusing.cpp
#include <string>
#include <unordered_set>

class Customer
{
    private:
        std::string name;
    public:
        Customer(std::string const& n) : name(n) { }
        std::string getName() const { return name; }
};

struct CustomerEq {
    bool operator() (Customer const& c1, Customer const& c2) const {
        return c1.getName() == c2.getName();
    }
};

struct CustomerHash {
    std::size_t operator() (Customer const& c) const {
        return std::hash<std::string>()(c.getName());
    }
};

// define class that combines operator() for variadic base classes:
template<typename... Bases>
struct Overloader : Bases...
{
    using Bases::operator()...; // OK since C++17
};

int main()
{
    // combine hasher and equality for customers in one type:
    using CustomerOP = Overloader<CustomerHash,CustomerEq>;
    std::unordered_set<Customer,CustomerHash,CustomerEq> coll1;
    std::unordered_set<Customer,CustomerOP,CustomerOP> coll2;
    // ...
}
```

`Overloader`通过可变参数模板实现了多重继承，同时使用`using Bases::operator()...`引入了各个基类中的调用运算符`()`。

## 4.5 总结

1. 参数包可以让模板处理任意数量的参数
2. 编译器通过递归的方式处理参数包，所以需要一个不含可变参数的模板作为递归终止条件
3. `sizeof...`运算符可以求出参数包中参数数量
4. 可变参数模板的典型应用是转发任意数量和类型的参数
5. 通过使用折叠表达可以实现某种操作应用于参数包中所有参数
