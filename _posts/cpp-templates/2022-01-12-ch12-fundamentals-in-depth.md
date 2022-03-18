---
layout: post
title: 《C++ Templates》第12章 深入模板
categories: Reading
tags: C++
---

# 12 深入模板

## 12.1 参数化声明

C++支持四种类型的模板，可以声明在命名空间中，具体为：

1. 类模板
2. 函数模板
3. 变量模板
4. 别名模板

```cpp
// details/definitions1.hpp
template<typename T>    // a namespace scope class template
class Data {
    public:
        static constexpr bool copyable = true;
        // ...
};

template<typename T>    // a namespace scope function template
void log (T x) {
    // ...
}

template<typename T>    // a namespace scope variable template (since C++14)
T zero = 0;

template<typename T>    // a namespace scope variable template (since C++14)
bool dataCopyable = Data<T>::copyable;

template<typename T>    // a namespace scope alias template
using DataList = Data<T*>;
```

当这四种模板声明在类中时，就变成了：

1. 嵌套类模板
2. 成员函数模板
3. 静态数据成员模板
4. 成员别名模板

```cpp
// details/definitions2.hpp
class Collection {
    public:
        template<typename T>    // an in-class member class template definition
        class Node {
            // ...
        };

        template<typename T>    // an in-class (and therefore implicitly inline)
        T* alloc() {            // member function template definition
            // ...
        }

        template<typename T>    // a member variable template (since C++14)
        static T zero = 0;

        template<typename T>    // a member alias template
        using NodePtr = Node<T>*;
};
```

除了成员别名模板，其余三种在类中声明的模板可以定义在类外：

```cpp
// details/definitions3.hpp
template<typename T>            // a namespace scope class template
class List {
    public:
        List() = default;       // because a template constructor is defined

        template<typename U>    // another member class template,
        class Handle;           // without its definition

        template<typename U>    // a member function template
        List (List<U> const&);  // (constructor)

        template<typename U>    // a member variable template (since C++14)
        static U zero;
};

template<typename T>            // out-of-class member class template definition
    template<typename U>
class List<T>::Handle {
    // ...
};

template<typename T>            // out-of-class member function template definition
    template<typename T2>
List<T>::List (List<T2> const& b)
{
    // ...
}

template<typename T>            // out-of-class static data member template definition
    template<typename U>
U List<T>::zero = 0;
```

联合体也可以是模板：

```cpp
template<typename T>
union AllocChunk {
    T object;
    unsigned char bytes[sizeof(T)];
};
```

### 12.1.1 虚成员函数（模板）

虚成员函数不能是模板，因为C++是通过虚表实现虚函数的，这要求在定义类时就要确定虚函数的个数，但是成员函数模板只有在调用时才能实例化，所以下面的定义会报错：

```cpp
template<typename T>
class Dynamic {
    public:
        virtual ~Dynamic();     // OK: one destructor per instance of Dynamic<T>
        template<typename T2>
        virtual void copy (T2 const&);
                                // ERROR: unknown number of instances of copy()
                                // given an instance of Dynamic<T>
};
```

### 12.1.2 模板和链接

模板的默认链接方式是`extern "C++"`的。模板一般是可以外部链接的，但是也有例外：

```cpp
template<typename T>        // refers to the same entity as a declaration of the
void external();            // same name (and scope) in another file

template<typename T>        // unrelated to a template with the same name in
static void internal();     // another file

template<typename T>        // redeclaration of the previous declaration
static void internal();

namespace {
    template<typename>      // also unrelated to a template with the same name
    void otherInternal();   // in another file, even one that similarly appears
}                           // in an unnamed namespace

namespace {
    template<typename>      // redeclaration of the previous template declaration
    void otherInternal();
}

struct {
    template<typename T> void f(T) {} // no linkage: cannot be redeclared
} x;
```

### 12.1.3 主模板

正常声明的模板称为主模板（Primary Templates）：

```cpp
template<typename T> class Box;                     // OK: primary template
template<typename T> class Box<T>;                  // ERROR: does not specialize
template<typename T> void translate(T);             // OK: primary template
template<typename T> void translate<T>(T);          // ERROR: not allowed for functions
template<typename T> constexpr T zero = T{};        // OK: primary template
template<typename T> constexpr T zero<T> = T{};     // ERROR: does not specialize
```

## 12.2 模板形参

C++支持3种模板形参：

- （类型）模板参数
- 非类型模板参数
- 模板的模板参数

### 12.2.1 类型模板形参

类型模板参数跟在`typename`和`class`后面，后续的类型模板参数需要用`,`分隔，可以`=`提供默认参数。

### 12.2.2 非类型模板形参

