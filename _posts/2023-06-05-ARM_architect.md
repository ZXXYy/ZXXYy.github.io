---
layout: post
title:      "区别ARM下的Armv9, A32, AArch64, Cortex..."
date:       2023-06-05 14:00:00
author:     zxy
math: true
categories: ["Coding", "System", "ARM"]
tags: ["ARM"]
post: true
---

Arm生态下的术语(terms)五花八门，时常更新...这直接导致刚刚接触的Arm的人会晕头转向，各种各样的名词扑面而来，完全分不清什么是什么。这篇post总结了Arm生态下的一些名词帮助理解。

### Recap

在解释Arm生态下的terms前，有必要重新回忆一下指令集、指令集架构、指令微架构、处理器核心等名词的意思。

- 指令集（Instruction Set）：**是一组机器指令**，用于告诉处理器执行特定的操作。它定义了处理器能够理解和执行的指令的集合，包括操作码（opcode）和操作数（operand）。
- 指令集架构（Instruction Set Architecture，简称ISA）：是对指令集的更高级别的描述。它**不仅定义了指令集本身，还包括了处理器的寄存器、内存模型、地址空间、异常处理机制等方面的规范**。ISA定义了处理器与软件之间的接口，决定了软件如何与硬件进行交互和利用处理器的功能。
- 微架构/处理器架构（Microarchitecture）：指令微架构是**指令集架构的具体实现方式**，它是在ISA的基础上，实现处理器硬件的细节和组织方式。某个处理器架构的指令集架构规定了它应该支持的指令集和指令的功能，但是具体的实现方式可以有很多种。不同的微架构设计可能在指令缓存大小、流水线深度、乱序执行能力、超标量技术、分支预测等方面有所差异。
- 处理器核心（CPU cores）：CPU内的一个独立处理单元，**是执行指令的物理单元**。每个核心能够独立执行指令，并能与多核处理器中的其他核心同时执行任务。每个核心通常拥有自己的执行资源，如算术逻辑单元（ALU）、寄存器和缓存。微架构定义了这些独立处理单元如何操作和与其他组件交互以执行计算任务，每个核心都可以有自己的微架构。
- SoC（System on a Chip）：**一种集成电路芯片**，将多个核心（如处理器核心）、存储器（如RAM、闪存等）、图形处理单元（GPU）、输入输出接口、外设控制器和其他功能模块集成在同一芯片上。

### Arm生态下的相关名词

**Arm Instruction Set Architecture**：一系列精简指令集架构 (Reduced Instruction Set Architectures, RISC) 

使用一种是ARMv来表示的都是指令集架构，v后的数字代表不同的版本。

