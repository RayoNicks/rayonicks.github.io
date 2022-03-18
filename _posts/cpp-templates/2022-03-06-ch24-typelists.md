---
layout: post
title: 《C++ Templates》第24章 类型列表
categories: Reading
tags: C++
---

# 24 类型列表

## 24.1 类型列表

类型列表就是存储类型的列表，和`std::list`类似也可以进行添加和删除等操作，但是区别在于添加操作不会对原始列表进行修改，而是创建一个新的列表，最常见的方式是通过模板实现：

```cpp
// typelist/typelist.hpp
template<typename... Elements>
class Typelist
{
};
```

获取类型列表中第一个类型：

```cpp
// typelist/typelistfront.hpp
template<typename List>
class FrontT;

template<typename Head, typename... Tail>
class FrontT<Typelist<Head, Tail...>>
{
    public:
        using Type = Head;
};

template<typename List>
using Front = typename FrontT<List>::Type;
```

移除类型列表中第一个类型：

```cpp
// typelist/typelistpopfront.hpp
template<typename List>
class PopFrontT;

template<typename Head, typename... Tail>
class PopFrontT<Typelist<Head, Tail...>> {
    public:
        using Type = Typelist<Tail...>;
};

template<typename List>
using PopFront = typename PopFrontT<List>::Type;
```

在类型列表首部添加一个新的类型：

```cpp
// typelist/typelistpushfront.hpp
template<typename List, typename NewElement>
class PushFrontT;

template<typename... Elements, typename NewElement>
class PushFrontT<Typelist<Elements...>, NewElement> {
    public:
        using Type = Typelist<NewElement, Elements...>;
};

template<typename List, typename NewElement>
using PushFront = typename PushFrontT<List, NewElement>::Type;
```

## 24.2 类型列表算法

### 24.2.1 索引算法

利用`PopFront<>`和递归可以实现获取类型列表中第N个类型的算法：

```cpp
// typelist/nthelement.hpp
// recursive case:
template<typename List, unsigned N>
class NthElementT : public NthElementT<PopFront<List>, N-1>
{
};

// basis case:
template<typename List>
class NthElementT<List, 0> : public FrontT<List>
{
};

template<typename List, unsigned N>
using NthElement = typename NthElementT<List, N>::Type;
```

### 24.2.2 查找算法

查找类型列表中最大位宽的算法如下：

```cpp
// typelist/largesttype.hpp
template<typename List>
class LargestTypeT;

// recursive case:
template<typename List>
class LargestTypeT
{
    private:
        using First = Front<List>;
        using Rest = typename LargestTypeT<PopFront<List>>::Type;
    public:
        using Type = IfThenElse<(sizeof(First) >= sizeof(Rest)), First, Rest>;
};

// basis case:
template<>
class LargestTypeT<Typelist<>>
{
    public:
        using Type = char;
};

template<typename List>
using LargestType = typename LargestTypeT<List>::Type;
```

递归终止条件模板可能会导致编译错误，为此可以添加`IsEmpty<>`进行封装：

```cpp
// typelist/typelistisempty.hpp
template<typename List>
class IsEmpty
{
    public:
        static constexpr bool value = false;
};

template<>
class IsEmpty<Typelist<>> {
    public:
        static constexpr bool value = true;
};
```

```cpp
// typelist/genericlargesttype.hpp
template<typename List, bool Empty = IsEmpty<List>::value>
class LargestTypeT;

// recursive case:
template<typename List>
class LargestTypeT<List, false>
{
    private:
        using Contender = Front<List>;
        using Best = typename LargestTypeT<PopFront<List>>::Type;
    public:
        using Type = IfThenElse<(sizeof(Contender) >= sizeof(Best)), Contender, Best>;
};

// basis case:
template<typename List>
class LargestTypeT<List, true>
{
    public:
        using Type = char;
};

template<typename List>
using LargestType = typename LargestTypeT<List>::Type;
```

### 24.2.3 追加算法

追加算法和`PushFrontT<>`类似：

```cpp
// typelist/typelistpushback.hpp
template<typename List, typename NewElement>
class PushBackT;

template<typename... Elements, typename NewElement>
class PushBackT<Typelist<Elements...>, NewElement>
{
    public:
        using Type = Typelist<Elements..., NewElement>;
};

template<typename List, typename NewElement>
using PushBack = typename PushBackT<List, NewElement>::Type;
```

但是也可以通过`Front<>`、`PushFront<>`、`PopFront<>`和`IsEmpty<>`实现：