非类型模板参数只能是：

- 整型或者枚举类型
- 指针类型
- 成员函数指针
- 左值引用
- `std::nullptr_t`
- `auto`或者`decltype(auto)`

### 12.2.3 模板的模板形参

当模板形参也可以用模板替代时，该模板参数就是模板的模板形参。

### 12.2.4 模板参数包

通过`...`可以声明可变参数模板，模板参数包可以包含多个类型模板参数、非类型模板参数和模板的模板参数：

```cpp
using IntTuple = Tuple<int>;                // OK: one template argument
using IntCharTuple = Tuple<int, char>;      // OK: two template arguments
using IntTriple = Tuple<int, int, int>;     // OK: three template arguments
using EmptyTuple = Tuple<>;                 // OK: zero template arguments

template<typename T, unsigned... Dimensions>
class MultiArray; // OK: declares a nontype template parameter pack

using TransformMatrix = MultiArray<double, 3, 3>; // OK: 3x3 matrix

template<typename T, template<typename,typename>... Containers>
void testContainers(); // OK: declares a template template parameter pack
```

对于类模板，模板参数包只能在最后。而对于函数模板可以有多个模板参数包，但是要保证用来分隔模板参数包的模板参数具有默认值或者可以被推导：

```cpp
template<typename... Types, typename Last>
class LastType; // ERROR: template parameter pack is not the last template parameter

template<typename... TestTypes, typename T>
void runTests(T value);     // OK: template parameter pack is followed
                            // by a deducible template parameter

template<unsigned...> struct Tensor;
template<unsigned... Dims1, unsigned... Dims2>
    auto compose(Tensor<Dims1...>, Tensor<Dims2...>); // OK: the tensor dimensions can be deduced
```

模板参数包也可以用来偏特化：

```cpp
template<typename...> Typelist;
template<typename X, typename Y> struct Zip;
template<typename... Xs, typename... Ys>
    struct Zip<Typelist<Xs...>, Typelist<Ys...>>;
        // OK: partial specialization uses deduction to determine
        // the Xs and Ys substitutions
```

嵌套的模板参数包也是合法的：

```cpp
template<typename... Ts, Ts... vals> struct StaticValues {};
// ERROR: Ts cannot be expanded in its own parameter list

template<typename... Ts> struct ArgList {
    template<Ts... vals> struct Vals {};
};
ArgList<int, char, char>::Vals<3, 'x', 'y'> tada;
```

### 12.2.5 默认模板实参

类的默认模板实参要出现在靠后的位置，而函数模板没有这个要求，但是都不能重复：

```cpp
template<typename T1, typename T2, typename T3,
typename T4 = char, typename T5 = char>
class Quintuple; // OK

template<typename T1, typename T2, typename T3 = char,
typename T4, typename T5>
class Quintuple; // OK: T4 and T5 already have defaults

template<typename T1 = char, typename T2, typename T3,
typename T4, typename T5>
class Quintuple;    // ERROR: T1 cannot have a default argument
                    // because T2 doesn’t have a default

template<typename R = void, typename T>
R* addressof(T& value); // OK: if not explicitly specified, R will be void

template<typename T = void>
class Value;

template<typename T = void>
class Value; // ERROR: repeated default argument
```

以下情况不允许使用默认模板实参：

- 偏特化
- 模板参数包
- 类外定义成员函数模板
- 友元类
- 友元函数声明（友元函数定义可以）

## 12.3 模板实参

### 12.3.1 函数模板实参

函数模板实参可以通过显示指定、推导或者默认形参的方式指定：

```cpp
// details/max.cpp

template<typename T>
T max (T a, T b)
{
    return b < a ? a : b;
}

int main()
{
    ::max<double>(1.0, -3.0);   // explicitly specify template argument
    ::max(1.0, -3.0);           // template argument is implicitly deduced to be double
    ::max<int>(1.0, 3.0);       // the explicit <int> inhibits the deduction;
                                // hence the result has type int
}
```

无法被推导的模板实参（例如函数返回类型）一般会被放在模板声明中靠前的位置：

```cpp
// details/implicit.cpp
template<typename DstT, typename SrcT>
DstT implicit_cast (SrcT const& x) // SrcT can be deduced, but DstT cannot
{
    return x;
}

int main()
{
    double value = implicit_cast<double>(-1);
}
```

### 12.3.2 类型模板实参

任何类型都可以作为类型模板实参，但是要保证代码是合法的：

```cpp
template<typename T>
void clear (T p)
{
    *p = 0;     // requires that the unary * be applicable to T
}

int main()
{
    int a;
    clear(a);   // ERROR: int doesn’t support the unary *
}
```

### 12.3.3 非类型模板实参

