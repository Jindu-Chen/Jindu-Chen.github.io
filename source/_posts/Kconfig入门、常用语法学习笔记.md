---
title: Kconfig入门、常用语法学习笔记
date: 2024-01-05 05:47:11
tags:
categories:
- [LiteOS]
- [Linux]
description: | 
    &emsp;&emsp;Kconfig作为一个配置管理工具，其不仅是Linux内核配置系统的一部分，也被广泛应用于其它开源项目，如Buildroot、U-Boot、OpenHarmony、BusyBox等。<br>
    &emsp;&emsp;要熟悉大型工程的配置和编译构建过程，那么需要对Kconfig有一定了解，本文为Kconfig之学习笔记，介绍Kconfig的概况、常用语法、代码示例解释、及对其于OpenHarmony的应用解析等。
---

**此文章大部为参考/复述整理编写，为私有理解回顾用**<br>

## 概况

Kconfig是Linux内核及许多其它系统广泛使用的一款配置管理工具，用于配置选择功能模块的编译与否<br>
- Kconfig是脚本语言，用于定义配置选项和配置菜单
- Kconfig文件分布在各级目录下，构成一个分布式的配置数据库
- 一般来说，`make menuconfig`通过make命令执行menuconfig目标，来调用Kconfig生成界面


**其单个配置项基本语法如下：**
```
config <option-name>                # config：配置选项的名称
        [depends on <expr>]         # depends on：该配置选项依赖的其它配置选项
        [select <expr>]             # select：其下所包含的选择项
        [default <expr>]            # default：该配置选项的默认值
        [prompt <string>]           # prompt：提示信息
        [help <string>]             # help：帮助信息
```
<br>

**简单示例如下：** 该Kconfig文件定义了两个配置选项：MY_FEATURE和MY_FEATURE_SUB_OPTION。MY_FEATURE是一个布尔类型的选项，默认值为y。MY_FEATURE_SUB_OPTION是一个布尔类型的选项，依赖于MY_FEATURE
```
config MY_FEATURE
        bool "My Feature"
        default y
        help
          This option enables my feature.

config MY_FEATURE_SUB_OPTION
        depends on MY_FEATURE
        bool "My Feature Sub Option"
        default n
        help
          This option enables my feature sub option.
```

在使用make menuconfig命令生成配置界面后，可以看到**以下配置选项**：
```
My Feature [y]
    * My Feature Sub Option [n]
```
用户可以选择将MY_FEATURE选项设置为y或n。如果将MY_FEATURE选项设置为y，则会显示MY_FEATURE_SUB_OPTION选项。用户可以选择将MY_FEATURE_SUB_OPTION选项设置为y或n

### Linux内核之make menuconfig

以下部分复述自参考文章<br>

`menuconfig`目标是Linux内核的顶层Makefile中的目标之一，定义如下：
```
config_cmd = menuconfig
config_cmd_help = Configure the kernel
```
该目标设置指定了`make menuconfig`命令执行步骤如下：
- 调用make命令的config_cmd目标
- config_cmd目标会调用`scripts/kconfig/Makefile:mconf`命令来生成配置界面
- mconf命令会解析Kconfig文件，并将配置选项显示在配置界面中，mconf代码位于`//script/Kconfig/mconf.c`
  > 解析Kconfig：./mconf Kconfig
  > 生成对话框：init_dialog
  > 配置界面：conf

<br>

在`make menuconfig`之后生成.config文件
- 在编译链接时，Makefile会读取.config文件中的内容，从而决定是否编译或者链接相关代码
- **此外，系统会将所有的选项以宏的形式保存在Linux内核根目录下的`include/generated/autoconf.h`文件，此头文件被包含于工程源文件中，用于决定源文件的条件编译选项**


## Kconfig常用语法

在Kconfig文件中，如果配置项名字为`XXX`，则生成的.config文件中，会展示其变量为CONFIG_XXX
- 如果设置了`CONFIG_=LINUX`，则变量名展示为`LINUX_XXX`
- not set，则表示配置选项未设置或依赖于其他配置选项（此其他选项未设置）
- .config文件如下所示，设置了`CONFIG_=LOSCFG`，去掉前缀的变量即为Kconfig文件中的配置项名字
```
LOSCFG_DEBUG_TOOLS=y
LOSCFG_MEM_DEBUG=y
# LOSCFG_DRIVERS is not set
```

