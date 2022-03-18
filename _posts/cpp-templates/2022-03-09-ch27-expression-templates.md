---
layout: post
title: 《C++ Templates》第27章 表达式模板
categories: Reading
tags: C++
---

# 27 表达式模板

表达式模板用来支持针对整个数组的数值运算。

## 27.1 临时量和循环分割

在引入表达式模板之前，先看一个数组运算的例子：

```cpp
// exprtmpl/sarray1.hpp
#include <cstddef>
#include <cassert>

template<typename T>
class SArray {
    public:
        // create array with initial size
        explicit SArray (std::size_t s)
            : storage(new T[s]), storage_size(s) {
                init();
        }

        // copy constructor
        SArray (SArray<T> const& orig)
            : storage(new T[orig.size()]), storage_size(orig.size()) {
                copy(orig);
        }

        // destructor: free memory
        ~SArray() {
            delete[] storage;
        }

        // assignment operator
        SArray<T>& operator= (SArray<T> const& orig) {
            if (&orig!=this) {
                copy(orig);
            }
            return *this;
        }

        // return size
        std::size_t size() const {
            return storage_size;
        }

        // index operator for constants and variables
        T const& operator[] (std::size_t idx) const {
            return storage[idx];
        }
        T& operator[] (std::size_t idx) {
            return storage[idx];
        }
    protected:
        // init values with default constructor
        void init() {
            for (std::size_t idx = 0; idx<size(); ++idx) {
                storage[idx] = T();
            }
        }
        // copy values of another array
        void copy (SArray<T> const& orig) {
            assert(size()==orig.size());
            for (std::size_t idx = 0; idx<size(); ++idx) {
                storage[idx] = orig.storage[idx];
            }
        }
    private:
        T* storage;                 // storage of the elements
        std::size_t storage_size;   // number of elements
};
```

```cpp
// exprtmpl/sarrayops1.hpp
// addition of two SArrays
template<typename T>
SArray<T> operator+ (SArray<T> const& a, SArray<T> const& b)
{
    assert(a.size()==b.size());
    SArray<T> result(a.size());
    for (std::size_t k = 0; k<a.size(); ++k) {
        result[k] = a[k]+b[k];
    }
    return result;
}

// multiplication of two SArrays
template<typename T>
SArray<T> operator* (SArray<T> const& a, SArray<T> const& b)
{
    assert(a.size()==b.size());
    SArray<T> result(a.size());
    for (std::size_t k = 0; k<a.size(); ++k) {
        result[k] = a[k]*b[k];
    }
    return result;
}

// multiplication of scalar and SArray
template<typename T>
SArray<T> operator* (T const& s, SArray<T> const& a)
{
    SArray<T> result(a.size());
    for (std::size_t k = 0; k<a.size(); ++k) {
        result[k] = s*a[k];
    }
    return result;
}

// multiplication of SArray and scalar
// addition of scalar and SArray
// addition of SArray and scalar
// ...
```

```cpp
// exprtmpl/sarray1.cpp
#include "sarray1.hpp"
#include "sarrayops1.hpp"

int main()
{
    SArray<double> x(1000), y(1000);
    // ...
    x = 1.2*x + x*y;
}
```

算法不够高效的原因在于：

1. 创建了3个临时量，大小都为1000
2. 表达式`x = 1.2*x + x*y`产生了6000次读和3000次写

早期的解决办法是使用运算赋值运算符来避免创建临时量以节省构造和析构：

```cpp
// exprtmpl/sarrayops2.hpp
// additive assignment of SArray
template<typename T>
SArray<T>& SArray<T>::operator+= (SArray<T> const& b)
{
    assert(size()==orig.size());
    for (std::size_t k = 0; k<size(); ++k) {
        (*this)[k] += b[k];
    }
    return *this;
}

// multiplicative assignment of SArray
template<typename T>
SArray<T>& SArray<T>::operator*= (SArray<T> const& b)
{
    assert(size()==orig.size());
    for (std::size_t k = 0; k<size(); ++k) {
        (*this)[k] *= b[k];
    }
    return *this;
}

// multiplicative assignment of scalar
template<typename T>
SArray<T>& SArray<T>::operator*= (T const& s)
{
    for (std::size_t k = 0; k<size(); ++k) {
        (*this)[k] *= s;
    }
    return *this;
}
```

```cpp
// exprtmpl/sarray2.cpp
#include "sarray2.hpp"
#include "sarrayops1.hpp"
#include "sarrayops2.hpp"

int main()
{
    SArray<double> x(1000), y(1000);
    // ...
    // process x = 1.2*x + x*y
    SArray<double> tmp(x);
    tmp *= y;
    x *= 1.2;
    x += tmp;
}
```

缺点在于写起来很繁琐。最理想的方式是编译器可以将代码转化为下面的形式：

