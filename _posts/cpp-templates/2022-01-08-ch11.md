---
layout: post
title: 《C++ Templates》第11章 泛型库
categories: Reading
tags: C++
---

# 11 泛型库

## 11.1 可调用对象类型

C++中的可调用对象类型（callable type）包括：

- 函数指针
- 重载了调用运算符的类
- 返回函数指针和函数引用的类的成员函数

### 11.1.1 函数对象

下面是类似`std::for_each`功能的模板：

```cpp
// basics/foreach.hpp
template<typename Iter, typename Callable>
void foreach (Iter current, Iter end, Callable op)
{
    while (current != end) {    // as long as not reached the end
        op(*current);           // call passed operator for current element
        ++current;              // and move iterator to next element
    }
}
```

```cpp
// basics/foreach.cpp
#include <iostream>
#include <vector>
#include "foreach.hpp"

// a function to call:
void func(int i)
{
    std::cout << "func() called for: " << i << '\n';
}

// a function object type (for objects that can be used as functions):
class FuncObj {
    public:
    void operator() (int i) const { // Note: const member function
        std::cout << "FuncObj::op() called for: " << i << '\n';
    }
};

int main()
{
    std::vector<int> primes = { 2, 3, 5, 7, 11, 13, 17, 19 };
    foreach(primes.begin(), primes.end(),   // range
            func);                          // function as callable (decays to pointer)
    foreach(primes.begin(), primes.end(),   // range
            &func);                         // function pointer as callable
    foreach(primes.begin(), primes.end(),   // range
            FuncObj());                     // function object as callable
    foreach(primes.begin(), primes.end(),   // range
            [] (int i) {                    // lambda as callable
                std::cout << "lambda called for: " << i << '\n';
            });
}
```

- 由于`foreach<>`中的第三个参数是传值的，传递函数对象`func`会退化为指向该函数的指针，因此第一个调用和第二个调用时一样的
- 当传递重载了调用运算符`()`的类对象`FuncObj`时，`op(*current)`等价于`op.operator()(*current)`，注意要将调用运算符声明为`const`的
- 可调用对象也可以是lambda表达式，也可以称为闭包（closures）

### 11.1.2 成员函数中的额外参数

对于成员函数，由于其中存在隐式的参数`this`，无法通过前面的方式进行调用，所以C++17提供了`std::invoke`来解决这个问题：

```cpp
// basics/foreachinvoke.hpp
#include <utility>
#include <functional>

template<typename Iter, typename Callable, typename... Args>
void foreach (Iter current, Iter end, Callable op, Args const&... args)
{
    while (current != end) {    // as long as not reached the end of the elements
        std::invoke(op,         // call passed callable with
                    args...,    // any additional args
                    *current);  // and the current element
        ++current;
    }
}
```

```cpp
// basics/foreachinvoke.cpp
#include <iostream>
#include <vector>
#include <string>
#include "foreachinvoke.hpp"

// a class with a member function that shall be called
class MyClass {
    public:
        void memfunc(int i) const {
            std::cout << "MyClass::memfunc() called for: " << i << '\n';
        }
};

int main()
{
    std::vector<int> primes = { 2, 3, 5, 7, 11, 13, 17, 19 };

    // pass lambda as callable and an additional argument:
    foreach(primes.begin(), primes.end(),           // elements for 2nd arg of lambda
            [](std::string const& prefix, int i) {  // lambda to call
                std::cout << prefix << i << '\n';
            },
            "- value: ");                           // 1st arg of lambda

    // call obj.memfunc() for/with each elements in primes passed as argument
    MyClass obj;
    foreach(primes.begin(), primes.end(),   // elements used as args
            &MyClass::memfunc,              // member function to call
            obj);                           // object to call memfunc() for
}
```

如果传递给`std::invoke`的可调用对象是成员函数指针，则会将可变参数中的第一个参数作为`this`指针，后面的参数照常传递。

### 11.1.3 包装函数调用

`std::invoke`的另一个应用是包装函数调用：

```cpp
// basics/invoke.hpp
#include <utility>      // for std::invoke()
#include <functional>   // for std::forward()

template<typename Callable, typename... Args>
decltype(auto) call(Callable&& op, Args&&... args)
{
    return std::invoke(std::forward<Callable>(op),      // passed callable with
                        std::forward<Args>(args)...);   // any additional args
}
```

