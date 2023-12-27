---
title: 国民技术N32G452芯片移植RT-Thread-Nano
date: 2023-11-28 11:37:51
tags:
categories:
- [RTOS]
- [MCU]
description: 记录移植rt-thread的过程，以及一些会导致移植程序运行异常的注意点
---




## 移植rt-thread，程序编译成功但无法运行的注意点

### rt-thread的内存布局之堆的初始化

- rt-thread的堆初始化，以__bss_end为起始地址、以芯片的内存顶部地址为结束地址，其内存布局顺序有别于常规的`rw-data -> bss -> heap -> stack`顺序
- 因此需要关注rt-thread链接脚本的内存布局与其的堆初始化是否对应无误


