## Google C++ Style Guide

### Background

C++是Google开源项目中使用的主要语言之一。C++有非常多强大特性，同时这些特性也带来了复杂性，这让代码更容易出bug，也很难阅读和维护。

本编码风格指南的目标是为了管理C++的复杂性，规定了编码时什么该做，什么不该做，这样可以保证代码可维护，同时也能更好地使用语言特性。

*编码风格（style）*，也可以理解为可读性，是Google在使用C++时所遵循的一些约定，或许叫约定更合适。

Google的大部分开源项目都遵循这个标准。

本指南不是C++教程：读者必须熟悉C++才可以。

#### Goals of the Style Guide

本指南的目的有很多，这些目的也是这份指南存在的原因。发布这份指南，可以引起Google社区中的开发者讨论和关注，也可以让开发者们了解为什么要指定这份指南。如果能明白指南中说的每一条规则是什么，也就能知道该规则什么时候可以去掉，或者有什么样的替代方案。

本编码风格指南的目标包括：
1. 应该发挥其应有的作用，不能是摆设...
2. 服务的是代码读者而不是作者...
3. 尽量和已有代码尽量保持一致...
4. 尽量和C++社区保持一致...
5. 避免C++中容易出错的特性...
6. 避免C++中技巧性过高和难以维护的特性...
7. 让开发者认识到指南所覆盖的范围...
8. 在必要时可以被优化...

本指南旨在提供合理限制下的最大编码指导。指南中参考了整个Google C++社区的既定惯例，而不仅仅是某个项目、某些团队和某个人的偏好。要辩证的看待技巧性过高和不常见的特性：没有禁止和允许并不相同。如果不确定可以问领导。

好了现在开始吧~

### C++ Version

本编码风格指南目前适用于C++17，不包括C++20（除designated initializers）。编码风格指南所针对的C++版本会随着时间逐步更新。

不要使用非标准的扩展！

使用较新特性（C++14，C++17）时要考虑可移植性。

### Header Files

通常来说，每个`.cc`文件应该有对应的`.h`文件。某些情况下可以没有，例如单元测试和只包括`main()`的`.cc`文件。

正确使用头文件可以很大的提高代码可读性和性能，同时也会大幅减小代码的尺寸。

下面的规则可以避免使用头文件时会遇到的陷阱。

#### Self-contained Headers

头文件应该是独立的，以`.h`结尾。用于包含的非头文件应该以`.inc`结尾，并且谨慎使用。

所有的头文件都应该是独立的。用户和代码重构工具在包含头文件时不必遵守特定的规则（注：这里是从独立的角度来阐述的，而不是说包含头文件可以不遵守规则）。头文件内应该有哨兵（hearder guards），也应该包含所有必须的头文件。

