---
layout: post
title: 《C++ Templates》第20章 基于类型属性的重载
categories: Reading
tags: C++
---

# 20 基于类型属性的重载

## 20.1 算法特化

算法特化（algorithm specialization）是为某种特定类型设计的算法，通常比泛型算法性能高。在定义了特化的算法后，一般只需借助重载解析规则便可以自动调用特化的算法，例如：

```cpp
template<typename T>
void swap(T& x, T& y)
{
    T tmp(x);
    x = y;
    y = tmp;
}

template<typename T>
void swap(Array<T>& x, Array<T>& y)
{
    swap(x.ptr, y.ptr);
    swap(x.len, y.len);
}
```

但也不是所有的特化算法都可以简单的转换为函数模板，例如下面的迭代器前进的例子：

```cpp
template<typename InputIterator, typename Distance>
void advanceIter(InputIterator& x, Distance n)
{
    while (n > 0) { // linear time
        ++x;
        --n;
    }
}

template<typename RandomAccessIterator, typename Distance>
void advanceIter(RandomAccessIterator& x, Distance n) {
    x += n; // constant time
}
```

两个`advanceIter()`模板只有模板参数名字不同，编译器将无法解析。

## 20.2 通过标签实现函数模板的分发 {#Tag-Dispatching}

引入迭代器标签后就可以正确解析了：

```cpp
template<typename Iterator, typename Distance>
void advanceIterImpl(Iterator& x, Distance n, std::input_iterator_tag)
{
    while (n > 0) { // linear time
        ++x;
        --n;
    }
}

template<typename Iterator, typename Distance>
void advanceIterImpl(Iterator& x, Distance n, std::random_access_iterator_tag) {
    x += n; // constant time
}

template<typename Iterator, typename Distance>
void advanceIter(Iterator& x, Distance n)
{
    advanceIterImpl(x, n, typename std::iterator_traits<Iterator>::iterator_category());
}
```

标准库中迭代器的分类如下：

```cpp
namespace std {
    struct input_iterator_tag { };
    struct output_iterator_tag { };
    struct forward_iterator_tag : public input_iterator_tag { };
    struct bidirectional_iterator_tag : public forward_iterator_tag { };
    struct random_access_iterator_tag : public bidirectional_iterator_tag { };
}
```

## 20.3 启用和禁用函数模板

通过`EnableIf<>`可以实现启用和禁用函数模板，原理依然是SFINAE规则：

```cpp
// typeoverload/enableif.hpp
template<bool, typename T = void>
struct EnableIfT {
};

template<typename T>
struct EnableIfT<true, T> {
    using Type = T;
};

template<bool Cond, typename T = void>
using EnableIf = typename EnableIfT<Cond, T>::Type;
```

```cpp
template<typename Iterator>
constexpr bool IsRandomAccessIterator =
    IsConvertible<typename std::iterator_traits<Iterator>::iterator_category,
                    std::random_access_iterator_tag>;

template<typename Iterator, typename Distance>
EnableIf<IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x, Distance n) {
    x += n; // constant time
}

template<typename Iterator, typename Distance>
EnableIf<!IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x, Distance n)
{
    while (n > 0) { // linear time
        ++x;
        --n;
    }
}
```

注意同时也要禁用其它重载函数，否则会导致模糊调用。

### 20.3.1 多条件下的算法特化

有些迭代器还可以后退，所以现在的需求变成了：

- 对于随机访问迭代器，可以在常量时间内前进和后退
- 对于双向迭代器，可以在线性时间内前进和后退
- 对于输入迭代器，只能在线性时间内前进

```cpp
// typeoverload/advance2.hpp
#include <iterator>

// implementation for random access iterators:
template<typename Iterator, typename Distance>
EnableIf<IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x, Distance n) {
    x += n; // constant time
}

template<typename Iterator>
constexpr bool IsBidirectionalIterator =
    IsConvertible<typename std::iterator_traits<Iterator>::iterator_category,
                    std::bidirectional_iterator_tag>;

// implementation for bidirectional iterators:
template<typename Iterator, typename Distance>
EnableIf<IsBidirectionalIterator<Iterator> && !IsRandomAccessIterator<Iterator>>
advanceIter(Iterator& x, Distance n) {
    if (n > 0) {
        for ( ; n > 0; ++x, --n) {  // linear time
        }
    } else {
        for ( ; n < 0; --x, ++n) {  // linear time
        }
    }
}

// implementation for all other iterators:
template<typename Iterator, typename Distance>
EnableIf<!IsBidirectionalIterator<Iterator>>
advanceIter(Iterator& x, Distance n) {
    if (n < 0) {
        throw "advanceIter(): invalid iterator category for negative n";
    }
    while (n > 0) { // linear time
        ++x;
        --n;
    }
}
```