非类型模板实参可以是：

- 其它非类型模板形参
- 在编译时被计算的整型常量
- 变量或者函数的地址（函数或者数组可以不加取地址运算符`&`）
- 上面三种的引用
- 成员指针（类的非静态数据成员和非静态成员函数），此时非类型模板参数的类型必须是成员指针类型
- 空指针（对于指针和成员指针类型的非类型模板实参）

一些合法的例子如下：

```cpp
template<typename T, T nontypeParam>
class C;

C<int, 33>* c1;             // integer type

int a;
C<int*, &a>* c2;            // address of an external variable

void f();
void f(int);
C<void (*)(int), f>* c3;    // name of a function: overload resolution selects
                            // f(int) in this case; the & is implied

template<typename T> void templ_func();
C<void(), &templ_func<double>>* c4; // function template instantiations are functions

struct X {
    static bool b;
    int n;
    constexpr operator int() const { return 42; }
};

C<bool&, X::b>* c5;         // static class members are acceptable variable/function names

C<int X::*, &X::n>* c6;     // an example of a pointer-to-member constant

C<long, X{}>* c7;           // OK: X is first converted to int via a constexpr conversion
                            // function and then to long via a standard integer conversion
```

一些不合法的例子如下：

```cpp
template<typename T, T nontypeParam>
class C;

struct Base {
    int i;
} base;

struct Derived : public Base {
} derived;

C<Base*, &derived>* err1;   // ERROR: derived-to-base conversions are not considered

C<int&, base.i>* err2;      // ERROR: fields of variables aren’t considered to be variables

int a[10];
C<int*, &a[0]>* err3;       // ERROR: addresses of array elements aren’t acceptable either
```

### 12.3.4 模板的模板实参

C++17之前要求模板的模板实参要完全和模板的模板实参匹配，不能有多余的默认模板参数，但是C++17中取消了这个限制：

```CPP
#include <list>
    // declares in namespace std:
    // template<typename T, typename Allocator = allocator<T>>
    // class list;
template<typename T1, typename T2, template<typename> class Cont> // Cont expects one parameter
class Rel {
    // ...
};

Rel<int, double, std::list> rel; // ERROR before C++17: std::list has more than one template parameter
```

### 12.3.5 模板实参的等价性

只有在两个模板实参两两相等的情况下才是完全等价，这样实例化的结果就是一样的，比较的过程不考虑类型别名和常量表达式的计算：

```cpp
template<typename T, int I>
class Mix;

using Int = int;

Mix<int, 3*3>* p1;
Mix<Int, 4+5>* p2; // p2 has the same type as p1
```

但是函数模板实例化的函数不会和普通函数、虚函数、编译器隐式合成的构造函数和运算符等价。

## 12.4 可变参数模板

### 12.4.1 参数包展开

参数包展开是指将参数包分成一个个参数的过程，例如下面代码中的`Types...`：

```cpp
template<typename... Types>
class MyTuple : public Tuple<Types...> {
    // extra operations provided only for MyTuple
};

MyTuple<int, float> t2; // inherits from Tuple<int, float>
```

### 12.4.2 参数包展开的时机

参数包展开会发生在：

- 定义多重继承的基类时
- 构造函数中初始化基类时
- 函数参数调用时
- `std::initializers`中
- 类模板和函数模板中
- 函数抛出的异常中（从C++11开始不再支持）
- 声明对齐时
- lambda表达式的捕获列表中
- 函数类型的参数中
- `using`

下面给出几个常见的参数包展开的例子，第一个发生在多重继承和函数参数调用时：

```cpp
template<typename... Mixins>
class Point : public Mixins... {    // base class pack expansion
        double x, y, z;
    public:
        Point() : Mixins()... { }   // base class initializer pack expansion

        template<typename Visitor>
        void visitMixins(Visitor visitor) {
            visitor(static_cast<Mixins&>(*this)...); // call argument pack expansion
        }
};

struct Color { char red, green, blue; };
struct Label { std::string name; };
Point<Color, Label> p;              // inherits from both Color and Label
```

下一个是发生在定义类模板的非类型模板参数时：

```cpp
template<typename... Ts>
struct Values {
    template<Ts... Vs>
    struct Holder {
    };
};

int i;
Values<char, int, int*>::Holder<'a', 17, &i> valueHolder;
```

### 12.4.3 函数参数包

下面是一个在构造函数中初始化基类时展开参数包的例子：

```cpp
template<typename... Mixins>
class Point : public Mixins...
{
        double x, y, z;
    public:
        // default constructor, visitor function, etc. elided
        Point(Mixins... mixins)         // mixins is a function parameter pack
            : Mixins(mixins)... { }     // initialize each base with the supplied mixin value

        struct Color { char red, green, blue; };
        struct Label { std::string name; };
        Point<Color, Label> p({0x7F, 0, 0x7F}, {"center"});
};
```

