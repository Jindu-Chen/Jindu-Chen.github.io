---
title: OpenHarmony-LiteOS-M内核之编译构建及移植适配过程解析
date: 2023-12-29
tags:
categories:
- [LiteOS]
- [OpenHarmony]
- [RTOS]
description: 要基于OpenHarmony-V4.0-Release源码工程进行LiteOS-M轻量系统的设备开发，那么熟悉了解其编译构建过程是必要的；本文以STM32/N32G452的LiteOS移植工程为例，剖析其业务代码、内核代码的调用编译构建流程
---


阅读本文，须先参阅[**GN与Ninja入门**](/2023/12/24/GN和ninja的区别、联系和应用开发)

本文内容大部为参考OpenHarmony相关站点文章，基于个人在移植适配/开发过程中的理解，作进一步的整理，其中必然有许多不足或前期理解错误之处，仅作参考用即可

## 前言

### 记于2023-12-29
- 就目前而言，网上关于LiteOS-M的相关开发移植资料都是相当的少，官方文档也是写得不清不楚的，如果不是项目需求，估计没人愿意来做LiteOS-M的移植适配工作。。
- 鄙人初次接触，此前参考了社区的STM32工程移植示例，成功移植LiteOS-M并在gcc+make开发环境运行
- 但由于政策或者是项目需要，单纯的内核源码移植其实没什么用，跟别的RTOS内核没啥两样，由此，接触了一个新概念：**XTS子系统**，要获得开放原子基金会的认证，那么就需要通过这个XTS兼容性测评
- 要把这个XTS测试组件编译进工程里面，鄙人目前一头雾水，特别还是基于国民技术N32G452芯片移植并编译测试，目前还在熟悉状态，考虑两种方式：
  - 基于 https://gitee.com/openharmony/kernel_liteos_m 所拉取的源码工程，进行熟悉构建开发
  - 基于 https://gitee.com/link?target=https%3A%2F%2Frepo.huaweicloud.com%2Fopenharmony%2Fos%2F4.0-Release%2Fcode-v4.0-Release.tar.gz 整个OpenHarmony全量代码工程进行熟悉开发
- 但其社区移植工程都已经过时，不适配Harmony4.0 Release，编译无法通过，要想移植，只能自己摸索、熟悉GN+Ninja的构建方式，从头理顺开发过程
  - 此前已经做了一部分的工作，包括命令行开发环境搭建、各种依赖搭建，但没有系统地进行整理，可以说是个遗憾
  - 另外感觉**移植编译XTS子系统、通过兼容性测评**这个时间周期会较长，而且是没有任何指导文档下的独立开发，本人对此预研项目持悲观态度，目前也不能把过多时间精力投入此处，且做且看吧！

---

### 记于2024-01-04

- 进展远超预期
- 截止今天已经完成了soc、board、vendor的移植适配工作，主要归功于华为云-连志安的博客，参考其对于GD32的移植范例及教程，快速地完成了国民技术的适配，包括底层驱动以及最简工程测试验证
- 不禁感叹，**前期的文献检索调研**是相当重要的
- 目前只剩下XTS子系统的编译链接，即可开始OpenHarmony兼容性测评认证工作
- 参照零星示例，尝试在vendor/xxx/config.json添加了xts子系统，同时在BUILD.gn添加链接选项，可看到xts组件有编译，但没链接进bin文件，对整个XTS子系统添加、编译链接流程思路还是不清晰，需要时间
- 捋清XTS添加核心步骤：生成相应的测试静态库、添加链接项或者修改链接脚本使得静态库能加入可执行文件、确保应用程序中有函数调用执行相关的测试操作

---

### 记于2024-01-11

- 如果从习惯于集成开发环境之开发者角度来看，那OpenHarmony的轻量系统编译构建体系简直是一个大坑，复杂麻烦，相互依赖、嵌套调用关系多，需要熟悉的使用工具也多
- 而且在OpenHarmony-V不断迭代发布情况下，其内相应的配置工程范例却没有更新适配，编译一堆报错。依赖关系还得开发者自己解决
- 目前已经基于国民技术N32G452完成了内核子系统、部分子系统的适配，当然离通过兼容性测试还有一段距离

---


## 轻量系统编译构建-命令执行简述

### hb命令

&emsp;&emsp;hb工具是一个命令行工具，其使用Python语言编写，能够调用gn和ninja工具来生成OpenHarmony的二进制文件。<br>
- hb的源代码在`build/lite/hb`路径下，其入口函数为`__main__.py中的main()`，支持命令有：`set、env、build、clean`
- **hb命令的具体实现位于`//build/lite/hb_internal`中**


### hb set

- `hb set`的作用为生成产品配置文件，其会在工程根目录下生成ohos_config.json文件
- 执行`hb set`时，脚本会遍历`//vendor/${product_company}/${product_name}`目录下的`config.json`文件，给出可选的产品选项
  > config.json文件中，product_name表示产品名，device_company和board用于关联出//device/board/<device_company>/<board>目录，匹配该目录下的<any_dir_name>/config.gni文件
  > 其中<any_dir_name>目录名可以是任意名称，但建议将其命名为适配内核名称（如：liteos_m、liteos_a、linux）
  > hb命令如果匹配到了多个config.gni，会将其中的kernel_type和kernel_version字段与vendor/<device_company>下config.json文件中的字段进行匹配，从而确定参与编译的config.gni文件


