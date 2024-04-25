---
title: 嵌入式单片机移植CmBacktrace
date: 2024-01-01
tags:
categories:
- [嵌入式]
- [MCU]
description: CmBacktrace是针对Cortex-M内核芯片提供程序运行异常时的堆栈回溯工具/中间件，将其移植到单片机中，可以在程序异常时打印相应的故障信息包括函数调用链、变量值、寄存器值等，大大缩短bug排查时间。本文介绍CmBacktrace的移植适配过程、原理解析、实际应用以及一些扩展问题思考等。
---


工程源码移植见：[CmBacktrace源码移植教程](https://github.com/Jindu-Chen/CmBacktrace_Adapt)


## 前言


## 移植适配与应用


**将相关源码加入工程**

**适配信息打印接口，如串口**

注释掉原有的HardFault中断服务函数

链接脚本中也要增加适配其宏定义（如果没有的话），如栈的起始结束地址`_bss`、代码段`__text_end`的起始结束地址等

**调用其初始化接口**

Demo应用

### 程序报错及解析

addr2line -e app.elf -a -f 080154c2 0800a3b2 08009092


### addrline2工具

`addr2line` 是一个用于将程序地址转换为源代码文件名及行号的工具，其属于`GNU Binutils`工具包中的一部分。
> `GNU Binutils`是GNU项目的一部分，是一套用于创建、修改和分析二进制文件的工具集合。其包括：汇编器（as）、链接器（ld）、反汇编器（objdump）、调试器、库管理器、对象查看器、地址转换工具（addr2line）等。
> 如WSL安装了GCC，则会包含整个GNU Binutils工具集。 如安装了`arm-none-eabi-`工具链，则会包含`arm-none-eabi-addr2line`。



## 原理解析

要理清CmBacktrace的实现原理，首先要**了解`ARM Cortex-M`内核的堆栈布局与管理、以及该内核对异常的处理流程**。

而CmBacktrace即是根据C语言的栈帧结构、以及Cortex-M内核的工作/异常处理的原理，当触发异常时，对堆栈进行解析实现回溯，而后将信息输出至终端，如串口。

如何通过cortex-m4内核的cpu寄存器值分析当前的栈帧调用链


### 堆栈布局与管理

### 异常处理与上下文保存

### CmBacktrace的工作原理


## 注意点与扩展思考

### 其它如Cortex-A内核如何回溯的？

### RTOS下移植CmBacktrace有何不同？

### 程序崩溃跳转至HardFault状态了，为何还能正常控制外设如DMA-UART打印故障信息？

## 参考站点