```cpp
// typelist/genericpushback.hpp
template<typename List, typename NewElement, bool = IsEmpty<List>::value>
class PushBackRecT;

// recursive case:
template<typename List, typename NewElement>
class PushBackRecT<List, NewElement, false>
{
        using Head = Front<List>;
        using Tail = PopFront<List>;
        using NewTail = typename PushBackRecT<Tail, NewElement>::Type;
    public:
        using Type = PushFront<Head, NewTail>;
};

// basis case:
template<typename List, typename NewElement>
class PushBackRecT<List, NewElement, true>
{
    public:
        using Type = PushFront<List, NewElement>;
};

// generic push-back operation:
template<typename List, typename NewElement>
class PushBackT : public PushBackRecT<List, NewElement> { };

template<typename List, typename NewElement>
using PushBack = typename PushBackT<List, NewElement>::Type;
```

两种方法虽然效果一样，但是在编译时间上有很大差别，原因在于第一种方法只需要实例化多个`Typelist<>`就可以，而第二种方法还会实例化`PushBackRecT<>`、`PushFrontT<>`、`FrontT<>`和`PopFrontT<>`。

### 24.2.4 反转算法

反转算法的一种实现方式是通过`Front<>`取出类型列表首部类型，然后递归反转剩下的类型列表，最后再进行组合：

```cpp
// typelist/typelistreverse.hpp
template<typename List, bool Empty = IsEmpty<List>::value>
class ReverseT;

template<typename List>
using Reverse = typename ReverseT<List>::Type;

// recursive case:
template<typename List>
class ReverseT<List, false>
    : public PushBackT<Reverse<PopFront<List>>, Front<List>> { };

// basis case:
template<typename List>
class ReverseT<List, true>
{
    public:
        using Type = List;
};
```

`Reverse<>`也可以用来实现`PopBack<>`（感觉有些多此一举）：

```cpp
// typelist/typelistpopback.hpp
template<typename List>
class PopBackT {
    public:
        using Type = Reverse<PopFront<Reverse<List>>>;
};

template<typename List>
using PopBack = typename PopBackT<List>::Type;
```

### 24.2.5 变换算法 {#Transforming-a-Typelist}

变换算法是将类型列表中的类型进行变换生成新的类型列表的算法，实现方法是先对首部类型进行类型变换，然后递归的对去除首部的类型列表进行变换，最终再拼接：

```cpp
// typelist/transform.hpp
template<typename List, template<typename T> class MetaFun, bool Empty = IsEmpty<List>::value>
class TransformT;

// recursive case:
template<typename List, template<typename T> class MetaFun>
class TransformT<List, MetaFun, false>
    : public PushFrontT<typename TransformT<PopFront<List>, MetaFun>::Type,
                        typename MetaFun<Front<List>>::Type>
{
};

// basis case:
template<typename List, template<typename T> class MetaFun>
class TransformT<List, MetaFun, true>
{
    public:
        using Type = List;
};

template<typename List, template<typename T> class MetaFun>
using Transform = typename TransformT<List, MetaFun>::Type;
```

变换元函数`MetaFun<>`可以是下面的形式：

```cpp
// typelist/addconst.hpp
template<typename T>
struct AddConstT
{
    using Type = T const;
};

template<typename T>
using AddConst = typename AddConstT<T>::Type;
```

### 24.2.6 累加算法

累加算法可以在遍历类型列表中的每一个类型的过程中使用类型函数`F`进行计算：

```cpp
// typelist/accumulate.hpp
template<typename List,
            template<typename X, typename Y> class F,
            typename I,
            bool = IsEmpty<List>::value>
class AccumulateT;

// recursive case:
template<typename List,
            template<typename X, typename Y> class F,
            typename I>
class AccumulateT<List, F, I, false>
    : public AccumulateT<PopFront<List>, F, typename F<I, Front<List>>::Type>
{
};

// basis case:
template<typename List,
            template<typename X, typename Y> class F,
            typename I>
class AccumulateT<List, F, I, true>
{
    public:
        using Type = I;
};

template<typename List,
            template<typename X, typename Y> class F,
            typename I>
using Accumulate = typename AccumulateT<List, F, I>::Type;
```

累加算法也可以用来查找类型列表中位宽最大的类型：

```cpp
// typelist/largesttypeacc0.hpp
template<typename T, typename U>
class LargerTypeT
    : public IfThenElseT<sizeof(T) >= sizeof(U), T, U>
{
};

template<typename Typelist>
class LargestTypeAccT
    : public AccumulateT<PopFront<Typelist>, LargerTypeT, Front<Typelist>>
{
};

template<typename Typelist>
using LargestTypeAcc = typename LargestTypeAccT<Typelist>::Type;
```

当向`LargestTypeAcc<>`传递空类型列表时会出现编译错误，改进方法如下：

