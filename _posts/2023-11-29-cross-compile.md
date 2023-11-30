---
title:      "ARM交叉编译工具链与编译选项"
date:       2023-11-29 09:00:00
author:     zxy
math: true
categories: ["Coding", "System", "ARM"]
tags: ["ARM", "Compiler"]
post: true
---

## Toolchain是什么

Toolchain是一组将源代码转换为可执行程序的工具的集合。主要包括：

1. Binutils（Binary Utilities）: 一组用于创建和管理二进制程序、目标文件、库、性能分析数据以及汇编源代码的编程工具集。主要包括：

   - assembler汇编器`as`：把汇编语言转化为机器码
   - linker链接器`ld`: 将多个obj文件、库文件和其他必要的资源组合在一起，生成最终的可执行程序
   - `objcopy`: 用于复制和转换obj文件的工具。可以用来在不同的可执行文件格式之间进行转换，或提取目标文件的特定部分。
   - `objdump`: 反汇编obj文件，提供obj文件的汇编代码、头部信息和节（section）的详细信息
   - `strip`, `readelf`, `ar`, `size`等

2. Compiler编译器: 将高级编程语言（如C、C++）翻译成机器代码的工具。它接收源代码文件并生成相应的obj文件，比如GCC。

3. Run-time Libraries运行时库：各种支持运行时期间的功能的库文件，例如标准C库、数学库等。这些库在程序执行期间被动态链接到程序中。**在编译用户空间的程序时会链接运行时库，但是在编译内核时不需要运行时库。**

   对于Unix系统来说，Run-time library就是基于POSIX规范的标准化应用程序接口（API），由C语言定义。它是应用程序与操作系统内核之间的主要接口。有很多不同的C library可以根据不同的场景进行选择：

   - libc: Linux下的ANSI C的函数库，包括`<stdlib.h>`，`<stdio.h>`，`<string.h>`，`<time.h>`等

   - glibc: GNU C library
   - musl libc：轻量化的C library，适用于RAM和存储有限的情况
   - newlib: 一个面向嵌入式系统的C library
   - uClibc-ng: micro controller C library

4. Linux headers

5. Debugger: 提供了诸如设置断点、查看变量内容、跟踪程序执行流程等功能，例如gdb。

## ABI是什么

在构建Toolchain时，我们需要规定二进制文件生成的规范，比如数据类型的大小、布局和对齐，函数调用时参数应该怎么传递，函数返回值如何返回，如何进行系统调用等。这些规范都由**ABI（Application binary interface）**来规定。ABI 的设计目标是简化系统的软件开发和移植过程，使得不同的编译器、库和工具链能够生成和使用符合同一规范的二进制文件。

在一个系统中，kernel和所有的应用程序都应该遵循同一个ABI规范。

由于嵌入式系统的发展，ARM架构在2000年末提出了**EABI(Extended Application Binary Interface)**的规范，用于定义在嵌入式系统中生成和运行二进制程序的接口。ARM之前定义的规范，也就被称为**OABI (Old Application Binary Interface)**。

EABI的规范又根据浮点数的传递方式分为两类

- eabi: 通过通用整数寄存器传递
- eabihf (hf for Hard-Float)：通过硬件的浮点寄存器传递，速度快但是不兼容没有FPU（ floating point unit）的CPU

## Toolchain编译流程

Toolchain将源代码转换为可执行程序的流程如下图所示：

![compilation](/assets/img/in-post/2023-11-29-cross-compile/compile.png)

在嵌入式系统的构建中，我们也需要通过上述的流程，使用toolchain来构建嵌入式Linux系统的三个部分：the bootloader, the kernel, and the root filesystem。通常，用于 Linux 的工具链是基于 GNU 项目的组件构建的。

## Case Study：ARM交叉编译链

根据不同的架构、开发商、目标设备上运行的操作系统、abi等可以有各种不同的toolchain。这一节我们先了解工具链的命名规则，再看看常见的ARM交叉编译链。

### 工具链的命名规则

一般来说，Toolchain的命名规则为： `[arch]-[vendor]-[kernel]-[system]-`

![toolchain](/assets/img/in-post/2023-11-29-cross-compile/toolchain.png)

- arch: 表示目标芯片架构，比如 32 位的 Arm 架构对应的 arch 为 arm，64 位的 Arm 架构对应的 arch 为 aarch64。
  - 如果 CPU 支持两种字节序模式，可以通过添加 "**el**" 表示小端（little-endian），或者添加 "**eb**" 表示大端（big-endian）来区分。例如，小端MIPS 架构可以用 "mipsel" 表示，而大端ARM 架构可以用 "armeb" 表示。
- vendor：工具链提供商，大部分工具链名字里面都没有包含这部分。常见的工具链提供商有buildroot, poky, unknown, none.
- kernel ：编译出来的可执行文件(目标文件)针对的操作系统，比如 Linux, uclinux, bare（无OS）。
- system: 交叉编译链所选择的库函数和目标映像规范，如gnu, gnueabi, gnueabihf, musleabi, or musleabihf等。
  - gnu = gnu C library + oabi
  - gnueabi = gnu C library+ eabi
  - musleabi = musl C library + eabi