```cpp
int main()
{
    SArray<double> x(1000), y(1000);
    // ...
    for (int idx = 0; idx<x.size(); ++idx) {
        x[idx] = 1.2*x[idx] + x[idx]*y[idx];
    }
}
```

减少到只有2000次读和1000次写。

## 27.2 将表达式作为模板实参

`x = 1.2*x + x*y`产生临时量和读写次数多的原因在于编译器是根据表达式的优先级进行计算的，如果将整个表达式的计算推迟到最终的赋值运算，就可以生成最优的代码了，也就是在计算`1.2*x + x*y`的过程中保留结果的生成过程，这可以通过下面的类型保留：

```cpp
A_Add<A_Mult<A_Scalar<double>,Array<double>>,
        A_Mult<Array<double>, Array<double>>>
```

### 27.2.1 表达式模板

表达式模板`A_Add<>`、`A_Mult<>`和`A_Scalar<>`定义如下：

```cpp
// exprtmpl/exprops1.hpp
#include <cstddef>
#include <cassert>

// include helper class traits template to select whether to refer to an
// expression template node either by value or by reference
#include "exprops1a.hpp"

// class for objects that represent the addition of two operands
template<typename T, typename OP1, typename OP2>
class A_Add {
    private:
        typename A_Traits<OP1>::ExprRef op1; // first operand
        typename A_Traits<OP2>::ExprRef op2; // second operand
    public:
        // constructor initializes references to operands
        A_Add (OP1 const& a, OP2 const& b)
            : op1(a), op2(b) {
        }

        // compute sum when value requested
        T operator[] (std::size_t idx) const {
            return op1[idx] + op2[idx];
        }

        // size is maximum size
        std::size_t size() const {
            assert (op1.size()==0 || op2.size()==0 || op1.size()==op2.size());
            return op1.size()!=0 ? op1.size() : op2.size();
        }
};

// class for objects that represent the multiplication of two operands
template<typename T, typename OP1, typename OP2>
class A_Mult {
    private:
        typename A_Traits<OP1>::ExprRef op1; // first operand
        typename A_Traits<OP2>::ExprRef op2; // second operand
    public:
        // constructor initializes references to operands
        A_Mult (OP1 const& a, OP2 const& b)
            : op1(a), op2(b) {
        }

        // compute product when value requested
        T operator[] (std::size_t idx) const {
            return op1[idx] * op2[idx];
        }

        // size is maximum size
        std::size_t size() const {
            assert (op1.size()==0 || op2.size()==0 || op1.size()==op2.size());
            return op1.size()!=0 ? op1.size() : op2.size();
        }
};
```

```cpp
// exprtmpl/exprscalar.hpp
// class for objects that represent scalars:
template<typename T>
class A_Scalar {
    private:
        T const& s; // value of the scalar
    public:
        // constructor initializes value
        constexpr A_Scalar (T const& v)
            : s(v) {
        }

        // for index operations, the scalar is the value of each element
        constexpr T const& operator[] (std::size_t) const {
            return s;
        }

        // scalars have zero as size
        constexpr std::size_t size() const {
            return 0;
        };
};
```

```cpp
// exprtmpl/exprops1a.hpp
// helper traits class to select how to refer to an expression template node
// - in general by reference
// - for scalars by value
template<typename T> class A_Scalar;
// primary template
template<typename T>
class A_Traits {
    public:
        using ExprRef = T const&; // type to refer to is constant reference
};

// partial specialization for scalars
template<typename T>
class A_Traits<A_Scalar<T>> {
    public:
        using ExprRef = A_Scalar<T>; // type to refer to is ordinary value
};
```

`A_Scalar::operator[]`返回`s`只是为了方便。

### 27.2.2 数组类型

`A_Add<>`和`A_Mult<>`的模板参数`OP1`和`OP2`既可以是数组，也可以是数组的运算结果，书中为了统一，定义了新的数组表示类模板`Array<>`，其中仍然使用`SArray<>`来存储数据：