上面的代码通过`decltype(auto)`实现返回引用。如果还需要对`std::invoke`的返回值进行处理然后再返回，则应该写为下面的样子：

```cpp
decltype(auto) ret{std::invoke(std::forward<Callable>(op), std::forward<Args>(args)...)};
// ...
return ret;
```

但是`decltype(auto)`结果不能是`void`，因为`void`不完整类型，应该写为下面的样子：

```cpp
// basics/invokeret.hpp
#include <utility>      // for std::invoke()
#include <functional>   // for std::forward()
#include <type_traits>  // for std::is_same<> and invoke_result<>

template<typename Callable, typename... Args>
decltype(auto) call(Callable&& op, Args&&... args)
{
    if constexpr(std::is_same_v<std::invoke_result_t<Callable, Args...>, void>) {
        // return type is void:
        std::invoke(std::forward<Callable>(op),
                    std::forward<Args>(args)...);
        // ...
        return;
    }
    else {
        // return type is not void:
        decltype(auto) ret{std::invoke(std::forward<Callable>(op),
                                        std::forward<Args>(args)...)};
        // ...
        return ret;
    }
}
```

## 11.2 实现泛型库的其它方法

### 11.2.1 类型特征

C++标准库提供了一些对类型进行操作的模板，可以修改类型：

```cpp
#include <type_traits>

template<typename T>
class C
{
    // ensure that T is not void (ignoring const or volatile):
    static_assert(!std::is_same_v<std::remove_cv_t<T>,void>,
                    "invalid instantiation of class C for void type");
    public:
        template<typename V>
        void f(V&& v) {
            if constexpr(std::is_reference_v<T>) {
                // ... // special code if T is a reference type
            }
            if constexpr(std::is_convertible_v<std::decay_t<V>,T>) {
                // ... // special code if V is convertible to T
            }
            if constexpr(std::has_virtual_destructor_v<V>) {
                // ... // special code if V has virtual destructor
            }
        }
};

std::remove_const_t<int const&> // yields int const&

std::remove_const_t<std::remove_reference_t<int const&>>    // int
std::remove_reference_t<std::remove_const_t<int const&>>    // int const

std::decay_t<int const&> // yields int

make_unsigned_t<int>        // unsigned int
make_unsigned_t<int const&> // undefined behavior (hopefully error)

add_rvalue_reference_t<int>         // int&&
add_rvalue_reference_t<int const>   // int const&&
add_rvalue_reference_t<int const&>  // int const& (lvalue-ref remains lvalue-ref)

is_copy_assignable_v<int>   // yields true (generally, you can assign an int to an int)
is_assignable_v<int,int>    // yields false (can’t call 42 = 42)

is_assignable_v<int&,int&> // yields true

is_swappable_v<int>             // yields true (assuming lvalues)
is_swappable_v<int&,int&>       // yields true (equivalent to the previous check)
is_swappable_with_v<int,int>    // yields false (taking value category into account)
```

### 11.2.2 std::addressof

`std::addressof`可以取出变量和函数的地址：

```cpp
template<typename T>
void f (T&& x)
{
    auto p = &x; // might fail with overloaded operator &
    auto q = std::addressof(x); // works even with overloaded operator &
    // ...
}
```

### 11.2.3 std::declval

`std::declval`可以在不创建对象的情况下使用某种类型的对象：

```cpp
// basics/maxdefaultdeclval.hpp
#include <utility>

template<typename T1, typename T2,
            typename RT = std::decay_t<decltype(true ? std::declval<T1>()
                                                        : std::declval<T2>())>>
RT max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

### 11.1.3 完美转发临时变量

通过将模板中的临时量声明为`auto&&`类型可以实现对模板中临时量的完美转发：

```cpp
template<typename T>
void foo(T x)
{
    auto&& val = get(x);
    // ...
    // perfectly forward the return value of get() to set():
    set(std::forward<decltype(val)>(val));
}
```

## 11.4 引用类型的模板形参

只有显示指定模板实参为引用类型时，模板参数才会变成引用类型：

```cpp
// basics/tmplparamref.cpp
#include <iostream>

template<typename T>
void tmplParamIsReference(T) {
    std::cout << "T is reference: " << std::is_reference_v<T> << '\n';
}

