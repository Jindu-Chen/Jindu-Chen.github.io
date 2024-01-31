---
title: ARM架构及相关概念
date: 2023-12-04
tags:
categories:
description: 记录ARM架构体系、指令集、寄存器等相关的一些概念知识
---


## ARM指令集

### Thumb指令集
其是ARM指令集的一个子集，16位的代码宽度


## 哈佛架构与冯诺依曼架构



## CPU寄存器


## Cortex-M内核

在 Cortex-M 内核中，中断和异常都由 NVIC（Nested Vectored Interrupt Controller）来管理。NVIC 是一个硬件模块，负责处理来自外部或内部的所有中断和异常
<br>

中断是指来自外部或内部事件的请求，如定时器超时、外部引脚中断
<br>

异常是CPU执行程序发生的错误处理（如：内存访问非法、除零）或者主动软件触发异常，包括PendSV、HardFault等异常
<br>


## 杂项

#### glue_7、glue_7t 和 eh_frame

- glue_7 和 glue_7t 是 ARM 架构中用于将 ARM 代码和 Thumb 代码粘合在一起的代码。

- ARM 架构支持两种指令集：ARM 指令集和 Thumb 指令集。ARM 指令集是 32 位指令集，而 Thumb 指令集是 16 位指令集。为了兼容两种指令集，ARM 架构中引入了一种称为 Thumb 嵌入 的机制。

- Thumb 嵌入允许 ARM 代码中嵌入 Thumb 指令，反之亦然。当 ARM 代码中嵌入 Thumb 指令时，连接器会生成 glue_7 或 glue_7t  section 来将 ARM 代码和 Thumb 代码粘合在一起。

- eh_frame 是异常处理框架（Exception Handling Frame）的缩写。异常处理框架是用于支持异常处理的一种结构。当程序发生异常时，异常处理框架会保存程序的运行状态，并将程序转到异常处理程序。

- 异常处理框架通常包含以下信息：
    程序的堆栈指针（SP）
    程序的返回地址（LR）
    异常发生时的寄存器值

- eh_frame section 包含了异常处理框架的信息。当程序发生异常时，连接器会将 eh_frame section 加载到内存中，以便异常处理程序可以访问异常处理框架的信息
