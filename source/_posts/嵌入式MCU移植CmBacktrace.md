---
title: 嵌入式单片机移植CmBacktrace
date: 2024-02-01
tags:
categories:
- [嵌入式]
- [MCU]
description: CmBacktrace是针对Cortex-M内核芯片提供程序运行异常时的堆栈回溯工具/中间件，将其移植到单片机中，可以在程序异常时打印相应的故障信息包括函数调用链、变量值、寄存器值等，大大缩短bug排查时间。本文介绍CmBacktrace的移植适配过程、原理解析、实际应用以及一些扩展问题思考等。
---


工程源码移植见：[CmBacktrace源码移植教程-github](https://github.com/Jindu-Chen/CmBacktrace_Adapt)

官方源码地址 : https://github.com/armink/CmBacktrace


**未完待续...**


## 前言

暂略

## 移植适配与应用

- 将相关源码加入工程

- 适配信息打印接口，如串口，将`printf`输出重定向至串口

- 注释掉原有的HardFault中断服务函数，避免与CmBacktrace源码的HardFault中断服务函数冲突

- 链接脚本中也要增加适配其宏定义（如果没有的话），如栈的起始结束地址`_bss`、代码段`__text_end`的起始结束地址等

- 在程序初始化时，调用CmBacktrace的初始化接口


### 程序报错及解析

当程序运行错误时，CmBacktrace会轮询栈空间，将其中指向的内容（为falsh代码段区间的函数地址部分）打印出来，此即函数地址调用链。

用户使用 addr2line 工具，配合 elf 文件，将函数地址转换为函数名，则可知函数调用链。

如：`addr2line -e app.elf -a -f 080154c2 0800a3b2 08009092`


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


核心代码如下：
```c
size_t cm_backtrace_call_stack(uint32_t *buffer, size_t size, uint32_t sp) {
    uint32_t stack_start_addr = main_stack_start_addr, pc;
    size_t depth = 0, stack_size = main_stack_size;
    bool regs_saved_lr_is_valid = false;

    ......

    /* copy called function address */
    for (; sp < stack_start_addr + stack_size; sp += sizeof(size_t)) {
        /* the *sp value may be LR, so need decrease a word to PC */
        pc = *((uint32_t *) sp) - sizeof(size_t);
        /* the Cortex-M using thumb instruction, so the pc must be an odd number */
        if (pc % 2 == 0) {
            continue;
        }
        /* fix the PC address in thumb mode */
        pc = *((uint32_t *) sp) - 1;
        if ((pc >= code_start_addr + sizeof(size_t)) && (pc <= code_start_addr + code_size) && (depth < CMB_CALL_STACK_MAX_DEPTH)
                /* check the the instruction before PC address is 'BL' or 'BLX' */
                && disassembly_ins_is_bl_blx(pc - sizeof(size_t)) && (depth < size)) {
            /* the second depth function may be already saved, so need ignore repeat */
            if ((depth == 2) && regs_saved_lr_is_valid && (pc == buffer[1])) {
                continue;
            }
            buffer[depth++] = pc;
        }
    }

    return depth;
}
```
以上函数传入 栈顶SP 地址，而后主要进入一个循环，从栈顶往栈底回溯，每4个字节读取一次，**如果其存放的数据是属于代码区的**（说明此是一个函数地址，大概率是函数调用链的地址，当然也可能是函数中的一个函数指针），则将其地址存入数组，并继续回溯。


## 注意点与扩展思考


### 其它如Cortex-A内核如何回溯的？


### RTOS与裸机下的CmBacktrace有何不同？

裸机：只需关注当前执行流的堆栈跟踪，不需要考虑多线程或任务切换

RTOS：需要考虑多线程或任务切换，需要考虑线程切换时的堆栈跟踪。要捕获当前运行任务的堆栈回溯信息，还要获取其他任务的堆栈信息。涉及到与 RTOS 内核的API交互，获取当前任务指针、任务堆栈基址、栈顶指针等信息


### 程序崩溃跳转至HardFault状态了，为何还能正常控制外设如DMA-UART打印故障信息？


## 参考站点


- [STM32上的backtrace原理与分析](https://zhuanlan.zhihu.com/p/512902251)
