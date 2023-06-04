---
layout:     post
title:      "GDB常用指令"
date:       2023-06-04 21:00:00
author:     zxy
categories: ["Coding", "GDB"]
tags: ["debugging", "tools"]
mathjax: true
---

GNU调试器（英语：GNU Debugger，缩写：gdb）是一个由GNU开源组织发布的、UNIX/LINUX操作 系统下的、基于命令行的、功能强大的程序调试工具。
总的来说，gdb有以下4个功能：
- 启动程序，并指定可能影响其行为的所有内容
- 使程序在指定条件下停止
- 检查程序停止时发生了什么
- 更改程序中的内容，以便纠正一个bug的影响

gdb需要被调试的程序中含有debugging的信息，因此在用gcc编译文件时，需要增加`-g`选项来提供gdb需要的调试信息。

```shell
gcc [other flags] -g <source files> -o <output file>
```
使用`objcopy`来分离调试信息
```shell
objcopy --only-keep-debug <output file> xxx.debug
```
使用`strip`来删除原文件里的调试信息
```shell
strip --strip-debug --strip-unneeded <output file>
```

## GDB常见指令

### 一般设置
(gdb) start：单步执行，运行程序，停在第一执行语句
(gdb) run：重新开始运行文件，简写r
- run-text：加载文本文件
- run-bin：加载二进制文件

(gdb) add-symbol-file filename address：从filename中读取额外的symbol table信息，并加载到指定address，filename被动态加载到address时可以使用


### 断点设置相关指令
(gdb) break 设置断点

- 断在具体的函数 `break func`
- 断在某一行 `break filename:num`
- 断在某一程序地址处 `b *address`
- 条件断点 `break filename:num if i >= ARRAYSIZE`

(gdb) watch my_var 在变量上设置断点，当该变量被修改(write)时断住程序
(gdb) rwatch *<memory address> 在指定内存上设置断点，当该内存被访问(read)时断住程序
- `rwatch *(int *)0xfeedface` 指定watch的内存长度

(gdb) awatch *0xfeedface 在指定内存上设置断点，当该内存被访问/修改(read/write)时断住程序
(gdb) info breakpoints 显示所有声明的断点
(gdb) delete 删除断点

### 程序执行流程相关常用指令
(gdb) continue：运行程序直到下一个断点，简写c
(gdb) next：单步调试（逐过程，函数直接执行，把函数当成一条指令执行），简写n
- nexti：在汇编指令层面的next ，简写ni

(gdb) step：单步调试（逐过程，进入函数执行），简写s
- stepi：在汇编指令层面的step ，简写si

(gdb) finish：结束当前函数，返回到函数调用点
(gdb) until lineNo： 跳出loop，运行程序直到lineNo

### 打印当前程序状态相关常用指令
(gdb) backtrace：查看函数的调用的栈帧和层级关系，简写bt
(gdb) frame：切换函数的栈帧，简写f
(gdb) print：打印值及地址，简写p
- 以16进制的方式打印变量值 `p/x my_var`
- 打印指针指向的内存值 `p *p`

(gdb) info：查看函数内部局部变量的数值，简写i
- 查看寄存器的值i register xxx 
- i registers

(gdb) x: 打印指定地址的数据
- 以16进制打印pc指向的地址值 `x/x $pc`
- 打印pc指向的5条指令内容 `x/5i $pc`
- 打印testArray中的字符串 `x/s testArray`

(gdb) show：展示debugger的信息
(gdb) display：追踪查看具体变量值
(gdb) layout <xxx>：打开TUI(Text User Interface)模式，在debug的时候可以在额外窗口中展示源码内容
- layout asm： 展示汇编和命令行窗口
- layout src： 展示源码和命令行窗口
- layout regs：展示寄存器+源码+命令行窗口
- layout split：展示汇编+源码+命令行窗口

(gdb) focus <name>：当有多个窗口时，切换窗口
- focus [next|prev]: activate next/prev window for scrolling
- focus cmd: activate command window for scrolling，上下键可以用来切换gdb命令

### 其他指令
(gdb) help <command>：查看command相关用法
(gdb) set logging file xxx.txt：gdb输出打印到指定文件
(gdb) set logging [on|off]：打开/关闭logging
(gdb) set logging overwrite [on|off]


