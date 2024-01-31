---
title: Linux U-Boot移植概述-基于I-MX6ULL
date: 2023-12-05
tags:
categories:
- [Linux]
- [I.MX6ULL]
description: 记录下基于I.MX6ULL的u-boot移植流程，作为U-Boot移植入门的简单总结回顾，不涉及深层知识
---


## U-Boot简介

是一段用于启动Linux内核的bootloader裸机程序，其主要作用是：初始化必要的外设，将Linux内核从源存储拷贝到内存中，最终启动内核


### 基于I.MX6ULL的U-Boot的一些开发概念

1. I.MX通过置位BOOT引脚进行配置选择从SD卡、EMMC、NAND FLASH等等的启动方式
<br>

2. 生成的U-Boot.bin文件需要加上头部（IVT、DCD等数据），然后再烧录到启动存储中
<br>

3. I.MX上电后先跑芯片内置BootROM程序，把U-Boot加载到内存后，再跳转执行U-Boot
<br>

4. U-Boot提供命令行交互界面，其可以选择系统从网络启动、从EMMC启动等等
<br>

5. 通常情况下，是参考原厂开发板做硬件，而后在原厂提供的BSP包上做修改
<br>



### U-Boot在I.MX6ULL中的启动流程简述


## 补充

### u-boot对内核的参数传递示例

```
/* boot从网络启动 */
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.14.18:/home/chenjd/linux/nfs/rootfs,proto=tcp rw ip=192.168.14.99:192.168.14.18:192.168.14.1:255.255.255.0::eth0:off'
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
saveenv
boot
```

```
/* boot从emmc启动 */
setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
setenv bootcmd 'fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
saveenv
boot
```