---
layout: post
title: 《C++ Templates》第19章 类型特征模板
categories: Reading
tags: C++
---

# 19 类型特征模板

## 19.1 序列累加的例子

### 19.1.1 固定类型特征

假设有一个对容器中元素累加的模板：

```cpp
// traits/accum1.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

template<typename T>
T accum (T const* beg, T const* end)
{
    T total{}; // assume this actually creates a zero value
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}

#endif // ACCUM_HPP
```

分别使用该模板对整型数组和字符进行累加：

```cpp
// traits/accum1.cpp
#include "accum1.hpp"
#include <iostream>

int main()
{
    // create array of 5 integer values
    int num[] = { 1, 2, 3, 4, 5 };

    // print average value
    std::cout << "the average value of the integer values is "
                << accum(num, num+5) / 5
                << '\n';

    // create array of character values
    char name[] = "templates";
    int length = sizeof(name)-1;

    // (try to) print average character value
    std::cout << "the average value of the characters in \""
                << name << "\" is "
                << accum(name, name+length) / length
                << '\n';
}
```

输出结果为：

```log
the average value of the integer values is 3
the average value of the characters in "templates" is -5
```

由于`char`所表示的数据范围较小，累加的过程可能存在溢出或者截断，所以会出现负值。当然可以通过将返回类型也作为模板参数解决这个问题，但是会比较麻烦。

另一种方式是为不同的类型`T`关联一个返回值的类型，这个相关联的类型可以视为`T`的类型特征。原文：

>An alternative approach to the extra parameter is to create an association between each type T for which accum() is called and the corresponding type that should be used to hold the accumulated value. This association could be considered characteristic of the type T, and therefore the type in which the sum is computed is sometimes called a trait of T.

类型特征可以通过模板特化实现：

```cpp
// traits/accumtraits2.hpp
template<typename T>
struct AccumulationTraits;

template<>
struct AccumulationTraits<char> {
    using AccT = int;
};

template<>
struct AccumulationTraits<short> {
    using AccT = int;
};

template<>
struct AccumulationTraits<int> {
    using AccT = long;
};

template<>
struct AccumulationTraits<unsigned int> {
    using AccT = unsigned long;
};

template<>
struct AccumulationTraits<float> {
    using AccT = double;
};
```

然后将模板修改为：

```cpp
// traits/accum2.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

#include "accumtraits2.hpp"

template<typename T>
auto accum (T const* beg, T const* end)
{
    // return type is traits of the element type
    using AccT = typename AccumulationTraits<T>::AccT;

    AccT total{}; // assume this actually creates a zero value
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}

#endif // ACCUM_HPP
```

这就会得到正确的输出：

```log
the average value of the integer values is 3
the average value of the characters in "templates" is 108
```

### 19.1.2 值特征

类型特征的解决办法很好的解决了返回类型的问题，但是却不能保证初值`total`是一个合理值，为此可以再为类型`T`提供值特征：

```cpp
// traits/accumtraits3.hpp
template<typename T>
struct AccumulationTraits;

template<>
struct AccumulationTraits<char> {
    using AccT = int;
    static AccT const zero = 0;
};

template<>
struct AccumulationTraits<short> {
    using AccT = int;
    static AccT const zero = 0;
};

template<>
struct AccumulationTraits<int> {
    using AccT = long;
    static AccT const zero = 0;
};
// ...
```

并用如下代码改写模板：

```cpp
// traits/accum3.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

#include "accumtraits3.hpp"

template<typename T>
auto accum (T const* beg, T const* end)
{
    // return type is traits of the element type
    using AccT = typename AccumulationTraits<T>::AccT;

    AccT total = AccumulationTraits<T>::zero; // init total by trait value
    while (beg != end) {
    total += *beg;
    ++beg;
    }
    return total;
}

#endif // ACCUM_HPP
```

但是C++只允许类的静态常量数据成员是整型或者枚举类型，对于浮点型，应该使用`constexpr`；对于自定义类型，只能通过静态函数的方式：

```cpp
// traits/accumtraits4.hpp
template<typename T>
struct AccumulationTraits;

template<>
struct AccumulationTraits<char> {
    using AccT = int;
    static constexpr AccT zero() {
        return 0;
    }
};

template<>
struct AccumulationTraits<short> {
    using AccT = int;
    static constexpr AccT zero() {
        return 0;
    }
};

template<>
struct AccumulationTraits<int> {
    using AccT = long;
    static constexpr AccT zero() {
        return 0;
    }
};

template<>
struct AccumulationTraits<unsigned int> {
    using AccT = unsigned long;
    static constexpr AccT zero() {
        return 0;
    }
};

template<>
struct AccumulationTraits<float> {
    using AccT = double;
    static constexpr AccT zero() {
        return 0;
    }
};
// ...
```

```cpp
// traits/accumtraits4bigint.hpp
template<>
struct AccumulationTraits<BigInt> {
    using AccT = BigInt;
    static BigInt zero() {
        return BigInt{0};
    }
};
```

不过在C++17中，对于自定义类型还可以写为`inline`的形式：

```cpp
template<>
struct AccumulationTraits<BigInt> {
    using AccT = BigInt;
    inline static BigInt const zero = BigInt{0}; // OK since C++17
};
```

### 19.1.3 将类型特征作为模板参数 {#Parameterized-Traits}

类型特征也可以作为模板参数：

```cpp
// traits/accum5.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

#include "accumtraits4.hpp"

template<typename T, typename AT = AccumulationTraits<T>>
auto accum (T const* beg, T const* end)
{
    typename AT::AccT total = AT::zero();
    while (beg != end) {
        total += *beg;
        ++beg;
    }
    return total;
}

#endif // ACCUM_HPP
```

## 19.2 类型特征和策略

累加模板的过程可以抽象为遍历序列中的每个元素，并和当前得到的累加值进行加法求和，这里将求和操作称为策略（policy），因此可以将模板策略参数化，通过模板参数来控制求和、求积的过程：

```cpp
// traits/accum6.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

#include "accumtraits4.hpp"
#include "sumpolicy1.hpp"

template<typename T,
            typename Policy = SumPolicy,
            typename Traits = AccumulationTraits<T>>
auto accum (T const* beg, T const* end)
{
    using AccT = typename Traits::AccT;
    AccT total = Traits::zero();
    while (beg != end) {
        Policy::accumulate(total, *beg);
        ++beg;
    }
    return total;
}

#endif // ACCUM_HPP
```

```cpp
// traits/sumpolicy1.hpp
#ifndef SUMPOLICY_HPP
#define SUMPOLICY_HPP

class SumPolicy {
    public:
        template<typename T1, typename T2>
        static void accumulate (T1& total, T2 const& value) {
            total += value;
        }
};

#endif // SUMPOLICY_HPP
```