## GDB脚本的使用
1. 变量
- 访问程序参数信息 $argc, $arg0, $arg1
- 自定义变量 `set $my_var=0`

2. 用户自定义命令
```shell
define commandName  
    statement  
    ......  
end 
```
其中statement可以是任意gdb命令，比如下面的一个自定义命令
```shell
define adder
  set $i = 0
  set $sum = 0
  while $i < $argc
    eval "set $sum = $sum + $arg%d", $i
    set $i = $i + 1
  end
  print $sum
end
```
3. 使用`document commandname`来说明用户**已经定义**的命令commandname的功能，使用help命令可以获得该命令说明。
```shell
document adder  
    add up all arguments 
end 
```
4. 给breapoint设置自定的command
```shell
# at entry point - cmd1
b main
commands 1
  print argc
  continue
end
```
5. 脚本文件的注释以#开头
6. gdb中执行脚本
- 在运行的gdb中要使用source命令，例如：`source xxx.gdb`
- 在gdb启动时，指定运行的脚本`gdb <exe_file> --command=xxx.gdb`
- 在`.gdbinit`中添加自定义的命令，gdb会在启动之后执行`.gdbinit`

## GDB插件
1. [gef](https://github.com/hugsy/gef) 强烈推荐
2. [pwndbg](https://github.com/pwndbg/pwndbg)


## GDB常见报错
1. Cannot access memory at address xxx
2. To be continued...

## QEMU与GDB的交互
QEMU支持通过 gdb 的远程连接工具（“gdbstub”）使用 gdb。
1. QEMU使用tcp协议的1234端口与GDB进行通信，以`qemu-system-riscv64`为例：
`qemu-system-riscv64 -nographic -machine virt -kernel vmlinux -bios default -S -s`
- `-s`选项表示QEMU在TCP协议的1234端口上监听来自 gdb 的传入连接
- `-S`选项表示在gdb接入之前，guest不会被启动；即需在gdb中启动程序

2. 在GDB中，通过`target remote localhost:1234`接入QEMU

> 注意：GDB只能通过**虚拟地址**访问QEMU的内存

## Useful GDB Reference website

1. [100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/)
2. [Debugging with GDB](https://sourceware.org/gdb/onlinedocs/gdb/index.html)
3. [Extending GDB](https://sourceware.org/gdb/onlinedocs/gdb/Define.html#Define)
4. [gdb-customize it the way you want](http://www.haifux.org/lectures/210/index.html)
5. [GNU GDB Debugger Command Cheat Sheet](http://www.yolinux.com/TUTORIALS/GDB-Commands.html)

## Reference

1. [https://www.cs.umd.edu/~srhuang/teaching/cmsc212/gdb-tutorial-handout.pdf](https://www.cs.umd.edu/~srhuang/teaching/cmsc212/gdb-tutorial-handout.pdf)
2. [https://cs.brown.edu/courses/cs033/docs/guides/gdb.pdf](https://cs.brown.edu/courses/cs033/docs/guides/gdb.pdf)
3. [https://www.qemu.org/docs/master/system/gdb.html](https://www.qemu.org/docs/master/system/gdb.html)
4. [https://stackoverflow.com/questions/866721/how-to-generate-gcc-debug-symbol-outside-the-build-target](https://stackoverflow.com/questions/866721/how-to-generate-gcc-debug-symbol-outside-the-build-target)
5. [https://stackoverflow.com/questions/10748501/what-are-the-best-ways-to-automate-a-gdb-debugging-session](https://stackoverflow.com/questions/10748501/what-are-the-best-ways-to-automate-a-gdb-debugging-session)
5. [https://hellogcc.github.io/blog/2014-07-02-%E5%A6%82%E4%BD%95%E5%86%99gdb%E5%91%BD%E4%BB%A4%E8%84%9A%E6%9C%AC/index.html](https://hellogcc.github.io/blog/2014-07-02-%E5%A6%82%E4%BD%95%E5%86%99gdb%E5%91%BD%E4%BB%A4%E8%84%9A%E6%9C%AC/index.html)