### 20.3.2 将EnableIf作为默认模板参数 {#Where-Does-the-EnableIf-Go}

`EnableIf<>`一般写在函数模板的返回类型处，对于没有返回类型的函数，`EnableIf<>`可以作为默认的模板实参：

```cpp
// typeoverload/container1.hpp
#include <iterator>
#include "enableif.hpp"
#include "isconvertible.hpp"

template<typename Iterator>
constexpr bool IsInputIterator =
    IsConvertible<
                    typename std::iterator_traits<Iterator>::iterator_category,
                    std::input_iterator_tag>;

template<typename T>
class Container {
    public:
        // construct from an input iterator sequence:
        template<typename Iterator,
                    typename = EnableIf<IsInputIterator<Iterator>>>
        Container(Iterator first, Iterator last);

        // convert to a container so long as the value types are convertible:
        template<typename U, typename = EnableIf<IsConvertible<T, U>>>
        operator Container<U>() const;
};
```

### 20.3.3 编译时if

借助C++17的编译时`if`可以避免使用`EnableIf<>`：

```cpp
// typeoverload/advance3.hpp
template<typename Iterator, typename Distance>
void advanceIter(Iterator& x, Distance n) {
    if constexpr(IsRandomAccessIterator<Iterator>) {
        // implementation for random access iterators:
        x += n;                         // constant time
    }
    else if constexpr(IsBidirectionalIterator<Iterator>) {
        // implementation for bidirectional iterators:
        if (n > 0) {
            for ( ; n > 0; ++x, --n) {  // linear time for positive n
            }
        } else {
            for ( ; n < 0; --x, ++n) {  // linear time for negative n
            }
        }
    }
    else {
        // implementation for all other iterators that are at least input iterators:
        if (n < 0) {
            throw "advanceIter(): invalid iterator category for negative n";
        }
        while (n > 0) {                 // linear time for positive n only
            ++x;
            --n;
        }
    }
}
```

### 20.3.4 Concepts

使用concept可以最简单的实现启用和禁用模板，但可惜的是还没有支持：

```cpp
// typeoverload/container4.hpp
template<typename T>
class Container {
    public:
        // construct from an input iterator sequence:
        template<typename Iterator>
        requires IsInputIterator<Iterator>
        Container(Iterator first, Iterator last);

        // construct from a random access iterator sequence:
        template<typename Iterator>
        requires IsRandomAccessIterator<Iterator>
        Container(Iterator first, Iterator last);

        // convert to a container so long as the value types are convertible:
        template<typename U>
        requires IsConvertible<T, U>
        operator Container<U>() const;
};
```

## 20.4 基于类型属性的类模板特化

### 20.4.1 启用和禁用特化的类模板

