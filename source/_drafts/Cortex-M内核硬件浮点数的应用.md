---
title: Cortex-M4F内核硬件浮点单元应用
date:
tags:
categories:
- [MCU]
- [嵌入式]
description: 如何开启FPU
---


## 前言

经常性的在链接选项中加入硬件浮点FPU的连接后，会导致编译失败

只要定义全局宏定义标识符__FPU_PRESENT 以及__FPU_USED 为 1，那么就可以开启硬件 FPU

## Gcc + Make环境下开启FPU

-D__FPU_PRESENT=1 \
-D__FPU_USED=1

-mfpu=fpv4-sp-d16 -mfloat-abi=hard \


## Keil环境下开启FPU

使能宏定义，Keil界面勾选


## 参考站点

