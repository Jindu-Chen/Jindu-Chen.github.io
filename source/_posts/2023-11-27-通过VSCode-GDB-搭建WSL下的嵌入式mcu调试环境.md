---
title: 通过VSCode + GDB 搭建WSL下的嵌入式mcu调试环境
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
description: 简述在windows中，使用VSCode登陆WSL，实现在WSL环境下通过GDB + J-Link的GDB调试方法
---


# 软件环境准备

- Windows下J-Link软件安装
- WSL
- VSCode
  - 插件安装：C/C++, WSL, 



# 操作步骤

1. 将ARM仿真器与目标嵌入式设备连线
2. 在windows下J-Link的安装目录下，找到JLinkGDBServer.exe，点击打开，选择目标调试嵌入式设备


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

# 补充说明



