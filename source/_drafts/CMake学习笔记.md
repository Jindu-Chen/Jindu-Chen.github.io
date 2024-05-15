---
title: CMake学习及应用笔记
date: 
tags:
categories:
- [C语言]
description: 
---



## 概述


GNU Make：一个构建工具，通过读取Makefile文件来管理项目、执行命令行构建过程。

Makefile：是一个文本文件，描述了如何从源代码生成目标文件（如可执行文件或库）的规则，以及依赖关系，主要用于C/C++，或其它语言，只要其构建过程可以通过命令行工具实现即可

CMake：更高一层的跨平台构建系统，其不直接执行构建，而是生成特定平台的构建文件如Makefile。类似CMake的工具有 automake、autoconf 等
> 通过编写一种与平台无关的 CMakeLists.txt 文件来制定整个工程的编译流程，再根据具体的编译平台，生成本地化的 Makefile 和工程文件，最后执行 make 编译


在`Linux`平台使用`CMake`生成`Makefile`文件并编译的流程如下：
- 编写`CMake`配置文件`CMakeLists.txt`
- 执行命令 `cmake PATH` 或者 `ccmake PATH` 生成 `Makefile`（ccmake 和 cmake 的区别在于前者提供了一个交互式的界面）
    > PATH 是 CMakeLists.txt 所在的目录。
- 使用 `make` 命令进行编译


## CMake简单应用示例

### 单个源文件示例

```c
// ./main.c
#include <stdio.h>

int main()
{
 printf("Hello World!\n");
 return 0;
}
```

```
# ./CMakeLists.txt

# 定义项目的名称为 HELLO，作用包括：命名项目、自动设置与项目相关的CMake变量，如PROJECT_NAME、为之后可能的指令提供一个作用域的界定
project(HELLO)

# add_executable 命令，指示CMake生成一个可执行程序
# hello 是指定生成的可执行文件的名字（不包含文件扩展名）
# 指定源文件： ./main.c 是指定源文件的路径，将源文件添加到可执行文件 hello 的构建中
add_executable(hello ./main.c)
```

当前目录下存在`main.c、CMakeLists.txt`文件，而后执行`cmake ./`，会在**执行命令的当前路径下**生成相关文件或目录如：CMakeCache.txt、CmakeFiles、cmake_install.cmake、Makefile
```
CMakeCache.txt： 用于存储项目配置的变量和用户选择的设置。
CMakeFiles： CMake 会创建一个名为 CMakeFiles 的目录，用于存放构建过程中的中间文件，包括配置信息、依赖关系分析、编译命令等。
cmake_install.cmake： 由 CMake 生成的脚本，用于指导安装过程。
    当执行 make install 命令时，这个脚本会被调用来将构建好的目标文件（如可执行文件、库、头文件等）复制到指定的安装目录。通常包含安装路径、文件权限设置等信息
Makefile： 用于指导 make 工具如何构建项目。这个 Makefile 包含了编译、链接和其他构建步骤的规则。执行 make 命令时，Make 会读取这个 Makefile 来执行构建任务
```

最后执行 make 编译工程，生成可执行文件`hello`，通过命令`file hello`可查看文件属性

输入`./hello`运行程序

---


## CMakeLists.txt语法规则

### 基础语法、语句



### 常用命令

### 常用变量

## CMake进阶应用

### 定义函数

### 宏定义

### 文件操作

### 设置交叉编译


## 注意项及扩展思考


### 遵循`Out-of-Source Build`一定要进入源码下的build目录下cmake吗？

保持源码目录干净：通过在单独的 build 目录中执行构建过程，可以避免在源代码目录中产生编译生成的中间文件、对象文件和可执行文件。这样，源代码目录保持整洁，易于管理和版本控制。

支持多构建配置：可以在不同的 build 目录中进行不同配置的构建（例如 Debug 和 Release 版本），每个配置都保存在独立的目录中，互不影响。

便于清理：当需要重新构建项目或者清理构建产物时，只需删除对应的 build 目录，而不会误删源代码文件。

便于跨平台和跨编译器构建：对于需要在不同平台上或使用不同编译器构建的项目，可以在不同的 build 目录中使用不同的 CMake 配置选项，比如指定不同的工具链文件。

执行 cmake 命令时，通常会指定源代码目录作为参数，告诉 CMake 在哪里找到 CMakeLists.txt 文件，例如：


cd build
cmake ..

### 工具链配置文件*.cmake应用

CMAKE_TOOLCHAIN_FILE 是 CMake 中的一个关键变量，用于指定一个工具链文件（通常以 .cmake 结尾）的路径，这个文件包含了针对特定目标平台的编译器和链接器的配置信息。工具链文件通常用于交叉编译，即在一种架构的机器上构建另一种架构的软件。例如，在 x86 PC 上构建 ARM 架构的嵌入式系统软件。

.. 表示上一级目录，假设源代码位于当前目录的上级目录
`cmake -DCMAKE_TOOLCHAIN_FILE=path/to/toolchain.cmake ..`

CMakeLists.txt 文件中设置： 在文件顶部添加以下行：
`set(CMAKE_TOOLCHAIN_FILE path/to/toolchain.cmake)`



### 如何进行arm-none-eabi-交叉编译？生成bin文件？



## 开发应用示例




## 参考站点


- [Modern CMake 简体中文版](https://modern-cmake-cn.github.io/Modern-CMake-zh_CN/)
- [CMake 良心教程，教你从入门到入魂](https://zhuanlan.zhihu.com/p/500002865)
- [全网最细的CMake教程！(强烈建议收藏)](https://zhuanlan.zhihu.com/p/534439206)
- [摆脱MDK,用cmake改造嵌入式软件开发体验](https://segmentfault.com/a/1190000019934297)