#### 调用解决方案包Kconfig的具体步骤

- `hb set`根据配置项的名称找到对应的 Kconfig 配置项。
- 将配置项的值写入到 Kconfig 配置项的变量中。
- 系统在`//kernel/liteos_m`使用 make menuconfig 命令配置，结果保存于 `//vendor/${product_company}/${product_name}/kernel_configs/debug.config` 文件，其包含配置项的值。
- 编译时使用 gn 命令将 debug.config 文件中的配置项读取到 gn 文件中

<br>

`//kernel/liteos_m/Kconfig`文件片段如下：移植开发板适配时，需要在board和soc目录下，添加补充相应的Kconfig文件
```
orsource "../../device/board/*/Kconfig.liteos_m.shields"
orsource "../../device/board/$(BOARD_COMPANY)/Kconfig.liteos_m.defconfig.boards"
orsource "../../device/board/$(BOARD_COMPANY)/Kconfig.liteos_m.boards"
orsource "../../device/soc/*/Kconfig.liteos_m.defconfig"
orsource "../../device/soc/*/Kconfig.liteos_m.series"
orsource "../../device/soc/*/Kconfig.liteos_m.soc"
```



### hb build (copy)

- 配置好产品解决方案、芯片开发板解决方案后，即可执行`hb set -> hb build`进行编译
- hb build的作用是根据配置信息生成编译指导文件build.ninja。
- 代码路径为：build/lite/hb/build/build.py
  - 调用 gn_build 使用 gn gen生成 *.ninja 文件
  - 调用 ninja_build 使用 ninja -w dupbuild=warn -C 生成 *.o *.so *.bin 等最后的文件


### build lite 配置目录梳理

复述自：https://bbs.huaweicloud.com/blogs/336722

```
build/lite
├── components                  # 组件描述文件
├── hb                          # hb pip安装包源码
├── make_rootfs                 # 文件系统镜像制作脚本
├── config                      # 编译配置项
│   ├── component               # 组件相关的模板定义
│   ├── kernel                  # 内核相关的编译配置
│   └── subsystem               # 子系统编译配置
├── platform                    # ld脚本
├── testfwk                     # 测试编译框架
└── toolchain                   # 编译工具链配置，包括：编译器路径、编译选项、链接选项等
```

- build/lite/ohos_var.gni
  - 定义了所有部件的全局变量
  - 用于读取产品解决方案的配置文件config.json中的配置项，解析为gn变量
  - 被build\lite\config\BUILDCONFIG.gn包含导入import


- build\lite\config\BUILDCONFIG.gn
  - 用于配置编译构建，其会import导入产品解决方案和芯片开发板解决方案的配置文件
  - 解析开发板配置文件config.gni

## LiteOS-M内核之GN编译构建流程总览

编译构建过程参考：https://bbs.huaweicloud.com/blogs/336721

- `hb build`->`build/lite/hb_internal/build/build.py`
- 读取板子配置，包括编译工具链、编译链接选项等
- 读取的配置文件有：
  - `build/lite/ohos_var.gni`：定义应用于所有组件的全局变量
  - `device/{board}/{company}/liteos_m/config.gni`：编译LiteOS-M内核的配置 -- 需要前提配置好的芯片开发板解决方案
  - `vendor/{company}/{board}/config.json`：板子所应用的全量配置表，包括子系统、组件等 -- 需要提前配置好的产品解决方案
- 调用gn gen命令，读取产品配置（板级配置、内核、组件等），生成out目录和ninja文件
- 调用`ninja -C out/{company}/{product}/`开始编译
- 打包、制作可执行文件
<br>

**`device/{company}/{board}/liteos_m/config.gni`文件**：
- 工程构建的全局配置文件，主要配置CPU型号、交叉编译工具链及全局编译、链接参数等信息
- 其内部分参数如下：
  ```
  board_opt_flags : 编译器相关选项，一般为芯片架构、浮点类型、编译调试优化等级等选项。
  board_asmflags  ：汇编编译选项，与Makefile中的ASFLAGS变量对应。
  board_cflags    ：C代码编译选项，与Makefile中的CFLAGS变量对应。
  board_cxx_flags ：C++代码编译选项，与Makefile中的CXXFLAGS变量对应。
  board_ld_flags  ：链接选项，与Makefile中的LDFLAGS变量对应。
  ```

---