```cpp
// typelist/largesttypeacc.hpp
template<typename T, typename U>
class LargerTypeT
    : public IfThenElseT<sizeof(T) >= sizeof(U), T, U>
{
};

template<typename Typelist, bool = IsEmpty<Typelist>::value>
class LargestTypeAccT;

template<typename Typelist>
class LargestTypeAccT<Typelist, false>
    : public AccumulateT<PopFront<Typelist>, LargerTypeT, Front<Typelist>>
{
};

template<typename Typelist>
class LargestTypeAccT<Typelist, true>
{
};

template<typename Typelist>
using LargestTypeAcc = typename LargestTypeAccT<Typelist>::Type;
```

### 24.2.7 插入排序

插入排序的思路是首先取出类型列表的首部类型，然后对剩下的类型列表进行排序，最终再将首部类型插入到合适的位置：

```cpp
// typelist/insertionsort.hpp
template<typename List,
            template<typename T, typename U> class Compare,
            bool = IsEmpty<List>::value>
class InsertionSortT;

template<typename List,
            template<typename T, typename U> class Compare>
using InsertionSort = typename InsertionSortT<List, Compare>::Type;

// recursive case (insert first element into sorted list):
template<typename List,
            template<typename T, typename U> class Compare>
class InsertionSortT<List, Compare, false>
    : public InsertSortedT<InsertionSort<PopFront<List>, Compare>,
                            Front<List>, Compare>
{
};

// basis case (an empty list is sorted):
template<typename List,
            template<typename T, typename U> class Compare>
class InsertionSortT<List, Compare, true>
{
    public:
        using Type = List;
};
```

在有序类型列表中插入类型的模板`InsertSortedT<>`依然是通过递归的方式实现的：

```cpp
// typelist/insertsorted.hpp
#include "identity.hpp"
template<typename List, typename Element,
            template<typename T, typename U> class Compare,
            bool = IsEmpty<List>::value>
class InsertSortedT;

// recursive case:
template<typename List, typename Element,
            template<typename T, typename U> class Compare>
class InsertSortedT<List, Element, Compare, false>
{
        // compute the tail of the resulting list:
        using NewTail = typename IfThenElse<Compare<Element, Front<List>>::value,
                                            IdentityT<List>,
                                            InsertSortedT<PopFront<List>, Element, Compare>>::Type;
        // compute the head of the resulting list:
        using NewHead = IfThenElse<Compare<Element, Front<List>>::value,
                                    Element,
                                    Front<List>>;
    public:
        using Type = PushFront<NewTail, NewHead>;
};

// basis case:
template<typename List, typename Element,
            template<typename T, typename U> class Compare>
class InsertSortedT<List, Element, Compare, true>
    : public PushFrontT<List, Element>
{
};

template<typename List, typename Element,
            template<typename T, typename U> class Compare>
using InsertSorted = typename InsertSortedT<List, Element, Compare>::Type;
```

当`Compare<T, U>::value`为`true`时，在排序后的类型列表中`T`会出现在`U`的前面，测试代码如下：

```cpp
// typelist/insertionsorttest.hpp
template<typename T, typename U>
struct SmallerThanT {
    static constexpr bool value = sizeof(T) < sizeof(U);
};

void testInsertionSort()
{
    using Types = Typelist<int, char, short, double>;
    using ST = InsertionSort<Types, SmallerThanT>;
    std::cout << std::is_same<ST,Typelist<char, short, int, double>>::value << '\n';
}
```

## 24.3 非类型的类型列表

这个名字有点绕，实际意思就是在编译时存储值的列表，为此需要为每个值构造一个类型：

```cpp
// typelist/ctvalue.hpp
template<typename T, T Value>
struct CTValue
{
    static constexpr T value = Value;
};
```

对前几个素数的编译时值列表求积的代码为：

```cpp
// typelist/multiply.hpp
template<typename T, typename U>
struct MultiplyT;

template<typename T, T Value1, T Value2>
struct MultiplyT<CTValue<T, Value1>, CTValue<T, Value2>> {
    public:
        using Type = CTValue<T, Value1 * Value2>;
};

template<typename T, typename U>
using Multiply = typename MultiplyT<T, U>::Type;
```

```cpp
using Primes = Typelist<CTValue<int, 2>, CTValue<int, 3>,
                        CTValue<int, 5>, CTValue<int, 7>,
                        CTValue<int, 11>>;

Accumulate<Primes, MultiplyT, CTValue<int, 1>>::value
```

如果编译时值列表中的值都是同一类型的话，那么也可以使用下面的模板：

```cpp
// typelist/cttypelist.hpp
template<typename T, T... Values>
using CTTypelist = Typelist<CTValue<T, Values>...>;
```

或者不借助`TypeList<>`，直接实现`ValueList<>`：

