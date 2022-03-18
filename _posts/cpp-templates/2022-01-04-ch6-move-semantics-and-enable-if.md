---
layout: post
title: 《C++ Templates》第6章 移动语义和std::enable_if
categories: Reading
tags: C++
---

# 6 移动语义和std::enable_if

这应该是我印象里最好的关于移动语义定义了：移动操作可以将拷贝和赋值操作优化为内部资源的偷取，这是因为被移动的对象是一个将要被释放的对象。原文：

>You can use it to optimize copying and assignments by moving (“stealing”) internal resources from a source object to a destination object instead of copying those contents. This can be done provided the source no longer needs its internal value or state (because it is about to be discarded).

## 6.1 完美转发 {#Perfect-Fowarding}

完美转发是为了保证在模板实例化时仍然保持参数的属性：

- 左值仍然是左值
- 常量仍然是常量
- 被移动的对象仍然是可以被移动的

所以可能需要把代码写成这个样子：

```cpp
// basics/move1.cpp
#include <utility>
#include <iostream>

class X {
    // ...
};

void g (X&) {
    std::cout << "g() for variable\n";
}
void g (X const&) {
    std::cout << "g() for constant\n";
}
void g (X&&) {
    std::cout << "g() for movable object\n";
}

// let f() forward argument val to g():
void f (X& val) {
    g(val);             // val is non-const lvalue => calls g(X&)
}
void f (X const& val) {
    g(val);             // val is const lvalue => calls g(X const&)
}
void f (X&& val) {
    g(std::move(val));  // val is non-const lvalue => needs std::move() to call g(X&&)
}
int main()
{
    X v;                // create variable
    X const c;          // create constant
    f(v);               // f() for nonconstant object calls f(X&) => calls g(X&)
    f(c);               // f() for constant object calls f(X const&) => calls g(X const&)
    f(X());             // f() for temporary calls f(X&&) => calls g(X&&)
    f(std::move(v));    // f() for movable variable calls f(X&&) => calls g(X&&)
}
```

注意在`void f (X&& val)`中仍然需要`std::move`，这是因为C++默认不传递右值的属性。当使用`val`时，仍然是非常量左值，和`void f (X& val)`中的`val`属性是一样的。如果默认传递右值，那么当该值第一次作为某函数的实参时，函数返回后该值可能就失效了。原文：

>The fact that move semantics is not automatically passed through is intentional and important. If it weren’t, we would lose the value of a movable object the first time we use it in a function.

上面的代码写为模板的形式会是这个样子：

```cpp
template<typename T>
void f (T val) {
    g(T);
}
```

但是只对左值版本有效（详见第1章中的推导规则）。为了支持右值版本，应该写成下面的样子：

```cpp
template<typename T>
void f (T&& val) {
    g(std::forward<T>(val)); // perfect forward val to g()
}
```

由于`std::move`不是模板，所以要用`std::forward`来转发潜在的移动语义。

注意：模板中的`T&&`和普通函数中的`X&&`是不一样的：

- 只有可以被移动的对象才可以作为参数类型为`X&&`的函数的参数
- `T&&`表明模板的参数的类型是转发引用（forwarding reference），或者在C++17之前叫做通用引用（universal reference），实参可以是常量左值、非常量左值和可移动对象。即使模板函数中的`typename T::iterator&&`也只是声明变量的类型为右值引用，而不是转发引用

完整程序如下：

```cpp
// basics/move2.cpp
#include <utility>
#include <iostream>

class X {
    // ...
};

void g (X&) {
    std::cout << "g() for variable\n";
}
void g (X const&) {
    std::cout << "g() for constant\n";
}
void g (X&&) {
    std::cout << "g() for movable object\n";
}

// let f() perfect forward argument val to g():
template<typename T>
void f (T&& val) {
    g(std::forward<T>(val)); // call the right g() for any passed argument val
}

int main()
{
    X v;                // create variable
    X const c;          // create constant
    f(v);               // f() for variable calls f(X&) => calls g(X&)
    f(c);               // f() for constant calls f(X const&) => calls g(X const&)
    f(X());             // f() for temporary calls f(X&&) => calls g(X&&)
    f(std::move(v));    // f() for move-enabled variable calls f(X&&) => calls g(X&&)
}
```

## 6.2 特殊成员函数模板 {#Special-Member-Function-Template}

可以将构造函数模板化：

```cpp
// basics/specialmemtmpl1.cpp
#include <utility>
#include <string>
#include <iostream>

class Person
{
    private:
        std::string name;
    public:
        // constructor for passed initial name:
        explicit Person(std::string const& n) : name(n) {
            std::cout << "copying string-CONSTR for '" << name << "'\n";
        }
        explicit Person(std::string&& n) : name(std::move(n)) {
            std::cout << "moving string-CONSTR for '" << name << "'\n";
        }
        // copy and move constructor:
        Person (Person const& p) : name(p.name) {
            std::cout << "COPY-CONSTR Person '" << name << "'\n";
        }
        Person (Person&& p) : name(std::move(p.name)) {
            std::cout << "MOVE-CONSTR Person '" << name << "'\n";
        }
};

int main()
{
    std::string s = "sname";
    Person p1(s);               // init with string object => calls copying string-CONSTR
    Person p2("tmp");           // init with string literal => calls moving string-CONSTR
    Person p3(p1);              // copy Person => calls COPY-CONSTR
    Person p4(std::move(p1));   // move Person => calls MOVE-CONST
}
```