### 12.4.4 多重和嵌套参数包展开

多个参数包可以同时展开：

```cpp
template<typename F, typename... Types>
void forwardCopy(F f, Types const&... values) {
    f(Types(values)...);
}
```

上面的代码中，模板参数包`Types`和函数参数包`values`通过一一的类型转换进行了展开。

参数包也可以嵌套展开：

```cpp
template<typename... OuterTypes>
    class Nested {
        template<typename... InnerTypes>
        void f(InnerTypes const&... innerValues) {
            g(OuterTypes(InnerTypes(innerValues)...)...);
    }
};
```

### 12.4.5 零长度参数包的展开

零长度的参数包将被忽略：

```cpp
template<>
class Point : {
    Point() : { }
};

template<typename T, typename... Types>
void g(Types... values) {
    T v(values...);
}
```

也就是`Point`将没有基类，`v`将被通过默认构造函数初始化，但是不会被编译器解析为函数定义。

### 12.4.6 折叠表达式

从C++17开始，可以使用二元运算符作用域参数包中的所有参数，例如：

```cpp
template<typename... T> bool g() {
    return (trait<T>() && ... && true);
}
```

当参数包为空时，`&&`运算符结果为`true`，`||`运算符结果为`false`，`,`运算符结果为`void`。

## 12.5 友元

### 12.5.1 类模板的友元类

友元类声明中麻烦的在于指定某个类模板的特化结果作为友元：

```cpp
template<typename T>
class Node;

template<typename T>
class Tree {
    friend class Node<T>;
    // ...
};
```

上面的代码要求在实例化`Tree<T>`时，`Node<T>`也要是可见的的，否则就会报错，但是普通类却没关系：

```cpp
template<typename T>
class Tree {
    friend class Factory;   // OK even if first declaration of Factory
    friend class Node<T>;   // error if Node isn’t visible
};
```

### 12.5.2 类模板的友元函数

函数名后面存在`<>`的函数模板可以被声明为类的友元，但是不能定义该函数模板：

```cpp
template<typename T1, typename T2>
void combine(T1, T2);

class Mixer {
    friend void combine<>(int&, int&);
                        // OK: T1 = int&, T2 = int&
    friend void combine<int, int>(int, int);
                        // OK: T1 = int, T2 = int
    friend void combine<char>(char, int);
                        // OK: T1 = char T2 = int
    friend void combine<char>(char&, int);
                        // ERROR: doesn’t match combine() template
    friend void combine<>(long, long) { ... }
                        // ERROR: definition not allowed!
};
```

对于没有`<>`的友元函数声明：

- 如果没有被作用域运算符`::`修饰，就当做友元函数声明，也可以进行定义
- 如果被作用域运算符`::`修饰，则根据匹配规则匹配友元函数

具体见下例：

```cpp
void multiply(void*); // ordinary function

template<typename T>
void multiply(T); // function template

class Comrades {
    friend void multiply(int) { }               // defines a new function ::multiply(int)
    friend void ::multiply(void*);              // refers to the ordinary function above, not to the multiply<void*> instance
    friend void ::multiply(int);                // refers to an instance of the template
    friend void ::multiply<double*>(double*);   // qualified names can also have angle brackets, but a template must be visible
    friend void ::error() { }                   // ERROR: a qualified friend cannot be a definition
};
```

前面两个例子都是普通类，对于类模板规则是一样的：

```cpp
template<typename T>
class Node {
    Node<T>* allocate();
    // ...
};

template<typename T>
class List {
    friend Node<T>* Node<T>::allocate();
    // ...
};
```

友元函数也可以定义在类模板中，多用于友元函数依赖于类模板的模板参数：

```cpp
template<typename T>
class Creator {
    friend void feed(Creator<T>) {  // every T instantiates a different function ::feed()
    // ...
    }
};

int main()
{
    Creator<void> one;
    feed(one);                      // instantiates ::feed(Creator<void>)
    Creator<double> two;
    feed(two);                      // instantiates ::feed(Creator<double>)
}
```

### 12.5.3 友元模板

当需要将模板实例化的所有结果都定义为友元时，需要使用友元模板：

```cpp
class Manager {
    template<typename T>
        friend class Task;

    template<typename T>
        friend void Schedule<T>::dispatch(Task<T>*);

    template<typename T>
        friend int ticket() {
            return ++Manager::counter;
        }
        static int counter;
};
```

友元模板只能是主模板。

## 12.6 后记
