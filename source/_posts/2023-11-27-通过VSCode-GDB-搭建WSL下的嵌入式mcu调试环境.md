---
title: 通过VSCode + GDB 搭建WSL下的嵌入式mcu调试环境
date: 2023-11-28 12:00:00
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

- windows下J-Link软件安装
- WSL
- VSCode



# 操作步骤



launch.json文件
```
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
            "program": "${workspaceFolder}/build/app.elf",
            "miDebuggerPath": "/home/chenjd/harmony/gcc-arm-none-eabi-9-2019-q4-major/bin/arm-none-eabi-gdb",
            "miDebuggerServerAddress": "localhost:2331",
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb"
        },
    ]
}
```

# 补充说明



