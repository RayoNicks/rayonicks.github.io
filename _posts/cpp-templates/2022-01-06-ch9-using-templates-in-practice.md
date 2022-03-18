---
layout: post
title: 《C++ Templates》第9章 模板在实际中的应用
categories: Reading
tags: C++
---

# 9 模板在实际中的应用

## 9.1 模板的包含模型

### 9.1.1 链接错误

一般来说C++代码的组织方式为：

- 在头文件中声明类和定义类型
- 全局变量和函数声明在头文件中，定义在源文件中

自然而然地会用相同的方式定义和使用函数模板：

```cpp
// basics/myfirst.hpp
#ifndef MYFIRST_HPP
#define MYFIRST_HPP

// declaration of template
template<typename T>
void printTypeof (T const&);

#endif // MYFIRST_HPP
```

```cpp
// basics/myfirst.cpp
#include <iostream>
#include <typeinfo>
#include "myfirst.hpp"

// implementation/definition of template
template<typename T>
void printTypeof (T const& x)
{
    std::cout << typeid(x).name() << '\n';
}
```

```cpp
// basics/myfirstmain.cpp
#include "myfirst.hpp"

// use of the template
int main()
{
    double ice = 3.0;
    printTypeof(ice); // call function template for type double
}
```

上面的使用方式会报链接错误，这是因为模板定义和触发实例化模板的代码在两个文件中。编译器并不知道编译`printTypeof(ice)`时需要实例化模板，而当编译另一个文件时，由于没有代码触发模板实例化，模板自然也就不会被实例化。原文：

>Unfortunately, in the previous example, these two pieces of information are in files that are compiled separately. Therefore, when our compiler sees the call to printTypeof() but has no definition in sight to instantiate this function for double, it just assumes that such a definition is provided elsewhere and creates a reference (for the linker to resolve) to that definition. On the other hand, when the compiler processes the file myfirst.cpp, it has no indication at that point that it must instantiate the template definition it contains for specific arguments.

### 9.1.2 在头文件中定义模板

解决上面问题的方法就是在头文件中完整的定义模板：

```cpp
// basics/myfirst2.hpp
#ifndef MYFIRST_HPP
#define MYFIRST_HPP

#include <iostream>
#include <typeinfo>

// declaration of template
template<typename T>
void printTypeof (T const&);

// implementation/definition of template
template<typename T>
void printTypeof (T const& x)
{
    std::cout << typeid(x).name() << '\n';
}

#endif // MYFIRST_HPP
```

将模板定义在头文件中的方式就是模板的包含模型（inclusion model），这会带来两个问题：

- 包含额外的头文件（也就是定义模板时使用的<iostream>和<typeinfo>）
- 如果两个翻译单元都实例化了同一模板函数，则可能会出现重复定义

## 9.2 模板和内联

将函数声明为`inline`并不一定真正进行内联，而只是让同一个函数可以被定义很多次，对于模板来说也是这个道理。

## 9.3 预编译头文件

C++头文件编译的很慢，预编译头文件技术就是提取公共头文件提前编译好。

## 9.4 理解模板编译错误

一般来说模板错误都会超级复杂，本节给了两个例子。

第一个是类型错误：

```cpp
// basics/errornovel1.cpp
#include <string>
#include <map>
#include <algorithm>

int main()
{
    std::map<std::string,double> coll;
    // ...
    // find the first nonempty string in coll:
    auto pos = std::find_if (coll.begin(), coll.end(),
                            [] (std::string const& s) {
                                return s != "";
                            });
}
```

产生的编译错误如下（看到这我不由自主地笑了，傻叉编译器确实都是这样子的）：