- ARMv9：目前(2023)最新的Arm指令集架构，新增了可扩展SIMD vector (SVE2)和矩阵操作 (SME/SME2)等。
- [ARMv8](https://en.wikichip.org/wiki/arm/armv8): 2011年发布，引入了新的64位操作，通过AArch64 execution state来执行64位操作，AArch32 execution state来兼容32位操作（后面会介绍）。
  - ARMv8.1, ARMv8.2, ARMv8.3... 小版本修改
- ARMv7: 2004年发布，同年也引入了profiles的概念（后面会介绍）。该指令集架构采用了Thumb-2技术（后面会介绍），支持Neon技术扩展。
- ...
- ARMv4：32位处理器架构，支持基本的32位寻址和数据操作。
  - ARMv4T，ARMv4的拓展，引入了Thumb指令集

---

**[ARM architecture profiles](https://developer.arm.com/documentation/dui0471/m/key-features-of-arm-architecture-versions/arm-architecture-profiles)**：Arm architecture定义了不同的架构特征用于不同的场景。总共有A, R, M三种profiles，分别是：“Application Profile“--通过MMU支持虚拟内存，手机、电脑、服务器上基本上用的都是这种profile；“Real-time profile”--用于有实时系统要求的应用；“Microcontroller profile”--经常集成到FPGA中，用于低功耗要求的应用。

- Armv9-A/Armv8-A：Armv9/Armv8架构的A profile
- Armv8-R: Armv8架构的R profile

---

**ARM instruction set**

- A64：Armv8-A引入的64位指令集，是32位的定长指令，但它支持64位的寻址和数据操作。
- A32=ARM：ARMv7-A和之前版本的32位指令集，是32位的定长指令，仅支持32位的寻址和数据操作。
- T32=Thumb-2: 一种混合指令集模式，结合了16位Thumb指令集和32位ARM指令集，由ARMv6t2引入。T32指令集旨在提供更好的代码密度和较低的功耗。
- T16=Thumb: 仅使用16位指令，由ARMv4t引入，用于减小代码大小和功耗。

---

**Execution states**

- AArch64: 由Armv8引入，用来支持64位寄存器（31 个通用寄存器、专用 64位SP、只能通过分支或异常写入的64位PC以及一个零值伪寄存器）和寻址。只支持运行A64 instruction set。

  > 由于历史原因， [macOS](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms), [Windows](https://learn.microsoft.com/en-us/windows/arm/overview), [the Linux kernel](https://lore.kernel.org/lkml/CA+55aFxL6uEre-c=JrhPfts=7BGmhb2Js1c2ZGkTH8F=+rEWDg@mail.gmail.com/)(这个链接很搞笑hhh)等会使用**arm64**/**ARM64**来指代AArch64。arm64 Linux指增加了AArch64支持的Linux kernel

- AArch32: 用来指代之前的32位的功能（15个32位通用寄存器，没有专用SP，PC可写），运行A32 and T32 instruction sets

---

**Arm-designed microarchitectures**：Arm不仅仅设计Arm architecture，还实现这些架构，这些实现的架构被称为microarchitecture， 即ARM生产的处理器架构。

ARM从2004年以后对处理器架构的命名做了重大修改。也就是从ARM11处理器之后，也就是从ARMv7架构开始引入Cortex 这个名字。和Arm的指令集架构一样，Arm的处理器也分成A、R、M三个系列，分别代表三种不同的应用领域。

| 指令集架构 | 处理器家族                                                   |
| ---------- | ------------------------------------------------------------ |
| ARMv1      | [ARM1](https://zh.wikipedia.org/w/index.php?title=ARM1&action=edit&redlink=1) |
| ARMv2      | [ARM2](https://zh.wikipedia.org/w/index.php?title=ARM2&action=edit&redlink=1)、[ARM3](https://zh.wikipedia.org/w/index.php?title=ARM3&action=edit&redlink=1) |
| ARMv3      | ARM6、[ARM7](https://zh.wikipedia.org/wiki/ARM7)             |
| ARMv4      | [StrongARM](https://zh.wikipedia.org/wiki/StrongARM)、[ARM7TDMI](https://zh.wikipedia.org/wiki/ARM7TDMI)、[ARM9](https://zh.wikipedia.org/wiki/ARM9)TDMI |
| ARMv5      | [ARM7EJ](https://zh.wikipedia.org/w/index.php?title=ARM7EJ&action=edit&redlink=1)、[ARM9E](https://zh.wikipedia.org/w/index.php?title=ARM9E&action=edit&redlink=1)、[ARM10E](https://zh.wikipedia.org/w/index.php?title=ARM10E&action=edit&redlink=1)、[XScale](https://zh.wikipedia.org/wiki/XScale) |
| ARMv6      | [ARM11](https://zh.wikipedia.org/w/index.php?title=ARM11&action=edit&redlink=1)、[ARM Cortex-M](https://zh.wikipedia.org/wiki/ARM_Cortex-M) |
| ARMv7      | [ARM Cortex-A](https://zh.wikipedia.org/w/index.php?title=ARM_Cortex-A&action=edit&redlink=1)、[ARM Cortex-M](https://zh.wikipedia.org/wiki/ARM_Cortex-M)、[ARM Cortex-R](https://zh.wikipedia.org/w/index.php?title=ARM_Cortex-R&action=edit&redlink=1) |
| ARMv8      | Cortex-A35、Cortex-A50系列、Cortex-A70系列、Cortex-X1        |
| ARMv9      | Cortex-A510、Cortex-A710、Cortex-X2                          |

- Cortex/Neoverse系列，它们都是 Arm设计的Arm微架构，wiki上整理了一个最近的[ARM指令集架构-微架构-产品](https://en.wikipedia.org/wiki/Template:Application_ARM-based_chips)对应的表格

  ![](/assets/img/in-post/2023-06-06-ARM_chips.png)

- [ARM指令架构集跟处理器架构家族的关系](https://en.wikipedia.org/wiki/ARM_architecture_family#Cores)

  ![](/assets/img/in-post/2023-06-06-ARM_arch_core.png)

​	

## Reference
1. [Disambiguating Arm, Arm ARM, Armv9, ARM9, ARM64, AArch64, A64, A78, ...](https://nickdesaulniers.github.io/blog/2023/03/10/disambiguating-arm/)

2. [Linker notes on AArch64](https://maskray.me/blog/2023-03-05-linker-notes-on-aarch64)

3. [Linker notes on AArch32](https://maskray.me/blog/2023-04-23-linker-notes-on-aarch32)

4. [ARM 架构家族史](https://broadgeek.com/2021/11/21/d179/)

5. [ARM architecture family wiki](https://en.wikipedia.org/wiki/ARM_architecture_family)

6. [Comparison of ARM processors](https://en.wikipedia.org/wiki/Comparison_of_ARM_processors)

7. [List of ARM processors](https://en.wikipedia.org/wiki/List_of_ARM_processors)

8. [ARM Overview](https://wiki.osdev.org/ARM_Overview)

   