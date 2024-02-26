---
title: Linux内核移植概述--基于I.MX6ULL
date: 2023-12-20
tags:
categories:
- [Linux]
- [I.MX6ULL]
description: 个人对Linux内核移植流程的总结、浅薄理解
---

## 基本知识概要



## Linux内核移植步骤简述

- 在Linux内核文件中，找到参考的板子配置文件——一般是半导体厂商基于其开发板的配置文件
- 以此份配置文件为模板，编译生成对应的zImage和.dtb文件
- 应用厂商开发板的zImage和.dtb文件，在自己的板子上启动Linux内核，查看能否正常启动
- 一般情况下，参考官方开发板设计的硬件，都能够正常启动
- 根据硬件与开发板的不同，修改相应的驱动；官方Linux内核一般会提供好各种相应的驱动，我们只需要修改设备树的一些配置信息即可




## Linux内核移植详解步骤——I.MX6ULL

### 在Linux内核中添加私有板子配置

- 复制一份xxx_defconfig配置文件，并重命名，将其作为私有的默认配置文件
  - 通过在工程根目录下执行`make xxx_defconfig`配置Linux内核选项
- 添加私有板子对应设备树文件
  - 创建私有.dts文件：`cd arch/arm/boot/dts` -> `cp imx6ull-14x14-evk.dts imx6ull-xxx-emmc.dts`
  - 添加至编译：`vim arch/arm/boot/dts/Makefile`，在`dtb-$(CONFIG_SOC_IMX6ULL)`配置项中加入`imx6ull-xxx-emmc.dts`，使得编译设备树时，会从imx6ull-xxx-emmc.dts编译出对应的imx6ull-xxx-emmc.dtb文件
- 编译得到zImage镜像文件和imx6ull-xxx-emmc.dtb设备树文件
  - 清理工程：`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean`
  - 配置Linux内核：`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- xxx_defconfig`
  - 可选，打开图形化配置界面：`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig`
  - 执行编译：`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- all -j16`
    - 可选的，将上述指令添加到脚本中，进行自动化执行，避免重复的输入
  - **编译完成后，在`arch/arm/boot`下生成zImage，在`arch/arm/boot/dts`下生成imx6ull-xxx-emmc.dtb文件**

### 内核测试

- 启动板子，在u-boot命令模式中，加载所编译的zImage和.dtb文件
  - `tftp 80800000 zImage`
  - `tftp 83000000 imx6ull_xxx_emmc.dtb`
  - `bootz 80800000 - 83000000`

- 出现以下相似内容，表示Linux内核启动成功
```
...
Booting Linux on physical CPU 0x0
Linux version 4.1.15 ...
...
```

#### 一些疑惑解答
- 在boot加载Linux内核时，所使用到的地址80800000和83000000是怎么来的？是固定地址？
  - 地址80800000是内核映像的起始地址，boot使用memcpy()将内核映像从存储设备加载到内存地址80800000中
  - 地址83000000是内核栈的起始地址，其是内核存储临时数据的区域，boot将内核栈初始化为地址83000000
    - 设备树是ARM平台上的重要配置文件，其描述了系统资源的布局和属性；内核启动时需要访问设备树，以根据其参数初始化硬件设备
    - 83000000是ARM平台上u-boot内核栈的常用起始地址，位于内存的低端，将设备树加载在此地址 -> 确保内核在访问设备树时不会发生内存访问冲突
  - 在u-boot中，此二地址可以通过配置文件修改
    ```
    # configs/<board>defconfig
    CONFIG_SYS_LOAD_ADDR = 0x80800000
    CONFIG_SYS_INIT_SP_ADDR = 0x83000000
    ```


## Linux内核顶层Makefile详解


### 顶层Makefile布局

版本号

MAKEFLAGS 变量

命令输出

静默输出

设置编译结果输出目录

代码检查

模块编译

设置目标架构和交叉编译器

调用 scripts/Kbuild.include 文件

交叉编译工具变量设置

头文件路径变量

导出变量

### make xxx_defconfig 过程


## Linux内核启动流程




## 参考站点