有了完美转发，也可以将接受`std::string`的构造函数写成模板形式：

```cpp
// basics/specialmemtmpl2.hpp
#include <utility>
#include <string>
#include <iostream>

class Person
{
    private:
        std::string name;
    public:
        // generic constructor for passed initial name:
        template<typename STR>
        explicit Person(STR&& n) : name(std::forward<STR>(n)) {
            std::cout << "TMPL-CONSTR for '" << name << "'\n";
        }
        // copy and move constructor:
        Person (Person const& p) : name(p.name) {
            std::cout << "COPY-CONSTR Person '" << name << "'\n";
        }
        Person (Person&& p) : name(std::move(p.name)) {
            std::cout << "MOVE-CONSTR Person '" << name << "'\n";
        }
};
```

但是此时`Person p3(p1)`会报错，因为根据模板匹配规则，`p1`不是常量，所以`Person(STR&& n)`比`Person (Person const& p)`匹配度更高，但是`std::string`却无法通过`Person`进行构造，所以需要通过`std::enable_if`来禁止匹配模板函数。

## 6.3 std::enable_if

从C++11开始，可以使用`std::enable_if`来在某些条件下禁掉函数模板：

```cpp
template<typename T>
typename std::enable_if<(sizeof(T) > 4)>::type
foo() {
}
```

上面的代码通过`typename`来提示编译器后面的`T`是模板参数。`std::enable_if`的含义为：

- 当表达式为假（`T`类型的大小小于等于`4`）时，`std::enable_if::type`是未定义的，但是根据模板的“代换失败不是错误（Substitution Failure Is Not An Error，SFINAE）”原则，`foo()`的模板定义就被忽略了
- 当表达式为真时，如果没有第二个模板参数，则`std::enable_if::type`是`void`，否则就是第二个模板参数的类型

从C++14开始，也可以简写为下面的样子：

```cpp
template<typename T>
std::enable_if_t<(sizeof(T) > 4)>
foo() {
}
```

## 6.4 使用std::enable_if

使用`std::enable_if`改写的`Person`类如下：

```cpp
// basics/specialmemtmpl3.hpp
#include <utility>
#include <string>
#include <iostream>
#include <type_traits>

template<typename T>
using EnableIfString = std::enable_if_t<std::is_convertible_v<T,std::string>>;

class Person
{
    private:
        std::string name;
    public:
        // generic constructor for passed initial name:
        template<typename STR, typename = EnableIfString<STR>>
        explicit Person(STR&& n)
        : name(std::forward<STR>(n)) {
            std::cout << "TMPL-CONSTR for '" << name << "'\n";
        }
        // copy and move constructor:
        Person (Person const& p) : name(p.name) {
            std::cout << "COPY-CONSTR Person '" << name << "'\n";
        }
        Person (Person&& p) : name(std::move(p.name)) {
            std::cout << "MOVE-CONSTR Person '" << name << "'\n";
        }
};
```

```cpp
// basics/specialmemtmpl3.cpp
#include "specialmemtmpl3.hpp"
int main()
{
    std::string s = "sname";
    Person p1(s);               // init with string object => calls TMPL-CONSTR
    Person p2("tmp");           // init with string literal => calls TMPL-CONSTR
    Person p3(p1);              // OK => calls COPY-CONSTR
    Person p4(std::move(p1));   // OK => calls MOVE-CONST
}
```

注意在C++14中要写为这样子：

```cpp
template<typename T>
using EnableIfString = std::enable_if_t<std::is_convertible<T, std::string>::value>;
```

而在C++11中要写为这样子：

```cpp
template<typename T>
using EnableIfString
    = typename std::enable_if<std::is_convertible<T, std::string>::value>::type;
```

还可以用`std::is_constructible`：

```cpp
template<typename T>
using EnableIfString = std::enable_if_t<std::is_constructible_v<std::string, T>>;
```

在使用这种条件模板时，一定要使用对应的语义，如`STR`类型可以转换为`std::string`则模板定义有效，而不能是`STR`类型不能转换为`Person`。原文：

>If you wonder why we don’t instead check whether STR is “not convertible to Person,” beware: We are defining a function that might allow us to convert a string to a Person. So the constructor has to know whether it is enabled, which depends on whether it is convertible, which depends on whether it is enabled, and so on. Never use enable_if in places that impact the condition used by enable_if. This is a logical error that compilers do not necessarily detect.

不能通过`std::enable_if`禁掉编译器合成的拷贝和移动构造函数，除非拷贝构造函数被删除了：

```cpp
class C
{
    public:
        // ...
        // user-define the predefined copy constructor as deleted
        // (with conversion to volatile to enable better matches)
        C(C const volatile&) = delete;

        // implement copy constructor template with better match:
        template<typename T>
        C (T const&) {
            std::cout << "tmpl copy constructor\n";
        }
        // ...
};
```

## 6.5 使用concepts简化enable_if

未标准化的内容，不知道在说啥。

## 6.6 总结

1. 可以通过转发引用`T&&`和`std::forward`来实现完美转发
2. 完美转发的成员函数模板可能比普通的函数匹配度更高
3. 通过`std::enable_if`可以在编译期间禁掉某些模板函数
4. 通过`std::enable_if`可以禁掉匹配度更高的构造函数模板和赋值运算符模板，从而让编译器优先匹配隐式合成的构造函数
5. 通过`std::enable_if`和`delete`可以模板化编译器合成的构造函数
6. Concepts will allow us to use a more intuitive syntax for requirements on function templates