```cpp
// typelist/valuelist.hpp
template<typename T, T... Values>
struct Valuelist {
};

template<typename T, T... Values>
struct IsEmpty<Valuelist<T, Values...>> {
    static constexpr bool value = sizeof...(Values) == 0;
};

template<typename T, T Head, T... Tail>
struct FrontT<Valuelist<T, Head, Tail...>> {
    using Type = CTValue<T, Head>;
    static constexpr T value = Head;
};

template<typename T, T Head, T... Tail>
struct PopFrontT<Valuelist<T, Head, Tail...>> {
    using Type = Valuelist<T, Tail...>;
};

template<typename T, T... Values, T New>
struct PushFrontT<Valuelist<T, Values...>, CTValue<T, New>> {
    using Type = Valuelist<T, New, Values...>;
};

template<typename T, T... Values, T New>
struct PushBackT<Valuelist<T, Values...>, CTValue<T, New>> {
    using Type = Valuelist<T, Values..., New>;
};
```

定义了`IsEmpty<>`、`FrontT<>`、`PopFrontT<>`、`PushFrontT`后就可以进行插入排序了：

```cpp
// typelist/valuelisttest.hpp
template<typename T, typename U>
struct GreaterThanT;

template<typename T, T First, T Second>
struct GreaterThanT<CTValue<T, First>, CTValue<T, Second>> {
    static constexpr bool value = First > Second;
};

void valuelisttest()
{
    using Integers = Valuelist<int, 6, 2, 4, 9, 5, 2, 1, 7>;
    using SortedIntegers = InsertionSort<Integers, GreaterThanT>;
    static_assert(std::is_same_v<SortedIntegers, Valuelist<int, 9, 7, 6, 5, 4, 2, 2, 1>>,
                    "insertion sort failed");
}
```

### 24.3.1 使用auto推导非类型模板参数

从C++17开始，由于`auto`可以用来推导非类型模板参数的类型，所以`CTValue<>`还有更简单的定义：

```cpp
// typelist/ctvalue17.hpp
template<auto Value>
struct CTValue
{
    static constexpr auto value = Value;
};
```

## 24.4 利用参数包展开优化算法

对于[24.2.5](#Transforming-a-Typelist)中的变换算法，还可以利用参数包展开来进行优化：

```cpp
// typelist/variadictransform.hpp
template<typename... Elements, template<typename T> class MetaFun>
class TransformT<Typelist<Elements...>, MetaFun, false>
{
    public:
        using Type = Typelist<typename MetaFun<Elements>::Type...>;
};
```

书中还给出了一个根据索引提取类型列表中指定类型的例子：

```cpp
// typelist/select.hpp
template<typename Types, typename Indices>
class SelectT;

template<typename Types, unsigned... Indices>
class SelectT<Types, Valuelist<unsigned, Indices...>>
{
    public:
        using Type = Typelist<NthElement<Types, Indices>...>;
};

template<typename Types, typename Indices>
using Select = typename SelectT<Types, Indices>::Type;
```

## 24.5 Cons风格的类型列表

假如C++不支持可变参数模板，那么类型列表就只能写成LISP中的样子了：

```cpp
// typelist/cons.hpp
class Nil { };

template<typename HeadT, typename TailT = Nil>
class Cons {
    public:
        using Head = HeadT;
        using Tail = TailT;
};
```

```cpp
// typelist/consfront.hpp
template<typename List>
class FrontT {
    public:
        using Type = typename List::Head;
};

template<typename List>
using Front = typename FrontT<List>::Type;
```

```cpp
// typelist/conspushfront.hpp
template<typename List, typename Element>
class PushFrontT {
    public:
        using Type = Cons<Element, List>;
};

template<typename List, typename Element>
using PushFront = typename PushFrontT<List, Element>::Type;
```

```cpp
// typelist/conspopfront.hpp
template<typename List>
class PopFrontT {
    public:
        using Type = typename List::Tail;
};

template<typename List>
using PopFront = typename PopFrontT<List>::Type;
```

```cpp
// typelist/consisempty.hpp
template<typename List>
struct IsEmpty {
    static constexpr bool value = false;
};

template<>
struct IsEmpty<Nil> {
    static constexpr bool value = true;
};
```

定义了这些基础操作后，就可以进行插入排序了：

```cpp
// typelist/conslisttest.hpp
template<typename T, typename U>
struct SmallerThanT {
    static constexpr bool value = sizeof(T) < sizeof(U);
};

void conslisttest()
{
    using ConsList = Cons<int, Cons<char, Cons<short, Cons<double>>>>;
    using SortedTypes = InsertionSort<ConsList, SmallerThanT>;
    using Expected = Cons<char, Cons<short, Cons<int, Cons<double>>>>;
    std::cout << std::is_same<SortedTypes, Expected>::value << '\n';
}
```

## 24.6 后记
