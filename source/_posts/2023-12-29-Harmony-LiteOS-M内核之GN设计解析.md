---
title: OpenHarmony-LiteOS-M内核之GN设计解析
date: 2023-12-26
tags:
categories:
- [LiteOS]
description: 要基于OpenHarmony源码工程进行LiteOS-M开发，那么熟悉了解其编译构建过程是必要的；本文以STM32的LiteOS移植工程为例，剖析其业务代码、内核代码的调用编译构建流程
---


[GN与Ninja入门](/2023/12/28/GN和ninja的区别、联系和应用开发)


## LiteOS-M内核之GN设计


### 编译入口

- LiteOS-M的编译入口定义在`//build/lite/components/kernel.json`，目标为`//kernel/liteos_m:kernel`，其在`//kernel/liteos_m/BUILD.gn`中定义
- 在产品的config.json配置文件中进行如下编写，表示要编译构建LiteOS-M内核
  ```
    {
    "subsystem": "kernel",
    "components": [
    { "component": "liteos_m", "features":[] }
    ]
    },
  ```