```cpp
// exprtmpl/exprarray.hpp
#include <cstddef>
#include <cassert>
#include "sarray1.hpp"

template<typename T, typename Rep = SArray<T>>
class Array {
    private:
        Rep expr_rep; // (access to) the data of the array
    public:
        // create array with initial size
        explicit Array (std::size_t s)
            : expr_rep(s) {
        }

        // create array from possible representation
        Array (Rep const& rb)
            : expr_rep(rb) {
        }

        // assignment operator for same type
        Array& operator= (Array const& b) {
            assert(size()==b.size());
            for (std::size_t idx = 0; idx<b.size(); ++idx) {
                expr_rep[idx] = b[idx];
            }
            return *this;
        }

        // assignment operator for arrays of different type
        template<typename T2, typename Rep2>
        Array& operator= (Array<T2, Rep2> const& b) {
            assert(size()==b.size());
            for (std::size_t idx = 0; idx<b.size(); ++idx) {
                expr_rep[idx] = b[idx];
            }
            return *this;
        }

        // size is size of represented data
        std::size_t size() const {
            return expr_rep.size();
        }

        // index operator for constants and variables
        decltype(auto) operator[] (std::size_t idx) const {
            assert(idx<size());
            return expr_rep[idx];
        }
        T& operator[] (std::size_t idx) {
            assert(idx<size());
            return expr_rep[idx];
        }

        // return what the array currently represents
        Rep const& rep() const {
            return expr_rep;
        }
        Rep& rep() {
            return expr_rep;
        }
};
```

`const`版本的`Array::operator[]`返回`decltype(auto)`的原因是`A_Mult::operator[]`和`A_Add::operator[]`可能返回临时量，此时推导结果为非引用类型。

### 27.2.3 表达式模板的运算符

表达式模板的运算符只需要将表示计算结果的数组以引用的方式传递到返回值中就可以了：

```cpp
// exprtmpl/exprops2.hpp
// addition of two Arrays:
template<typename T, typename R1, typename R2>
Array<T,A_Add<T,R1,R2>> operator+ (Array<T,R1> const& a, Array<T,R2> const& b) {
    return Array<T,A_Add<T,R1,R2>>(A_Add<T,R1,R2>(a.rep(),b.rep()));
}

// multiplication of two Arrays:
template<typename T, typename R1, typename R2>
Array<T,A_Mult<T,R1,R2>> operator* (Array<T,R1> const& a, Array<T,R2> const& b) {
    return Array<T,A_Mult<T,R1,R2>>(A_Mult<T,R1,R2>(a.rep(), b.rep()));
}

// multiplication of scalar and Array:
template<typename T, typename R2>
Array<T,A_Mult<T,A_Scalar<T>,R2>> operator* (T const& s, Array<T,R2> const& b) {
    return Array<T,A_Mult<T,A_Scalar<T>,R2>>(A_Mult<T,A_Scalar<T>,R2>(A_Scalar<T>(s), b.rep()));
}

// multiplication of Array and scalar, addition of scalar and Array
// addition of Array and scalar:
// ...
```

### 27.2.4 表达式求值 {#Review}

改进后的代码为：

```cpp
int main()
{
    Array<double> x(1000), y(1000);
    // ...
    x = 1.2*x + x*y;
}
```

由于运算符的优先级没有改变，因此编译器的计算顺序如下：

1. `1.2*x`返回`Array<double, A_Mult<double, A_Scalar<double>, SArray<double>>>`类型的对象，虽然还是临时对象，但是该临时对象中是`x`的引用，不会带来拷贝
2. `x*y`返回`Array<double, A_Mult<double, SArray<double>, SArray<double>>>`类型的临时对象
3. `1.2*x + x*y`返回的临时对象类型为：

```cpp
Array<double,
    A_Add<double,
        A_Mult<double, A_Scalar<double>, SArray<double>>,
        A_Mult<double, SArray<double>, SArray<double>>>>
```

接下来会调用`Array`的成员模板`operator=`，循环中的`expr_rep[idx] = b[idx]`会由于`b[idx]`的递归而最终展开为`(1.2*x[idx]) + (x[idx]*y[idx])`。

### 27.2.5 表达式模板的赋值操作

`A_Add<>`和`A_Mult<>`是不能出现在表达式的左边的，因为其中的下标运算符返回的是过期值，但是某些表达式模板应该是可以返回左值的，例如`A_Subscript<>`：

```cpp
// exprtmpl/exprops3.hpp
template<typename T, typename A1, typename A2>
class A_Subscript {
    public:
        // constructor initializes references to operands
        A_Subscript (A1 const& a, A2 const& b)
            : a1(a), a2(b) {
        }

        // process subscription when value requested
        decltype(auto) operator[] (std::size_t idx) const {
            return a1[a2[idx]];
        }
        T& operator[] (std::size_t idx) {
            return a1[a2[idx]];
        }

        // size is size of inner array
        std::size_t size() const {
            return a2.size();
        }
    private:
        A1 const& a1; // reference to first operand
        A2 const& a2; // reference to second operand
};
```

这个模板只支持`A_Subscript::a2`中存储的是整数，或许在对称密码算法置换中用到。

## 27.3 表达式模板的性能和限制

实际上[27.2.4](#Review)的分析并不是正确的，因为`[]`的在递归的过程中会进行函数调用，但是因为函数都很短小，所以内联可以解决问题。同时表达式模板还要求赋值的过程是不能覆盖后续计算需要的原数据。

## 27.4 后记
