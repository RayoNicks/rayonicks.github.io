---
layout: post
title: 《汇编器与加载器》前言、绪论、简史和分类
categories: Reading
tags: system
---

## 前言

我的计算机生涯要追溯到1960年，那时候高级语言还很罕见，所以我要从IBM 7040汇编语言开始介绍计算机。大约使用汇编语言一年后，我开始学些Fortran，不过我仍然很喜欢汇编器，同时我发现关于这个领域的文章很少，特别是相较于编译器来说。我最熟悉的关于汇编器和加载器的文献是[[1、2、3、64](#Reference)]。虽然有很多汇编语言的书籍，不过它们只讲汇编器的功能，而不讲汇编器的实现。

出现这种情况的原因是大概从19世纪50年代中期到19世纪70年代中期汇编语言开始衰落。Fortran和其它高级语言的光芒盖过了汇编语言。随着越来越多的程序员开始使用高级语言，汇编器也逐渐退出程序员的视野。不过1975年左右出现的微处理器给汇编语言带来了新的生机。

早期没有编译器，程序员们必须使用汇编语言，甚至是更低级的机器语言。不过没过多久，大约到19世纪90年代，就出现了很多针对微型计算机的编译器，但是汇编器并没有衰落，因为从本质上讲，所有运行在现在计算机上的软件开发系统都包含一个汇编器。第8章描述的汇编器就是一个最典型的例子。

文献[[5-7](#Reference)]中介绍了3个运行在CP/M机器上的Z-80汇编器，这些汇编器不仅没有过时，反而仍然是最先进的、支持重定位、宏汇编和条件汇编的现代汇编器。此外，文献[[5、6](#Reference)]还介绍了链接器和加载器。这些汇编器反映了80年代早期程序员们对Z-80和CP/M的关注点。针对现代x86和680x0处理器的汇编器也继承了这些传统，程序员们使用更加现代和复杂的汇编器来为特定的过程优化代码。

对于加载器就有些不同了。虽然一直在使用加载器，但是由于对程序员是透明的，所以程序员们几乎意识不到它的存在，自然相关的文献就更少了。

本书和其它汇编书籍的最大区别在于本书并不介绍某种汇编语言的语法，而是介绍汇编器和加载器的设计与实现。本书假设读者具备计算机和编程相关知识，因此会解释汇编器和加载器的工作原理。即使书中的例子是基于一种假设的出来的汇编语言，但是书中的讨论的理论都是通用的。具体的例子可以参见文献[[5、6、7、13、26、27、30、31、32、35、37、39、101](#Reference)]。

写作本书的初衷在于许多年前我的学生们向我抱怨说这个领域的资料太少了。由于我每学期都教授汇编器和加载器相关知识，所以我开始积累课堂笔记。随着笔记的增多，我本来准备发表一篇解释性的论文，但是发现自己太忙了，同时一篇论文也容不下这么多的内容，所以最后就写了这本书。

这是一本专业计算机教材，同时也是为系统程序员准备的，也可以作为系统编程或者计算机组成课程的补充材料。

第1章介绍了一遍扫描和两遍扫描的汇编器、一些重要概念——例如绝对定位和相对定位目标文件还有汇编器的一些特性——例如本地符号表和多位置计数器（multiple location counters）。

第2章介绍了符号表的数据结构。

第3章介绍了一些实际汇编器中伪指令的格式、含义和实现。

第4章介绍了汇编器中的宏汇编技术和条件汇编技术。为了完整，本章试图囊括所有的宏汇编特性及实现方法，但是并没有关注某种特定汇编语言的宏汇编语法。

第5章介绍了列表文件，第6章介绍了反汇编，还有三种特殊类型的汇编。一些读者肯定会对元汇编（meta-assemblers）和高级汇编（high-level assemblers）感兴趣。虽然这不是什么新鲜技术，但是却罕为人知。

第7章介绍了加载器。这一章中包含一个一遍扫描加载器的例子，还有动态加载、自举加载（bootstrap loader）、覆盖（overlay）和其它特性。

第8章介绍了4个最先进的汇编器，并进行了对比。

为了更像一本教材，每一章都包含一些练习题，答案在附录C。每一章的最后还包括复习题和项目。有些复习题很简单，有些则很难，需要同学们从教科书中寻找答案。项目可以作为编程作业，有些也是很简单，有些则稍微复杂一些。项目最好按照顺序完成，因为前后具有关联性。一些关于寻址模式的资料可以从附录A中得到。

参考文献用方括号表示。

需要特别关注的内容在每一段的开头，例如一些基本概念和其它重要的内容。

我要感谢来自英格兰国家物理实验（National Physical Laboratory）的B.A.Wichmann，他向我提供了P-516高级汇编语言、BABBAGE语言和GE 4000微机族语言的资料，这是我在收集和分析本书资料过程中收到的唯一资料。我还要感谢来自橡树岭国家实验室（Oak Ridge National Labs）的Johnny Tolliver，他提供的`MakeIndex`程序在形成本书的参考文献时起到了至关重要的作用。

## 绪论

我们首先要给汇编器下个定义，不幸的是计算机科学不像数学一样是一门严谨的学科，所以很多定义并不精确。我最喜欢关于汇编器的定义应该是这个：

汇编器的功能是将符号语言表示的源指令逐条翻译为机器可以识别的目标指令。

>An assembler is a translator that translates source instructions (in symbolic language) into target instructions(in machine language), on a one to one basis.

注意，每一条源指令只翻译为一条目标指令。

这个定义的优点在于清楚的说明了汇编器的翻译过程，不过还是不太准确，因为汇编器的功能并不只限于翻译，还为编写程序提供了很多便利——也就是通常说的指令（也包括伪指令）。本书将在第3章和第4章中讨论所有重要的指令。

汇编器的另一个定义为：

汇编器的功能是将面向机器的语言翻译为机器语言。

>An assembler is a translator that translates a machineoriented language into machine language.

这个定义借鉴了编译器的定义：编译器的功能是翻译面向问题的语言，或者机器无关的语言。相比于前者，这个定义没有提及逐条翻译，而这其实是汇编器最重要的特点之一。

学习汇编器的原因在于汇编器的工作过程反映了机器的体系结构，因此汇编语言也非常依赖于机器的内部组成。体系结构方面的特性，例如内存字大小、数制系统、编码方式、索引寄存器、通用寄存器，影响着汇编语言的编写方式和汇编器的翻译方式。这一事实说明了为什么还有研究汇编器的必要、为什么汇编语言相关课程依然是计算机学科的必修课程。

世界上第一个汇编器是assemble-go，它的功能仅仅是将代码汇编到内存中然后开始执行。不过人们很快的就意识到，链接是一个非常重要的功能。计算机科学的先驱们很早就提出了库（routine library）的概念，所以它们需要汇编器可以定位、加载和将其链接到主程序中，汇编器因此而得名（准确来说应该要组装器）[[4](#Reference)]。如今，汇编器只保留了翻译的功能，并且一次只翻译一个汇编程序，而定位、加载和链接等其它功能由加载器负责。

现代汇编器包含两种输入和两种输出，第一种输入是触发汇编器的命令，命令中一般包含需要汇编的源代码文件，还包含一些在如何进行汇编的信息，例如：

- 目标文件和列表文件的名字
- 是否在终端中打印列表文件
- 遇到错误是否继续进行汇编
- 不打印但是保存列表文件
- 是否启用宏
- 符号表的大小提示

本书后面将会逐一解释这些术语。下面给出了一个启用宏和的VAX汇编器命令：

```
MACRO /SHOW=MEB /LIST /DEBUG ABC
```

这条命令启动了汇编器程序，指出了源代码文件名为abc.mar（后缀被省略了）、在扩展宏时要显示二进制行、要创建列表文件、在汇编的过程中要生成调试信息。

再看一个MASM汇编器的例子：

```
MASM /d /Dopt=5 /MU /V
```

这条命令向汇编器传递了一个值为`5`的`opt`变量，并且告诉汇编器要将源代码文件中的所有字母转换为大写，同时在列表文件中要包含一些额外的信息。

汇编器的第二个输入是源文件，包含一系列符号化的指令和伪指令。汇编器会将这些指令逐条翻译为机器指令，而对于伪指令，汇编器会用特定的方式去解释它们。现代汇编器一般支持上百条伪指令，从最简单的`ORG`（用于指出起始地址），到复杂的`MACRO`。本书第3章和第4章将会介绍常见的伪指令。

汇编器的第一个输出是目标文件，这也是最重要的输出，它包含了将要被机器执行的机器指令。目标文件是汇编器和加载器系统中重要的组成部分，因为它实现了一次汇编后的多次运行。此外，目标文件中也可以包含汇编器需要传递给加载器的一些信息，从而选择程序的加载方式，这些信息可以被称为加载器伪指令，本书第3章和第7章将会讨论加载器伪指令。汇编器也可以不输出目标文件。

汇编器的第二个输出是列表文件，源文件的中的每一行产生列表文件的规则为：

- 定位计数器（The Location Counter）
- 源代码行号
- 机器指令
- 伪指令

一般列表文件生成后会被输出，然后就被删掉了，所以程序员也可以选择不生成或者不打印列表文件。有一些伪指令是针对列表文件的，比如可以跳过生成列表文件的一部分，是否打印页首部，控制宏的展开等。

列表文件通常包含交叉引用信息，存在例外的是MASM汇编器将交叉引用信息生成到了单独的文件中，并且要通过额外的程序来查看交叉引用信息。交叉引用信息包含程序中所有符号被定义的位置和被引用的位置。

*练习1 为什么需要跳过生成列表文件的一部分，或者不打印列表文件？*

前面提到的assemble-go汇编器是不生成目标文件的，它只生成机器指令和列表文件。assemble-go是一遍汇编器，理论上是可以生成目标文件的，只是目标文件中使用的都是绝对地址，这就降低了目标文件的灵活性。

现在很多汇编器都是两遍汇编器，它们产生的目标文件是可以被重定位的，因此可以被链接和加载。

顾名思义，加载器的作用是将程序加载到内存中，不过现代加载器的作用远不止于此，它们的主要任务包括加载、重定位、链接和启动程序。现代加载器可以读取多个目标文件，将它们逐个加载到内存中，重定位每个目标文件，最终将所有单独的目标文件链接成一个可执行模块，并从程序入口点开始执行。这样的加载器有几个优点，其中最重要的优点是能够编写和组装包含多个独立部分的程序。

将一个大型程序拆分成多个部分是有利的，因为每一部分可以由不同的程序员编写，每个程序员只关注自己的部分就可以了，甚至不同的部分都可以使用不同的语言编写。通常情况下，主程序是由高级语言编写的，其余例程用汇编语言编写。每一部分被分别汇编（编译），产生多个目标文件，所以汇编器（编译器）只能看到一部分，不能看到整个程序，只有加载器可以看到所有的部分，并将这些部分组装成一个完整的程序。同时汇编器在也是不知道输入源程序是完整的程序还是只是一部分的，所以它假设程序从零地址开始，并基于该假设进行汇编。在加载器加载每一部分之前，加载器会根据当前可用的内存区域和前一个加载的对象文件的地址确定当前部分的开始地址，然后加载程序，确保所有的指令装入正确的内存地址。这个调整程序中内存地址的过程称为重定位。

由于汇编器一次只处理一个程序，因此它是不能链接多个程序的。当汇编器汇编主程序的源文件时，它是不知道其它的源文件中可能包含了主程序中需要调用的例程的，结果就是它不知道调用指令的目标地址是什么，这样产生出的目标文件中就会包含一些无法填充的洞（holes）。但是由于加载器是可以看到所有的源程序的，所以它会填充这些洞中的值。这个过程称为链接。

在程序可以被运行之前，需要对其进行翻译（汇编和编译）、加载、重定位和链接，这些工作由汇编器（编译器）和加载器分别进行。同时具备两种功能的汇编加载系统（dual assmebler-loader systems）也是存在的。和上述过程存在区别的就是解释执行的程序，例如BASIC和APL，这些语言编写的程序的运行需要解释器，而不是汇编器和加载器。划分汇编器和加载器的首要原因在于可以分别开发程序的不同部分，详细的原因在这里不予给出，不过我们在这里会按照重要性给出一些汇编加载系统的优点：

- 程序可以被划分为多个部分进行分别开发，甚至使用不同的语言
- 汇编器的体积主要取决于符号表和宏表的大小，分别汇编源程序的每一部分可以使汇编器的体积很小
- 如果汇编很慢的话，那么当程序被修改后只重新汇编修改的一部分就可以了，不过事实一般是汇编很快，加载很慢
- 将汇编和加载分开使得加载器可以直接从库中加载汇编后的例程，这通常被认为是一个优点，但其实并不是这样子的，因为如果库中就是源代码的话也是可以的，而且源代码的体积通常会比目标文件的体积小

上面第二条感觉有些牵强，原文：

>It keeps the assembler small. This is an important advantage. The size of the assembler depends on the size of its internal tables (especially the symbol table and the macro definition table). An assembler designed to assemble large programs is large because of its large tables. Separate assembly makes it possible to assemble very large programs with a small assembler.

## 汇编器和加载器简史

延迟存储自动电子计算器（Electronic Delay Storage Automatic Calculator，EDSAC）是最早的具备存储程序能力的计算机，它由剑桥大学的Maurice Wilkes和W. Renwick在1949年发明[[4、8、97](#Reference)]。从EDSAC被发明的那天起，它就配有一个名为Initial Orders的汇编器，这个汇编器通过一组在ROM中的旋转式电话选择器实现，以符号指令作为输入，每一条符号指令由三部分组成，前两部分是字母形式的助记符和一个十进制地址，而第三部分是由程序员预设的12个常量之一，该常量在汇编时被添加到该地址。

有趣的是，Wilkes也是第一个提出标签、第一个使用宏和第一个开发子例程的，不过他当时称标签为浮点地址（floating addresses），称宏为综合顺序（synthetic orders）。

文献[[65](#Reference)]中给出了如何在早期汇编语言中使用标签的方法。IBM 650计算机在1953年左右问世，并配有一个和如今的汇编器非常类似的汇编器。符号优化和汇编程序（Symbolic Optimizer and Assembly Program, SOAP）是第一个使用传统方法进行符号汇编的程序，但是它最大的特点在于高效的计算下一条指令的地址。IBM 650数字计算机中配有磁鼓内存，程序存储在内存中。每一条指令都从内存中读取，并且每一条指令中都包含下一条指令的地址。为了提升执行速率，每一条指令都应该存储在上一条指令读取完后磁头所在的位置。SOAP通过每一条指令的执行时间计算该地址。第7章将会描述更多的细节，同时也有一个相关编程项目。

IBM 704是第一批较为成功的商业计算机中的一个，它包含支持浮点计算的硬件和索引寄存器。IBM 704在1956年问世，同年，联合飞机公司（United Aircraft Corp）的Roy Nutt为IBM 704编写了联合飞机公司符号汇编器（United Aircraft Symbolic Assembly Program，UASAP-1）。起初它只是一个简单的汇编器，功能就是逐条翻译指令，其余由程序员自由发挥。后来，IBM的用户组织SHARE，采用了该汇编器的后续版本[[9](#Reference)]，并将组织成员们贡献的代码一同发布给了组织成员。UASAP为了早期汇编器的作者们提供了汇编器的实现思路，并且器设计原则仍然被如今的汇编器所采用。UASAP的后续版本也支持宏汇编技术[[62](#Reference)]。

同年，R.Goldfinger为IBM 702/705计算机开发了IBM Autocoder汇编器[[10](#Reference)]。这是第一个支持宏的汇编器。Autocoder汇编器被广泛使用，并最终被开发成具有大量可选安装宏库的大型系统。

另一个较早出现的汇编器是1958年由M. E. Conway为UNISAP I & II计算机开发的UNISAP汇编器[[47](#Reference)]。这是一个1.5遍扫描汇编器，是第一个支持本地标签的汇编器。这些概念都将在第1章中描述。

到19世纪50年代末，IBM发布了7000系列计算机，该计算机配有符号编码和翻译（Symbolic Coder And Translator，SCAT）宏汇编器，同时该汇编器也具有现代汇编器的所有特性：支持伪指令、支持可扩展的宏库并且生成可重定位的目标文件。

SCAT汇编器最初是为IBM 709计算机开发的[[56](#Reference)]，经修改后被移植到了IBM 7090计算机，该计算机上的另一个强大的汇编器是通用汇编系统（Generalized Assembly System，GAS）[[58](#Reference)]。

宏汇编的概念是由很多人同时提出的，不过McIlroy[[22](#Reference)]应该是是第一个提出宏汇编和条件汇编的，并且在GAS汇编器中提供了实现。文献[[60](#Reference)]中介绍了宏定义表的处理方法。

为IBM 704-709-7090计算机开发的加载器[[59](#Reference)]是首批全功能加载器之一，它支持重定位和链接。

Ferguson最早提出了元汇编技术[[24](#Reference)]。高级汇编的概念由Wirth提出[[61](#Reference)]，后来被NCR的某个无名的开发人员进行了扩展，最终形成了NEAT/3语言[[85、86](#Reference)]。

下面这张图展示了汇编器和加载器发展过程中经历的主要阶段。

![history](/assets/images/assemblers-and-loaders/history.jpg)

## 汇编器和加载器分类

- 一遍汇编器（One-pass Assembler）：只读取源文件一次
- 两遍汇编器（Two-pass Assembler）：读取源文件两次
- 驻留内存汇编器（Resident Assembler）：永久加载到内存中的汇编器，一般是加载到ROM中，功能很简单，仅支持很少的伪指令，也不支持宏，也是一遍汇编器。上面三种汇编器会在第1章中介绍
- 宏汇编器（Macro-Assembler）：支持宏汇编，参见第4章
- 交叉汇编器（Cross-Assembler）：在某台机器上运行、但是为另一台机器生成代码的汇编器。为了方便移植，这种汇编器一般使用高级语言实现，用来在大型机器上为小型机器生成目标代码
- 元汇编器（Meta-Assembler）：可以汇编不同指令集的汇编器
- 反汇编器（Disassembler）：功能和汇编器相反，将机器码转换为汇编程序
- 高级汇编器（high-level assembler）：具备某些高级语言特性的汇编语言的汇编器，这种汇编语言也可以被认为是高级语言。交叉汇编器、元汇编器、反汇编器和高级汇编器将在第6章中介绍
- 微指令汇编器（Micro-Assembler）：用来汇编微指令，和普通汇编器没什么本质上的区别，而且微指令也不是用来对微机编程的

上述汇编器也是可以组合的，例如存在交叉宏汇编器和微指令驻留内存汇编器。

- 自举加载器（Bootstrap Loader）：开始部分的代码用来加载剩余部分，或者加载另外的加载器，一般存在于ROM中
- 绝对地址加载器（Absolute Loader）：只能加载包含绝对地址的目标文件，目标文件中的程序有固定的内存地址
- 重定位加载器（Relocating Loader）：能够加载可重定位目标文件的加载器，同一个程序可以被加载到不同内存地址
- 链接加载器（Linking Loader）：可以链接被分成多个部分的程序，并将所有部分都加载到内存中
- 链接编辑器（Linkage Editor）：执行链接和重定位的功能，并生成一个可以被重定位加载器加载的模块。第7章中将讨论所有类型的加载器。

## 参考文献 {#Reference}

```ref
1. Barron, D. W., Assemblers and Loaders, 3rd ed., New York, N.Y.: American Elsevier 1968.
2. Kent, W., Assembler Language Macroprogramming, ACM Computing Surveys 1,4(Dec. 1969) 183–196.
3. Presser, L., and J. R. White, Linkers and Loaders, ACM Computing Surveys 4,3(Sep. 1972) 149–167.
4. Wilkes, M. V., D. J. Wheeler, and S. Gill, The Preparation of Programs for an Electronic Digital Computer. Reading, MA.: Addison-Wesley, 1951.5. Z80ASM Ver. 1.05 from SLR Systems, Butler, PA. 1984.
5. Z80ASM Ver. 1.05 from SLR Systems, Butler, PA. 1984.
6. ASMZ80 Ver 3.6C from Relational Memory Systems, San Jose, CA. 1984.
7. MOPI Ver.2.0 from Voice Operated Computer Systems, Minneapolis, Minn. 1984.
8. Wilkes, M. V., The EDSAC, MTAC 4,(1950) p. 61. Also reprinted in Randall, B., The Origins of Digital Computers, Springer Verlag, Berlin, 1982.13. Signetics 2650 Microprocessor Manual. Sunnyvale, CA.: Signetics Corp., 1977.
9. Melcher, W. P., SHARE Assembler UASAP 3-7. SHARE distribution 564, 1958.
10. Goldfinger R., The IBM Type 705 Autocoder. Proc. East Joint Comp. Conf., San Francisco, 1956.
13. Signetics 2650 Microprocessor Manual. Sunnyvale, CA.: Signetics Corp., 1977.
22. McIlroy, M. D., Macro Instruction Extensions of Compiler Languages, in Comm. ACM 3,(4), p. 214 (1960).
24. Ferguson, D. E., The Evolution of the Meta-Assembly Program, Comm. ACM 9, p. 190 (1966).
26. IBM System/360 Operating System Assembler Language, IBM Form No. GC28-6514.
27. IBM System/360 OS/VS and DOS/VS Assembler Language, IBM Form No. GC33-4010.
30. CDC COMPASS Version 3 Reference Manual, #60492600.
31. SUN Microsystems Assembler Language Reference Manual, part #800-1179.
32. PDP-11 Macro-11 Language Reference Manual, Order #AA-5075B-TC.
35. ASM 86 Language Reference Manual, Intel Corp., Order #121703.
37. IBM PC Macro Assembler, IBM #6172234.
39. NOVA Computer Assembler Manual, Data General Corp. #093-000017.
47. Conway, M. E., UNISAP, Symbolic Assembly Program for UNIVAC I and UNIVAC II, The Computer Center,Case Institute of Technology, Cleveland, 1958.
56. Boehm, E. M., and T. B. Steel, The SHARE 709 System, Machine Implementation and Symbolic Programming, J. ACM 6,2,134–140 (Apr. 1959).
58. Mealy, G. H., A Generalized Assembly System, in Rosen S., Programming Systems and Languages, New York, NY.: McGraw-Hill, 1969, pp. 535–559.
59. McCarthy, J. et. al., The Linking Segment Subprogram Language and Linking Loader, ibid pp. 572–581.
60. Greenwald, I. D., Handling of Macro Instructions, Comm. ACM 2,11,21–23(1959).
61. Wirth, N., PL360, A Programming Language for the 360 Computers, J. ACM 15,(1),37–75(Jan. 1968).
62. Barnett, M., Macro Directive Approach to High-Speed Computing, Solid State Physics Research Group, MIT, Cambridge, Mass.: 1959.64. Calingaert, P., Program Translation Fundamentals, Rockville, MD.: Computer Science Press, 1987.
64. Calingaert, P., Program Translation Fundamentals, Rockville, MD.: Computer Science Press, 1987.
65. IBM 7090 Data Processing System, Reference Manual, IBM Form No. A22-6528.
85. NCRCentury NEAT/3 Ref. Manual, Dayton, OH.: NCR Corp., Binder #0210, 1968.
86. NCRCentury NEAT/3 Programming Text, Dayton, OH.: NCR Corp., Binder #0274, 1968.
97. Lavington, S., Early British Computers, Bedford, MA.: Digital Press, 1980.
101. Flores Ivan Assemblers and BAL, Englewood Cliffs, NJ.: Prentice-Hall, 1971.
```
