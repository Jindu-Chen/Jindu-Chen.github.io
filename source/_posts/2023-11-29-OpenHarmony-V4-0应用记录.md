---
title: OpenHarmony-V4.0使用记录
date: 2023-11-29 12:00:01
tags:
categories:
- [LiteOS]
description: 参考社区的STM32之Ninja编译方式的工程范例，移植搭建基于LiteOS的N32WB452工程，记录过程中碰到的一些编译纠错、代码编写及构建等相关信息，以供检索回溯
---



## 编译纠错


- 提示以下信息，
    ```
    ../../../kernel/liteos_m/kernel/src/los_init.c:54:10: fatal error: los_exc_info.h: No such file or directory
    54 | #include "los_exc_info.h"
        |          ^~~~~~~~~~~~~~~~
    compilation terminated.
    [43/265] gcc cross compiler obj/kernel/liteos_m/kal/posix/src/posix.pthread_cond.o
    ```