int main()
{
    std::cout << std::boolalpha;
    int i;
    int& r = i;
    tmplParamIsReference(i);        // false
    tmplParamIsReference(r);        // false
    tmplParamIsReference<int&>(i);  // true
    tmplParamIsReference<int&>(r);  // true
}
```

下面的模板包括一个模板参数，以及该类型的非类型模板参数，还为非类型模板参数提供了默认参数值：

```cpp
// basics/referror1.cpp
template<typename T, T Z = T{}>
class RefMem {
    private:
        T zero;
    public:
        RefMem() : zero{Z} {
    }
};

int null = 0;

int main()
{
    RefMem<int> rm1, rm2;
    rm1 = rm2;              // OK

    RefMem<int&> rm3;       // ERROR: invalid default value for Z
    RefMem<int&, 0> rm4;    // ERROR: invalid default value for Z

    extern int null;
    RefMem<int&,null> rm5, rm6;
    rm5 = rm6;              // ERROR: operator= is deleted due to reference member
}
```

上面代码错误的原因是：

- 引用类型无法默认初始化
- 不能用`0`等常量值初始化引用类型
- 编译器无法为包含引用类型的类合成默认的拷贝赋值运算符

引用类型的非类型模板参数非常容易出错：

```cpp
// basics/referror2.cpp
#include <vector>
#include <iostream>

template<typename T, int& SZ>           // Note: size is reference
class Arr {
    private:
        std::vector<T> elems;
    public:
        Arr() : elems(SZ) {             // use current SZ as initial vector size
        }
        void print() const {
            for (int i=0; i<SZ; ++i) {  // loop over SZ elements
                std::cout << elems[i] << ' ';
            }
        }
};

int size = 10;

int main()
{
    Arr<int&,size> y;   // compile-time ERROR deep in the code of class std::vector<>

    Arr<int,size> x;    // initializes internal vector with 10 elements
    x.print();          // OK
    size += 100;        // OOPS: modifies SZ in Arr<>
    x.print();          // run-time ERROR: invalid memory access: loops over 120 elements
}
```

## 11.5 延迟求值

编写泛型库时，要考虑到模板是否会支持不完整类型。例如对于下面的类模板和实例化代码：

```cpp
template<typename T>
class Cont {
    private:
        T* elems;
    public:
        // ...
};

struct Node
{
    std::string value;
    Cont<Node> next; // only possible if Cont accepts incomplete types
};
```

由于`Node`是不完整类型，`Cont`模板要能够处理不完整类型：

```cpp
template<typename T>
class Cont {
    private:
        T* elems;
    public:
        // ...
        typename std::conditional<std::is_move_constructible<T>::value,
                                    T&&,
                                    T&
                                    >::type
        foo();

        template<typename D = T>
        typename std::conditional<std::is_move_constructible<D>::value,
                                    T&&,
                                    T&
                                    >::type
        foo();
}
```

由于`std::is_move_constructible`必须用完整类型实例化，同时模板只在需要时才进行实例化，所以第二种声明方式可以使得`Cont<Node>`不会报错。

## 11.6 编写泛型库的注意事项

1. 使用转发引用来转发模板参数，使用`auto&&`声明需要转发的模板临时变量
2. 当模板参数被声明为转发引用时，要保证转发到的模板的调用参数类型为传引用类型
3. `std::address`可以取出依赖于模板参数的对象的地址
4. 成员函数模板可能会比编译器隐式合成的构造函数和赋值运算符匹配度更高
5. 当模板参数是C风格字符串且传引用类型类型时，考虑使用`std::decay`
6. 当模板参数为非常量引用时，需要注意参数可能被`const`修饰
7. 注意处理模板参数被显示的指定为引用类型的情况
8. 注意模板是否可以处理不完整类型
9. 为C风格数组和字符串进行特化重载

第2条原文：

>When parameters are declared as forwarding references, be prepared that a template parameter has a reference type when passing lvalues.

## 11.7 小结

1. 可调用对象包括函数、函数指针、函数对象、仿函数（functor）和lambda表达式
2. 类的调用运算符`()`应该为`const`的
3. `std::invoke`可以处理可调用对象为成员函数
4. 使用`decltype(auto)`实现返回类型的完美转发
5. 标准库提供了类型识别和类型修改相关的模板（type traits）
6. 使用`std::declval`提取没有被求值的表达式的类型
7. 使用`auto&&`实现模板中临时量的完美转发
8. 注意处理模板参数被显示的指定为引用类型的情况
9. 通过模板可以推迟表达式的求值时间