## 菜单

```
menu "test menu"
    config TEST_MENU_A
        tristate "menu test A"

    config TEST_MENU_B
        bool "menu test B"
        default n                           
endmenu
```

## 选择

```
choice
    prompt "The maximal size of fifo"
    default USB_FIFO_512
    
    config USB_FIFO_512
        bool "512"
        
    config USB_FIFO_1024
        bool "1024"
        
    config USB_FIFO_3072
        bool "3072"

endchoice
```

## menuconfig

menuconfig XXX和config XXX类似， 唯一不同的是该选项除了能设置y/m/n外，还可以实现菜单效果(能回车进入该项内部)


```
menuconfig M
  if M
      config C1
      config C2
  endif
```

---

以下为Kconfig文件示例：其定义了一个名为My Menu的菜单，菜单下包含四个配置选项：MY_FEATURE_1、MY_FEATURE_2、MY_FEATURE_3和MY_FEATURE_4。其中，MY_FEATURE_4依赖于MY_FEATURE_1
```
menu "My Menu"

config MY_FEATURE_1
        bool "My Feature 1"
        default y
        help
          This option enables my feature 1.

config MY_FEATURE_2
        bool "My Feature 2"
        default n
        help
          This option enables my feature 2.

config MY_FEATURE_3
        bool "My Feature 3"
        default n
        help
          This option enables my feature 3.

config MY_FEATURE_4
        depends on MY_FEATURE_1
        bool "My Feature 4"
        default n
        help
          This option enables my feature 4.

endmenu
```
在使用make menuconfig命令生成配置界面后，可以看到以下配置选项：
```
My Menu
    * My Feature 1 [y]
    * My Feature 2 [n]
    * My Feature 3 [n]
    * My Feature 4 [n]
```
用户可以选择将菜单下的任意一个配置选项设置为y或n。如果将MY_FEATURE_1选项设置为y，则会显示MY_FEATURE_4选项。用户可以选择将MY_FEATURE_4选项设置为y或n。


## 进阶操作

### source语句

source 语句用于读取另一个文件中的 Kconfig 文件

### if语句

if语句：用于根据条件来显示或隐藏配置选项<br>

以下示例，如果CONFIG_CPU_32选项设置为n，则MY_FEATURE选项才会显示在配置界面中
```
if !CONFIG_CPU_32
        config MY_FEATURE
                bool "My Feature"
                default y
                help
                  This option enables my feature, which is only available on 64-bit CPUs.
endif
```

### tristate语句

用于定义三态配置选项<br>

以下示例，MY_FEATURE选项可以设置为y、n或m
```
config MY_FEATURE
        tristate "My Feature"
        default n
        help
          This option enables my feature
```


## OpenHarmony之Kconfig应用

在OpenHarmony轻量系统的编译链接配置过程中，有如下应用：
- 用户在`//kernel/liteos_m/`目录下`make menuconfig`，结果生成于`//vendor/${company}/${company_product}/kernel_configs/debug.config`文件中（前提是先创建该文件）
- 而此debug.config文件的宏配置会公开可见于整个GN构建工程，使得系统可以进行选择性的条件编译链接
- 另外，会有mkconfig.sh脚本将此配置文件转化为C语言的宏定义，写入`los_config.h`文件，此效果等同于在`target_config.h`文件声明配置相关宏定义



## 注意项及疑惑解答

- 一定是生成.config文件吗，这个文件名是否可改
- .config文件的配置是如何导入到C源文件的宏定义的
- Kconfig.xx.xx文件类型
- 如何构建
- .json文件的用法，其属性与Kconfig的联系？


## 参考站点

- [menuconfig 和、Kconfig 介绍及例子解析！](https://zhuanlan.zhihu.com/p/517418914)
- [Make menuconfig生成文件](https://www.cnblogs.com/hellokitty2/p/7587251.html)

