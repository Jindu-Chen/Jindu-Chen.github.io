---
title: Makefile学习笔记
date: 2023-12-15
tags:
categories:
- [Linux]
- [嵌入式]
description: |
  &emsp;&emsp;日常的嵌入式软件开发中不可避免地要接触和熟悉各种各样大大小小的工程，它们的构建编译方式当然也不是一成不变的，包括并不限于 Keil IDE集成环境、GCC + Make、SCons + GCC、GN/Ninja + GCC/RISCV 等等。<br>
  &emsp;&emsp;在这个开发工作过程中，积累了一些碎片化知识，怀着所学所遇皆记录以供回顾的想法，写下这篇文章，记录下个人对 Makefile 的认知学习及实际开发中的应用。
---


## 前言

Makefile 问世并流行多年，互联网上早已拥有海量的相关学习文章、视频及书籍资料。因此这篇文章主要以个人的理解角度，来阐述 Makefile 的学习之路。

## 基础知识/概念

### 符号
- 符号`$`
  表示引用定义变量的值 
- 符号`$@`
  目标的集合
- 符号`$<`
  依赖文件集合中的第一个文件；通常依赖文件是以("%")定义的，所以其表示为符合模式的一系列文件

### Makefile格式

1. 直接输入命令make，系统会默认以第一条规则的目标为最终目标; 也可以指定生成目标，比如make main、make clean


```
目标....： 依赖文件集合...
    命令1
    命令2
    ......
```

### 规则模式与语法

- 模式规则

- 常用函数

## 进阶应用

### Makefile父子文件配置

多目录管理

文件间变量如何共享？

如何组织工程

### make menuconfig 原理

### Automake 原理

Automake是GNU构建系统的一部分，它旨在简化软件项目的构建过程。Automake通过简化Makefile的编写，提供了一种更加高级和可移植的方式来描述软件项目的构建规则。

Automake使用一种称为"Makefile.am"的文件来描述构建规则，然后通过"aclocal"、"autoconf"和"automake"等工具生成最终的Makefile。

使用Automake可以使得构建规则更加清晰、简洁，并且具有更好的可移植性。

Automake还提供了一些高级功能，如支持多目录构建、自动化依赖关系管理、自动生成文件等。

这些功能使得Automake成为一个强大的构建工具，特别适用于大型和复杂的软件项目。

## 注意项及疑惑解答

### Makefile 文件与 .mk 文件的区别与联系？

在实际使用中，通常将makefile文件的扩展名命名为.mk或者Makefile。在GNU Make工具中，.mk文件通常用于包含一组相关的规则，而Makefile文件通常用于包含整个项目的构建规则。

实际上，.mk文件和Makefile文件之间没有本质区别，它们都是Make工具所使用的构建规则文件。

通常情况下，.mk文件用于包含一组相关的规则，而Makefile文件用于包含整个项目的构建规则，但这只是一种惯例，实际上可以根据项目的需要来命名文件。

无论是.mk文件还是Makefile文件，它们都是Make工具所使用的构建规则文件，可以包含变量定义、目标规则、依赖关系等内容。

因此，在实际使用中，可以根据项目的需要来选择使用.mk文件或者Makefile文件，并没有严格的规定。


### 在Makefile中，是如何将每个需要编译生成.o目标文件与.c源文件关联起来的

```
# addprefix用于在每个元素前面添加一个前缀：(OUTPUT_DIR)/
# (C_SOURCES:.c=.o)将C源文件的后缀.c替换为.o，使得C_SOURCES中的每个.c都对应相应的.o文件
C_OBJECTS = $(addprefix $(OUTPUT_DIR)/, $(C_SOURCES:.c=.o))

$(OUTPUT_DIR)/%.o: %.c              # 模式规则，要生成.o文件，则要找到对应的.c
    mkdir -p $(dir $@)              # 如果目录不存在，则创建；(dir@)表示目标文件$@的目录部分
    $(CC) $(INCLUDE) $(CFLAGS) -MMD -c $< -o $@
```
<br>

### ./configure命令

configure脚本由autoconf工具生成，在make命令之前先执行配置；执行此命令，配置编译器、库、工具等参数，并根据此参数生成一个Makefile

<br>
    
### 编译、包含依赖文件，避免重复编译、提高效率

在编译命令中加入`-MMD`选项，告诉编译器生成依赖关系文件
在Makefile里的目标文件编译规则命令之后，加入`-include $(wildcard $(OUTPUT_DIR)/*/*.d)`
    

## 参考站点

