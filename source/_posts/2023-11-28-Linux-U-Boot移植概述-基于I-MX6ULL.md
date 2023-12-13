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