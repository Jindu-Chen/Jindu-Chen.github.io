---
title: OpenHarmony-V4.0-LiteOS-M鸿蒙轻内核开发调试
date: 2023-12-20
tags:
categories:
- [OpenHarmony]
- [LiteOS]
description: 参考社区的STM32之Ninja编译方式的工程范例，移植搭建基于LiteOS的N32WB452工程，记录其过程中的一些编译纠错、代码功能编写及构建、XTS子系统测试等相关信息，以供检索回溯
---

## 基于命令行开发环境搭建

复述官方文档，仅作参考，当在源码根目录下键入`hb help`，命令执行成功，基本完成前期环境的搭建

- [安装库和工具集](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/quick-start/quickstart-pkg-install-package.md)
- 下载OpenHarmony-V4.0-Release全量代码，可通过git拉取或者下载压缩包解压
  - 在源码根目录下执行prebuilts脚本，安装编译器及二进制工具--`bash build/prebuilts_download.sh`
- [安装编译工具](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/quick-start/quickstart-pkg-install-tool.md)
- 开发基于Cortex-M内核工程，需要安装`arm-none-eabi-`工具链，并声明工具链路径至`~/.bahsrc`。此处不过多赘述
- 键入`hb set`选择目标开发设备，键入`hb build`执行编译，编译成功则表示环境构建无误

## 轻量系统开发的前期流程

- 参考Gitee的OpenHarmony项目搭建命令行开发环境，熟悉基本的操作流程
- 熟悉GN、ninja的文件编写、和源码构建过程，了解子系统、组件等基本概念
  - 子系统是逻辑概念，其具体由对应的组件构成
  - 组件是可复用的软件单元，其包含源码、配置文件、资源文件和编译脚本
  - 通过配置文件、编译链接脚本，使得同一组件可以在不同的目标架构上运行
- 参考社区的移植流程，根据其工程范例进一步了解轻量系统的构建开发原理


## 目标工程的编译构建流程

- hb set
- hb build如何调用构建

## 子系统添加和构建





## 编译纠错

- 编译成功，但生成的可执行bin文件明显偏小，查看Kconfig配置文件是否有选项没写对，导致的没编译链接

- 提示以下信息：进入`kernel/liteos_m`，然后make menuconfig，选中兼容项，失能musl库，改为new lib。
```
[OHOS ERROR] arm-none-eabi-gcc: fatal error: /home/jd_chen/Downloads/gcc-arm-none-eabi-10-2020-q4-major/bin/../lib/gcc/arm-none-eabi/10.2.1/../../../../arm-none-eabi/lib/nano.specs: attempt to rename spec 'link' to already defined spec 'nano_link'
[OHOS ERROR] compilation terminated.
```



- 出现如下编译错误，在vfs_fs.c添加头文件<stdbool.h>包含即可
    ```
    [OHOS ERROR] [56/340] gcc cross compiler obj/kernel/liteos_m/components/fs/vfs/vfs.vfs_fs.o
    [OHOS ERROR] FAILED: obj/kernel/liteos_m/components/fs/vfs/vfs.vfs_fs.o 
    ...
    [OHOS ERROR] ../../../kernel/liteos_m/components/fs/vfs/vfs_fs.c:214:43: error: unknown type name 'bool'
    [OHOS ERROR]   214 | static int VfsPathCheck(const char *path, bool isFile)
    [OHOS ERROR]       |                                           ^~~~
    [OHOS ERROR] ../../../kernel/liteos_m/components/fs/vfs/vfs_fs.c: In function 'VfsOpen':
    [OHOS ERROR] ../../../kernel/liteos_m/components/fs/vfs/vfs_fs.c:244:9: error: implicit declaration of function 'VfsPathCheck' [-Werror=implicit-function-declaration]
    [OHOS ERROR]   244 |     if (VfsPathCheck(path, TRUE) != LOS_OK) 
    [OHOS ERROR]       |         ^~~~~~~~~~~~
    [OHOS ERROR] cc1: all warnings being treated as errors
    ```



## 参考站点

- [Gitee-OpenHarmony基于命令行入门](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/quick-start/Readme-CN.md#/openharmony/docs/blob/master/zh-cn/device-dev/quick-start/quickstart-pkg-sourcecode.md)