分析`build/lite/BUILD.gn`文件中`ohos`元目标如下：
```
group("ohos") {
  deps = []
  if (ohos_build_target == "") {
    # Step 1: Read product configuration profile.
    # 读取/vendor/{company}/{board}/config.json
    product_cfg = read_file("${product_config_path}/config.json", "json")   

    parts_targets_info = read_file(
            "${root_build_dir}/build_configs/parts_info/parts_modules_info.json",
            "json")

    # Step 2: Loop subsystems configured by product.
    # 遍历板子的子系统配置
    foreach(product_configed_subsystem, product_cfg.subsystems) {
      subsystem_name = product_configed_subsystem.subsystem

      # 判断XTS子系统相关，暂未搞懂
      if (build_xts || (!build_xts && subsystem_name != "xts")) {
        # Step 3: Read OS subsystems profile.
        subsystem_parts_info = {
......}}}}
}
```

---

## 编译构建调用/适配过程

主要分为以下几部分：
- `vendor/{company}/{board}/`--产品配置方案
- `device/soc/`--芯片解决方案
- `device/board/`--开发板解决方案
- `kernel/liteos_m`--轻量系统内核


## 产品解决方案代码

**内核子系统适配：**
- 在`//vendor/{company}/{product}/config.json`文件添加内核子系统及配置如下：
```
    "subsystems": [ 
        {
            "subsystem": "kernel",
            "components": [
                {
                    "component": "liteos_m"
                }
            ]
        }
    ],
```

## 芯片解决方案代码


## 硬件板解决方案代码


**`target_config.h`文件适配**:
- 在`//kernel/liteos_m/kernel/include/los_config.h`文件中，有包含一个名为target_config.h的头文件，如果没有这个头文件，则会编译出错
- 该头文件主要定义一些与soc芯片相关的宏定义，可参考其它解决方案的target_config.h配置即可，或者可以根据编译错误提示进行添加或者删减
- 在何处添加该头文件由用户决定，建议是存放于`//device/board/${company_product}`目录下
- 而后在`device/{board}/{company}/{company_board}/liteos_m/config.gni`添加该头文件的路径包含即可
- 配置系统时钟、系统节拍
  ```
  #define OS_SYS_CLOCK                                        SystemCoreClock   // SystemCoreClock为N32G452库的宏定义，具体可由用户设定，如72000000
  #define LOSCFG_BASE_CORE_TICK_PER_SECOND                    (1000UL)
  ```

**适配printf打印函数**:
此处不过多赘述，用户自行实现重定向`int printf(char const  *fmt, ...)`函数即可
<br>


## 子系统适配

### OpenHarmony全量仓工程目录解析

建议大致了解整个工程目录框架


### 启动恢复子系统适配

大致思路如下：<br>

对于Cortex-M内核来说，要适配此启动恢复子系统，实际上是要有完成两个工程的适配，分别是bootloader、app
- bootloader是个相对独立的工程，其包含芯片启动.S文件、system_xxx.c文件、main()函数几部分，主要作用就是完成应用程序升级功能。当然，前期适配过程中，只需要在初始化系统时钟，配置必要外设，然后**直接设置app应用程序的栈顶指针，跳转至app程序段开始即可**
- app是独立的应用程序工程，此部分同样包含启动.S文件、等等相应部分
- 芯片上电先跑bootloader，然后跳转到app
- 当然，上述工程文件基本位于`//device/board/{company}/{company_board}`下，具体如何构建由开发者决定
- 最后需要通过 **`merge_bin`** 将两个bin文件合并成最终的烧录bin文件

<br>
在`//vendor/{company}/{product}/config.json`新增对应配置选项
```
{
      "subsystem": "startup",
      "components": [
        {
          "component": "bootstrap_lite",
          "features": []
        },
        {
          "component": "syspara_lite",
          "features": []
        }
      ]
}
```
另外，bootstrap_lite部件需要在`//device/board/{company}/{board}/xxx.ld`链接脚本添加相关的段 -- 关键词：**灌段**


### DFX子系统适配

其是一个提升软件质量设计的工具集，主要包含两部分内容：
- DFR（可靠性）：提供流水日志、故障分析等功能
- DFT（可测试性）：提供测试桩、桩机、测试工具等功能


## 移植思路总结

此部分内容直接转述自参考文章：https://developer.huawei.com/consumer/cn/blog/topic/03889745165640019

移植类型可以分为三部分：ARCH、SoC、Board
- ARCH为架构级别移植，代码位于kernel/liteos_m/arch
- SoC为对已支持的架构做芯片厂商级的移植，代码位于device/soc
- Board为对芯片下不同板子的移植，代码位于device/board
- 另，还有厂商配置相关代码，位于vendor，其主要用于编译系统、HDF配置(驱动框架的配置描述文件)：用于描述驱动程序的硬件资源、功能特性，使用Key-Value格式-由属性节点与子节点组成

移植路线：**//vendor -> //device/board -> //device/soc -> ARCH**


## 参考站点

- [移植案例与原理 - build lite编译构建过程](https://bbs.huaweicloud.com/blogs/336721)
- [移植案例与原理 - build lite配置目录全梳理](https://bbs.huaweicloud.com/blogs/336722)
- [从零开始移植OpenHarmony轻量系统](https://developer.huawei.com/consumer/cn/blog/topic/03893164050570049)
- [轻量系统STM32F407芯片移植案例](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/porting/porting-stm32f407-on-minisystem-eth.md)