如果头文件声明了`inline`函数或者模板，那就应该提供定义（也可以通过包含其它的文件来引入定义）。不要把这些定义移动到单独的文件（`-inl.h`）里，那是过去的做法，现在不允许了。如果一个模板的所有实例都只在一个`.cc`文件中，无论是通过[显式](https://en.cppreference.com/w/cpp/language/class_template#Explicit_instantiation)的方式还是因为模板定义在该文件中，那么模板可以定义在该文件中。

只有极少情况下会出现头文件不独立的情况，这种头文件不在文件开头包含，一般是在代码中间。这些文件可以不用添加哨兵，也尽量不要加入其它的依赖。这种文件一般以`.inc`结尾。谨慎使用这类头文件，优先考虑独立的头文件。

#### The `#define` Guard

所有头文件都应该添加`#define`哨兵来避免重复包含，格式遵循`<PROJECT>_<PATH>_<FILE>_H_`。

为了确保哨兵的唯一性，名称应该使用相对于项目根目录的全路径。例如`foo/src/bar/baz.h`内的哨兵应该为：

```cpp
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

// ...

#endif  // FOO_BAR_BAZ_H_
```

#### Include What You Use

只有引用了其它文件中定义的符号时才应该直接包含声明了或者定义了那个符号的头文件。其它情况下不要包含那个文件。

不要依赖传递包含。这样就可以移除不必要的头文件定义。例如，如果`foo.cc`引用了`bar.h`中的符号，即使`foo.h`已经包含了`bar.h`，也应该直接包含`bar.h`。

#### Forward Declarations

避免前向声明，采用直接包含的方式（见上一节）。

**定义**：

前向声明指的是声明但不定义。例如：

```cpp
// In a C++ source file:
class B;
void FuncInB();
extern int variable_in_b;
ABSL_DECLARE_FLAG(flag_in_b);
```

**前向声明的优点**：
- 前向声明节省编译时间
- 前向声明避免重复编译头文件

**前向声明的缺点**：
- 前向声明隐藏了依赖关系，当头文件发生变化时可以避免一些重复编译
- 前向声明使得一些自动化工具难以定位定义了该符号的模块
- 前向声明会给库的升级带来不兼容的问题。前向声明的函数或者模板使得头文件作者不能修改API符号，例如提升整数类型，添加默认模板参数或者迁移命名空间
- `std::`中的前向声明会产生未定义的行为
- 使用前向声明时，很难确定该声明是不是必要的，如果只是简单的讲前向声明更改为`#include`，可能会修改代码行为（注：这一点深有感触），例子如下：

```cpp
// b.h:
struct B {};
struct D : B {};

// good_user.cc:
#include "b.h"
void f(B*);
void f(void*);
void test(D* x) { f(x); }  // Calls f(B*)
```

如果将`#include "b.h"`换为`B`和`D`的前向声明，`test()`将会调用`f(void*)`。

- 前向声明了一个头文件中的多个符号很罗嗦
- 使用前向声明会使代码变慢变复杂

**结论**：

避免前向声明。

#### Inline Functions

只在函数小于10行时添加`inline`。

**定义**：

内联函数是指让编译器将函数调用展开为函数代码。

**内联的优点**：

如果内联函数够小的话，代码会更高效，可以内联访问器和修改器（注：就是访问类成员的函数）和其它短小、性能关键的函数。

**内联的缺点**：

过度使用内联会使程序变慢。内联可能会让代码体积变大，也可能变小。例如内联访问器可以很好的减小代码体积，而内联一个大函数会增大代码体积。在现代处理器上，小体积代码能更好的利用指令缓存。

**结论**：

经验法则证明如果函数超过10行就不要内联。特别注意，析构函数通常比看到的要长！

尽量不要内联包含选择结构的函数，除非选择结构从不执行。

需要注意的是即使声明了内联，编译器也可能不会内联，例如虚函数和递归函数通常不会被内联。不要内联频繁使用的递归函数。如果内联虚函数，一般是将其写在类定义内，可能是为了方便，也可能是为了方便写文档。

#### Names and Order of Includes

头文件的包含顺序为：
1. 相关头文件
2. C系统头文件
3. C++标准库头文件
4. 其它库头文件
5. 项目自定义头文件

包含项目自定义的头文件应该包含文件在项目中的路径，但是不要使用`.`或者`..`。例如包含`google-awesome-project/src/base/logging.h`的方式为：

```cpp
#include "base/logging.h"
```

对于头文件`dir2/foo2.h`，如果实现在`dir/foo.cc`中，测试在`dir/foo_test.cc`中，头文件的顺序应该为：
1. `dir2/foo2.h`
2. 空行
3. `#include <unistd.h>`等
4. 空行
5. `#include <algorithm>`
6. 空行
7. 其它库（注：应该是第三方库）的`.h`文件
8. 空行
9. 本项目中的头文件

用空格区分每一组。

按照上述顺序，如果`dir2/foo2.h`中缺少必要的头文件，`dir/foo.cc`和`dir/foo_test.cc`会编译失败。这个规则保证了错误首先出现在`dir2/foo2.h`的作者上，其它人不会无辜躺枪。

通常`dir/foo.cc`和`dir2/foo2.h`应该在同一目录中（例如`base/basictypes_test.cc`和`base/basictypes.h`），也允许在不同目录。

C++中的C头文件（`c`开头无`.h`后缀）和C头文件可以互相替代，项目中保持一直即可。

每一组中的头文件应该按照字典序排列。老旧项目可能不遵守这个，方便的时候应该修改（注：根据经验，修改可能会引起编译错误，因为头文件互相依赖，并不独立）。

例如，`google-awesome-project/src/foo/internal/fooserver.cc`中的包含顺序应该为：

```cpp
#include "foo/server/fooserver.h"

#include <sys/types.h>
#include <unistd.h>

#include <string>
#include <vector>

#include "base/basictypes.h"
#include "foo/server/bar.h"
#include "third_party/absl/flags/flag.h"
```

**例外情况**：系统相关的头文件有时需要条件编译，这时可以把这些头文件放到后面。同时还要保证条件编译的作用域尽可能的小。例如：

```cpp
#include "foo/public/fooserver.h"

#include "base/port.h"  // For LANG_CXX11.

#ifdef LANG_CXX11
#include <initializer_list>
#endif  // LANG_CXX11
```

### Scoping

#### Namespaces

除少数例外，应该在命名空间中包含代码。命令空间名字应该唯一，也可以包含路径。不要使用`using`指示（例如`using namespace foo`）。不要使用内联命名空间。对于匿名命名空间，参见[Internal Linkage]()。

**定义**：

命名空间将项目中全部的符号分散到了多个独立的作用域中，以很好的避免命名冲突。

**优点**：

命名空间降低了大型项目中符号冲突的问题，也缩短了很多符号的长度。

例如，如果两个不用的项目都定义了`Foo`类型，就可能会造成编译错误或者运行时错误。分别放到两个命名空间`project1::Foo`和`project2::Foo`中可以避免这个问题，同时在各自的命名空间中也可以直接使用`Foo`。

内联命名空间将内部名称暴露在外层作用域中，例如：

```cpp
namespace outer {
inline namespace inner {
  void foo();
}  // namespace inner
}  // namespace outer
```

`outer::inner::foo()`和`outer::foo()`可以互换，内联命名空间主要用于ABI兼容。

**缺点**：

命名空间使得符号名字复杂化。

特别是内联命名空间，它破坏了命名空间的规则。内联命名空间之应该用在版本控制中。

在必须要使用全修饰的符号名称，嵌套过深的命名空间使得名字变得非常长。

**结论**：

按照如下规则使用命名空间：
- 遵循[Namespace Names]()中的命名规则
- 在多行命名控件的`}`后加上注释
- 不要在命名空间中包含`#include`和前向声明，例子如下：

```cpp
// In the .h file
namespace mynamespace {

// All declarations are within the namespace scope.
// Notice the lack of indentation.
class MyClass {
 public:
  // ...
  void Foo();
};

}  // namespace mynamespace
```

```cpp
// In the .cc file
namespace mynamespace {

// Definition of functions is within scope of the namespace.
void MyClass::Foo() {
  // ...
}

}  // namespace mynamespace
```

```cpp
#include "a.h"

ABSL_FLAG(bool, someflag, false, "a flag");

namespace mynamespace {

using ::foo::Bar;

// Code goes against the left margin.

}  // namespace mynamespace
```

- 关于[Protocol Buffer Packages](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated#package)的内容
- 不要声明`std`中的内容，因为不保证可移植性，应该直接包含标准库的头文件。
- 不要使用`using`指示
- 不要在头文件中使用命名空间别名，除非是内部使用的命名空间，因为别名会变成外部链接的，例如：

```cpp
// Shorten access to some commonly used names in .cc files.
namespace baz = ::foo::bar::baz;
```

```cpp
// Shorten access to some commonly used names (in a .h file).
namespace librarian {
namespace impl {  // Internal, not part of the API.
namespace sidetable = ::pipeline_diagnostics::sidetable;
}  // namespace impl

inline void my_inline_function() {
  // namespace alias local to a function (or method).
  namespace baz = ::foo::bar::baz;
  // ...
}
}  // namespace librarian
```

- 不要使用内联命名空间

#### Internal Linkage

如果是`.cc`中内部使用的函数，确保它们是内部链接的，即添加`static`关键字或者将其放入匿名命名空间中。不要在`.h`中引用这些符号。

**定义**：

将符号变为内部链接的方法是将符号放在匿名命名空间中。对于函数和变量，还可以为其添加`static`。通过这些方式可以保证这些符号不会被另一个文件中的代码所引用。即使另一个文件中有同名符号也能保证正常工作。

**结论**：

鼓励在`.cc`中为不暴露的接口使用内部链接。不要在`.h`中使用内部链接。

在编码格式上，匿名命名空间和具名命名空间一致：

```cpp
namespace {
// ...
}  // namespace
```

#### Nonmember, Static Member, and Global Functions

优先将非成员函数放在命名空间中，避免使用全局函数。不要用单独的类型来组织多个静态变量。类的静态方法应该和类本身，或者类的静态数据成员关联。

**优点**:

将非成员函数放在匿名空间中可以避免污染全局命名空间。

**缺点**：

当非成员函数和静态函数访问外部资源，或者他们有很强依赖的时候，将其作为类成员可能更有意义。

**结论**：

有时的确需要定义不和类实例相关联的函数，这时可以定义静态函数或者使用非成员函数。非成员函数不应该依赖外部变量，同时应该定义在命名空间中。与其定义单纯用来组织静态函数的类，不如直接给这些符号一个公共的前缀。

只在单个`.cc`中定义和使用的非成员函数应该是内部链接的。（注：结合前面，这种函数应该是在匿名命名空间中）。

#### Local Variables

尽可能缩小变量的作用域，并在声明时初始化变量。

C++允许在任何地方初始化变量。鼓励在局部作用域中靠近变量第一次使用的位置声明变量，这样可以更方便的找到变量的定义和初值，特别的，应该对变量进行初始化，而不是定义后再赋值，例如：

```cpp
int i;
i = f();      // Bad -- initialization separate from declaration.

int j = g();  // Good -- declaration has initialization.

std::vector<int> v;
v.push_back(1);  // Prefer initializing using brace initialization.
v.push_back(2);

std::vector<int> v = {1, 2};  // Good -- v starts initialized.
```

`if`、`while`和`for`中使用的变量应该定义在子句中，以保证该变量的有效范围只在对应的作用域中：

```cpp
while (const char* p = strchr(str, '/')) str = p + 1;
```

如果变量是对象，那么定义在局部作用域中就会频繁的调用构造函数和析构函数：

```cpp
// Inefficient implementation:
for (int i = 0; i < 1000000; ++i) {
  Foo f;  // My ctor and dtor get called 1000000 times each.
  f.DoSomething(i);
}
```

更好的方式是定义在局部作用域之外：

```cpp
Foo f;  // My ctor and dtor get called once each.
for (int i = 0; i < 1000000; ++i) {
  f.DoSomething(i);
}
```

#### Static and Global Variables

禁止定义具有[静态存储周期](http://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration)的对象，除非析构函数式平凡的，也就是说析构函数不做任何事情，例如析构成员和析构基类。换句话说，如果对象的类型中没有用户定义类型，没有虚函数，且基类也满足这个条件，则认为对象的析构函数式平凡的。函数内定义的静态变量应该使用动态初始化。不鼓励为类的静态变量或者命名空间内的变量使用动态初始化，也就是说某些特定情况下可以使用，例子见下面。

经验法则证明满足后面条件的全局变量可以添加`constexpr`。

**定义**：

每个变量都有存储周期，也就是生命周期。生命周期从初始化开始，一直延续到程序退出的变量具有静态存储周期。这些对象包括：作用域在整个命名空间级别的变量，类的静态变量和函数内的静态变量。函数内的静态变量在第一次执行该语句时被初始化，其余类型均在程序启动时初始化。所有具有静态存储周期的变量在程序退出时被销毁。

**优点**：

全局和静态变量的用户包括：具名常量，辅助数据结构，命令行参数，日志，注册机制和后台基础设施等等。

**缺点**：

动态初始化的，或者具有非平凡析构函数的全局和静态变量会导致难以调试的bug。不同翻译单元中的变量初始化顺序不固定，析构顺序也不固定（除非析构完全的时构造的逆过程）。如果一个静态变量的初始化过程会引用另外一个静态变量，很有可能会引用一个未初始化的变量，也有可能引用了一个已经析构的变量。如果程序启动了没有在退出后销毁的线程，新的线程很有可能访问已经析构了的对象（注：见下一节的`thread_local`）。

**结论**：

##### Decision on destruction

如果析构函数是平凡的，则可以认为析构函数没有真正运行，否则就有可能访问生命周期已经结束的变量。因此，只允许将具有平凡析构函数的对象定义为静态变量。基本类型（例如指针和`int`）具有平凡析构函数，具有平凡析构函数的数组也是为可以平凡析构。同时被`constexpr`修饰的变量也认为时可以平凡析构的。正确的例子：

```cpp
const int kNum = 10;  // Allowed

struct X { int n; };
const X kX[] = {{1}, {2}, {3}};  // Allowed

void foo() {
  static const char* const kMessages[] = {"hello", "world"};  // Allowed
}

// Allowed: constexpr guarantees trivial destructor.
constexpr std::array<int, 3> kArray = {1, 2, 3};
```

错误的例子如下：

```cpp
// bad: non-trivial destructor
const std::string kFoo = "foo";

// Bad for the same reason, even though kBar is a reference (the
// rule also applies to lifetime-extended temporary objects).
const std::string& kBar = StrCat("a", "b", "c");

void bar() {
  // Bad: non-trivial destructor.
  static std::map<int, int> kData = {{1, 0}, {2, 0}, {3, 0}};
}
```

注意引用并不是对象，所以不受上述限制（从析构方面考虑的一些限制），但是不适用于引用动态初始化的对象。有一个例外，就是允许函数内的静态引用，例如`static T& t = *new T`。（注：这一部分的结论应该从静态对象是否会析构，以及析构函数是否是平凡的角度来考虑。）

##### Decision on initialization

对于初始化，除了考虑构造函数的执行问题，还要考虑赋值初始化。下面的例子除了`n`，其余的初始化值都是不确定的。

```cpp
int n = 5;    // Fine
int m = f();  // ? (Depends on f)
Foo x;        // ? (Depends on Foo::Foo)
Bar y = g();  // ? (Depends on g and on Bar::Bar)
```

这里引用C++标准中的常量初始化（constant initialization）概念，也就是指初始化表达式是常量，还有构造函数应该是`constexpr`修饰，例如：

```cpp
struct Foo { constexpr Foo(int) {} };

int n = 5;  // Fine, 5 is a constant expression.
Foo x(2);   // Fine, 2 is a constant expression and the chosen constructor is constexpr.
Foo a[] = { Foo(1), Foo(2), Foo(3) };  // Fine
```

允许常量初始化。常量初始化的具有静态存储周期的变量应该用`constexpr`修饰，或者[ABSL_CONST_INIT](https://github.com/abseil/abseil-cpp/blob/03c1513538584f4a04d666be5eb469e3979febba/absl/base/attributes.h#L540)。非局部的、具有静态存储周期的、没有被`constexpr`修饰变量认为是动态初始化的，代码审查时要重点关注。下面是一个反例：

```cpp
// Some declarations used below.
time_t time(time_t*);      // Not constexpr!
int f();                   // Not constexpr!
struct Bar { Bar() {} };

// Problematic initializations.
time_t m = time(nullptr);  // Initializing expression not a constant expression.
Foo y(f());                // Ditto
Bar b;                     // Chosen constructor Bar::Bar() not constexpr.
```

不鼓励，或者说禁止动态初始化非局部变量，但是如果这些变量之间没有依赖也可以用，例如下面的例子是允许的：

```cpp
int p = getpid();  // Allowed, as long as no other static variable
                   // uses p in its own initialization.
```

可以动态初始胡静态局部变量。

##### Common patterns

- 全局字符串：使用`constexpr`修饰的`string_view`，或者字符数组，或者指向字面值的字符指针。字符串字面值具有静态存储周期，一般情况足够了。
- `map`、`set`和其它动态容器：如果需要静态的，或者固定大小的集合，例如固定大小的查找表，那么不推荐使用标准库的容器，因为它们不是平凡析构的。应该使用平法类型的数组代替这些容器，例如多维数组（实现`int`到其它的映射），或者成对的数组（注：可能说多个不用类型的具有对应关系的数组）。对于小集合，线性查找效率足够高，也可以借助[absl/algorithm/container.h](https://github.com/abseil/abseil-cpp/blob/master/absl/algorithm/container.h)。如果要使用二分查找，定义时就可以排好序。如果一定要使用标准库容器，考虑使用函数内的静态变量（注：应该是因为不会存在初始化和析构的顺序问题）。
- 自定义类型：记得为自定义类型的构造函数添加`constexpr`，以及确保析构函数是平凡的
- 以上都不适用的情况下，定义一个函数内的静态指针或者引用来引用一个堆上的对象，例如`static const auto& impl = *new T(args...);`。

#### `thread_local` Variables

`thread_local`变量的初值必须可以在编译期被计算，或者使用[ABSL_CONST_INIT](https://github.com/abseil/abseil-cpp/blob/03c1513538584f4a04d666be5eb469e3979febba/absl/base/attributes.h#L540)。对于线程局部变量，首选[thread_local](https://en.cppreference.com/w/c/thread/thread_local)的方式。

**定义**：

变量可以为线程定义，例如`thread_local Foo foo = ...;`。这种方式定义的变量有多个副本，每个线程访问的是不同的副本，和上一节的静态和全局变量有些类似。这些变量可以被定义在命名空间和函数内，也可以作为类的静态变量，但是不能是成员变量。

`thread_local`变量的初始化方式和静态变量类似，区别在于要为每个线程初始化一次。所以`thread_local`变量可以定义在函数内，但是定义在其它地方的依然存在初始化顺序问题。

`thread_local`变量在线程终止（注：原文是terminate，可能说的是没有`join()`）之前不会被销毁，所以没有析构顺序的问题。

**优点**：
- 线程局部变量只会被当前线程访问，所以没有竞争问题
- `thread_local`是唯一的标准库支持的定义线程局部变量的方式

**缺点**：
- 访问`thread_local`变量会有额外的开销
- 除了保证了线程安全，`thread_local`和全局变量几乎一样
- `thread_local`变量有存储开销
- `thread_local`不能用于类的静态成员函数
- `thread_local`可能不如编译器扩展效率高

**结论**：

函数内的`thread_local`变量没有安全问题，可以不加限制的使用。可以通过下面的方式实现类或者命名空间级别的安全`thread_local`变量：

```cpp
Foo& MyThreadLocalFoo() {
  thread_local Foo result = ComplicatedInitialization();
  return result;
}
```

类或者命名空间级别的`thread_local`变量的初值必须可以在编译期被计算，使用[ABSL_CONST_INIT](https://github.com/abseil/abseil-cpp/blob/03c1513538584f4a04d666be5eb469e3979febba/absl/base/attributes.h#L540)可以强制实现，如`ABSL_CONST_INIT thread_local Foo foo = ...;`。

如果要使用线程局部变量，优先选择`thread_local`。

### Classes

类是C++代码的主要组成部分，这一节列出了对于类，应该做什么和不应该做什么。

#### Doing Work in Constructors

避免在构造函数中调用虚函数。如果构造函数不能报告错误，构造过程中尽量不要出错。

**定义**：

构造函数的函数体中可以进行任何初始化。

**优点**：
- 不需要考虑类对象是否有效
- 如果构造函数可以完全初始化一个对象，可以当作`const`对象，且可以用在标准库的容器和算法中

**缺点**：
- 构造函数中的虚函数调用不会分发到派生类中
- 没有办法在构造函数中报告错误，注意异常是被禁止的[]()
- 继续上一点，如果构造失败，需要通过类似`bool IsValid()`的函数报告错误，但是经常会忘记
- 不能取构造函数的地址

**结论**：

不应该在构造函数中调用虚函数。构造函数出错时，可以直接终结程序，或者使用工厂模式。避免对没有其它状态的对象调用`Init()`方法。

#### Implicit Conversions

不要定义隐式转换。在类型转换运算符和单参数构造函数前加`explict`。

**定义**：

隐式转换是指在需要一种类型的变量时，可以用另一种类型的变量代替。

除了语言内置转换外，用户也可以自定义类型转换，例如将转换为`bool`可以写为`operator bool()`。接受其它类型的单参数构造函数也可以视为隐式类型转换。

`explict`可以保证在只有目标类型确定的情况下上述两种用户定义的转换才有效。这个规则对列表初始化依然有效：

```cpp
class Foo {
  explicit Foo(int x, double y);
  ...
};

void Func(Foo f);

Func({42, 3.14});  // Error
```

准确来说上面的代码不属于隐式转换的类型，但是编译器也会按照同样的规则处理。

**优点**：
- 隐式转换省去了强制类型转换的麻烦
- 隐式转换可以代替重载，例如接受`string_view`类型的函数的实参可以是`std::string`或者`const char*`
- 列表初始化的方式更加精确

**缺点**：
- 隐式转换隐藏了类型不匹配带来的bug
- 隐式转换的情况下，很难找到真正调用的函数
- 单参数构造函数很有可能被用于隐式转换，这可能和初衷相违背
- 如果两个类型可以互相转换，可能会出现模糊调用
- 列表初始化同样有上述问题，特别是单参数的列表

**结论**：

除了拷贝构造函数和移动构造函数外，类型转换运算符和单参数构造函数应该添加`explict`关键字。

多参数构造函数可以忽略`explict`，以`std::initializer_list`为单参数的函数也可以省略`explict`。

#### Copyable and Movable Types

定义`public`方法的类应该考虑对象的拷贝特性和移动特性。

**定义**：

可移动的类型指的是可以通过临时量初始化的类型。

可拷贝类型指的是可以通过同类型对象初始化的类型，同时还要保证被拷贝的对象不发生变化。`std::unique_ptr<int>`是可移动类型，而不是可拷贝类型，因为源对象被拷贝后就失效了。`int`和`string`是可移动类型，也是可拷贝类型。

对于用户自定义类型，拷贝语义由拷贝构造函数和拷贝赋值运算符决定；移动语义由移动构造函数和移动赋值运算符决定，如果不存在的话，则也有拷贝构造函数和拷贝赋值运算符决定。

拷贝和移动操作会被编译器隐式调用。

**优点**：

可拷贝和可移动的对象使得API更加的简单，因为可以通过通过参数传入和用返回值返回。同时也不用像指针和引用一样考虑对象的所有权，生命周期等问题。还能隔离函数内外的变量，使得代码更易理解和维护，更有助于编译器进行优化。这些对象可以用于传值的API中，例如容器算法等。

拷贝和移动相关的函数可以自定义，也可以由编译器生成（这样可以保证所有对象都被操作）。拷贝和移动构造函数也可以避免堆上操作。

移动操作可以实现所有权的转义，某些情况下使得代码更简单。

**缺点**：

某些类型不应该被拷贝，如单例、作用域相关的类型和强耦合性类型（`std::mutex`）。可拷贝的基类可能会带导致[对象切片（object slicing）](https://en.wikipedia.org/wiki/Object_slicing)的错误。

拷贝构造是隐式的，可能会新手混淆（因为某些语言默认传引用）。拷贝还会有性能开销。

**结论**：

类应该特别处理拷贝和移动操作。

可拷贝的类型应该定义拷贝操作，只可移动类型应该定义移动操作，不可拷贝和移动类型应该删除拷贝和移动操作。可拷贝类型也应该定义移动操作来提升性能。可以显式的声明或者删除拷贝构造函数、移动构造函数、拷贝赋值运算符和移动赋值运算符。例子如下：

```cpp
class Copyable {
 public:
  Copyable(const Copyable& other) = default;
  Copyable& operator=(const Copyable& other) = default;

  // The implicit move operations are suppressed by the declarations above.
  // You may explicitly declare move operations to support efficient moves.
};

class MoveOnly {
 public:
  MoveOnly(MoveOnly&& other) = default;
  MoveOnly& operator=(MoveOnly&& other) = default;

  // The copy operations are implicitly deleted, but you can
  // spell that out explicitly if you want:
  MoveOnly(const MoveOnly&) = delete;
  MoveOnly& operator=(const MoveOnly&) = delete;
};

class NotCopyableOrMovable {
 public:
  // Not copyable or movable
  NotCopyableOrMovable(const NotCopyableOrMovable&) = delete;
  NotCopyableOrMovable& operator=(const NotCopyableOrMovable&)
      = delete;

  // The move operations are implicitly disabled, but you can
  // spell that out explicitly if you want:
  NotCopyableOrMovable(NotCopyableOrMovable&&) = delete;
  NotCopyableOrMovable& operator=(NotCopyableOrMovable&&)
      = delete;
};
```

仅在显而易见的情况下，可以省略一些声明：
- 如果没有`private`成员，那么类型的可拷贝性和可移动性是是由`public`成员决定的
- 如果基类不可拷贝和移动，派生类也不行。对于用于接口的基类，可拷贝性和可移动性是不确定的，在派生类中也不确定
- 构造函数和赋值运算符应该成对出现

如果类型的拷贝语义和移动语义不确定，那么就认为类型是不可拷贝和不可移动的。对于可拷贝类型，移动操作只是为了提升性能，但是定义移动操作很有可能会引入bug，所以要慎重！如果类型提供拷贝操作，记得检查默认实现的正确性。

为了避免类型被切片，考虑将基类定义为抽象的：将构造函数和析构函数定义为`protected`的，添加纯虚函数。尽量避免从非抽象类派生。

#### Structs vs. Classes

仅在用于保存数据的情况下使用`struct`，其余情况下使用`class`。

虽然在C++中，`struct`和`class`几乎一样，但是本指南中二者区别对待。

`struct`只用来保存数据，且所有成员都是`public`的。成员之间不能有关联，因为直接访问某个成员会破坏这种关联。构造函数，析构函数和辅助的方法可以定义在`struct`中，但是不能为成员变量之间强加关联。

如果类型需要更多的函数和内在的不变性，使用`class`。在不确定的情况下，使用`class`。

为了和STL保持一致，无状态类型应该使用`struct`，例如类型萃取，模板元编程和仿函数。

最后提示一下，`struct`和`class`中成员变量应该按照不同的方式命名，见[变量命名]()。

#### Structs vs. Pairs and Tuples

如果需要组织在一起的类型有明确的含义，优先使用`struct`而不是`pair`和`tuple`。

虽然`pair`和`tuple`可以少写一些代码，但是`.first`、`.second`和`std::get<X>`的方式会降低可读性。虽然C++14开始支持`std::get<Type>`，但是变量名字还是比变量类性更可读。

在不具有明确含义的情况下，可以使用`pair`和`tuple`。

#### Inheritance

组合比继承更好。总是使用`public`继承。

**定义**：

派生类包括基类定义的所有数据和方法。接口继承是从抽象基类（没有状态或方法的基类）继承，其它都是实现继承。

**优点**：

实现继承通过复用基类代码来减少代码大小。由于继承是编译时声明，所以编译器可以检测错误。接口继承强制类暴露特定API，在这种情况下，如果没有定义抽象基类中的方法，编译器会报错。

**缺点**：

实现继承中，派生类的代码分布在基类和派生类之间，会造成阅读上的困难。派生类不能重写非虚函数，因此不能更改实现。

多重继承会带来性能开销（从单继承到多继承的性能下降通常会比从普通函数到虚函数的性能下降更大），并且因为它有可能导致“菱形”继承。

**结论**：

所有的继承都应该是`public`的。如果要使用`private`继承，应该将基类定义为成员。可以使用`final`来使类不能被继承。

不要过度使用实现继承。复合有时更合适。在“是（is-a）”的情况下使用继承：如果`Bar`是一种`Foo`，则`Bar`是`Foo`的派生类。

限制定义`protected`成员的数量，这也就限制了可以从派生类访问的对象的数量。根据[Access Control]()中的规则，数据成员应该是`private`的。

在重写虚函数（也包括虚析构函数）时，使用`override`或者`final`关键字。重写虚函数时不要使用`virtual`，因为如果被标记为`override`或者`final`的函数没有覆盖基类中的虚函数，会有编译错误。这两个关键字可以很方便的表明该函数是否是虚函数。

允许多重继承，但是不鼓励多重实现继承。

#### Operator Overloading

清晰明确的使用运算符重载。

**定义**：

在一个参数是用户自定义类型的情况下，C++允许重载通过`operator`重载内置运算符的含义。`operator`也可以用于定义新的字面值和类型转换。

**优点**：

运算符重载可以使用户定义类型更容易被理解。重载运算符应该和原有运算符含义相同。

用户自定义字面值是创建用户自定义类型对象的简单方式。

**缺点**：
- 想要正确的重载运算符并不简单，如果不当会引入bug
- 过多的使用运算符重载会使代码变得混乱，特别是当某些运算符含义发生变化时
- 运算符重载同样有着函数重载中相同的问题
- 高级代码分析工具才能分析出运算符重载
- 如果传递给运算符的参数类型不对，可能会调用错误的函数
- 重载`&&`、`||`和`,`会带来语义上的变化
- 如果两个文件中提供了同一运算符的不同定义，可能会出现运行时错误
- 用户自定义字面值可能会带来混淆，比如通过用户自定义字面值简化`std::string_view("Hello World")`为`"Hello World"sv`
- 用户自定义字面值创建的类型不能被命名空间修饰，所以要使用被禁用的`using`指示

**结论**：

重载的运算符应该和原有含义保持一致。

只为自己的类型重载运算符，准确的是，在同一个`.h`、`.cc`和命名空间中定义全部重载的运算符，以减少出现多重定义的情况。尽量不要将重载运算符和模板产生关联，因为要考虑的内容太多了。如果重载了一个运算符，就要重载相关的所有运算符，例如重载`<`后，也要重载`>`。

对于二元运算符，优先实现为非成员函数。如果是成员函数，则只有二元运算符的右参数可能会发生类型转化，左右参数行为不一致会出现问题。

在该重载运算符的地方应该重载，比如不要用`Equals()`代替`==`、不要用`CopyFrom()`代替`=`、不要用`PrintTo()`代替`<<`。如果类型本身不具备某种运算符下的含义，就不要重载该运算符。可以使用仿函数代替，比如用`std::set`存储时，可以用仿函数而不必定义`<`。

不要重载`&&`、`||`和`,`，也不要重载一元运算符。不要引入用户自定义字面量，也不要使用用户自定义字面量。

类型转换运算符内容见[Implicit Conversions]()，赋值运算符的内容见[Copyable and Movable Types]()，流运算符的内容见[Streams]()。

#### Access Control

类的数据成员应该是`private`的，除非是常量。

在[Google Test](https://github.com/google/googletest)中，`.cc`中定义的测试类的数据成员可以是`protected`。如果测试类不在`.cc`中，应该是`private`的。

#### Declaration Order

相互关联的成员声明放在一起，总是先定义`public`成员。

类的定义中，先是`public`，然后是`protected`，最后是`private`。没有的为空。

同种访问类型的成员中，将相互关联的成员声明放在一起，按照下面的顺序：
1. 类型和类型别名，例如`typedef`、`using`、`enum`、嵌套类和`friend`类型
2. 静态常量
3. 工厂函数
4. 构造函数和赋值运算符
5. 析构函数
6. 其他函数，静态方法、成员函数和友元函数
7. 数据成员，静态变量和成员变量

### Functions

#### Inputs and Outputs

函数的输出参数可以通过参数和返回值传递。

优先使用返回值，因为可读性更高，性能也更高。

优先返回值，其次返回引用。在不空的情况下，避免返回指针。

参数可以输入参数，也可以是输出参数，或者既是输入参数也是输出参数。不可省略的输入参数应该传值，或者常量引用，而不可省略的输出参数应该传引用。用`std::optional`包装可省略的输入参数。如果不可省略的实参可能是引用，应该用`const`指针。可省略的输出参数没有`const`。
>Generally, use std::optional to represent optional by-value inputs, and use a const pointer when the non-optional form would have used a reference. Use non-const pointers to represent optional outputs and optional input/output parameters.

避免定义必须使用常量引用的函数，因为常量引用绑定的是临时量。为了避免生命周期的问题，可以通过拷贝，或者使用`const`指针。

关于参数顺序，输入参数在前，输出参数在后，即使新添加参数也保持这个规则。变参函数顺序随意。

#### Write Short Functions

函数应该尽可能的短而精悍。

有时长函数是必须的，所以没有函数长度的硬性规定。如果函数超过40行，可以考虑拆分。

即使函数现在正常工作，但是添加新功能后可能会引入bug，所以还是要尽可能的短，因为短的更易读，也更方便测试。

不要担心修改过长的函数。如果现在的函数已经很难读，且很难复用，就应该考虑拆分。

#### Function Overloading

只有在调用时就能明白在调用哪个重载函数的情况下才使用重载函数（也包括构造函数）。

**定义**：

如果重载函数分别以`const std::string&`和`const char*`为参数，可以直接使用`std::string_view`版本。

**优点**：

重载允许同名函数使用不同的参数。重载是模板所必需的。

基于`const`和引用的不同重载函数会带来性能上的提升。

**缺点**：

复杂的重载牵扯到C++内部复杂的重载解析规则。如果派生类重载了基类的函数，也很难确定会调用对象。

**结论**：

当不同的重载函数之间没有明显的语义差别时才使用重载。如果可以用简单的语言描述重载函数之间的区别，那么重载是有意义的。

#### Default Arguments

当默认参数始终具有相同值时，允许在非虚函数中使用默认参数。如果默认参数带来的可读性优点没有超过下面所述的缺点，则应该使用重载。

**优点**：

使用默认参数的原因是大部分情况下的参数都可以使用默认值，仅在少数情况下需要传递默认参数。如果用重载代替，则需要为少数情况定义额外的函数。同时相比于重载，默认参数也很好的区分了必要参数和可选参数。

**缺点**：

默认参数是另一种形式的重载，重载的缺点默认参数同样具备。

虚函数中的默认参数值和对象类型相关，不能保证所有的默认值都一样。

默认参数也是参数，每次调用都会发生传参。理解代码时，一般都假设每次调用时默认参数的值是相同的。

在使用函数指针的情况下，默认参数会导致混淆，因为函数指针可以指向同类型的任意函数。

**结论**：

禁止在虚函数中使用默认参数，同样在其它可能出现默认值不一致的地方也禁止使用默认参数。

其它情况下可以使用默认参数。如果不确定，则应该使用重载。

#### Trailing Return Type Syntax

只有在返回类型不明确的情况下使用尾置返回类型。

**定义**：

C+允许两种方式的函数声明：

```cpp
int foo(int x);

auto foo(int x) -> int;
```

对于普通函数来说这个不影响，影响的是类成员函数和函数模板。

**优点**：

尾置返回类型是lambda表达式中声明返回类型的唯一方式。虽然有时编译器可以推断lambda表达式的返回类型，但是显式的指出返回类型可以提升可读性。

在模板函数中只能使用位置返回类型：

```cpp
template <typename T, typename U>
auto add(T t, U u) -> decltype(t + u);
```

否则会很麻烦：

```cpp
template <typename T, typename U>
decltype(declval<T&>() + declval<U&>()) add(T t, U u);
```

**缺点**：

尾置返回类型在其它语言中没有，可能会造成理解困难（这太牵强了）。

原有的函数声明方式大量存在，为了保持一致性应该少用尾置返回类型。

**结论**：

大部分情况下不要用尾置返回类型，仅在lambda表达式中使用。对于函数模板的问题，最好是将返回类型也作为模板参数。麻烦的写法不推荐，见[Template Metaprogramming]()。

### Google-Specific Magic

本节介绍几种技巧和工具。

#### Ownership and Smart Pointers

对于动态分配的对象，尽量保持单一所有权。转义所有权时，使用智能指针。

**定义**：

讨论所有权是为了管理动态分配的对象。首次在堆上分配对象的变量获得对象的所有权，并要保证在不需要时应该释放资源。所有权可以共享，这时拥有所有权最久的变量负责资源的释放。即使所有权不能被共享，也应该能够被转移。

只能指针通过重载`*`和`->`模拟指针的行为。`std::unique_ptr`表达的是不能共享所有权，对象在`std::unique_ptr`退出作用域时删除。`std::unique_ptr`不能被拷贝，但是可以移动。`std::shared_ptr`表达的是可以共享所有权，`std::shared_ptr`可以拷贝，拥有所有权最久的`std::shared_ptr`负责释放资源。

**优点**：
- 抛开所有权很难管理堆上对象
- 转移所有权代价小
- 转移所有权免去了共享生命周期的问题
- 上述两种智能指针可以提升可读性
- 智能指针简化了所有权的管理
- 使用`std::shared_ptr`可以避免深拷贝

**缺点**：
- 所有权的转义需要通过指针，即使是智能指针，也比使用普通变量复杂
- 值语义带来的性能下降被夸大了，所以综合看所有权带来的有时并不明显
- 转移所有权的API使得调用者使用单一内存管理模型
- 智能指针释放内存的位置并不清楚
- `std::unique_ptr`实现的所有权转移利用了移动语义，不易于理解
- 共享所有权需要带来额外的运行时开销
- 共享所有权有时会出现循环引用，导致资源不能释放
- 智能指针并不能解决普通指针的问题

**结论**：

如果要动态分配对象，尽量不要转义所有权。如果需要使用动态分配的对象，考虑使用指针和引用。如果要转义所有权，显式的使用`std::unique_ptr`：

```cpp
std::unique_ptr<Foo> FooFactory();
void FooConsumer(std::unique_ptr<Foo> ptr);
```

尽量不要设计共享所有权的模型。一般这么做是出于性能问题，但是不要想当然，并且保证共享所有权的对象是常量`std::shared_ptr<const Foo>`。

不要使用`std::auto_ptr`，应该使用`std::unique_ptr`。

#### cpplint

[cpplint.py](https://raw.githubusercontent.com/google/styleguide/gh-pages/cpplint/cpplint.py)。

### Other C++ Features

#### Rvalue References

仅在下述情况中使用右值引用。

**定义**：

右值引用是只能绑定到临时量的引用。普通函数中用`&&`表示右值引用。

模板函数中的`&&`表明转发引用。

**优点**：
- 移动构造函数可以将拷贝操作替换为移动操作，与其拷贝大量数据，移动操作只需要指针的修改
- 右值引用可以实现可移动但是不可拷贝的类型
- `std::move`可以使得标准库中其它组件性能更高，例如`std::unique_ptr`
- 模板中的转发引用可以实现完美转发（转发参数的引用属性）

**缺点**：
- 右值引用还没有完全推广，转发引用的语法令人费解
- 右值引用经常被误用
- 右值引用反直觉，一般来说调用函数的实参在函数返回后依然有效

**结论**：

仅在下列情况使用引用：
- 定义移动操作时
- 会改变`this`的函数中
- 支持完美转发
- 定义函数的右值引用参数版本

#### Friends

允许使用友元类型和友元函数。

友元应该定义在同一文件中。友元主要用在构造中（注：可能指的是工厂模式）。测试类可以是被测试类的友元。

虽然友元打破了封装的边界，但是有些情况下比`public`方法更好。更多的情况是使用`public`方法进行交互。

#### Exceptions

不使用异常。

**优点**：
- 异常可能允许更高级别的代码决定如何处理深度嵌套函数中“基本不可能发生”错误
- 现代语言中都有异常机制
- 有些第三方库使用了异常，如果关闭会很那集成
- 异常可以让构造函数报错
- 测试框架中包含异常

**缺点**：
- 使用`throw`抛出异常时，必须每一个外层函数中考虑异常
- 异常会扰乱程序的执行流，不便于理解，也不便于调试
- 异常安全的代码需要RAII的支持，还需要在适当的地方处理所有的异常，这样做不值当
- 打开异常会使二进制程序变大
- 异常可能被滥用（注：感觉这几点中只有最后一个有道理）

**结论**：

在新项目中使用异常的好处大于成本，而在已有项目中会影响大部分代码。如果异常传播的范围太广，超出了项目的范围，也会带来集成问题。由于Google的大部分代码都不处理异常，所以很难在新代码中启用异常。

鉴于Google现有代码中几乎不使用异常，在新代码中使用异常的会有很大的代价。并且Google认为异常的替代方案，例如错误码和断言不比异常差。

不适用异常不是固执已见，而是从实际出发的，特别是兼容问题。如果从头再来，或许会允许异常。

`std::exception_ptr`和`std::nested_exception`也不允许。

Windows代码是个[例外]()。

#### noexcept

在必要的地方使用`noexcept`。

**定义**：

`noexcept`用于指出函数是否会抛出异常。如果`noexcept`函数抛出了异常，程序会退出。

`noexcept`在编译器进行检查。

**优点**：
- 为移动操作声明`noexcept`可以提高效率，例如`std::vector<T>::resize()`
- 启用异常的情况下，`noexcept`可以用于编译优化

**缺点**：
- 如果按照前述禁用了异常，`noexcept`很可能不对
- 去掉`noexcept`有代价

**结论**：

`noexcept`可以带来性能上的优化，因为一旦抛出了异常就可以结束程序了。同样`noexcept`也会为移动操作带来性能上的提升。

如果异常被完全禁用，可以考虑无条件`noexcept`，否则使用带简单条件的`noexcept`，也就是在某些情况下允许函数抛出异常，比如使用`std::is_nothrow_move_constructible`检查移动操作等。一般情况下，异常都是由于内存分配失败引起的，而且如果分配失败了，程序都不选择恢复它。即使是其它异常，也应该保证接口的简单性，可以单纯的假设不支持某种异常（注：更多内容见原文）。

#### Run-Time Type Information (RTTI)

避免使用RTTI。

**定义**：

RTTI是指用`typeid`和`dynamic_cast`可以在运行时获取类型信息。

**优点**：

RTTI的替代方案需要修改甚至重新设计类的继承层次，但是可行性很低。

RTTI在单元测试中很有用。

在使用抽象对象时，RTTI很有用：

```cpp
bool Base::Equal(Base* other) = 0;
bool Derived::Equal(Base* other) {
  Derived* that = dynamic_cast<Derived*>(other);
  if (that == nullptr)
    return false;
}
```

**缺点**：

如果经常需要在运行时获取类型信息，这说明代码设计有问题，特别是继承层次有问题。

如果使用RTTI，代码的执行逻辑会取决于所传入的类型，降低可维护性。

**结论**：

RTTI经常被滥用，在正式代码中使用时应该三思。在单元测试中可以使用。如果代码确实需要根据类型来执行，这里提供两个备选方案：
- 考虑是否可以用虚函数解决
- 如果代码不属于对象，考虑双重分派解决方案，例如Visitor模式

如果代码逻辑可以保证基类的指针和引用确实是派生类，可以使用`dynamic_cast`，不过此时也可以用`static_cast`。

如果代码执行非常依赖于所传入的类型，说明实现思路有问题：

```cpp
if (typeid(*data) == typeid(D1)) {
  // ...
} else if (typeid(*data) == typeid(D2)) {
  // ...
} else if (typeid(*data) == typeid(D3)) {
  //...
}
```

如果加入了新的类型，上述代码会很复杂。

不要手动实现RTTI的替代方法。

#### Casting

应该使用`static_cast`，不要使用C语言形式的强制类型转换（转换到`void`除外）。只有当`T`是类时，才使用`T(x)`。

**定义**：

C++引入了新的强制类型转换。

**优点**：

C语言形式的转换存在歧义，既可以表示合法的类型转换（`(int)3.5`），也表示强制类型转换（`(int)"hello"`）。C++的列表初始化和强制转换方式避免了这种歧义。

**缺点**：

语法太长。

**结论**：

如果需要显式的强制类型转换，使用C++的方式：
- 转换整数时使用初始化列表，例如`int64_t{x}`，因为肯定不会向下转换
- 使用`absl::implicit_cast`实现派生类向基类的准换
- 使用`static_cast`代替C语言形式的类型转换，包括基类和派生类之间的互相转换
- 使用`const_cast`移除`const`
- 使用`reinterpret_cast`转换指针
- 使用`absl::bit_cast`转换二进制表示的数据

关于`dynamic_cast`见上一节。

#### Streams

在合适的地方使用流，并尽量简单。只在输出用户可见值时重载和使用`<<`。

**定义**：

流是C++提供的标准IO方式，由`<iostream>`定义。

**优点**：

流提供的`<<`和`>>`比`printf`更加统一，且更方便。

`std::cin`、`std::cout`、`std::cerr`和`std::clog`支持控制台。

**缺点**：
- 对流内部的改变会作用到其后所有的操作上
- 流将控制和实际数据混在了一起，重载也能会导致不清晰的调用
- 流对国际化的本地化的支持存在问题
- 流API太复杂
- 重载`<<`会延长编译时间

**结论**：

典型使用流的情况是输出一些给开发者看的内容。如果是为了记录日志，应该用专用的日志库。

避免使用流接收用户的输入，或者接收不可信的数据。对于国际化和本地化，应该使用相应的库。

如果使用流，避免使用会改变流内部状态的API，比如修改进制、对齐方式和精度，因为这会影响到后续的操作。

仅在对象会输出可读信息时才重载`<<`，避免使用`<<`暴露对象内部细节。如果要打印调试信息，用专门的函数，比如`DebugString()`。

#### Preincrement and Predecrement

在不需要后置自增语义的情况下，使用前置自增（也同样适用于自减）.

**定义**：

在需要自增自减变量时，考虑是否需要使用自增或者自减之后的值。

**优点**：

后置自增返回自增之前的值，虽然可以压缩代码，但是不便于理解。前置自增可读性更高，也没有效率上的损失，甚至由于不需要返回旧值，性能可能更高。

**缺点**：

传统C语言使用后置自增。

**结论**：

使用前置版本，除非代码必须使用旧值。

#### Use of const

在函数定义中适时的使用`const`。有时`constexpr`比`const`更合适。

**定义**：

通过`const`可以使得变量不可修改。类的方法中的`const`表明该方法不会更改类的内部状态。

**优点**：

`const`有助于理解变量的使用方式，从而可以更好的编写代码。

**缺点**：

`const`参数会将常量属性一直传递下去。当遇到不接收`const`参数的函数时，必须要使用`const_cast`才行。

**结论**：

强烈推荐适时地在函数参数、方法和全局变量添加`const`属性。`const`划分出了可写的变量，有助于编写线程安全的代码。关于`const`要注意这几点：
- 如果函数不会修改指针和引用，应该添加`const`，例如`const T&`和`const T*`
- 传值的参数不添加`const`
- 除非方法会修改类的内部状态，否则为方法添加`const`

##### Where to put the const

推荐使用`const int* foo`而不是`int* const foo`的方式。`const`可以立即为形容词，`int`时名词，在英语中形容词应该在名词前面。

#### Use of constexpr

使用`constexpr`定义纯常量和使用常量初始化的对象。

**定义**：

纯常量表示可以在编译器确定的变量，这些变量在编译时就能确定值。也可以为函数添加`constexpr`来初始化变量。

**优点**：

`constexpr`可以定义更加复杂的常量，例如通过表达式和函数初始化的变量。

**缺点**：

将变量过早定义为`constexpr`可能会给二次开发带来麻烦。如果将函数定义为`constexpr`，则函数返回值必须可以在编译时计算，可能会引入一些难以理解的代码。

**结论**：

使用`constexpr`来定义纯常量和一些必要的函数。不要给复杂的函数添加`constexpr`。不要使用`constexpr`强制内联。

#### Integer Types

一般来说用`int`，或者`<cstdint>`中的整型。如果`int`装不下，使用`int64_t`。即使使用了更长的类型，也要考虑回绕问题。

**定义**：

C++没有规定`int`的位宽，一般默认`short`占16位，`int`占32位，`long`占32位，`long long`占64位

**优点**：

写法统一。

**缺点**：

不同体系结构上位宽不一样。

**结论**：

如果需要考虑位宽，使用`<cstdint>`中定义的类型。不要使用除`int`外的内置类型。适当情况下可以使用`size_t`和`ptrdiff_t`。

使用`int`时，一般都不会太大，比如用在循环控制变量。假设`int`占32位。

对于可能的大整数，使用`int64_t`。

不应该使用无符号版本（例如`uint32_t`），除非用作掩码。不要假设无符号整型永远为正！

在表示大小时，保证整型宽度可以容纳所有可能大小。

特别注意整形转换问题。

##### On Unsigned Integers

无符号整型用作掩码或者模数。由于历史的原因，C++使用无符号整型表示容器大小，但是委员会的很多人认为这是一个错误，但是目前无法修复。无符号算数会产生溢出，和有符号整数运算机制不同，错误很难发现。

混合使用无符号整型和有符号整型会导致很多问题。所以本指南建议：尽量使用迭代器，不要混用无符号整型和有符号整型，尽量避免无符号整型。更不要假设无符号整型是非负数。

#### 64-bit Portability

在打印、比较和结构体对齐时要考虑64位的问题。

- `printf()`在64位平台上有问题，所以尽量避免传递整型（特别是表明位宽的整型）到`printf`相关的函数（注：原因见原文）
- `sizeof(void *) != sizeof(int)`
- 对于在磁盘中存储的结构体，考虑整型的对其问题
- 使用`{}`创建64位整型常量

#### Preprocessor Macros

避免定义宏，特别是在头文件中。用`inline`、`enum`和`const`代替宏。宏应该以项目为前缀。不要用宏定义C++的API。

宏会扩展，所以和源文件中的代码不一样。还会带来未定义的行为。

用宏定义C++ API会带来很多问题，如果报错的话，错误信息会很复杂；重构工具更新宏也很麻烦；所以不允许将宏用于定义API：

```cpp
class WOMBAT_TYPE(Foo) {
  // ...

 public:
  EXPAND_PUBLIC_WOMBAT_API(Foo)

  EXPAND_WOMBAT_COMPARISONS(Foo, ==, <)
};
```

好在C++不像C一样非常依赖宏。宏可以用`inline`和`const`代替，长变量名可以定义引用，也不要用宏来条件编译代码。

当然宏也具备独特的功能，例如拼接字符串。使用宏前，考虑下是否有替代方式。

如果必须要用，遵循下面的规则：
- 不要在`.h`中定义宏
- 仅在使用前`#define`，使用后马上`#undef`
- 不要`#undef`现有宏，定义新的
- 尽量不要使用扩展到不平衡的C++结构的宏
- 不要用`##`生成名字

不要将头文件中的宏暴露到头文件外（通过`#undef`取消定义）。如果要使用全局宏，要加入大写的项目名称前缀。

#### 0 and nullptr/NULL

空指针使用`nullptr`，空字符使用`\0`。

#### sizeof

优先使用`sizeof(varname)`而不是`sizeof(type)`。

如果修改了变量类型，`sizeof(varname)`依然正确。不和变量关联的情况下使用`sizeof(type)`。正确的例子：

```cpp
MyStruct data;
memset(&data, 0, sizeof(data));

if (raw_size < sizeof(int)) {
  LOG(ERROR) << "compressed record not big enough for count: " << raw_size;
  return false;
}
```

错误的例子：

```cpp
memset(&data, 0, sizeof(MyStruct));
```

#### Type Deduction (including auto)

使用类型推导是为了让不熟悉项目的人更好的理解，而不是为了避免写复杂的名称。

**定义**：

C++编译器可以推导类型的情况包括：

[函数模板实参推导](https://en.cppreference.com/w/cpp/language/template_argument_deduction)，调用函数模板时可以不用指出实参的类型：

```cpp
template <typename T>
void f(T t);

f(0);  // Invokes f<int>(0)
```

[`auto`变量推导](https://en.cppreference.com/w/cpp/language/auto)，用`auto`替代变量的类型（auto也可以被`const`修饰）：

```cpp
auto a = 42;  // a is an int
auto& b = a;  // b is an int&
auto c = b;   // c is an int
auto d{42};   // d is an int, not a std::initializer_list<int>
```

[函数返回类型推导](https://en.cppreference.com/w/cpp/language/function#Return_type_deduction)，`auto`和`decltype(auto)`可以用于函数的返回类型：

```cpp
auto f() { return 0; }  // The return type of f is int
```

lambda表达式使用同样的方式推导，但是不用写`auto`。[Trailing Return Type Syntax]()中提到的`auto`不用于推导，只是用来占位的。

[泛型lambda表达式](https://isocpp.org/wiki/faq/cpp14-language#generic-lambdas)，lambda表达式的参数类型可以是`auto`，这样的lambda表达式其实是一个函数模板：

```cpp
// Sort `vec` in decreasing order
std::sort(vec.begin(), vec.end(), [](auto lhs, auto rhs) { return lhs > rhs; });
```

[lambda表达式的捕获列表](https://isocpp.org/wiki/faq/cpp14-language#lambda-captures)，捕获列表中可以定义新变量，但是不用声明类型，此时编译器会进行推导：

```cpp
[x = 42, y = "foo"] { ... }  // x is an int, and y is a const char*
```

[类模板实参推导](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction)，见[Class Template Argument Deduction]()。

[结构化绑定](https://en.cppreference.com/w/cpp/language/structured_binding)，使用`auto`声明`tuple`、`struct`和`array`时，可以声明字段的名字，而不声明变量的名字，这种情况下不能指出变量的类型，只能用`auto`：

```cpp
auto [iter, success] = my_map.insert({key, value});
if (!success) {
  iter->second = value;
}
```

此时`auto`也可以和`const`、`&`和`&&`组合使用，但是这些修饰作用于匿名变量，而不是每一个绑定。

**优点**：

- C++的名字会很长，特别是模板和命名空间的名字，使用`auto`会方便很多
- 名字在小范围内多次重复时，会降低可读性，用`auto`代替更好
- 有时让编译器推导类型会更安全

**缺点**：

显式的写明类型让代码更清晰，例如从下面的代码中不能很方便的知道`foo`和`i`的类型：

```cpp
auto foo = x.add_foo();
auto i = y.Find(key);
```

类型推导和引用关联时，程序员必须清楚推导结果中是否包含引用。

当类型推导用于接口中时，改变实参就会改变所调用的函数。

**结论**：

仅在能让代码可读性更好更安全的情况下使用类型推导，而不要为了不写长名字而使用类型推导。使用类型推导时，应该考虑到：虽然类型信息对开发者来说是可有可无的，但是对于刚接触代码的人来说类型信息是必要的。

下面看具体情况。

##### Function template argument deduction

函数模板实参推导是允许的，这是模板的默认使用方式。

##### Local variable type deduction

对于局部变量，使用类型推导以简化过长的名字可以让代码更清晰：

```cpp
std::unique_ptr<WidgetWithBellsAndWhistles> widget_ptr =
    std::make_unique<WidgetWithBellsAndWhistles>(arg1, arg2);
absl::flat_hash_map<std::string,
                    std::unique_ptr<WidgetWithBellsAndWhistles>>::const_iterator
    it = my_map_.find(key);
std::array<int, 6> numbers = {4, 8, 15, 16, 23, 42};
```

```cpp
auto widget_ptr = std::make_unique<WidgetWithBellsAndWhistles>(arg1, arg2);
auto it = my_map_.find(key);
std::array numbers = {4, 8, 15, 16, 23, 42};
```

上例中，很明显`it`表示迭代器。如果容器的类型和键的类型没有值的类型重要，那就可以这么写：

```cpp
if (auto it = my_map_.find(key); it != my_map_.end()) {
  WidgetWithBellsAndWhistles& widget = *it->second;
  // Do stuff with `widget`
}
```

如果`auto`可以工作，就不要使用`decltype(auto)`。

##### Return type deduction

仅在函数很短，`return`语句很少的情况下使用类型推导，否则很难一眼看出函数的返回类型。同时不要在接口中使用类型推导，因为此时不得不将实现也暴露给用户。

##### Parameter type deduction

泛型lambda表达式中的参数类型推导和调用者相关，所以可能不会很方便的确定实参的类型。

##### Lambda init captures

见[Lambda Expressions]()。

##### Structured bindings

结构化绑定可以很好的提升可读性，比如`pair`和`tuple`中的变量没有名字，但是结构化绑定可以赋予变量有意义的名字。不过话说回来，本指南不推荐使用`pair`和`tuple`，所以结构化绑定只应该用在强制返回`pair`和`tuple`的地方。

如果结构化绑定的是结构体，有时可能会为具体的业务提供更加清晰的名字，但是新的名字和结构体中字段的对应关系就会有些不清楚，所以建议这么写：

```cpp
auto [/*field_name1=*/bound_name1, /*field_name2=*/bound_name2] = ...
```

#### Class Template Argument Deduction

仅对类模板所支持的类型使用类模板实参推导。

**定义**：

[类模板实参推导](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction)，编译器可以推导类模板中实参的类型，例如：

```cpp
std::array a = {1, 2, 3};  // `a` is a std::array<int, 3>
```

这个代码能工作是因为有如下的推导指引（将多个参数的`array`推导为类型第1个参数类型，数量为参数总量的`array`）：

```cpp
namespace std {
template <class T, class... U>
array(T, U...) -> std::array<T, 1 + sizeof...(U)>;
}
```

主模板（相对于特化的类模板）中的构造函数也作为推导指引。

根据推导指引推导的类型作为推导的结果。

**优点**：

类模板实参推导可以省去写类的名字。

**缺点**：

构造函数提供的隐式推导指引可能带来错误的结果，但是添加显式推导指引也会影响依赖于隐式推导指引的代码。

类模板实参推导和`auto`有着同样的缺点，因为它们都从变量的初值中来推导变量的类型。

**结论**：

除非类模板明确指明了支持的实参类型，也就是类作者提供了推导指引，否则不要用类模板实参推导。

#### Designated Initializers

以和C++20兼容的方式使用指定初始化。

**定义**：

[Designated initializers](https://en.cppreference.com/w/cpp/language/aggregate_initialization#Designated_initializers)可以指定初始化简单结构中的变量：

```cpp
struct Point {
  float x = 0.0;
  float y = 0.0;
  float z = 0.0;
};

Point p = {
  .x = 1.0,
  .y = 2.0,
  // z will be 0.0
};
```

未被指定的字段会按照传统的方式被初始化。

**优点**：

指定初始化可以很方便的初始化简单的结构体。

**缺点**：

指定初始化时C的语法，之前一直作为C++的扩展，在C++20中才被标准化。

标准C++的指定初始化比C语言更严格，要求指定初始化时变量的顺序和结构体变量定义的顺序一致。

**结论**：

以兼容C++20的方式使用指定初始化，也就是指定初始化的顺序和结构体定义的顺序一致。

#### Lambda Expressions

适当的使用lambda表达式。如果lambda表达式使用会超过当前作用域，显示的捕获当前作用域中的变量。

**定义**：

lambda表达式用于创建匿名函数，以当作参数传入：

```cpp
std::sort(v.begin(), v.end(), [](int x, int y) {
  return Weight(x) < Weight(y);
});
```

lambda表达式也允许捕获当前作用域中的变量，可以通过名字显式捕获，也可以隐式捕获：

```cpp
int weight = 3;
int sum = 0;
// Captures `weight` by value and `sum` by reference.
std::for_each(v.begin(), v.end(), [weight, &sum](int x) {
  sum += weight * x;
});
```

隐式捕获会捕获所有在函数体中用到的变量（包括`this`指针）：

```cpp
const std::vector<int> lookup_table = ...;
std::vector<int> indices = ...;
// Captures `lookup_table` by reference, sorts `indices` by the value
// of the associated element in `lookup_table`.
std::sort(indices.begin(), indices.end(), [&](int a, int b) {
  return lookup_table[a] < lookup_table[b];
});
```

捕获变量时还可以提供显式的初始化，这就可以以值的方式捕获仅可移动的对象：

```cpp
std::unique_ptr<Foo> foo = ...;
[foo = std::move(foo)] () {
  ...
}
```

上面的捕获方式实际上不捕获作用域中的任何变量，只是定义了一个为lambda表达式使用的变量（`foo`的类型由编译器推导，见[Type Deduction (including auto)]()）：

```cpp
[foo = std::vector<int>({1, 2, 3})] () {
  ...
}
```

**优点**：
- lambda表达式定义的函数对象可以很方便的用于标准库提供的算法
- 适当的使用隐式捕获可以减少代码中的冗余
- lambda表达式、`std::function`和`std::bind`组合可以作为一种很方便的回调机制

**缺点**：
- 当lambda表达式超出捕获的变量的作用域时，捕获的变量很可能已经失效了
- 默认的值捕获方式会带来空悬指针的问题，因为值捕获的指针不是深拷贝
- 捕获会声明新的变量，但是没有指出变量的类型，降低可读性
- 初始化捕获中会用到类型推导，所以也会有类型推导带来的缺点
- 超长的匿名函数使得代码很难理解（为什么我在这里想到了[微软SEAL中的CKKS？？？](https://github.com/microsoft/SEAL/blob/main/native/src/seal/evaluator.cpp)）

**结论**：
- 按照下面的描述使用lambda表达式
- 如果lambda表达式会超出当前作用域，使用显式捕获。不要这样用：

```cpp
{
  Foo foo;
  ...
  executor->Schedule([&] { Frobnicate(foo); })
  ...
}
// BAD! The fact that the lambda makes use of a reference to `foo` and
// possibly `this` (if `Frobnicate` is a member function) may not be
// apparent on a cursory inspection. If the lambda is invoked after
// the function returns, that would be bad, because both `foo`
// and the enclosing object could have been destroyed.
```

应该这样用：

```cpp
{
  Foo foo;
  ...
  executor->Schedule([&foo] { Frobnicate(foo); })
  ...
}
// BETTER - The compile will fail if `Frobnicate` is a member
// function, and it's clearer that `foo` is dangerously captured by
// reference.
```

- 仅在lambda表达式的生命周期比捕获的变量短时，可以使用隐式引用捕获
- 当为较短的lambda表达式捕获变量时，只有当要捕获的变量一目了然并且不会捕获`this`时可以使用默认值捕获。当非静态成员函数中的lambda表达式引用了非静态的成员变量时，会隐式捕获`this`，这是应该引用捕获。当使用值捕获时，lambda表达式的函数体不要太长
- 不要在捕获是初始化捕获的值，不如在函数体中定义变量，甚至不如用普通函数
- 关于参数和返回类型，参见[Type Deduction (including auto)]()

#### Template Metaprogramming

避免模板元编程。

**定义**：

模板元编程是指利用C++模板实例化的图灵完备机制进行编译时计算。

**优点**：

模板元编程让函数接口更加灵活，保证类型安全，也能更高效。如果没有Google Test，`std::tuple`、`std::function`和Boost不能工作。

**缺点**：

一般只有非常有经验的人才理解元编程，很多程序员都不理解，所以元编程会大大降低可读性，也会使得代码难以调试。

模板元编程会使得编译器报奇奇怪怪的错误。

模板元编程的代码很难使用工具进行大规模的重构。模板代码会大量展开，很难确定哪些是必要的。很多重构工具是基于抽象语法树的（AST）的，所以只能理解模板展开后的代码。

**结论**：

元编程使得语言用起来更方便，但是也会让人自作聪明。最好在一些低级别的组件中使用，这样可以把维护的负担分摊到使用的时候（不太明白什么意思）。

使用元编程之前应该考虑：团队当中是否有一半以上的人能理解？如果退出了项目，这部分代码是否还可以维护？非C++程序员调用该代码时，是否可以理解编译错误？如果在项目中使用了递归实例化、类型列表、元函数、表达式模板、SFINAE、通过`sizeof`进行重载解析等技巧，说明已经用过了。

即使用了元编程，也应该少用，并且将复杂度降到最低。同时应该将元编程隐藏到实现细节中，这样至少头文件是可读的。要对使用元编程的代码进行注释，比如应该如何使用，生成的代码是什么样子。此外还要注意，因为元编程的编译错误很难理解，所以也应该注释。

#### Boost

仅允许使用Boost中部分库。

**定义**：

[Boost](https://www.boost.org/)。

**优点**：

Boost库是高质量的C++代码，弥补了标准库中很多缺失的功能。

**缺点**：

某些Boost库鼓励使用元编程和函数式编程。

**结论**：

Google只允许使用部分Boost库，具体见原文[Boost](https://google.github.io/styleguide/cppguide.html#Boost)。

#### Other C++ Features

与Boost一样，现代C++中有一些阻碍可读性特性。

**结论**：

除了指南中描述的，下面的禁用：
- `<ratio>`
- `<cfenv>`
- `<filesystem>`，因为有bug

#### Nonstandard Extensions

不允许使用非标准的扩展功能。

**定义**：

编译器提供的非标准功能，例如GCC的`__attribute__`和内联汇编等等。

**优点**：
- 非标准扩展提供了C++不提供的功能
- 使用非标准扩展可以提升性能

**缺点**：
- 非标准扩展不跨平台
- 非标准扩展容易导致令人迷惑的行为
- 非标准扩展需要额外的前置知识

**结论**：
不要使用非标准扩展。

#### Aliases

外部链接别名用户也可以使用，所以应该有注释。

**定义**：

创建符号别名的方法有：

```cpp
typedef Foo Bar;
using Bar = Foo;
using other_namespace::Foo;
```

在新的代码中，应该使用`using`而不是`typedef`。

头文件中定义的别名是接口的一部分，而函数中定义的别名不是，`private`中的也不是，内部命名空间中的也不是。`.cc`中的别名也不是外部链接。本节仅讨论外部链接的别名。

**优点**：
- 别名缩短了长名字，提升可读性
- 修改外部链接别名的原始名称依然可以保证兼容

**缺点**：
- 外部链接的别名使模块变复杂
- 用户代码可能会对别名做一些潜在的假设
- 有时创建一个仅用于外部链接的别名会带来额外的维护成本
- 别名可能导致命名冲突
- 如果别名不常见，可能会降低可读性
- 不能保证同一类型只有一个别名

**结论**：

不要只是为了缩短一些名称而创建外部链接的别名，而应该是因为用户确实需要。

创建外部链接的别名后，应该为其写文档，内容包括原始类型是否会更改，是否还有其它潜在的兼容性假设。这样用户就能知道是否可以用这个别名完全代替原始类型，也会给用户代码的实现保留一定的自由度。

不要在命名空间中定义外部链接的别名，见[Namespaces]()。

别名的注释可以这么写：

```cpp
namespace mynamespace {
// Used to store field measurements. DataPoint may change from Bar* to some internal type.
// Client code should treat it as an opaque pointer.
using DataPoint = ::foo::Bar*;

// A set of measurements. Just an alias for user convenience.
using TimeSeries = std::unordered_set<DataPoint, std::hash<DataPoint>, DataPointComparator>;
}  // namespace mynamespace
```

下面的注释没有写明别名的用途，并且很多不应该让用户使用：

```cpp
namespace mynamespace {
// Bad: none of these say how they should be used.
using DataPoint = ::foo::Bar*;
using ::std::unordered_set;  // Bad: just for local convenience
using ::std::hash;           // Bad: just for local convenience
typedef unordered_set<DataPoint, hash<DataPoint>, DataPointComparator> TimeSeries;
}  // namespace mynamespace
```

如果是内部链接的别名，注释可以省略些：

```cpp
// In a .cc file
using ::foo::Bar;
```

### Inclusive Language

在代码中，使用包容的语言，不要使用可能存在攻击性的词汇，例如`master`和`slave`，`blacklist`和`whitelist`，`redline`等。不要使用可能存在性别歧视的词汇。

### Naming

代码中最应该保持一致的就是命名风格。程序员的大脑存在思维定势，符号的名字应该能准确的反应出所指代是类型，变量，函数，常量还是宏等。

命名可以很随意，但是保持一致很重要。

#### General Naming Rules

命名应该保证可读性，也包括其它团队的人。

命名应该反应对象的用途。不要担心名字太长，能让别人理解更重要。尽可能少的使用简称。不要删掉单词中的某些字母来简化。根据经验，在Wiki上有简称的词条可以使用简称。一般来说，名字的详细程度和该符号的可见性相关，比如对于只有5行的函数，可以使用`n`这种简单的变量：

```cpp
class MyClass {
 public:
  int CountFooErrors(const std::vector<Foo>& foos) {
    int n = 0;  // Clear meaning given limited scope and context
    for (const auto& foo : foos) {
      // ...
      ++n;
    }
    return n;
  }
  void DoSomethingImportant() {
    std::string fqdn = ...;  // Well-known abbreviation for Fully Qualified Domain Name
  }
 private:
  const int kMaxAllowedConnections = ...;  // Clear meaning within context
};
```

下面的命名就显得冗余了：

```cpp
class MyClass {
 public:
  int CountFooErrors(const std::vector<Foo>& foos) {
    int total_number_of_foo_errors = 0;  // Overly verbose given limited scope and context
    for (int foo_index = 0; foo_index < foos.size(); ++foo_index) {  // Use idiomatic `i`
      // ...
      ++total_number_of_foo_errors;
    }
    return total_number_of_foo_errors;
  }
  void DoSomethingImportant() {
    int cstmr_id = ...;  // Deletes internal letters
  }
 private:
  const int kNum = ...;  // Unclear meaning within broad scope
};
```

可以使用熟悉的简称，例如`i`作为循环变量，`T`作为模板形参。

单词是最小的命名单元，包括简称和俚语。对于大小写混用的名字，比如[驼峰命名](https://en.wikipedia.org/wiki/Camel_case)，会用所有大写字母作为缩写，但是命名中只有首字母大写，比如写为`StartRpc()`而不是`StartRPC()`。

类型模板参数见[Type Names]()，非类型模板参数见[Variable Names]()。

#### File Names

文件名应该小写，用下划线或者连字符分隔，如果没有规定，优先用连字符，例如：
- `my_useful_class.cc`
- `my-useful-class.cc`
- `myusefulclass.cc`
- `myusefulclass_test.cc`

源文件后缀为`.cc`，头文件后缀为`.h`，非独立头文件见[Self-contained Headers]()。

文件名字不要和已有文件冲突。

文件名应该具体，比如应该用`http_server_logs.h`而不是`logs.h`。头文件和源文件应该匹配。

#### Type Names

类型名首字母大写。对于多个单词组成的类名，每个单词的首字母大小，中间没有下划线，例如`MyExcitingClass`和`MyExcitingEnum`。

这条规则适用于`class`、`struct`、类型别名、`enum`和类型模板参数，例如：

```cpp
// classes and structs
class UrlTable { ...
class UrlTableTester { ...
struct UrlTableProperties { ...

// typedefs
typedef hash_map<UrlTableProperties *, std::string> PropertiesMap;

// using aliases
using PropertiesMap = hash_map<UrlTableProperties *, std::string>;

// enums
enum class UrlTableError { ...
```

#### Variable Names

变量名和成员变量名一律小写，使用下划线分割单词，类（不包括结构体）成员变量后加单独的下划线，例如：`a_local_variable`、`a_struct_data_member`和`a_class_data_member_`。

##### 通用命名规则

```cpp
std::string table_name;  // OK - lowercase with underscore.

std::string tableName;   // Bad - mixed case.
```

##### 类成员变量

无论是静态变量还是普通成员变量，都按照普通变量命名，并在尾部添加下划线：

```cpp
class TableInfo {
 private:
  std::string table_name_;  // OK - underscore at end.
  static Pool<TableInfo>* pool_;  // OK.
};
```

##### 结构体成员变量

无论是静态变量还是普通成员变量，都按照普通变量命名，注意尾部没有下划线：

```cpp
struct UrlTableProperties {
  std::string name;
  int num_entries;
  static Pool<UrlTableProperties>* pool;
};
```

#### Constant Names

`const`或者`constexpr`变量添加前缀`k`，命名方式参考类型命名。如果大小写混用的方式不能表达含义，再添加下划线：

```cpp
const int kDaysInAWeek = 7;
const int kAndroid8_0_0 = 24;  // Android 8.0.0
```

注意具有静态存储周期的变量（静态和全局变量）都应该这么命名（注：这一条和前面的变量命名方式相冲突，应该只适用于全局变量）。

#### Function Names

普通函数命名大小写混用。

函数命名的方式和类型名一样：首字母大写。对于多个单词组成的类名，每个单词的首字母大小，中间没有下划线：

```cpp
AddTableEntry()
DeleteUrl()
OpenFileOrDie()
```

暴露到外部的类和命名空间级别的常量应该按照函数命名。

类成员的访问器和修改器可以按照变量的方式命名，例如`int count()`和`void set_count(int count)`。

#### Namespace Names

命名空间名字小写，用下划线分隔。

顶层命名空间的名字应该为项目名，或者团队名字。

命名空间和变量命名方法一致，命名空间中的代码不需要提及命名空间的名字。

避免嵌套的命名空间和熟知的名字冲突。不要创建任何嵌套的`std`命名空间，尽量避免使用可能冲突的名字，不要嵌套过深的命名空间。

对于内部命名空间，最好加上文件名字以避免冲突。

#### Enumerator Names

枚举类型视为常量：

```cpp
enum class UrlTableError {
  kOk = 0,
  kOutOfMemory,
  kMalformedInput,
};
```

不要将枚举当作宏：

```cpp
enum class AlternateUrlTableError {
  OK = 0,
  OUT_OF_MEMORY = 1,
  MALFORMED_INPUT = 2,
};
```

#### Macro Names

根据[Preprocessor Macros]()，最好不要定义宏。如果定义，全部大写，并用下划线分隔，例如：

```cpp
#define ROUND(x) ...
#define PI_ROUNDED 3.0
```

#### Exceptions to Naming Rules

如果新定义的内容和标准库中内容类似，模拟标准库中的名字，例如`bigopen()`、`uint`、`bigpos`、`sparse_hash_map`和`LONGLONG_MAX`。

### Comments

注释是使代码可读的关键。本节内容说明了注释应该写什么，写到哪里。尽管注释非常重要，但是好的代码是自说明的。好的命名比注释更有意义。

#### Comment Style

`//`和`/**/`都可以，保持一致即可。

`//`更常用。

#### File Comments

文件开头应该有许可证信息。

如果文件声明、实现、或者测试了已经注释的内容，就没有必要写注释。其它情况文件应该有注释。

##### Legal Notice and Author Line

每个文件都应该有许可证信息。

如果对具有作者行的文件进行了重大更改，请考虑删除作者行。新文件通常不应包含版权声明或作者行。

##### File Contents

如果`.h`文件定义了多个抽象类，文件注释应该描述抽象类之间的关系，1到2行足够了。每个类的注释属于每个类，不属于文件注释。

不要在`.h`和`.cc`中注释重复的内容。

#### Class Comments

每一个不能自解释的`class`和`struct`都应该通过注释来说明用途和使用方法：

```cpp
// Iterates over the contents of a GargantuanTable.
// Example:
//    std::unique_ptr<GargantuanTableIterator> iter = table->NewIterator();
//    for (iter->Seek("foo"); !iter->done(); iter->Next()) {
//      process(iter->key(), iter->value());
//    }
class GargantuanTableIterator {
  // ...
};
```

类的注释应该阐明要怎么使用类，以及使用过程中的注意事项。特别要在注释中写出类所基于的假设。如果类可以被多个线程同时访问，要特别注明多线程环境下的注意事项。

类文档可以提供一小段代码来更好的展示类的用途。

关于类的使用方法的注释应该和接口定义在一起（`.h`中），关于类实现的注释应该和实现代码在一起（`.cc`中）。

#### Function Comments

声明部分的注释指出函数的用途，定义部分的注释重点关注函数的操作。

##### Function Declarations

每一个函数前都应该通过注释表明函数的功能和使用方法，只有非常简单的情况下可以省略。`.cc`文件中定义的`private`方法也应该有注释。函数注释应该用第三人称，使用动词短语，比如用Opens the file而不是Open the file（注：看来是要注意时态问题）。通常来说，声明中的注释不会描述函数是如何实现的，这些注释应该在函数实现的注释中。

声明处的注释可以包括：
- 输入输出参数
- 对于成员函数，是否会保存参数的引用或者指针
- 指针参数是否可以为空
- 对于输出参数，函数返回时其中的值
- 性能相关的使用问题

例子如下：

```cpp
// Returns an iterator for this table, positioned at the first entry
// lexically greater than or equal to `start_word`. If there is no
// such entry, returns a null pointer. The client must not use the
// iterator after the underlying GargantuanTable has been destroyed.
//
// This method is equivalent to:
//    std::unique_ptr<Iterator> iter = table->NewIterator();
//    iter->Seek(start_word);
//    return iter;
std::unique_ptr<Iterator> GetIterator(absl::string_view start_word) const;
```

记住不要注释重复和显而易见的内容。

派生类中的虚函数声明注释应该关注当前函数，不用过多提及基类中的函数。多数情况下不需要为派生类中的虚函数声明提供注释。

构造函数和析构函数的注释不能只写初始化或者销毁对象，而应该解释每一个参数，以及析构函数清理哪些东西。一般来说析构函数声明不需要注释。

##### Function Definitions

如果函数的实现用到了某种技巧，应该有注释。比如解释为什么要用这个技巧，而不用朴素的方法；比如为什么函数的前半部分需要锁，而后半部分不需要。

不要重复函数声明中的注释。可以简单重复下函数的在做什么，但是重点应该在实现上。

#### Variable Comments

一般来说，变量名字就是最好的注释。只在某些特定的情况下需要单独写注释。

##### Class Data Members

注释中应该指出成员变量之间的关系。

如果变量可以去特殊值，要在注释中写明特殊值的含义：

```cpp
private:
 // Used to bounds-check table accesses. -1 means
 // that we don't yet know how many entries the table has.
 int num_total_entries_;
```

##### Global Variables

每一个全局变量都应该有注释，注释中写明作用，以及为什么要作为全局变量：

```cpp
// The total number of test cases that we run through in this regression test.
const int kNumTestCases = 6;
```

#### Implementation Comments

##### Explanatory Comments

如果实现用到了某种技巧，应该有注释。复杂的实现也应该有注释。

##### Function Argument Comments

如果函数实参的意义不明确：
- 如果实参数是常量，并且多处使用这个常量，应该为其定义一个常量
- 尝试将`bool`类型参数更改为枚举类型
- 如果参数是一些配置选项，考虑传递一个对象来表示所有的选项，好处是通过名字引用的同时，可以减少参数数量，也方便扩展
- 用变量代替复杂的表达式
- 最后再给实参写注释

错误的例子：

```cpp
// What are these arguments?
const DecimalNumber product = CalculateProduct(values, 7, false, nullptr);
```

正确的例子：

```cpp
ProductOptions options;
options.set_precision_decimals(7);
options.set_use_cache(ProductOptions::kDontUseCache);
const DecimalNumber product =
    CalculateProduct(values, options, /*completion_callback=*/nullptr);
```

##### Don'ts

再次强调注释中不要写显而易见的东西，应该写为什么这么做。更好的方式是使得代码可以自我解释。

错误的例子：

```cpp
// Find the element in the vector.  <-- Bad: obvious!
if (std::find(v.begin(), v.end(), element) != v.end()) {
  Process(element);
}
```

改进后如下：

```cpp
// Process "element" unless it was already processed.
if (std::find(v.begin(), v.end(), element) != v.end()) {
  Process(element);
}
```

如果代码写的好，就不需要注释：

```cpp
if (!IsAlreadyProcessed(element)) {
  Process(element);
}
```

#### Punctuation, Spelling, and Grammar

注意注释中的标点、拼写和语法问题。

注释应该和文章一样规范。完整的句子比短句更容易理解。

尽管代码审查的过程中不会指出标点的问题，但是还是应该保证可读性。

#### TODO Comments

在临时的代码和解决方案或者不完美的方法添加`TODO`注释。

`TODO`应该全大写，后面写明作者、联系方式和问题等，这是为了日后更好的解决问题。当然添加`TODO`注释的人不一定是解决问题的人，所以写自己无所谓：

```cpp
// TODO(kl@gmail.com): Use a "*" here for concatenation operator.
// TODO(Zeke) change this to use relations.
// TODO(bug 12345): remove the "Last visitors" feature.
```

`TODO`中可以明确指出满足某些条件该做什么。

### Formatting

编码风格和格式化本可以随意，但是如果开发这遵循相同的模式，项目会更好维护。有些条款可能不会被人所接受，或者习惯起来还需要花一些时间，但是为了更好的写作，让所有开发者遵循相同的编码风格是非常有必要的。

#### Line Length

每行做多80字字符。

这条确实有争议，但是现有代码都是这么做的，所以还是要保持一致。

**优点**：

赞同的人认为强迫调整代码编辑器大小是很粗鲁的。有些人习惯于一屏显示多个代码，所以屏幕没有空间来容纳更多的字符。80就是传统的宽度。

**缺点**：

长一点的代码有利于阅读。行宽80是1960年的标准，现在有足够宽的显示器了。

**结论**：

80就是最大值。

如果：
- 换行会破坏可读性，例如命令或者URL；
- 过长的字符串字面值，例如包含URL，或者命令的帮助页面等，总之就是换行会降低可读性。除了测试代码，这种字符串字面值应该靠近文件开头定义；
- `#include`；
- 头文件哨兵；
- `using`声明；

那么一行可以超过80.

#### Non-ASCII Characters

尽量不用非ASCII码，如果用的话用UTF-8编码。

用户界面内容不应该在文件中硬编码，即使是英语，所以非ASCII字符应该是很少见的。有几种情况可以使用非ASCII字符，例如解析非ASCII分割符，单元测试，但是编码要用UTF-8。

也可以使用十六进制编码表示，例如`"\xEF\xBB\xBF"`或者`u8"\uFEFF"`。

`u8`前缀保证了字符串字面值会用UTF-8编码。如果字符串字面值中包含非ASCII字符，不要使用`u8`前缀，这可能导致解码错误。

不要使用`char16_t`和`char32_t`类型。只有在用Windows API的时候才可以使用`wchar_t`类型。

#### Spaces vs. Tabs

只用空格，不用制表符，缩进2空格（-_-）。

#### Function Declarations and Definitions

返回类型和函数名在同一行，如果放得下，参数也在同一行，例如：

```cpp
ReturnType ClassName::FunctionName(Type par_name1, Type par_name2) {
  DoSomething();
}
```

如果放不下，就这样：

```cpp
ReturnType ClassName::ReallyLongFunctionName(Type par_name1, Type par_name2,
                                             Type par_name3) {
  DoSomething();
}
```

如果第一个参数都放不下，就这样：

```cpp
ReturnType LongClassName::ReallyReallyReallyLongFunctionName(
    Type par_name1,  // 4 space indent
    Type par_name2,
    Type par_name3) {
  DoSomething();  // 2 space indent
}
```

有几点要注意：
- 名字要起好
- 必要时可以忽略参数名
- 如果返回类型和函数名一行放不下，将函数名放到新的一行
- 如果上一条成立，函数名前不缩进
- `(`和函数名在同一行
- 函数名和`(`之间没有空格
- `(`和参数之间没有空格
- `{`在函数头的最后一行，不新起一行
- `}`单独在函数的最后一行，或者和`{`在同一行
- `)`和`{`之间应该有空格
- 如果参数有多行，每行左对齐
- 默认缩进2空格（-_-）
- 换行的参数缩进4个空格（注：对应上面最后一个情况）

省略参数名的例子如下：

```cpp
class Foo {
 public:
  Foo(const Foo&) = delete;
  Foo& operator=(const Foo&) = delete;
};
```

不显而易见的省略要通过注释省略：

```cpp
class Shape {
 public:
  virtual void Rotate(double radians) = 0;
};

class Circle : public Shape {
 public:
  void Rotate(double radians) override;
};

void Circle::Rotate(double /*radians*/) {}
```

```cpp
// Bad - if someone wants to implement later, it's not clear what the
// variable means.
void Circle::Rotate(double) {}
```

属性和扩展为属性的宏应该在函数定义最前面（奇怪这里为啥有空格？）：

```cpp
  ABSL_ATTRIBUTE_NOINLINE void ExpensiveFunction();
  [[nodiscard]] bool IsOk();
```

#### Lambda Expressions

参数，函数体按照上述方式格式化，捕获列表按照逗号分隔的列表格式化。

引用捕获时，`&`和变量名之间没有空格。

短小的lambda表达式可以作为内联函数（注：下面的例子中，`return`缩进了2个空格）：

```cpp
std::set<int> to_remove = {7, 8, 9};
std::vector<int> digits = {3, 9, 1, 8, 4, 7, 1};
digits.erase(std::remove_if(digits.begin(), digits.end(), [&to_remove](int i) {
               return to_remove.find(i) != to_remove.end();
             }),
             digits.end());
```

#### Floating-point Literals

浮点变量必须有小数点，小数点两侧都要有数字，即使用`e`表示。这可以避免和整型混淆。浮点变量的初值可以写为整型，但是此时并不是整型。错误的例子：

```cpp
float f = 1.f;
long double ld = -.5L;
double d = 1248e6;
```

正确的例子：

```cpp
float f = 1.0f;
float f2 = 1;   // Also OK
long double ld = -0.5L;
double d = 1248.0e6;
```


#### Function Calls

如果一行放得下就写一行，如果放不下，就在新行写第一个参数，相对于上一行缩进4个空格，后面每行都是4个空格的缩进，同时尽可能保证调用的行最少。

一行放得下的情况：

```cpp
bool result = DoSomething(argument1, argument2, argument3);
```

如果一行放不下，可以将从第二个参数开始换行，新行的参数参照第一行的参数左对齐

```cpp
bool result = DoSomething(averyveryveryverylongargument1,
                          argument2, argument3);
```

也可以将第一个参数新起一行，参数的第一行相对于上一行缩进4个空格：

```cpp
if (...) {
  if (...) {
    bool result = DoSomething(
        argument1, argument2,  // 4 space indent
        argument3, argument4);
  }
```

不影响可读性的前提下，尽量将参数放在同一行以减少行数。虽然一行一个参数更便于修改，但是可读性比可修改性优先级更高。

如果一行多个参数降低可读性（一般是由于参数是个表达式），应该定义一个变量存储表达式的值：

```cpp
int my_heuristic = scores[x] * y + bases[x];
bool result = DoSomething(my_heuristic, x, y, z);
```

或者将这种参数单独放在一行，并加上注释：

```cpp
bool result = DoSomething(scores[x] * y + bases[x],  // Score heuristic.
                          x, y, z);
```

如果参数在单独行更可读，那就独占一行。总之就是可读性至上！

如果参数默认分组，可以根据分组进行换行：

```cpp
// Transform the widget by a 3x3 matrix.
my_widget.Transform(x1, x2, x3,
                    y1, y2, y3,
                    z1, z2, z3);
```

#### Braced Initializer List Format

像格式化实参一样格式化`{}`里面的内容。

如果`{}`跟着一个类型名或者变量名，将`{}`视为`()`来进行格式化，如果前面没有名字，认为是0长度名字：

```cpp
// Examples of braced init list on a single line.
return {foo, bar};
functioncall({foo, bar});
std::pair<int, int> p{foo, bar};

// When you have to wrap.
SomeFunction(
    {"assume a zero-length name before {"},
    some_other_function_parameter);
SomeType variable{
    some, other, values,
    {"assume a zero-length name before {"},
    SomeOtherType{
        "Very long string requiring the surrounding breaks.",
        some, other, values},
    SomeOtherType{"Slightly shorter string",
                  some, other, values}};
SomeType variable{
    "This is too long to fit all in one line"};
MyType m = {  // Here, you could also break before {.
    superlongvariablename1,
    superlongvariablename2,
    {short, interior, list},
    {interiorwrappinglist,
     interiorwrappinglist2}};
```

#### Conditionals

在`if`（也包括`else if`）和`(`之间添加空格，也在`)`和`{`之间添加空格，但是判断条件和`()`之间的内容无空格。如果判断条件中存在初始化语句，可以在初始化语句的`;`后添加空格或者换行。错误的例子如下：

```cpp
if(condition) {              // Bad - space missing after IF
if ( condition ) {           // Bad - space between the parentheses and the condition
if (condition){              // Bad - space missing before {
if(condition){               // Doubly bad

if (int a = f();a == 10) {   // Bad - space missing after the semicolon
```

总是在条件执行的语句中使用`{}`。在`{`之后换行，`}`单独一行。`else if`（也包括`else`）和`}`同行，例如：

```cpp
if (condition) {                   // no spaces inside parentheses, space before brace
  DoOneThing();                    // two space indent
  DoAnotherThing();
} else if (int a = f(); a != 3) {  // closing brace on new line, else on same line
  DoAThirdThing(a);
} else {
  DoNothing();
}
```

出于历史原因，如果只有`if`，并且整个`if`语句可以写为一行或者两行的情况下，可以不使用`{}`。在写为一行的情况下，在`)`和条件执行的语句之间添加空格；在写为两行的情况下，在`)`后换行，例子如下：

```cpp
if (x == kFoo) return new Foo();

if (x == kBar)
  return new Bar(arg1, arg2, arg3);

if (x == kQuz) { return new Quz(1, 2, 3); }
```

只有在`if`后的语句足够简单的情况下才能省略`{}`。对于复杂的判断条件，或者条件执行的语句过于复杂，就应该使用`{}`。

最后给出一些不允许的例子：

```cpp
// Bad - IF statement with ELSE is missing braces
if (x) DoThis();
else DoThat();

// Bad - IF statement with ELSE does not have braces everywhere
if (condition)
  foo;
else {
  bar;
}

// Bad - IF statement is too long to omit braces
if (condition)
  // Comment
  DoSomething();

// Bad - IF statement is too long to omit braces
if (condition1 &&
    condition2)
  DoSomething();
```

#### Loops and Switch Statements

`switch`中的每一种情况可以用`{}`。应该为`switch`中的fall through添加注释。如果循环中只有一条语句，可以省略`{}`。空循环应该用`{}`或者`continue`。

`switch`中`case`后的`{}`是可选的，如果有，应该是下面的样子：

```cpp
switch (var) {
  case 0: {  // 2 space indent
    ...      // 4 space indent
    break;
  }
  case 1: {
    // ...
    break;
  }
  default: {
    assert(false);
  }
}
```

如果`switch`中的变量不是枚举类型，应该添加`default`。如果`default`后的语句不应该被执行，应该视为错误，例子同上。

Fall through应该使用`[[fallthrough]]`。如果多个`case`对应同一个操作，不需要特别注明：

```cpp
switch (x) {
  case 41:  // No annotation needed here.
  case 43:
    if (dont_be_picky) {
      // Use this instead of or along with annotations in comments.
      [[fallthrough]];
    } else {
      CloseButNoCigar();
      break;
    }
  case 42:
    DoSomethingSpecial();
    [[fallthrough]];
  default:
    DoSomethingGeneric();
    break;
}
```

如果循环中只有一条语句，`{}`是可选的：

```cpp
for (int i = 0; i < kSomeNumber; ++i)
  printf("I love you\n");

for (int i = 0; i < kSomeNumber; ++i) {
  printf("I take it back\n");
}
```

空循环应该添加`{}`，或者只有一个`continue`，不能只有一个`;`，正确的例子：

```cpp
while (condition) {
  // Repeat test until it returns false.
}
for (int i = 0; i < kSomeNumber; ++i) {}  // Good - one newline is also OK.
while (condition) continue;  // Good - continue indicates no logic.
```

错误的例子：

```cpp
while (condition);  // Bad - looks like part of do/while loop.
```

#### Pointer and Reference Expressions

`.`和`->`运算符周围没有空格。`*`和`&`运算符后没有空格。下面是正确的例子：

```cpp
x = *p;
p = &x;
x = r.y;
x = r->y;
```

声明指针和引用类型时，`*`和`&`可以贴近类型，也可以贴近变量：

```cpp
// These are fine, space preceding.
char *c;
const std::string &str;
int *GetPointer();
std::vector<char *>

// These are fine, space following (or elided).
char* c;
const std::string& str;
int* GetPointer();
std::vector<char*>  // Note no space between '*' and '>'
```

两种都可以，但是一个文件应该使用同一中风格，修改已有文件时不要破坏这个规则。

非指针和引用类型的情况下，可以一行声明或者定义多个变量，正确的例子如下：

```cpp
// Fine if helpful for readability.
int x, y;
```

错误的例子如下：

```cpp
int x, *y;  // Disallowed - no & or * in multiple declaration
int* x, *y;  // Disallowed - no & or * in multiple declaration; inconsistent spacing
char * c;  // Bad - spaces on both sides of *
const std::string & str;  // Bad - spaces on both sides of &
```

#### Boolean Expressions

如果复杂的布尔表达式超出了行宽限制，保证在`&&`和`||`后换行：

```cpp
if (this_one_thing > this_other_thing &&
    a_third_thing == a_fourth_thing &&
    yet_another && last_one) {
}
```

Google倾向于上面的用法，但是`&&`和`||`在每行的开头也可以。

#### Return Values

不要在`return`后面的语句添加不必要的`()`。

`return`后的语句如果有些复杂，可以添加`()`以提升可读性，例如：

```cpp
return result;                  // No parentheses in the simple case.
// Parentheses OK to make a complex expression more readable.
return (some_long_condition &&
        another_condition);
```

错误的例子：

```cpp
return (value);                // You wouldn't write var = (value);
return(result);                // return is not a function!
```

#### Variable and Array Initialization

用`=`、`()`和`{}`初始化都可以：

```cpp
int x = 3;
int x(3);
int x{3};
std::string name = "Some Name";
std::string name("Some Name");
std::string name{"Some Name"};
```

注意，对于可以通过`std::initializer_list`构造的类型，非空的`{}`会优先匹配`std::initializer_list`，但是空的`{}`有优先匹配默认构造函数。如果要使用非`std::initializer_list`的构造函数，应该使用`()`进行初始化：

```cpp
std::vector<int> v(100, 1);  // A vector containing 100 items: All 1s.
std::vector<int> v{100, 1};  // A vector containing 2 items: 100 and 1.
```

还要注意，`{}`初始化的方式不允许按向下转型：

```cpp
int pi(3.14);  // OK -- pi == 3.
int pi{3.14};  // Compile error: narrowing conversion.
```

#### Preprocessor Directives

`#`应该是第一个字符。

即使预处理指令不在文件开头，`#`也应该是第一个字符，正确的例子：

```cpp
// Good - directives at beginning of line
  if (lopsided_score) {
#if DISASTER_PENDING      // Correct -- Starts at beginning of line
    DropEverything();
# if NOTIFY               // OK but not required -- Spaces after #
    NotifyClient();
# endif
#endif
    BackToNormal();
  }
```

错误的例子：

```cpp
// Bad - indented directives
  if (lopsided_score) {
    #if DISASTER_PENDING  // Wrong!  The "#if" should be at beginning of line
    DropEverything();
    #endif                // Wrong!  Do not indent "#endif"
    BackToNormal();
  }
```

#### Class Format

类中的变量按照`public`、`protected`和`private`的顺序组织，这三个关键字缩进1个空格，变量和函数参照三个关键字再缩进1个空格。例子如下：

```cpp
class MyClass : public OtherClass {
 public:      // Note the 1 space indent!
  MyClass();  // Regular 2 space indent.
  explicit MyClass(int var);
  ~MyClass() {}

  void SomeFunction();
  void SomeFunctionThatDoesNothing() {
  }

  void set_some_var(int var) { some_var_ = var; }
  int some_var() const { return some_var_; }

 private:
  bool SomeInternalFunction();

  int some_var_;
  int some_other_var_;
};
```

有几点需要注意：
- 继承写在同一行，行宽不超过80
- `public`、`protected`和`private`参照`class`关键字缩进1个空格
- 除了第一个关键字，每个关键字的上一行应该为空行。类定义简单的情况下可以不遵守这点
- 这三个关键字后面没有空行
- 按照`public`、`protected`和`private`顺序定义类
- 按照[Declaration Order]()中的顺序组织变量

#### Constructor Initializer Lists

构造函数的初始化列表可以在一行，也可以多行，后续行按照4个空格缩进。正确的例子如下：

```cpp
// When everything fits on one line:
MyClass::MyClass(int var) : some_var_(var) {
  DoSomething();
}

// If the signature and initializer list are not all on one line,
// you must wrap before the colon and indent 4 spaces:
MyClass::MyClass(int var)
    : some_var_(var), some_other_var_(var + 1) {
  DoSomething();
}

// When the list spans multiple lines, put each member on its own line
// and align them:
MyClass::MyClass(int var)
    : some_var_(var),             // 4 space indent
      some_other_var_(var + 1) {  // lined up
  DoSomething();
}

// As with any other code block, the close curly can be on the same
// line as the open curly, if it fits.
MyClass::MyClass(int var)
    : some_var_(var) {}
```

#### Namespace Formatting

命名空间中的内容不缩进。

也就是命名空间不会添加缩进层次：

```cpp
namespace {

void foo() {  // Correct.  No extra indentation within namespace.
  // ...
}

}  // namespace
```

所以这是错的：

```cpp
namespace {

  // Wrong!  Indented when it should not be.
  void foo() {
    // ...
  }

}  // namespace
```

#### Horizontal Whitespace

按照如下规则使用行内空格。不要在行尾添加空格。

##### General

```cpp
int i = 0;  // Two spaces before end-of-line comments.

void f(bool b) {  // Open braces should always have a space before them.
  // ...
}

int i = 0;  // Semicolons usually have no space before them.

// Spaces inside braces for braced-init-list are optional.  If you use them,
// put them on both sides!
int x[] = { 0 };
int x[] = {0};

// Spaces around the colon in inheritance and initializer lists.
class Foo : public Bar {
 public:
  // For inline function implementations, put spaces between the braces
  // and the implementation itself.
  Foo(int b) : Bar(), baz_(b) {}  // No spaces inside empty braces.
  void Reset() { baz_ = 0; }  // Spaces separating braces from implementation.
  // ...
```

##### Loops and Conditionals

```cpp
if (b) {          // Space after the keyword in conditions and loops.
} else {          // Spaces around else.
}

while (test) {}   // There is usually no space inside parentheses.
switch (i) {
for (int i = 0; i < 5; ++i) {

// Loops and conditions may have spaces inside parentheses, but this
// is rare.  Be consistent.
switch ( i ) {
if ( test ) {
for ( int i = 0; i < 5; ++i ) {

// For loops always have a space after the semicolon.  They may have a space
// before the semicolon, but this is rare.
for ( ; i < 5 ; ++i) {
  // ...

// Range-based for loops always have a space before and after the colon.
for (auto x : counts) {
  // ...
}

switch (i) {
  case 1:         // No space before colon in a switch case.
    ...
  case 2: break;  // Use a space after a colon if there's code after it.
```

##### Operators

```cpp
// Assignment operators always have spaces around them.
x = 0;

// Other binary operators usually have spaces around them, but it's
// OK to remove spaces around factors.  Parentheses should have no
// internal padding.
v = w * x + y / z;
v = w*x + y/z;
v = w * (x + z);

// No spaces separating unary operators and their arguments.
x = -5;
++x;
if (x && !y)
  // ...
```

##### Templates and Casts

```cpp
// No spaces inside the angle brackets (< and >), before
// <, or between >( in a cast
std::vector<std::string> x;
y = static_cast<char*>(x);

// Spaces between type and pointer are OK, but be consistent.
std::vector<char *> x;
```

#### Vertical Whitespace

少使用空行。

在函数之间不要使用多余两个空行，函数开头没有空行，结尾也没有空行。

基本原则是：如果一屏能容纳更多的代码，就更易读。

下面是使用空行的经验：
- 函数开头和结尾的空行并不能提升可读性
- 很长的if-else语句之间使用空行可以提升可读性
- 注释前的空行有助于提升可读性，因为注释的内容往往表达的是不同的想法
- 命名空间内的空行和命名空间之间的空行有助于提升可读性

### Exceptions to the Rules

虽然上面的内容是强制的，但是也可以变通。

#### Existing Non-conformant Code

重点是和已有的内容保持一致。

#### Windows Code

- 不要使用匈牙利命名法的前缀。源文件用`.cc`后缀
- 虽然Windows有很多`typedef`，但是还是应该使用更贴近标准C++的方式，例如用`const TCHAR *`代替`LPCTSTR`。
- 使用cl.exe时，将警告级别调到3或者更高，并且将警告视为错误
- 不要使用`#pragma once`，应该用头文件哨兵
- 不要使用非标准库中的扩展，例如`#pragma`和`__declspec`。可以使用`DLLIMPORT`和`DLLEXPORT`

另外一些建议：
- 不鼓励使用多继承（multiple implementation inheritance;）
- 如果可能，关闭异常
- 尽量不用预编译的头文件，例如`StdAfx.h`
- 资源头文件不受上述限制
