# 缘起

最近读到一篇文章，名字叫[《C++慢速指南》之打造工程友好型的C++项目](https://zhuanlan.zhihu.com/p/269603170)，作者提出了工程友好型C++项目的概念，其所具备的要素包括**单元测试**、**文档**和**自动化**，做到这三点后能明显提升项目的舒适度。如果想把项目打造成文中所述的“工程友好型项目”，除编码外还需要大量的前置知识，但是工作中受时间的制约，往往都是以实现功能为主要目的，很少关心其它的东西。

基于作者的观点，本文总结了我心目中的“工程友好型C++”项目所具有的特点。

# 编码（clang-format和Google C++ Style Guide）

## clang-format

首先是编码格式。每个人的编码风格都不一样（缩进、大括号的位置等等），所以项目中应该有统一的规范。格式化代码可以用`clang-format`，这是LLVM项目里的一个工具，只要项目根目录中存在名为`.clang-format`的文件，Visual Studio和Visual Studio Code就可以在键入一些特殊字符的时候自动格式化代码。`.clang-format`文件中各个字段的含义可以见[官方文档](https://clang.llvm.org/docs/ClangFormatStyleOptions.html)，用的最多的就是每行代码的宽度、缩进的大小还有在什么地方添加空格和换行。

## Google C++ Style Guide

`clang-format`内置的代码格式中，除了排在第一的LLVM外，就是Google的格式。Google不仅定义了代码格式，还定义了一套完整的编码规范。通读一遍[Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)后，感觉其中绝大分的规则是非常有道理的。正文规范中所讲的，只有了解了每条规则的意义，才能明白什么时候可以不遵守甚至改进这些规则。

关于Google的编码规范，这是我自己的[学习笔记]()。

# 构建（CMake）

C++的构建是一个老大难的问题，好在有CMake。CMake也分为传统CMake和现代CMake，分界线是版本3.0，推荐使用现代CMake，越新的越好。

重点是理解现代CMake中的传播（propagate）机制，也就是在最常见的`target_link_libraries()`命令中的`PRIVATE`、`INTERFACE`和`PUBLIC`背后的原理。

有两个视频讲的不错，[More Modern CMake](https://www.bilibili.com/video/BV1ea4y1v7mk/?vd_source=f7b3403b160f6c50cb65245afdb4043e)和[Oh No! More Modern CMake](https://www.bilibili.com/video/BV1fK411H7Nb/?vd_source=f7b3403b160f6c50cb65245afdb4043e)，还有作者关于两次演讲的[资料](https://github.com/Bagira80/More-Modern-CMake)。

在第一个视频中，作者提出了C++构建过程的两个概念，build requirement和usage requirement。Build requirement是构建某个targer时必须的内容，往往是源文件。如果是构建非对象库（object），例如通过`add_executable()`定义可执行文件或者通过`add_library()`定义静态库（static）或者动态库（shared）时，可能还需要链接其它库，这都属于构建依赖。Usage requirement就是使用某个target时需要的内容，此时只需要头文件就可以了。Build requirement通过`PRIVATE`关键字定义，不会继续传播；usage requirement通过`INTERFACE`关键字定义，会继续传播。所以，`target_link_libraries()`可以理解为：在构建左侧目标时，会依赖右侧的目标，右侧目标的usage requirement会随着`PRIVATE`和`INTERFACE`变成左侧目标的build requirement或者usage requirement。

第二个视频中，作者更加详细的说明了对象库（object）的依赖传递关系，还有现代CMake中使用`find_package()`查找已安装的第三方库（例如Boost）和使用`FetchContent_Declare()`和`FetchContent_MakeAvailable()`同时构建外部项目的方法。

# 文档（Doxygen）

文档应该说是项目中最容易忽略的东西了。因为项目迭代快，主要的精力往往集中于功能实现上，文档经常是事后补上的，所以大部分情况下文档是跟不上代码的，这就给新入项目的开发者带来很多的困难。

[Doxygen](https://www.doxygen.org/index.html)是根据代码中的注释生成API文档的工具，除了文字，还能生成各个实体之间的关系图。只要注释符合特定的格式，`doxygen`就能将其转换为html、Latex等格式的文档。使用Doxygen最大的优点是文档就在代码中，这就使得文档和代码紧紧的耦合在了一起。代码审查的时候顺便就可以检查文档，很大程度上避免了因文档和代码分离导致的文档落后于代码的问题。

Doxygen的官方文档写的非常通俗易懂：如果要教别人如何写文档，自己的文档就应该写清楚。Doxygen官网的html应该就是用`doxygen`生成的，在浏览时可以多注意下生成的html页面的效果。

虽然`doxygen`可以从多种格式的注释中提取文档，但是我还是更习惯JavaDoc风格的，也就是下面的格式：

```cpp
/**
 * ... text ...
 */
void func();

int var; /**< Detailed description after the member */
```

前面的是块注释，出现在`class`、`struct`、`union`、`enum`、`namespace`、宏和函数之前，后面的是行内注释，用来注释成员和参数。

Doxygen将注释分为两类，简明注释和详细注释，并且定义了自己的格式来从注释块中提取简明注释，不过我觉得不如显式的使用命令更方便。这里对常用的命令做一个总结，更多内容可以参考[官方文档](https://www.doxygen.nl/manual/commands.html)。另外，官网中的命令都是以`\`开头的，但是我更喜欢用`@`开头。

## 结构提示类（Section indicators）

这类命令用于表明所注释的对象的种类，还有可以控制调用图和引用图。用于提示所注释的对象的类型的命令包括：
- @class
- @concept，见C++20
- @def
- @dir，目录
- @enum
- @example
- @extends，继承
- @file
- @fn
- @headerfile
- @implements，继承
- @interface，接口类
- @namespace
- @overload，重载
- @private
- @protected
- @public
- @pure，纯虚函数
- @relates和@relatesalso，和其它实体关联
- @static
- @struct
- @typedef
- @union
- @var

其实只要注释严格的出现在所注释的对象之前，这些命令中的很多都是用不上的。

## 节提示类命令（Section indicators）

这些命令用于描述所注释的对象的属性，贴一个官网的[例子](https://www.doxygen.nl/manual/examples/author/html/class_some_nice_class.html)：

```cpp
/*! 
 *  @brief     Pretty nice class.
 *  @details   This class is used to demonstrate a number of section commands.
 *  @author    John Doe
 *  @author    Jan Doe
 *  @version   4.1a
 *  @date      1990-2011
 *  @pre       First initialize the system.
 *  @bug       Not all memory is freed when deleting an object of this class.
 *  @warning   Improper use can crash your application
 *  @copyright GNU Public License.
 */
class SomeNiceClass {};
```

因此这些都不能省略，个人认为比较有用的注释信息包括：
- @author(s)，作者信息
- @brief，简要描述
- @bug，文档中棕色显示
- @copyright
- @date和@showdate
- @deprecated
- @details，详细描述
- @exception
- @invariant
- @note，黄色突出显示
- @param
- @tparam，模板参数
- @post，后置条件
- @pre，前置条件，文档中绿色突出显示
- @remark，注释，类似MSDN
- @return
- @sa，see also
- @since，表明从那个版本开始引入
- @test，描述测试样例
- @todo
- @version
- @warning，文档中红色显示

## 链接类命令（Commands to create links）

用于在文档中创建链接，更多见[官方文档](https://www.doxygen.nl/manual/autolink.html)。

## 示例代码类命令（Commands for displaying examples）

这类命令用于在注释中讲解代码，可以用`@include`添加整段代码，也可以通过`@dontinclude`和一些配套命令实现逐行讲解，贴一个官方的[例子](https://www.doxygen.nl/manual/examples/include/html/pag_example.html)：

```cpp
/*! A test class. */
 
class Include_Test
{
  public:
    /// a member function
    void example();
};
 
/*! \page pag_example
 *  \dontinclude include_test.cpp
 *  Our main function starts like this:
 *  \skip main
 *  \until {
 *  First we create an object \c t of the Include_Test class.
 *  \skipline Include_Test
 *  Then we call the example member function 
 *  \line example
 *  After that our little test routine ends.
 *  \line }
 */

// include_test.cpp
void main()
{
  Include_Test t;
  t.example();
}
```

## 文档显示类命令（Commands for visual enhancements）

最后一部分命令用于调整界面显示，如字体、文字样式、列表、图片、emoji表情、消息序列图、UML和公式，更多的命令见官方文档。

# 测试（Google Test）

[视频1](https://www.bilibili.com/video/BV1ug411C73x/?vd_source=f7b3403b160f6c50cb65245afdb4043e)

[视频2](https://www.bilibili.com/video/BV1bW4y1h7yJ/?vd_source=f7b3403b160f6c50cb65245afdb4043e)

[视频3](https://www.bilibili.com/video/BV1cP411j7ri/?vd_source=f7b3403b160f6c50cb65245afdb4043e)

[Google Test官方文档](https://google.github.io/googletest/)

# 自动化（GitHub Actions）

前面都是本地的操作，这一步是线上的操作。本地编码完毕测试通过后，下一步就要合并到主分支进行发布了。在这之前，还包括代码审查、静态扫描、集成测试等环节，这些都是固定的流程，枯燥又无味，大部分可以通过自动化的方式来实现，这就是现在流行的CICD（Continuous Integration，Continuous Delivery）。

一般来说在大项目中，每个人的修改都是项目中的一部分，为了不影响其他的功能模块，还需要集成到完整的项目中进行测试。这里想要达到的效果是：在本地将代码push到远程的仓库后，可以自动触发构建和测试等自动化的工作。由于这个时间会比较长，其他开发者可以在这个时间里进行代码审查，一旦审查和自动化的测试通过，就可合并到主分支上进行发布了。

几乎所有的代码托管平台都提供了CICD的功能，这里简单介绍下GitHub自带的Actions。其余的代码托管平台，如GitLab，还有其它的CICD工具都是同样的原理。

在项目根目录中，新建`.github/workflow/`目录以保存yml文件，这样GitHub就可以自动检测并运行所有的工作流了。

有几个概念有助于理解：
- 工作流（workflow）：GitHub自动进行的一些列任务称为工作流，例如构建，测试，打包和部署
- 触发事件（event）：触发GitHub运行工作流的操作，例如push和pull request
- 作业（job）：一个工作流由多个作业组成，不同的作业在不同的环境中运行，如果不添加依赖关系，作业之间是并行的
- 步骤（step）：作业中的每一步，可以是GitHub中已有的Action，或者是一条命令
- 构件（artifacts）：自动构建后生成的文件，例如安装包和源码包等

对于C++项目，一般用CMake管理，GitHub Actions中提供了基于CMake项目的模板（CMake based projects）：

```yml
name: CMake

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

env:
  # Customize the CMake build type here (Release, Debug, RelWithDebInfo, etc.)
  BUILD_TYPE: Release

jobs:
  build:
    # The CMake configure and build commands are platform agnostic and should work equally well on Windows or Mac.
    # You can convert this to a matrix build if you need cross-platform coverage.
    # See: https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/managing-complex-workflows#using-a-build-matrix
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Configure CMake
      # Configure CMake in a 'build' subdirectory. `CMAKE_BUILD_TYPE` is only required if you are using a single-configuration generator such as make.
      # See https://cmake.org/cmake/help/latest/variable/CMAKE_BUILD_TYPE.html?highlight=cmake_build_type
      run: cmake -B ${{github.workspace}}/build -DCMAKE_BUILD_TYPE=${{env.BUILD_TYPE}}

    - name: Build
      # Build your program with the given configuration
      run: cmake --build ${{github.workspace}}/build --config ${{env.BUILD_TYPE}}

    - name: Test
      working-directory: ${{github.workspace}}/build
      # Execute tests defined by the CMake configuration.
      # See https://cmake.org/cmake/help/latest/manual/ctest.1.html for more detail
      run: ctest -C ${{env.BUILD_TYPE}}
```

- 通过`on`指定触发工作流的事件，模板中的是push和pull request，全部的触发事件见[Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)。更多触发方式见[Trigger a workflow](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow)。
- 通过`jobs`定义工作流中的工作。模板中只有一个工作，名字叫`build`，就是在最新的Ubuntu上通过CMake进行构建，还可以创建在Windows和Mac中进行构建的工作。
- 通过`steps`中定义工作的步骤。通过`uses: actions/checkout@v3`可以使用已有的action来将代码检出到触发工作流的分支上。如果已有的action有参数，可以通过`with`传递。通过`name`可以自定义后面的步骤，`run`中指明了每一步中执行的命令。

工作流的全部语法介绍可以在[官方文档](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)中找到。

默认情况下工作流是在GitHub提供的服务器中运行，但是也可以使用自己的服务器，甚至还可以使用容器。可以在工作流中创建pull request或者issue，但这时候要使用token。除了使用现有的action，也可以自定义action，更多内容见[官方文档](https://docs.github.com/en/actions)。