### 常见的ARM交叉编译链

arm交叉编译器下载：

- [linaro社区](https://releases.linaro.org/components/toolchain/binaries/)：老版本，版本范围 4.9 ~ 7.5
- [arm官网](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads) ： 新版本，版本范围 8.2 ~10.3

下面我们来看几个常见的例子：

- `arm-none-linux-gnueabi-gcc` 表示基于arm架构的，适用于任何CPU型号，linux系统下的，可编译出符合GNU规范以及EABI接口要求的，c语言的编译器。 

- `arm-none-eabi-gcc ` 表示基于arm架构的，适用于任何CPU型号（也可理解成不指定具体的工具链供应商），不带操作系统的裸机系统（包括 ARM Linux 的 boot、kernel，不适用于编译 Linux 应用 Application），可编译出符合嵌入式平台ABI接口要求的，c语言的编译器。
  -  一般适用用于 Arm Cortex-M/Cortex-R 平台，它使用的是 专用于嵌入式系统的newlib C 库。
  - 编译裸机的toolchain可以不用说明用的Runtime C library（也就是没有gnu这一项），因为在编译的时候不会用到C library
- `arm-linux-gnueabi-gcc` 和 `aarch64-linux-gnu-gcc` 适用于 Arm Cortex-A 系列芯片，前者针对 32 位芯片，后者针对 64 位芯片，它使用的是 glibc 库。也可以用来编译 u-boot、linux kernel 以及应用程序。
- `arm-linux-gnueabihf-gcc`:  表示基于arm架构的，适用于任何CPU型号，linux系统下的，可编译出符合GNU规范以及EABIHF接口要求的，c语言的编译器。 硬件的浮点寄存器传递传递浮点数。

>  `arm-linux-gnueabi-gcc`和`arm-linux-gnueabihf-gcc`的区别：
>
> - 其实这两个交叉编译器只不过是gcc的选项-mfloat-abi的默认值不同
>   - gnueabi表示用以下gcc选项`-mfloat-abi=soft` or `-mfloat-abi=softfp`编译
>   - gnueabihf表示用` -mfloat-abi=hard `编译，表示编译器和库可以使用硬件的浮点运算指令。
>
> - 既然只是编译选项不同，为什么要有两个不同的工具链呢？因为工具链包括了binutils，已经编译的libraries（libc）等，这两个不同的工具链表示工具链中的库支不支持使用浮点寄存器传参。

早期的arm处理器，因为在CPU内部没有专门的浮点运算硬件单元，所以对应的编译器遇到浮点运算的程序代码时，不会将它们翻译成使用FPU（floating point unit）计算的机器指令，而是将浮点运算翻译成在CPU的算术逻辑单元（ALU）上软件模拟的指令代码。后面的处理器开始逐渐集成FPU，对应的编译器选项`-mfloat-abi`也就有了三个：

- soft版：不使用FPU进行浮点运算，适合早期无FPU的ARM处理器
- softfp版： armel架构（对应的编译器为 arm-linux-gnueabi-gcc ）使用FPU进行浮点运算，但使用普通寄存器传参，编译器使用该版本，用来兼容没有FPU的处理器
- hard版：armhf架构（对应的编译器 arm-linux-gnueabihf-gcc ）使用FPU进行浮点运算，使用浮点寄存器传参，性能最高，但中断负荷最大，适用于现在高性能的集成FPU的处理器

其中后两者都要求arm里有fpu浮点运算单元，soft与后两者是兼容的，但softfp和hard两种模式互不兼容

不同的芯片厂商，发布自己不同处理器、不同平台的SDK时，使用的编译器版本可能不一样，一般我们建议，使用原厂的编译环境就可以了。而对于内核的学习，为了保持兼容性，建议使用softfp就可以了。

## Reference

1. Book: Mastering Embedded Linux Programming

2. [Stackoverflow: Difference between arm-eabi arm-gnueabi and gnueabi-hf compilers](https://stackoverflow.com/questions/26692065/difference-between-arm-eabi-arm-gnueabi-and-gnueabi-hf-compilers)
3. [交叉编译器 arm-linux-gnueabi 和 arm-linux-gnueabihf 的区别](https://www.cnblogs.com/xiaotlili/p/3306100.html)
4. [Wiki: GNU Binutils](https://en.wikipedia.org/wiki/GNU_Binutils)
5. [Cross Compiling ToolChains](https://education.sakshi.com/en/cseit/study-material/linux-system-programming/cross-compiling-tool-chains-44792)
6. [交叉编译器的命名规则及详细解释](https://blog.csdn.net/LEON1741/article/details/81537529)
7. [Stackoverflow: What's the difference between hard and soft floating point numbers?](https://stackoverflow.com/questions/3321468/whats-the-difference-between-hard-and-soft-floating-point-numbers)