和[20.3.2](#Where-Does-the-EnableIf-Go)一样，可以通过引入默认模板参数来匹配特化的类模板，本节中以实现一个字典为例：

```cpp
template<typename Key, typename Value, typename = void>
class Dictionary
{
    private:
        vector<pair<Key const, Value>> data;
    public:
        // subscripted access to the data:
        value& operator[](Key const& key)
        {
            // search for the element with this key:
            for (auto& element : data) {
                if (element.first == key) {
                    return element.second;
                }
            }

            // there is no element with this key; add one
            data.push_back(pair<Key const, Value>(key, Value()));
            return data.back().second;
        }
        // ...
};

template<typename Key, typename Value>
class Dictionary<Key, Value, EnableIf<HasLess<Key> && !HasHash<Key>>>
{
    private:
        map<Key, Value> data;
    public:
        value& operator[](Key const& key) {
            return data[key];
        }
        // ...
};

template<typename Key, typename Value>
class Dictionary<Key, Value, EnableIf<HasHash<Key>>>
{
    private:
        unordered_map<Key, Value> data;
    public:
        value& operator[](Key const& key) {
            return data[key];
        }
        // ...
};
```

### 20.4.2 通过标签匹配类模板

`advanceIter()`也可以借助类模板实现：

```cpp
// primary template (intentionally undefined):
template<typename Iterator,
            typename Tag =
                BestMatchInSet<typename std::iterator_traits<Iterator>::iterator_category,
                                std::input_iterator_tag,
                                std::bidirectional_iterator_tag,
                                std::random_access_iterator_tag>>
class Advance;

// general, linear-time implementation for input iterators:
template<typename Iterator>
class Advance<Iterator, std::input_iterator_tag>
{
    public:
        using DifferenceType = typename std::iterator_traits<Iterator>::difference_type;
        void operator() (Iterator& x, DifferenceType n) const
        {
            while (n > 0) {
                ++x;
                --n;
            }
        }
};

// bidirectional, linear-time algorithm for bidirectional iterators:
template<typename Iterator>
class Advance<Iterator, std::bidirectional_iterator_tag>
{
    public:
        using DifferenceType = typename std::iterator_traits<Iterator>::difference_type;
        void operator() (Iterator& x, DifferenceType n) const
        {
            if (n > 0) {
                while (n > 0) {
                    ++x;
                    --n;
                }
            } else {
                while (n < 0) {
                    --x;
                    ++n;
                }
            }
        }
};

// bidirectional, constant-time algorithm for random access iterators:
template<typename Iterator>
class Advance<Iterator, std::random_access_iterator_tag>
{
    public:
        using DifferenceType = typename std::iterator_traits<Iterator>::difference_type;
        void operator() (Iterator& x, DifferenceType n) const
        {
            x += n;
        }
}
```

`BestMatchInSet<>`的功能为找到最匹配的类型，这可以借助函数的重载机制：

```cpp
// construct a set of match() overloads for the types in Types...:
template<typename... Types>
struct MatchOverloads;

// basis case: nothing matched:
template<>
struct MatchOverloads<> {
    static void match(...);
};

// recursive case: introduce a new match() overload:
template<typename T1, typename... Rest>
struct MatchOverloads<T1, Rest...> : public MatchOverloads<Rest...> {
    static T1 match(T1);                    // introduce overload for T1
    using MatchOverloads<Rest...>::match;   // collect overloads from bases
};

// find the best match for T in Types...:
template<typename T, typename... Types>
struct BestMatchInSetT {
    using Type = decltype(MatchOverloads<Types...>::match(declval<T>()));
};

template<typename T, typename... Types>
using BestMatchInSet = typename BestMatchInSetT<T, Types...>::Type;
```

`MatchOverloads<>`通过递归的方式重载了多个`match()`，每个函数接受不同类型的参数。当在`decltype`对`match()`调用进行推导的过程中，就可以实现最佳匹配。个人认为实例化的过程应该为：

1. 实例化`BestMatchInSet<>`触发实例化`BestMatchInSetT<>`
2. 实例化`BestMatchInSetT<>`时会检查`Type`的定义，这会触发`decltype`中的推导，从而触发实例化`MatchOverloads<Types...>`
3. 实例化`MatchOverloads<T1, Rest...>`会递归实例化基类，然后在每个基类中定义`match()`，并在派生类中拉取基类的`match()`
4. 回到第2步，`decltype`中的推导会根据重载解析规则选择最匹配的`match()`，最终推导出`BestMatchInSet<>::Type`的类型
5. 回到第1步，引用`BestMatchInSetT<T, Types...>::Type`得到最终结果

书中关于这一部分的解释：

>The MatchOverloads template uses recursive inheritance to declare a match() function with each type in the input set of Types.

>Each instantiation of the recursive MatchOverloads partial specialization introduces a new match() function for the next type in the list. It then employs a using declaration to pull in the match() function(s) defined in its base class, which handles the remaining types in the list. When applied recursively, the result is a complete set of match() overloads corresponding to the given types, each of which returns its parameter type. 

>The BestMatchInSetT template then passes a T object to this set of overloaded match() functions and produces the return type of the selected (best) match() function. If none of the functions matches, the void returning basis case (which uses an ellipsis to capture any argument) indicates failure.

>To summarize, BestMatchInSetT translates a function-overloading result into a trait and makes it relatively easy to use tag dispatching to select among class template partial specializations.

## 20.5 可安全实例化的模板

如果将每一个模板参数都使用`EnableIf`进行限制，则当模板代换失败时SFINAE规则会起作用，不会导致编译失败，这样的模板称为可安全实例化的模板（instantiation-safe templates）。下面的代码不是可安全实例化的，因为无法保证类型`T`支持`<`运算符，也无法保证结果可以转换为`bool`：

```cpp
template<typename T>
T const& min(T const& x, T const& y)
{
    if (y < x) {
        return y;
    }
    return x;
}
```

为了保证类型`T`支持`<`运算符，需要实现`LessResultT<>`：

```cpp
// typeoverload/lessresult.hpp
#include <utility>      // for declval()
#include <type_traits>  // for true_type and false_type

template<typename T1, typename T2>
class HasLess {
    template<typename T> struct Identity;
    template<typename U1, typename U2> static std::true_type
        test(Identity<decltype(std::declval<U1>() < std::declval<U2>())>*);
    template<typename U1, typename U2> static std::false_type
        test(...);
    public:
        static constexpr bool value = decltype(test<T1, T2>(nullptr))::value;
};

template<typename T1, typename T2, bool HasLess>
class LessResultImpl {
    public:
        using Type = decltype(std::declval<T1>() < std::declval<T2>());
};

template<typename T1, typename T2>
class LessResultImpl<T1, T2, false> {
};

template<typename T1, typename T2>
class LessResultT
    : public LessResultImpl<T1, T2, HasLess<T1, T2>::value> {
};

template<typename T1, typename T2>
using LessResult = typename LessResultT<T1, T2>::Type;
```

并改进`min()`的定义：

```cpp
// typeoverload/min2.hpp
#include "isconvertible.hpp"
#include "lessresult.hpp"

template<typename T>
EnableIf<IsConvertible<LessResult<T const&, T const&>, bool>, T const&>
min(T const& x, T const& y)
{
    if (y < x) {
        return y;
    }
    return x;
}
```

测试代码：

```cpp
// typeoverload/min.cpp
#include "min.hpp"

struct X1 { };
bool operator< (X1 const&, X1 const&) { return true; }

struct X2 { };
bool operator<(X2, X2) { return true; }

struct X3 { };
bool operator<(X3&, X3&) { return true; }

struct X4 { };

struct BoolConvertible {
    operator bool() const { return true; } // implicit conversion to bool
};
struct X5 { };
BoolConvertible operator< (X5 const&, X5 const&)
{
    return BoolConvertible();
}

struct NotBoolConvertible { // no conversion to bool
};
struct X6 { };
NotBoolConvertible operator< (X6 const&, X6 const&)
{
    return NotBoolConvertible();
}

struct BoolLike {
    explicit operator bool() const { return true; } // explicit conversion to bool
};
struct X7 { };
BoolLike operator< (X7 const&, X7 const&) { return BoolLike(); }

int main()
{
    min(X1(), X1());    // X1 can be passed to min()
    min(X2(), X2());    // X2 can be passed to min()
    min(X3(), X3());    // ERROR: X3 cannot be passed to min()
    min(X4(), X4());    // ERROR: X4 cannot be passed to min()
    min(X5(), X5());    // X5 can be passed to min()
    min(X6(), X6());    // ERROR: X6 cannot be passed to min()
    min(X7(), X7());    // UNEXPECTED ERROR: X7 cannot be passed to min()
}
```

`X3`报错的原因在于`<`运算符不能比较两个临时对象，`X4`报错的原因在于没有匹配的`<`运算符，`X6`报错的原因在于`NotBoolConvertible`不能隐式转换为`bool`类型。

最有意思的是`X7`。`explicit`修饰的类型转换只在条件表达式、逻辑运算符和条件运算符`? :`中可以作为隐式转换，称为特定语境下的类型转换（contextually converted），而`IsConvertible<>`是借助`decltype`中的函数调用判断是否可以进行转换的，所以`X7`中的类型转换不起作用，为此需要引入`IsContextualBoolT<>`：

```cpp
// typeoverload/iscontextualbool.hpp
#include <utility>      // for declval()
#include <type_traits>  // for true_type and false_type

template<typename T>
class IsContextualBoolT {
    private:
        template<typename T> struct Identity;
        template<typename U> static std::true_type
            test(Identity<decltype(declval<U>() ? 0 : 1)>*);
        template<typename U> static std::false_type
            test(...);
    public:
        static constexpr bool value = decltype(test<T>(nullptr))::value;
};
template<typename T>
constexpr bool IsContextualBool = IsContextualBoolT<T>::value;
```

```cpp
// typeoverload/min3.hpp
#include "iscontextualbool.hpp"
#include "lessresult.hpp"

template<typename T>
EnableIf<IsContextualBool<LessResult<T const&, T const&>>, T const&>
min(T const& x, T const& y)
{
    if (y < x) {
        return y;
    }
    return x;
}
```

## 20.6 标准库中基于类型属性的重载

除了[20.2](#Tag-Dispatching)中已经提到的迭代器分类外，标准库中基于类型属性的重载还包括：

- 以某些类型调用`std::copy`时最终会调用`std::memcpy`或者`std::memmove`
- 以某些类型调用`std::fill`时最终会调用`std::memset`
- 避免调用平凡的析构函数

《STL源码剖析》一书中介绍了大量有关这部分的内容。

## 20.7 后记
