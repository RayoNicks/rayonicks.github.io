---
layout: post
title: 《C++ Templates》第10章 模板相关术语
categories: Reading
tags: C++
---

# 10 模板相关术语

## 10.1 类模板还是模板类（“Class Template” or “Template Class”?）

- 类模板（class template）表示这个类是一个模板，是一类类的统称
- 模板类（template class）有时指类模板，有时指类模板实例化后得到的类

书中一直都用函数模板（function template）、成员模板（member template），成员函数模板（member function template）和变量模板（variable template）。

## 10.2 代换、实例化和特化（Substitution, Instantiation, and Specialization）

- 代换（substitution）：使用模板实参替代模板形参的过程
- 模板实例化（instantiation）：通过代换生成类和函数定义的过程，根据是否代换全部参数分为完整实例化和不完整实例化
- 特化（specialization）：完整和不完整实例化模板得到的结果

## 10.3 声明和定义（Declarations versus Definitions）

- 声明（declaration）：只引入名字，不提供细节

```cpp
class C;        // a declaration of C as a class
void f(int p);  // a declaration of f() as a function and p as a named parameter
extern int v;   // a declaration of v as a variable
```

- 定义（definition）：提供具体的细节

```cpp
class C {};         // definition (and declaration) of class C

void f(int p) {     // definition (and declaration) of function f()
    std::cout << p << '\n';
}

extern int v = 1;   // an initializer makes this a definition for v

int w;              // global variable declarations not preceded by
                    // extern are also definitions
```

### 10.3.1 完整和不完整类型

不完整类型包括：

- 声明但未定义的类
- 没有定义长度的数组
- 数组元素类型是不完整的
- `void`
- 枚举类型的基类和枚举值是未定义的
- 用`const`和`volatile`修饰的上面的类型

其余都是完整类型。例子如下：

```cpp
class C;                // C is an incomplete type
C const* cp;            // cp is a pointer to an incomplete type
extern C elems[10];     // elems has an incomplete type
extern int arr[];       // arr has an incomplete type
// ...
class C { };            // C now is a complete type (and therefore cpand elems
                        // no longer refer to an incomplete type)
int arr[10];            // arr now has a complete type
```

## 10.4 一遍定义规则（The One-Definition Rule）

一遍定义规则可以简单解释为：

- 普通非内联函数（包括成员函数）、非内联全局变量和类静态成员在整个程序中只能定义一次
- 类类型、模板（包括偏特化但不包括特化）、内联函数和变量在每个翻译单元中只能定义一次

可链接实体（linkable entity）包括函数、成员函数、全局变量、静态数据成员，这些实体也可以是模板生成的。

## 10.5 模板实参和模板形参（Template Arguments versus Template Parameters）

- 模板形参（template parameter）：模板声明和定义中`template`后的参数
- 模板实参（template argument）：代换模板形参的实际参数，也就是实例化模板时`<>`中的参数，能够在编译时计算出的值
- 模板标识（template-id）：模板名称和模板参数的结合

## 10.6 总结

1. 使用术语类模板、函数模板和变量模板
2. 模板实例化是用模板实参替代模板形参得到常规类和常规函数的过程，得到的结果为模板的特化
3. 类型可以是完整的，也可以是不完整的
4. 非内联函数、成员函数、全局变量和静态数据成员在整个程序中只能定义一次
