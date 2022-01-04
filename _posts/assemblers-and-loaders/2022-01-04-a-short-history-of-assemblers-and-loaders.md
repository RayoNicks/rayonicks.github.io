---
layout: post
title: 《汇编器与加载器》汇编器和加载器简史
categories: Reading
tags: system
---

# 汇编器和加载器简史

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

# 参考文献 {#Reference}

```ref
4. Wilkes, M. V., D. J. Wheeler, and S. Gill, The Preparation of Programs for an Electronic Digital Computer. Reading, MA.: Addison-Wesley, 1951.
8. Wilkes, M. V., The EDSAC, MTAC 4,(1950) p. 61. Also reprinted in Randall, B., The Origins of Digital Computers, Springer Verlag, Berlin, 1982.
9. Melcher, W. P., SHARE Assembler UASAP 3-7. SHARE distribution 564, 1958.
10. Goldfinger R., The IBM Type 705 Autocoder. Proc. East Joint Comp. Conf., San Francisco, 1956.
22. McIlroy, M. D., Macro Instruction Extensions of Compiler Languages, in Comm. ACM 3,(4), p. 214 (1960).
24. Ferguson, D. E., The Evolution of the Meta-Assembly Program, Comm. ACM 9, p. 190 (1966).
47. Conway, M. E., UNISAP, Symbolic Assembly Program for UNIVAC I and UNIVAC II, The Computer Center,Case Institute of Technology, Cleveland, 1958.
56. Boehm, E. M., and T. B. Steel, The SHARE 709 System, Machine Implementation and Symbolic Programming, J. ACM 6,2,134–140 (Apr. 1959).
58. Mealy, G. H., A Generalized Assembly System, in Rosen S., Programming Systems and Languages, New York, NY.: McGraw-Hill, 1969, pp. 535–559.
59. McCarthy, J. et. al., The Linking Segment Subprogram Language and Linking Loader, ibid pp. 572–581.
60. Greenwald, I. D., Handling of Macro Instructions, Comm. ACM 2,11,21–23(1959).
61. Wirth, N., PL360, A Programming Language for the 360 Computers, J. ACM 15,(1),37–75(Jan. 1968).
62. Barnett, M., Macro Directive Approach to High-Speed Computing, Solid State Physics Research Group, MIT, Cambridge, Mass.: 1959.
65. IBM 7090 Data Processing System, Reference Manual, IBM Form No. A22-6528.
85. NCRCentury NEAT/3 Ref. Manual, Dayton, OH.: NCR Corp., Binder #0210, 1968.
86. NCRCentury NEAT/3 Programming Text, Dayton, OH.: NCR Corp., Binder #0274, 1968.
97. Lavington, S., Early British Computers, Bedford, MA.: Digital Press, 1980.
```