```cpp
// traits/accum6.cpp
#include "accum6.hpp"
#include <iostream>

class MultPolicy {
    public:
        template<typename T1, typename T2>
        static void accumulate (T1& total, T2 const& value) {
            total *= value;
        }
};

int main()
{
    // create array of 5 integer values
    int num[] = { 1, 2, 3, 4, 5 };

    // print product of all values
    std::cout << "the product of the integer values is "
                << accum<int,MultPolicy>(num, num+5)
                << '\n';
}
```

由于初值仍然为0，所以代码并不正确，有两种方法可以解决这个问题，一个是将初值作为策略的一部分，还有就是仿照标注库`std::accumulate`将初值作为参数。

### 19.2.1 类型特征和策略的区别

- 类型特征（trait）表示模板参数的额外属性：
  1. 可以不作为模板参数传递
  2. 在作为模板参数的情况下一般具有默认值，调用时基本不需要提供模板实参，见[19.1.3](#Parameterized-Traits)
  3. 一般和模板参数密切相关
  4. 一般是类型或者常量
  5. 一般通过类型特征模板获取
- 策略（policy）表示函数或者类型中可以配置的行为（个人认为重点在可配置）
  1. 大多数情况下要通过模板参数传递才能发挥作用
  2. 一般不提供默认模板实参，需要显示指定
  3. 一般和其它模板参数关系不大
  4. 一般通过成员函数实现
  5. 一般实现在普通类或者类模板

### 19.2.2 通过类模板实现策略

策略还可以实现为类模板，然后将其作为模板的模板参数：

```cpp
// traits/sumpolicy2.hpp
#ifndef SUMPOLICY_HPP
#define SUMPOLICY_HPP

template<typename T1, typename T2>
class SumPolicy {
    public:
        static void accumulate (T1& total, T2 const& value) {
            total += value;
        }
};

#endif // SUMPOLICY_HPP
```

```cpp
// traits/accum7.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

#include "accumtraits4.hpp"
#include "sumpolicy2.hpp"

template<typename T,
            template<typename,typename> class Policy = SumPolicy,
            typename Traits = AccumulationTraits<T>>
auto accum (T const* beg, T const* end)
{
    using AccT = typename Traits::AccT;
    AccT total = Traits::zero();
    while (beg != end) {
        Policy<AccT,T>::accumulate(total, *beg);
        ++beg;
    }
    return total;
}

#endif // ACCUM_HPP
```

### 19.2.3 组合使用类型特征和策略

当都通过模板参数传递类型特征和策略时，一般将策略放在类型特征之前，这是因为策略可能不使用默认模板实参，而类型特征一般使用默认模板实参。

### 19.2.4 通过迭代器实现累加

前面的累加模板都是通过指针实现的，标准库中更通用的方式是通过迭代器（书中的例子还是会导致溢出和截断）：

```cpp
// traits/accum0.hpp
#ifndef ACCUM_HPP
#define ACCUM_HPP

#include <iterator>

template<typename Iter>
auto accum (Iter start, Iter end)
{
    using VT = typename std::iterator_traits<Iter>::value_type;
    VT total{}; // assume this actually creates a zero value
    while (start != end) {
        total += *start;
        ++start;
    }
    return total;
}
#endif // ACCUM_HPP
```

## 19.3 类型函数

类型函数是以类型为参数并返回一个类型或者值的函数，如果将`sizeof`视为函数的话，那么其就是一个类型函数，下面的代码可以认为是`sizeof`的函数版本：

```cpp
// traits/sizeof.cpp
#include <cstddef>
#include <iostream>

template<typename T>
struct TypeSize {
    static std::size_t const value = sizeof(T);
};

int main()
{
    std::cout << "TypeSize<int>::value = "
                << TypeSize<int>::value << '\n';
}
```

### 19.3.1 获取容器中元素类型的模板

下面的偏特化模板可以获取容器中元素的类型：

```cpp
// traits/elementtype.hpp
#include <vector>
#include <list>

template<typename T>
struct ElementT;                    // primary template

template<typename T>
struct ElementT<std::vector<T>> {   // partial specialization for std::vector
    using Type = T;
};

template<typename T>
struct ElementT<std::list<T>> {     // partial specialization for std::list
    using Type = T;
};
// ...

template<typename T, std::size_t N>
struct ElementT<T[N]> {             // partial specialization for arrays of known bounds
    using Type = T;
};

template<typename T>
struct ElementT<T[]> {              // partial specialization for arrays of unknown bounds
    using Type = T;
};
// ...
```

使用方法为：

```cpp
// traits/elementtype.cpp
#include "elementtype.hpp"
#include <vector>
#include <iostream>
#include <typeinfo>

template<typename T>
void printElementType (T const& c)
{
    std::cout << "Container of "
                << typeid(typename ElementT<T>::Type).name()
                << " elements.\n";
}

int main()
{
    std::vector<bool> s;
    printElementType(s);
    int arr[42];
    printElementType(arr);
}
```

### 19.3.2 类型修饰模板（Transformation Traits）

类型修饰模板可以移除和添加类型中的引用、移除限定符和实现类型退化（decay）。

#### 移除引用

```cpp
// traits/removereference.hpp
template<typename T>
struct RemoveReferenceT {
    using Type = T;
};

template<typename T>
struct RemoveReferenceT<T&> {
    using Type = T;
};

template<typename T>
struct RemoveReferenceT<T&&> {
    using Type = T;
};
```

#### 添加引用

```cpp
// traits/addreference.hpp
template<typename T>
struct AddLValueReferenceT {
    using Type = T&;
};

template<typename T>
using AddLValueReference = typename AddLValueReferenceT<T>::Type;

template<typename T>
struct AddRValueReferenceT {
    using Type = T&&;
};

template<typename T>
using AddRValueReference = typename AddRValueReferenceT<T>::Type;
```

引用折叠规则适用于这里。由于`void`是不完整类型，所以需要实现以下特化（这里只是`AddLValueReferenceT<>`的例子）：

```cpp
template<>
struct AddLValueReferenceT<void> {
    using Type = void;
};

template<>
struct AddLValueReferenceT<void const> {
    using Type = void const;
};

template<>
struct AddLValueReferenceT<void volatile> {
    using Type = void volatile;
};

template<>
struct AddLValueReferenceT<void const volatile> {
    using Type = void const volatile;
};
```

#### 移除限定符

移除`const`限定：

```cpp
// traits/removeconst.hpp
template<typename T>
struct RemoveConstT {
    using Type = T;
};

template<typename T>
struct RemoveConstT<T const> {
    using Type = T;
};

template<typename T>
using RemoveConst = typename RemoveConstT<T>::Type;
```

同时移除`const`和`volatile`限定：

```cpp
// traits/removecv.hpp
#include "removeconst.hpp"
#include "removevolatile.hpp"

template<typename T>
struct RemoveCVT : RemoveConstT<typename RemoveVolatileT<T>::Type> {
};

template<typename T>
using RemoveCV = typename RemoveCVT<T>::Type;
```

`RemoveCVT<>`通过`RemoveVolatileT<>`移除了`volatile`限定，还通过继承的方式继承了`RemoveConstT::Type`。

#### 实现类型退化

当模板参数为传值类型时，会发生类型退化：

```cpp
// traits/passbyvalue.cpp
#include <iostream>
#include <typeinfo>
#include <type_traits>

template<typename T>
void f(T)
{
}

template<typename A>
void printParameterType(void (*)(A))
{
    std::cout << "Parameter type: " << typeid(A).name() << '\n';
    std::cout << "- is int: " << std::is_same<A,int>::value << '\n';
    std::cout << "- is const: " << std::is_const<A>::value << '\n';
    std::cout << "- is pointer: " << std::is_pointer<A>::value << '\n';
}

int main()
{
    printParameterType(&f<int>);
    printParameterType(&f<int const>);
    printParameterType(&f<int[7]>);
    printParameterType(&f<int(int)>);
}
```

输出为：

```log
Parameter type: i
- is int:     1
- is const:   0
- is pointer: 0
Parameter type: i
- is int:     1
- is const:   0
- is pointer: 0
Parameter type: Pi
- is int:     0
- is const:   0
- is pointer: 1
Parameter type: PFiiE
- is int:     0
- is const:   0
- is pointer: 1
```

模拟类型退化的模板为：

```cpp
template<typename T>
struct DecayT : RemoveCVT<T> {
};

template<typename T>
struct DecayT<T[]> {
    using Type = T*;
};

template<typename T, std::size_t N>
struct DecayT<T[N]> {
    using Type = T*;
};

template<typename R, typename... Args>
struct DecayT<R(Args...)> {
    using Type = R (*)(Args...);
};

template<typename R, typename... Args>
struct DecayT<R(Args..., ...)> {
    using Type = R (*)(Args..., ...);
};
```

```cpp
// traits/decay.cpp
#include <iostream>
#include <typeinfo>
#include <type_traits>
#include "decay.hpp"

template<typename T>
void printDecayedType()
{
    using A = typename DecayT<T>::Type;
    std::cout << "Parameter type: " << typeid(A).name() << '\n';
    std::cout << "- is int: " << std::is_same<A,int>::value << '\n';
    std::cout << "- is const: " << std::is_const<A>::value << '\n';
    std::cout << "- is pointer: " << std::is_pointer<A>::value << '\n';
}

int main()
{
    printDecayedType<int>();
    printDecayedType<int const>();
    printDecayedType<int[7]>();
    printDecayedType<int(int)>();
}
```

对于为什么需要第二个可变参数特化模板我也没看懂。原文：

>Strictly speaking, the comma prior to the second ellipsis (...) is optional but is provided here for clarity. Due to the ellipsis being optional, the function type in the first partial specialization is actually syntactically ambiguous: It can be parsed as either R(Args, ...) (a C-style varargs parameter) or R(Args... name) (a parameter pack). The second interpretation is picked because Args is an unexpanded parameter pack. We can explicitly add the comma in the (rare) cases where the other interpretation is desired.

### 19.3.3 谓词模板

谓词模板接受多个类型参数，返回一个布尔值。

#### 判断类型是否相同

`IsSameT<>`判断两个类型是否相同：

```cpp
// traits/issame0.hpp
template<typename T1, typename T2>
struct IsSameT {
    static constexpr bool value = false;
};

template<typename T>
struct IsSameT<T, T> {
    static constexpr bool value = true;
};
```

#### 将布尔值转换为类型

通过下面的模板可以实现将布尔值转换为类型：

```cpp
// traits/boolconstant.hpp
template<bool val>
struct BoolConstant {
    using Type = BoolConstant<val>;
    static constexpr bool value = val;
};

using TrueType = BoolConstant<true>;
using FalseType = BoolConstant<false>;
```

这样`IsSameT<>`就可以通过继承的方式实现：

```cpp
// traits/issame.hpp
#include "boolconstant.hpp"

template<typename T1, typename T2>
struct IsSameT : FalseType
{
};

template<typename T>
struct IsSameT<T, T> : TrueType
{
};
```

将布尔值转换为类型的好处是可以在编译时实现函数分发，相比于向函数中传递`true`和`false`从而实现在运行时执行不同分支的代码的方式来说会更快一些（我觉得是这样子）：

```cpp
// traits/issame.cpp
#include "issame.hpp"
#include <iostream>

template<typename T>
void fooImpl(T, TrueType)
{
    std::cout << "fooImpl(T,true) for int called\n";
}

template<typename T>
void fooImpl(T, FalseType)
{
    std::cout << "fooImpl(T,false) for other type called\n";
}

template<typename T>
void foo(T t)
{
    fooImpl(t, IsSameT<T,int>{}); // choose impl. depending on whether T is int
}

int main()
{
    foo(42);    // calls fooImpl(42, TrueType)
    foo(7.7);   // calls fooImpl(42, FalseType)
}
```

### 19.3.4 结果类型模板 {#Result-Type-Traits}

结果类型模板给出多个类型的运算结果。

例如对于加法，最简单的实现为：

```cpp
// traits/plus1.hpp
template<typename T1, typename T2>
struct PlusResultT {
    using Type = decltype(T1() + T2());
};

template<typename T1, typename T2>
using PlusResult = typename PlusResultT<T1, T2>::Type;
```

但是这种方式要求`T1`和`T2`可以被默认构造，为了消除这个限制，可以使用`std::declval`：

```cpp
// traits/plus2.hpp
#include <utility>

template<typename T1, typename T2>
struct PlusResultT {
    using Type = decltype(std::declval<T1>() + std::declval<T2>());
};

template<typename T1, typename T2>
using PlusResult = typename PlusResultT<T1, T2>::Type;
```

`std::declval`的定义如下：

```cpp
namespace std {
    template<typename T>
    add_rvalue_reference_t<T> declval() noexcept;
}
```

因为这个模板只用在`decltype`和`sizeof`，所以标准库并不提供定义。原文：

>This function template is intentionally left undefined because it’s only meant to be used within
decltype, sizeof, or some other context where no definition is ever needed.

## 19.4 基于SFINAE规则的类型特征模板

### 19.4.1 通过SFINAE规则实现函数重载 {#SFINAE-Out-Function-Overloads}

基于SFINAE规则实现的函数重载可以用于判断类型是否可以被默认构造：

```cpp
// traits/isdefaultconstructible1.hpp
#include "issame.hpp"

template<typename T>
struct IsDefaultConstructibleT {
    private:
        // test() trying substitute call of a default constructor for T passed as U:
        template<typename U, typename = decltype(U())>
            static char test(void*);
        // test() fallback:
        template<typename>
            static long test(...);
    public:
        static constexpr bool value
            = IsSameT<decltype(test<T>(nullptr)), char>::value;
};
```

使用方法为：

```cpp
IsDefaultConstructibleT<int>::value     // yields true

struct S {
    S() = delete;
};
IsDefaultConstructibleT<S>::value       // yields false
```

`test()`中不能使用`T`作为模板参数的原因在于`IsDefaultConstructibleT<T>::value`会先以参数`T`实例化类模板，在这个过程中会实例化`test()`的声明，如果`T`不能被默认构造，将会出现编译错误，使用`U`将实例化推迟到了引用`value`的时刻，也就是利用SFINAE规则的时刻。原文：

>This doesn’t work, however, because for any T, always, all member functions are substituted, so that for a type that isn’t default constructible, the code fails to compile instead of ignoring the first test() overload. By passing the class template parameter T to a function template parameter U, we create a specific SFINAE context only for the second test() overload.

另一种基于谓词模板的判断方式如下：

```cpp
// traits/isdefaultconstructible2.hpp
#include <type_traits>

template<typename T>
struct IsDefaultConstructibleHelper {
    private:
        // test() trying substitute call of a default constructor for T passed as U:
        template<typename U, typename = decltype(U())>
            static std::true_type test(void*);
        // test() fallback:
        template<typename>
            static std::false_type test(...);
    public:
        using Type = decltype(test<T>(nullptr));
};

template<typename T>
struct IsDefaultConstructibleT : IsDefaultConstructibleHelper<T>::Type {
};
```

### 19.4.2 通过SFINAE规则实现偏特化 {#SFINAE-Out-Partial-Specializations}

基于SFINAE规则实现的偏特化也可以用于判断类型是否可以被默认构造：

```cpp
// traits/isdefaultconstructible3.hpp
#include "issame.hpp"
#include <type_traits> // defines true_type and false_type

// helper to ignore any number of template parameters:
template<typename...> using VoidT = void;

// primary template:
template<typename, typename = VoidT<>>
struct IsDefaultConstructibleT : std::false_type
{
};

// partial specialization (may be SFINAE’d away):
template<typename T>
struct IsDefaultConstructibleT<T, VoidT<decltype(T())>> : std::true_type
{
};
```

### 19.4.3 基于SFINAE规则的泛型lambda表达式 {#Using-Generic-Lambdas-for-SFINAE}

基于SFINAE规则实现的泛型lambda表达式也可以用于判断类型是否可以被默认构造，但是很复杂：

```cpp
// traits/isvalid.hpp
#include <utility>

// helper: checking validity of f(args...) for F f and Args... args:
template<typename F, typename... Args,
            typename = decltype(std::declval<F>()(std::declval<Args&&>()...))>
std::true_type isValidImpl(void*);

// fallback if helper SFINAE’d out:
template<typename F, typename... Args>
std::false_type isValidImpl(...);

// define a lambda that takes a lambda f and returns whether calling f with args is valid
inline constexpr
auto isValid = [](auto f) {
                            return [](auto&&... args) {
                                        return decltype(isValidImpl<decltype(f),
                                                                    decltype(args)&&...
                                                                    >(nullptr)){};
                                    };
                            };

// helper template to represent a type as a value
template<typename T>
struct TypeT {
    using Type = T;
};

// helper to wrap a type as a value
template<typename T>
constexpr auto type = TypeT<T>{};

// helper to unwrap a wrapped type in unevaluated contexts
template<typename T>
T valueT(TypeT<T>); // no definition needed
```

使用方法为：

```cpp
constexpr auto isDefaultConstructible
    = isValid([](auto x) -> decltype((void)decltype(valueT(x))()) {
                });

isDefaultConstructible(type<int>)   // true (int is default-constructible)
isDefaultConstructible(type<int&>)  // false (references are not default-constructible)
```

一点点分析。`isDefaultConstructible`和`isValid()`都是常量表达式，可以在编译时计算。`isDefaultConstructible`是以lambda表达式`[](auto x) -> decltype((void)decltype(valueT(x))()) {}`调用`isValid()`返回的lambda表达式，代换后的结果为：

```cpp
constexpr auto isDefaultConstructible
    = [](auto&&... args) {
        return decltype(isValidImpl<decltype([](auto x) -> decltype((void)decltype(valueT(x))())),
                                    decltype(args)&&...
                                    >(nullptr)
                        ){};
    };
```

`isDefaultConstructible`是一个泛型lambda表达式，所以`auto&&... args`为转发引用。当以`TypeT<int>`类型或者`TypeT<int&>`类型的**临时对象**调用`isDefaultConstructible`时，推导结果为`TypeT<int>`和`TypeT<int&>`。由于上述`return`语句的最后是`{}`，所以`isDefaultConstructible`返回的将是`isValidImpl<>`模板的返回类型创建的临时对象，也就是`std::true_type`或者`std::false_type`类型的临时对象。

现在只需要确定`return`后面的`decltype`的推导结果就可以了。注意到这是一个函数调用，参数为`nullptr`，将匹配`isValidImpl(void*)`:

- 第1个参数包含泛型lambda表达式，编译器将创建一个重载了函数调用运算符模板的闭包类型：

```cpp
class SomeCompilerSpecificName
{
    public:
        SomeCompilerSpecificName();
        template<typename T>
        auto operator() (T x) -> decltype((void)decltype(valueT(x))())) const
        {
        }
};
```

- 第2个模板参数类型为`TypeT<int>&&`或者`TypeT<int&>&&`
- 第3个模板参数是`decltype`的推导结果，推导的对象为`std::declval<F>()(std::declval<Args&&>()...)`。`std::declval<F>()`会实例化闭包类型，但是并不会实例化函数调用运算符成员模板。`std::declval<Args&&>()...`将创建一个`TypeT<int>`或者`TypeT<int&>`类型的**临时对象**，并用该临时对象作为参数调用闭包类型，这时才会实例化闭包类型中的函数调用运算符成员模板，该模板参数类型为`TypeT<int>`或者`TypeT<int&>`。在推导返回类型的过程中，`decltype(valueT(x))`的结果为`int`或者`int&`，当尝试默认构造引用类型时就会失败，编译器转而寻求另一个版本的`isValidImpl<>`，这将使得`isDefaultConstructible`返回`std::false_type`类型的临时对象。

### 19.4.4 编译基于SFINAE规则的类型特征模板（SFINAE-Friendly Traits）

[19.3.4](#Result-Type-Traits)节中的traits/plus2.hpp解决了`T1`和`T2`需要默认构造才能相加求和的问题，但是还存在`+`未定义的问题，例如对于如下代码：

```cpp
template<typename T>
class Array {
    // ...
};

// declare + for arrays of different element types:
template<typename T1, typename T2>
Array<typename PlusResultT<T1, T2>::Type> operator+ (Array<T1> const&, Array<T2> const&);

class A {
};
class B {
};

void addAB(Array<A> arrayA, Array<B> arrayB) {
    auto sum = arrayA + arrayB; // ERROR: fails in instantiation of PlusResultT<A, B>
    // ...
}
```

在推导返回类型的过程中，如果`A`和`B`不能相加，则编译器会因为实例化`PlusResultT<T1, T2>::Type`失败报错，因为此时是在实例化类成员的过程中，而不是在代换的过程中，所以SFINAE规则并不适用。原文：

>The practical problem is not that this failure occurs with code that is clearly ill-formed like this (there is no way to add an array of A to an array of B) but that it occurs during template argument deduction for operator+, deep in the instantiation of PlusResultT<A,B>.

简单的解决办法是再实现一个重载版本：

```cpp
// declare generic + for arrays of different element types:
template<typename T1, typename T2>
Array<typename PlusResultT<T1, T2>::Type> operator+ (Array<T1> const&, Array<T2> const&);

// overload + for concrete types:
Array<A> operator+(Array<A> const& arrayA, Array<B> const& arrayB);

void addAB(Array<A> const& arrayA, Array<B> const& arrayB) {
    auto sum = arrayA + arrayB; // ERROR?: depends on whether the compiler instantiates PlusResultT<A,B>
    // ...
}
```

但这还是可能报错，原因是虽然重载版本更为匹配，但是标准并没有规定此时是否需要实例化模板，如果实例化了，就有可能报错。原文：

>This has a remarkable consequence: It means that the program may fail to compile even if we add a specific overload to adding A and B arrays, because C++ does not specify whether the types in a function template are actually instantiated if another overload would be better.

为了解决这个问题，需要把所有可能的失败挪到代换相关上下文中，而不是在函数体或者类定义中：

```cpp
// traits/hasplus.hpp
#include <utility>      // for declval
#include <type_traits>  // for true_type, false_type, and void_t

// primary template:
template<typename, typename, typename = std::void_t<>>
struct HasPlusT : std::false_type
{
};

// partial specialization (may be SFINAE’d away):
template<typename T1, typename T2>
struct HasPlusT<T1, T2, std::void_t<decltype(std::declval<T1>() + std::declval<T2>())>> : std::true_type
{
};
```

```cpp
// traits/plus3.hpp
#include "hasplus.hpp"

template<typename T1, typename T2, bool = HasPlusT<T1, T2>::value>
struct PlusResultT { // primary template, used when HasPlusT yields true
    using Type = decltype(std::declval<T1>() + std::declval<T2>());
};

template<typename T1, typename T2>
struct PlusResultT<T1, T2, false> { // partial specialization, used otherwise
};
```

编译器推导返回类型时，会先实例化`PlusResultT<T1, T2>`，如果第三个参数推导结果为`true`，则实例化主模板，代换顺利结束；否则实例化特化版本，由于特化版本没有`Type`成员，则函数模板的实例化以SFINAE规则结束。

## 19.5 IsConvertibleT<>

基于SFINAE规则的类型特征模板可以用来判断是否可以进行类型转换：

```cpp
// traits/isconvertible.hpp
#include <type_traits>  // for true_type and false_type
#include <utility>      // for declval

template<typename FROM, typename TO>
struct IsConvertibleHelper {
    private:
        // test() trying to call the helper aux(TO) for a FROM passed as F:
        static void aux(TO);
        template<typename F, typename T, typename = decltype(aux(std::declval<F>()))>
            static std::true_type test(void*);
        // test() fallback:
        template<typename, typename>
            static std::false_type test(...);
    public:
        using Type = decltype(test<FROM, TO>(nullptr));
};

template<typename FROM, typename TO>
struct IsConvertibleT : IsConvertibleHelper<FROM, TO>::Type {
};

template<typename FROM, typename TO>
using IsConvertible = typename IsConvertibleT<FROM, TO>::Type;

template<typename FROM, typename TO>
constexpr bool isConvertible = IsConvertibleT<FROM, TO>::value;
```

使用到的方法和[19.4.1](#SFINAE-Out-Function-Overloads)节中的类似。书中的代码有错误，调用`test()`时应该提供两个模板实参，这里是修改后的正确版本。

一般来说是不能转换为数组类型和指针类型的，所以还需要如下特化：

```cpp
template<typename FROM, typename TO, bool = IsVoidT<TO>::value
                                            || IsArrayT<TO>::value
                                            || IsFunctionT<TO>::value>
struct IsConvertibleHelper {
    using Type = std::integral_constant<bool, IsVoidT<TO>::value && IsVoidT<FROM>::value>;
};

template<typename FROM, typename TO>
struct IsConvertibleHelper<FROM,TO,false> {
    // ... // previous implementation of IsConvertibleHelper here
};
```

## 19.6 侦测类成员

基于SFINAE规则的模板还可以用来侦测某个类中是否定义了某个成员。

### 19.6.1 侦测类型成员

使用[19.4.2](#SFINAE-Out-Partial-Specializations)节中偏特化的方法可以侦测类中是否定义了`size_type`类型：

```cpp
// traits/hassizetype.hpp
#include <type_traits> // defines true_type and false_type

// helper to ignore any number of template parameters:
template<typename...> using VoidT = void;

// primary template:
template<typename, typename = VoidT<>>
struct HasSizeTypeT : std::false_type
{
};

// partial specialization (may be SFINAE’d away):
template<typename T>
struct HasSizeTypeT<T, VoidT<typename T::size_type>> : std::true_type
{
};
```

### 19.6.2 侦测任意类型成员 {#Detecting-Arbitrary-Member-Types}

为每个类型成员都定义上述的模板过于麻烦了，可以使用宏来简化：

```cpp
// traits/hastype.hpp
#include <type_traits> // for true_type, false_type, and void_t
#define DEFINE_HAS_TYPE(MemType)                                    \
    template<typename, typename = std::void_t<>>                    \
    struct HasTypeT_##MemType                                       \
        : std::false_type { };                                      \
    template<typename T>                                            \
    struct HasTypeT_##MemType<T, std::void_t<typename T::MemType>>  \
        : std::true_type { } // ; intentionally skipped
```

```cpp
// traits/hastype.cpp
#include "hastype.hpp"

#include <iostream>
#include <vector>

DEFINE_HAS_TYPE(value_type);
DEFINE_HAS_TYPE(char_type);

int main()
{
    std::cout << "int::value_type: "
                << HasTypeT_value_type<int>::value << '\n';
    std::cout << "std::vector<int>::value_type: "
                << HasTypeT_value_type<std::vector<int>>::value << '\n';
    std::cout << "std::iostream::value_type: "
                << HasTypeT_value_type<std::iostream>::value << '\n';
    std::cout << "std::iostream::char_type: "
                << HasTypeT_char_type<std::iostream>::value << '\n';
}
```

### 19.6.3 侦测非类型成员 {#Detecting-Nontype-Members}

用同样的方法也可以侦测类中的数据成员和成员函数：

```cpp
// traits/hasmember.hpp
#include <type_traits> // for true_type, false_type, and void_t

#define DEFINE_HAS_MEMBER(Member)                                       \
    template<typename, typename = std::void_t<>>                        \
    struct HasMemberT_##Member                                          \
        : std::false_type { };                                          \
    template<typename T>                                                \
    struct HasMemberT_##Member<T, std::void_t<decltype(&T::Member)>>    \
        : std::true_type { } // ; intentionally skipped
```

```cpp
// traits/hasmember.cpp
#include "hasmember.hpp"
#include <iostream>
#include <vector>
#include <utility>

DEFINE_HAS_MEMBER(size);
DEFINE_HAS_MEMBER(first);

int main()
{
    std::cout << "int::size: "
                << HasMemberT_size<int>::value << '\n';
    std::cout << "std::vector<int>::size: "
                << HasMemberT_size<std::vector<int>>::value << '\n';
    std::cout << "std::pair<int,int>::first: "
                << HasMemberT_first<std::pair<int,int>>::value << '\n';
}
```

上面的代码无法处理重载函数（可能是因为区分确定两个重载函数的地址），但是可以通过调用的方式来侦测：

```cpp
// traits/hasbegin.hpp
#include <utility>      // for declval
#include <type_traits>  // for true_type, false_type, and void_t

// primary template:
template<typename, typename = std::void_t<>>
struct HasBeginT : std::false_type {
};

// partial specialization (may be SFINAE’d away):
template<typename T>
struct HasBeginT<T, std::void_t<decltype(std::declval<T>().begin())>>
    : std::true_type {
};
```

同样的技巧也可以用于检测表达式：

```cpp
// traits/hasless.hpp
#include <utility>      // for declval
#include <type_traits>  // for true_type, false_type, and void_t

// primary template:
template<typename, typename, typename = std::void_t<>>
struct HasLessT : std::false_type
{
};

// partial specialization (may be SFINAE’d away):
template<typename T1, typename T2>
struct HasLessT<T1, T2, std::void_t<decltype(std::declval<T1>() < std::declval<T2>())>>
    : std::true_type
{
};
```

也可以用来检测多个条件：

```cpp
// traits/hasvarious.hpp
#include <utility>      // for declval
#include <type_traits>  // for true_type, false_type, and void_t

// primary template:
template<typename, typename = std::void_t<>>
struct HasVariousT : std::false_type
{
};

// partial specialization (may be SFINAE’d away):
template<typename T>
struct HasVariousT<T, std::void_t<decltype(std::declval<T>().begin()),
                                    typename T::difference_type,
                                    typename T::iterator>>
    : std::true_type
{
};
```

### 19.6.4 使用泛型lambda表达式侦测类型成员

和[19.6.2](#Detecting-Arbitrary-Member-Types)节中使用宏定义模板最相近的方法是使用[19.4.3](#Using-Generic-Lambdas-for-SFINAE)节中的`isValid()`：

```cpp
// traits/isvalid1.cpp
#include "isvalid.hpp"
#include <iostream>
#include <string>
#include <utility>

int main()
{
    using namespace std;
    cout << boolalpha;

    // define to check for data member first:
    constexpr auto hasFirst
        = isValid([](auto x) -> decltype((void)valueT(x).first) {
                    });
    cout << "hasFirst: " << hasFirst(type<pair<int,int>>) << '\n'; // true

    // define to check for member type size_type:
    constexpr auto hasSizeType
        = isValid([](auto x) -> typename decltype(valueT(x))::size_type {
                    });

    struct CX {
        using size_type = std::size_t;
    };
    cout << "hasSizeType: " << hasSizeType(type<CX>) << '\n'; // true

    if constexpr(!hasSizeType(type<int>)) {
        cout << "int has no size_type\n";
        // ...
    }

    // define to check for <:
    constexpr auto hasLess
        = isValid([](auto x, auto y) -> decltype(valueT(x) < valueT(y)) {
                    });

    cout << hasLess(42, type<char>) << '\n';                // yields false
    cout << hasLess(type<string>, type<string>) << '\n';    // yields true
    cout << hasLess(type<string>, type<int>) << '\n';       // yields false
    cout << hasLess(type<string>, "hello") << '\n';         // yields false
}
```

编译试了下，必须要添加编译选项`-std=c++17`，同时`hasLess(42, type<char>)`和`hasLess(type<string>, "hello")`结果为`false`，这是因为以`42`和`hello`作为参数调用`valueT()`时无法进行推导。

也可以通过`std::declval`来实现上述类型特征模板（原理上都是产生某种类型的临时对象）：

```cpp
// traits/isvalid2.cpp
#include "isvalid.hpp"
#include <iostream>
#include <string>
#include <utility>

constexpr auto hasFirst
    = isValid([](auto&& x) -> decltype((void)&x.first) {
                });
template<typename T>
using HasFirstT = decltype(hasFirst(std::declval<T>()));

constexpr auto hasSizeType
    = isValid([](auto&& x) -> typename std::decay_t<decltype(x)>::size_type {
                });
template<typename T>
using HasSizeTypeT = decltype(hasSizeType(std::declval<T>()));

constexpr auto hasLess
    = isValid([](auto&& x, auto&& y) -> decltype(x < y) {
                });

template<typename T1, typename T2>
using HasLessT = decltype(hasLess(std::declval<T1>(), std::declval<T2>()));

int main()
{
    using namespace std;
    cout << "first: " << HasFirstT<pair<int,int>>::value << '\n'; // true

    struct CX {
        using size_type = std::size_t;
    };

    cout << "size_type: " << HasSizeTypeT<CX>::value << '\n';   // true
    cout << "size_type: " << HasSizeTypeT<int>::value << '\n';  // false
    cout << HasLessT<int, char>::value << '\n';                 // true
    cout << HasLessT<string, string>::value << '\n';            // true
    cout << HasLessT<string, int>::value << '\n';               // false
    cout << HasLessT<string, char*>::value << '\n';             // true
}
```

## 19.7 其它类型特征模板

### 19.7.1 分支型模板

根据某种条件实例化主模板或者特化的模板称为分支模板，可以统一写为下面的形式：

```cpp
// traits/ifthenelse.hpp
#ifndef IFTHENELSE_HPP
#define IFTHENELSE_HPP

// primary template: yield the second argument by default and rely on
// a partial specialization to yield the third argument
// if COND is false
template<bool COND, typename TrueType, typename FalseType>
struct IfThenElseT {
    using Type = TrueType;
};

// partial specialization: false yields third argument
template<typename TrueType, typename FalseType>
struct IfThenElseT<false, TrueType, FalseType> {
    using Type = FalseType;
};

template<bool COND, typename TrueType, typename FalseType>
using IfThenElse = typename IfThenElseT<COND, TrueType, FalseType>::Type;
#endif // IFTHENELSE_HPP
```

下面是一个利用分支模板选择容纳`N`的最小整型的例子：

```cpp
// traits/smallestint.hpp
#include <limits>
#include "ifthenelse.hpp"

template<auto N>
struct SmallestIntT {
    using Type =
        typename IfThenElseT<N <= std::numeric_limits<char>::max(), char,
            typename IfThenElseT<N <= std::numeric_limits<short>::max(), short,
                typename IfThenElseT<N <= std::numeric_limits<int>::max(), int,
                    typename IfThenElseT<N <= std::numeric_limits<long>::max(), long,
                        typename IfThenElseT<N <= std::numeric_limits<long long>::max(),
                                                long long, // then
                                                void // fallback
                                            >::Type
                                        >::Type
                                    >::Type
                                >::Type
                            >::Type;
};
```

如果想要获取某种整型的无符号类型，可以将模板写为下面的形式：

```cpp
// yield T when using member Type:
template<typename T>
struct IdentityT {
    using Type = T;
};

// to make unsigned after IfThenElse was evaluated:
template<typename T>
struct MakeUnsignedT {
    using Type = typename std::make_unsigned<T>::type;
};

template<typename T>
struct UnsignedT {
    using Type = typename IfThenElse<std::is_integral<T>::value && !std::is_same<T,bool>::value,
                                        MakeUnsignedT<T>,
                                        IdentityT<T>
                                    >::Type;
};
```

### 19.7.2 侦测不抛出异常的成员

如果要判断某操作是否会抛出异常，例如移动构造函数是否会抛出异常，可以使用下面的模板：

```cpp
// traits/isnothrowmoveconstructible1.hpp
#include <utility>      // for declval
#include <type_traits>  // for bool_constant

template<typename T>
struct IsNothrowMoveConstructibleT
    : std::bool_constant<noexcept(T(std::declval<T>()))>
{
};
```

如果`T`类型没有移动构造函数（或者拷贝构造函数），则表达式`T(std::declval<T>())`是非法的，也就无法进行编译。借鉴[19.6.3](#Detecting-Nontype-Members)中的方法，可以在侦测移动是否抛出异常之前侦测是否可以移动：

```cpp
// traits/isnothrowmoveconstructible2.hpp
#include <utility>      // for declval
#include <type_traits>  // for true_type, false_type, and bool_constant<>

// primary template:
template<typename T, typename = std::void_t<>>
struct IsNothrowMoveConstructibleT : std::false_type
{
};

// partial specialization (may be SFINAE’d away):
template<typename T>
struct IsNothrowMoveConstructibleT<T, std::void_t<decltype(T(std::declval<T>()))>>
    : std::bool_constant<noexcept(T(std::declval<T>()))>
{
};
```

### 19.7.3 类型特征模板的缺点

类型特征模板起作用的原因是定义了类型成员`Type`，所以不得不显示的写出`::Type`来获取类型特征，为了简便可以使用别名模板来代替：

```cpp
template<typename T>
using RemoveCV = typename RemoveCVT<T>::Type;

template<typename T>
using RemoveReference = typename RemoveReferenceT<T>::Type;

template<typename T1, typename T2>
using PlusResult = typename PlusResultT<T1, T2>::Type;
```

对于值特征，也需要显示写出`::value`，为了简便可以使用变量模板来代替：

```cpp
template<typename T1, typename T2>
constexpr bool IsSame = IsSameT<T1,T2>::value;

template<typename FROM, typename TO>
constexpr bool IsConvertible = IsConvertibleT<FROM, TO>::value;
```

## 19.8 类型分类

类型特征模板也可以用来判断类型是内置类型、指针类型还是类类型。

### 19.8.1 判断内置类型

内置类型数量较少，可以通过穷举的方式提供所有的特化版本：

```cpp
// traits/isfunda.hpp
#include <cstddef>      // for nullptr_t
#include <type_traits>  // for true_type, false_type, and bool_constant<>

// primary template: in general T is not a fundamental type
template<typename T>
struct IsFundaT : std::false_type {
};

// macro to specialize for fundamental types
#define MK_FUNDA_TYPE(T)                               \
    template<> struct IsFundaT<T> : std::true_type {    \
    };

MK_FUNDA_TYPE(void)
MK_FUNDA_TYPE(bool)
MK_FUNDA_TYPE(char)
MK_FUNDA_TYPE(signed char)
MK_FUNDA_TYPE(unsigned char)
MK_FUNDA_TYPE(wchar_t)
MK_FUNDA_TYPE(char16_t)
MK_FUNDA_TYPE(char32_t)

MK_FUNDA_TYPE(signed short)
MK_FUNDA_TYPE(unsigned short)
MK_FUNDA_TYPE(signed int)
MK_FUNDA_TYPE(unsigned int)
MK_FUNDA_TYPE(signed long)
MK_FUNDA_TYPE(unsigned long)
MK_FUNDA_TYPE(signed long long)
MK_FUNDA_TYPE(unsigned long long)

MK_FUNDA_TYPE(float)
MK_FUNDA_TYPE(double)
MK_FUNDA_TYPE(long double)

MK_FUNDA_TYPE(std::nullptr_t)

#undef MK_FUNDA_TYPE
```

```cpp
// traits/isfundatest.cpp
#include "isfunda.hpp"
#include <iostream>

template<typename T>
void test (T const&)
{
    if (IsFundaT<T>::value) {
        std::cout << "T is a fundamental type" << '\n';
    }
    else {
        std::cout << "T is not a fundamental type" << '\n';
    }
}
int main()
{
    test(7);
    test("hello");
}
```

### 19.8.2 判断复合类型

这里的复合类型是指指针类型、引用类型、成员指针类型和数组类型：

#### 指针类型

```cpp
// traits/ispointer.hpp
template<typename T>
struct IsPointerT : std::false_type {       // primary template: by default not a pointer
};

template<typename T>
struct IsPointerT<T*> : std::true_type {    // partial specialization for pointers
    using BaseT = T; // type pointing to
};
```

#### 引用类型

```cpp
// traits/islvaluereference.hpp
template<typename T>
struct IsLValueReferenceT : std::false_type {       // by default no lvalue reference
};

template<typename T>
struct IsLValueReferenceT<T&> : std::true_type {    // unless T is lvalue references
    using BaseT = T; // type referring to
};
```

```cpp
// traits/isrvaluereference.hpp
template<typename T>
struct IsRValueReferenceT : std::false_type {       // by default no rvalue reference
};

template<typename T>
struct IsRValueReferenceT<T&&> : std::true_type {   // unless T is rvalue reference
    using BaseT = T; // type referring to
};
```

```cpp
// traits/isreference.hpp
#include "islvaluereference.hpp"
#include "isrvaluereference.hpp"
#include "ifthenelse.hpp"

template<typename T>
class IsReferenceT
    : public IfThenElseT<IsLValueReferenceT<T>::value,
                            IsLValueReferenceT<T>,
                            IsRValueReferenceT<T>
                            >::Type {
};
```

#### 数组类型

```cpp
// traits/isarray.hpp
#include <cstddef>

template<typename T>
struct IsArrayT : std::false_type {         // primary template: not an array
};

template<typename T, std::size_t N>
struct IsArrayT<T[N]> : std::true_type {    // partial specialization for arrays
    using BaseT = T;
    static constexpr std::size_t size = N;
};

template<typename T>
struct IsArrayT<T[]> : std::true_type {     // partial specialization for unbound arrays
    using BaseT = T;
    static constexpr std::size_t size = 0;
};
```

#### 成员指针类型

```cpp
// traits/ispointertomember.hpp
template<typename T>
struct IsPointerToMemberT : std::false_type {           // by default no pointer-to-member
};

template<typename T, typename C>
struct IsPointerToMemberT<T C::*> : std::true_type {    // partial specialization
    using MemberT = T;
    using ClassT = C;
};
```

### 19.8.3 识别函数类型

函数可能包含任意数量的参数，为此需要为类型特征模板提供一个参数包：

```cpp
// traits/isfunction.hpp
#include "../typelist/typelist.hpp"

template<typename T>
struct IsFunctionT : std::false_type {                      // primary template: no function
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params...)> : std::true_type {        // functions
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = false;
};

template<typename R, typename... Params>
struct IsFunctionT<R (Params..., ...)> : std::true_type {   // variadic functions
    using Type = R;
    using ParamsT = Typelist<Params...>;
    static constexpr bool variadic = true;
};
```

如果要处理`const`、`volatile`和引用等修饰符，还需要很多特化版本，具体见书。

### 19.8.4 判断类类型

类中一般都包含成员，所以可以通过成员指针来判断类类型：

```cpp
// traits/isclass.hpp
#include <type_traits>

template<typename T, typename = std::void_t<>>
struct IsClassT : std::false_type {         // primary template: by default no class
};

template<typename T>
struct IsClassT<T, std::void_t<int T::*>>   // classes can have pointer-to-member
    : std::true_type {
};
```

### 19.8.5 判断枚举类型 {#Determining-Enumeration-Types}

书中的是“非其它”的实现方式：

```cpp
// traits/isenum.hpp
template<typename T>
struct IsEnumT {
    static constexpr bool value = !IsFundaT<T>::value &&
                                    !IsPointerT<T>::value &&
                                    !IsReferenceT<T>::value &&
                                    !IsArrayT<T>::value &&
                                    !IsPointerToMemberT<T>::value &&
                                    !IsFunctionT<T>::value &&
                                    !IsClassT<T>::value;
};
```

## 19.9 提取策略的类型特征模板

### 19.9.1 传递只读类型的参数

向函数传递只读类型的参数有两种方法，分别为传值和传常量引用，可以通过下面类型特征模板简单判断哪种方式开销更小：

```cpp
// traits/rparam.hpp
#ifndef RPARAM_HPP
#define RPARAM_HPP

#include "ifthenelse.hpp"
#include <type_traits>
template<typename T>
struct RParam {
    using Type
        = IfThenElse<(sizeof(T) <= 2*sizeof(void*)
                            && std::is_trivially_copy_constructible<T>::value
                            && std::is_trivially_move_constructible<T>::value),
                        T,
                        T const&>;
};

#endif // RPARAM_HPP
```

使用方式如下：

```cpp
// traits/rparamcls.hpp
#include "rparam.hpp"
#include <iostream>

class MyClass1 {
    public:
        MyClass1 () {
        }
        MyClass1 (MyClass1 const&) {
            std::cout << "MyClass1 copy constructor called\n";
        }
};

class MyClass2 {
    public:
        MyClass2 () {
        }
        MyClass2 (MyClass2 const&) {
            std::cout << "MyClass2 copy constructor called\n";
        }
};

// pass MyClass2 objects with RParam<> by value
template<>
class RParam<MyClass2> {
    public:
        using Type = MyClass2;
};
```

```cpp
// traits/rparam1.cpp
#include "rparam.hpp"
#include "rparamcls.hpp"

// function that allows parameter passing by value or by reference
template<typename T1, typename T2>
void foo (typename RParam<T1>::Type p1, typename RParam<T2>::Type p2)
{
    // ...
}
int main()
{
    MyClass1 mc1;
    MyClass2 mc2;
    foo<MyClass1,MyClass2>(mc1,mc2);
}
```

但是不便之处在于编译器无法进行类型推导了，这是因为`RParam<T1>::Type`不是可推导的上下文，但是可以通过包装器（wrapper）包装一下：

```cpp
// traits/rparam2.cpp
#include "rparam.hpp"
#include "rparamcls.hpp"

// function that allows parameter passing by value or by reference
template<typename T1, typename T2>
void foo_core (typename RParam<T1>::Type p1, typename RParam<T2>::Type p2)
{
    // ...
}

// wrapper to avoid explicit template parameter passing
template<typename T1, typename T2>
void foo (T1 && p1, T2 && p2)
{
    foo_core<T1,T2>(std::forward<T1>(p1),std::forward<T2>(p2));
}

int main()
{
    MyClass1 mc1;
    MyClass2 mc2;
    foo(mc1,mc2); // same as foo_core<MyClass1,MyClass2>(mc1,mc2)
}
```

## 19.10 标准库中的类型特征模板

[19.8.5](#Determining-Enumeration-Types)节中的判断枚举类型的模板不太容易直接实现，类似的还有标准库中的`std::is_union`模板，为此编译器提供了一些支持。

## 19.11 后记

附录D中介绍了标准库中类型特征模板。
