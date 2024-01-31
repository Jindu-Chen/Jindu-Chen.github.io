---
title: 通过VSCode + GDB 搭建WSL下的嵌入式MCU调试环境
date: 2023-12-04
tags:
- [WSL]
- [VSCode]
- [GDB]
- [J-Link]
categories:
- [嵌入式]
- [环境搭建]
- [WSL]
description: 简述在Windows中，使用VSCode登陆WSL，实现在WSL环境下通过GDB + J-Link的STM32调试方法
---


## 前言

此前在Keil集成IDE环境下开发，编辑编译调试一体化，无须关心过多底层事项。但由于各种原因，需要经常在Linux环境开发，因此也就顺势把工作中MCU开发的相关项目搬到WSL下了，本人使用的是`arm-none-eabi-gcc`工具链，构建方式为`Make`，编辑开发方式为 VSCode + Remote Explorer，调试器硬件为 ARM V9 仿真器，调试接口为 SWD。

其实本人一直很少使用在线调试，一般是通过 串口 信息打印作为维测方法，只要在模板工程成功运行之后，基本就不需要在线调试了。但初次移植、或者搭建工程，程序异常跑飞，甚至串口都无信息输出的情况下，或许就需要在线调试查找原因了。

本文介绍一种简单的 通过 VSCode + J-Link 调试方法，此过程基本不涉及任何 GDB 命令，纯粹是 VSCode 集成的 GDB 界面操作。



## 软件环境准备

- Windows下J-Link软件安装
- WSL
- VSCode
  - 插件安装：C/C++, WSL, 

以上，前提条件是需要先搭建好在 WSL 下的编译开发环境，主要为`arm-none-eabi-`工具链安装。

配置调试的思路为：
- 在Windows下打开`JLinkGDBServer.exe`，配置将调试器成功连接目标板，J-Link本地流量端口通常为 2331
- 使用 VSCode 登陆 WSL，应用其`Run and Debug`功能，主要为配置`launch.json`文件，包括`*.elf`文件路径、`arm-none-eabi-gdb`命令路径等

完成上述两点操作，即可在 VSCode 环境下开启 GDB 调试了


## 操作步骤

1. 将ARM仿真器与目标嵌入式设备连线
2. 在Windows下J-Link的安装目录下，找到`JLinkGDBServer.exe`，点击打开，选择目标调试嵌入式设备


![GDB Server config](../pictures/2023-12-04%20J-Link.png)
<br>
![J-Link GDB Server](../pictures/2023-12-04%20j-link%20GDB.png)

3. 在Windows下使用VSCode登陆WSL，打开源码所在工程文件夹，作为VSCode工作区
4. 点击"Run and Debug"，选择GDB（C）调试，并提示需要配置launch.json文件，配置完成即可开始调试

```
#launch.json文件
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug with J-Link",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/app.elf",  // .elf文件路径
            "miDebuggerPath": "/home/chenjd/harmony/gcc-arm-none-eabi-9-2019-q4-major/bin/arm-none-eabi-gdb",   // WSL下的arm-none-eabi-gdb工具链路径
            "miDebuggerServerAddress": "localhost:2331",        // J-Link端口，通常为2331
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb"
        },
    ]
}
```

## 参考站点

- [gdb调试stm32的技巧](https://hhuysqt.github.io/gdb%E8%B0%83%E8%AF%95%E6%8A%80%E5%B7%A7/)