```log
In file included from /usr/include/c++/7/bits/stl_algobase.h:71:0,
                 from /usr/include/c++/7/bits/char_traits.h:39,
                 from /usr/include/c++/7/string:40,
                 from errornovel1.cpp:1:
/usr/include/c++/7/bits/predefined_ops.h: In instantiation of ‘bool __gnu_cxx::__ops::_Iter_pred<_Predicate>::operator()(_Iterator) [with _Iterator = std::_Rb_tree_iterator<std::pair<const std::__cxx11::basic_string<char>, double> >; _Predicate = main()::<lambda(const string&)>]’:
/usr/include/c++/7/bits/stl_algo.h:104:42:   required from ‘_InputIterator std::__find_if(_InputIterator, _InputIterator, _Predicate, std::input_iterator_tag) [with _InputIterator = std::_Rb_tree_iterator<std::pair<const std::__cxx11::basic_string<char>, double> >; _Predicate = __gnu_cxx::__ops::_Iter_pred<main()::<lambda(const string&)> >]’
/usr/include/c++/7/bits/stl_algo.h:161:23:   required from ‘_Iterator std::__find_if(_Iterator, _Iterator, _Predicate) [with _Iterator = std::_Rb_tree_iterator<std::pair<const std::__cxx11::basic_string<char>, double> >; _Predicate = __gnu_cxx::__ops::_Iter_pred<main()::<lambda(const string&)> >]’
/usr/include/c++/7/bits/stl_algo.h:3932:28:   required from ‘_IIter std::find_if(_IIter, _IIter, _Predicate) [with _IIter = std::_Rb_tree_iterator<std::pair<const std::__cxx11::basic_string<char>, double> >; _Predicate = main()::<lambda(const string&)>]’
errornovel1.cpp:13:30:   required from here
/usr/include/c++/7/bits/predefined_ops.h:283:11: error: no match for call to ‘(main()::<lambda(const string&)>) (std::pair<const std::__cxx11::basic_string<char>, double>&)’
  { return bool(_M_pred(*__it)); }
           ^~~~~~~~~~~~~~~~~~~~
/usr/include/c++/7/bits/predefined_ops.h:283:11: note: candidate: bool (*)(const string&) {aka bool (*)(const std::__cxx11::basic_string<char>&)} <conversion>
/usr/include/c++/7/bits/predefined_ops.h:283:11: note:   candidate expects 2 arguments, 2 provided
errornovel1.cpp:11:53: note: candidate: main()::<lambda(const string&)>
                             [] (std::string const& s) {
                                                     ^
errornovel1.cpp:11:53: note:   no known conversion for argument 1 from ‘std::pair<const std::__cxx11::basic_string<char>, double>’ to ‘const string& {aka const std::__cxx11::basic_string<char>&}’
```

后面就是解释了一下该怎么去读懂这个错误信息，不过如果有经验的话应该一眼就能看明白这个错误信息是啥的。

第二个例子是缺少`const`：

```cpp
// basics/errornovel2.cpp
#include <string>
#include <unordered_set>

class Customer
{
    private:
        std::string name;
    public:
        Customer (std::string const& n)
            : name(n) {
        }
        std::string getName() const {
            return name;
        }
};

int main()
{
    // provide our own hash function:
    struct MyCustomerHash {
        // NOTE: missing const is only an error with g++ and clang:
        std::size_t operator() (Customer const& c) {
            return std::hash<std::string>()(c.getName());
        }
    };

    // and use it for a hash table of Customers:
    std::unordered_set<Customer,MyCustomerHash> coll;
    // ...
}
```

g++产生的编译错误如下（人间真实）：

```log
In file included from /usr/include/c++/7/bits/hashtable.h:35:0,
                 from /usr/include/c++/7/unordered_set:47,
                 from errornovel2.cpp:2:
/usr/include/c++/7/bits/hashtable_policy.h: In instantiation of ‘struct std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash>’:
/usr/include/c++/7/type_traits:143:12:   required from ‘struct std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> >’
/usr/include/c++/7/type_traits:154:31:   required from ‘struct std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
/usr/include/c++/7/bits/unordered_set.h:98:63:   required from ‘class std::unordered_set<Customer, main()::MyCustomerHash>’
errornovel2.cpp:28:49:   required from here
/usr/include/c++/7/bits/hashtable_policy.h:87:34: error: no match for call to ‘(const main()::MyCustomerHash) (const Customer&)’
  noexcept(declval<const _Hash&>()(declval<const _Key&>()))>
           ~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~
errornovel2.cpp:22:21: note: candidate: std::size_t main()::MyCustomerHash::operator()(const Customer&) <near match>
         std::size_t operator() (Customer const& c) {
                     ^~~~~~~~
errornovel2.cpp:22:21: note:   passing ‘const main()::MyCustomerHash*’ as ‘this’ argument discards qualifiers
In file included from /usr/include/c++/7/bits/move.h:54:0,
                 from /usr/include/c++/7/bits/stl_pair.h:59,
                 from /usr/include/c++/7/bits/stl_algobase.h:64,
                 from /usr/include/c++/7/bits/char_traits.h:39,
                 from /usr/include/c++/7/string:40,
                 from errornovel2.cpp:1:
/usr/include/c++/7/type_traits: In instantiation of ‘struct std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’:
/usr/include/c++/7/bits/unordered_set.h:98:63:   required from ‘class std::unordered_set<Customer, main()::MyCustomerHash>’
errornovel2.cpp:28:49:   required from here
/usr/include/c++/7/type_traits:154:31: error: ‘value’ is not a member of ‘std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> >’
     : public __bool_constant<!bool(_Pp::value)>
                               ^~~~~~~~~~~~~~~~
In file included from /usr/include/c++/7/unordered_set:48:0,
                 from errornovel2.cpp:2:
/usr/include/c++/7/bits/unordered_set.h: In instantiation of ‘class std::unordered_set<Customer, main()::MyCustomerHash>’:
errornovel2.cpp:28:49:   required from here
/usr/include/c++/7/bits/unordered_set.h:98:63: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef __uset_hashtable<_Value, _Hash, _Pred, _Alloc>  _Hashtable;
                                                               ^~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:105:45: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::key_type key_type;
                                             ^~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:106:47: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::value_type value_type;
                                               ^~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:107:43: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::hasher hasher;
                                           ^~~~~~
/usr/include/c++/7/bits/unordered_set.h:108:46: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::key_equal key_equal;
                                              ^~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:109:51: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::allocator_type allocator_type;
                                                   ^~~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:114:45: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::pointer  pointer;
                                             ^~~~~~~
/usr/include/c++/7/bits/unordered_set.h:115:50: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::const_pointer const_pointer;
                                                  ^~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:116:47: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::reference  reference;
                                               ^~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:117:52: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::const_reference const_reference;
                                                    ^~~~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:118:46: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::iterator  iterator;
                                              ^~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:119:51: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::const_iterator const_iterator;
                                                   ^~~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:120:51: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::local_iterator local_iterator;
                                                   ^~~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:121:57: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::const_local_iterator const_local_iterator;
                                                         ^~~~~~~~~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:122:47: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::size_type  size_type;
                                               ^~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:123:52: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       typedef typename _Hashtable::difference_type difference_type;
                                                    ^~~~~~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:282:7: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       operator=(initializer_list<value_type> __l)
       ^~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:375:2: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
  emplace(_Args&&... __args)
  ^~~~~~~
/usr/include/c++/7/bits/unordered_set.h:419:7: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       insert(const value_type& __x)
       ^~~~~~
/usr/include/c++/7/bits/unordered_set.h:423:7: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       insert(value_type&& __x)
       ^~~~~~
/usr/include/c++/7/bits/unordered_set.h:478:7: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       insert(initializer_list<value_type> __l)
       ^~~~~~
/usr/include/c++/7/bits/unordered_set.h:679:7: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       equal_range(const key_type& __x)
       ^~~~~~~~~~~
/usr/include/c++/7/bits/unordered_set.h:683:7: error: ‘value’ is not a member of ‘std::__not_<std::__and_<std::__is_fast_hash<main()::MyCustomerHash>, std::__detail::__is_noexcept_hash<Customer, main()::MyCustomerHash> > >’
       equal_range(const key_type& __x) const
       ^~~~~~~~~~~
```

这里的错误其实就是`std::unordered_set`需要`MyCustomerHash`的调用运算符`()`是`const`成员函数。

所以作者给出的建议就是多用几个编译器试试。。。

## 9.5 后记

## 9.6 总结

1. 模板的包含模型是最广的应用方式
2. 只有在头文件中类外（也包括结构体外）定义的特化函数模板需要`inline`
3. 如果使用预编译头，要保证头文件的包含顺序是一致的
4. 排除模板错误非常麻烦
