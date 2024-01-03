---
title: OpenHarmony-LiteOS-M内核之GN设计编译构建过程解析
date: 2023-12-29
tags:
categories:
- [LiteOS]
- [RTOS]
description: 要基于OpenHarmony-V3.2-Release源码工程进行LiteOS-M轻量系统的设备开发，那么熟悉了解其编译构建过程是必要的；本文以STM32的LiteOS移植工程为例，剖析其业务代码、内核代码的调用编译构建流程
---


阅读本文，须先参阅[**GN与Ninja入门**](/2023/12/24/GN和ninja的区别、联系和应用开发)

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

### 记于2024-01-02

- 客观来说，理论上移植LiteOS-M到N32G452，并编译进行XTS子系统测试是可行的，不能畏惧不前
- 目前需要的知识储备为：**熟悉GN、Ninja工具，有深入的理解编译及链接原理，熟悉cortex-m架构，了解Python，熟悉RTOS内核实现**
- 首要任务是：**基于OpenHarmony全量工程的构建方式，把最简程序运行起来**

---

## 轻量系统编译构建-命令执行简述

### hb命令

&emsp;&emsp;hb工具是一个命令行工具，其使用Python语言编写，能够调用gn和ninja工具来生成OpenHarmony的二进制文件。<br>
- hb的源代码在`build/lite/hb`路径下，其入口函数为`__main__.py中的main()`，支持命令有：`set、env、build、clean`
- **hb命令的具体实现位于`//build/lite/hb_internal`中**


### hb set

- `hb set`的作用为生成产品配置文件，其会在工程根目录下生成ohos_config.json文件
- 其会读取`//vendor/`的配置，以供选择


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
  - `device/{company}/{board}/liteos_m/config.gni`：编译LiteOS-M内核的配置 -- 需要前提配置好的芯片开发板解决方案
  - `vendor/{company}/{board}/config.json`：板子所应用的全量配置表，包括子系统、组件等 -- 需要提前配置好的产品解决方案
- 调用gn gen命令，读取产品配置（板级配置、内核、组件等），生成out目录和ninja文件
- 调用`ninja -C out/{company}/{product}/`开始编译
- 打包、制作可执行文件

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
        }
        subsystem_parts_info = read_file(
                "${root_build_dir}/build_configs/mini_adapter/${subsystem_name}.json",
                "json")

        # Step 4: Loop components configured by product.
        foreach(product_configed_component,
                product_configed_subsystem.components) {
          # Step 5: Check whether the component configured by product is exist.
          component_found = false

          foreach(part_name, subsystem_parts_info.parts) {
            if (product_configed_component.component == part_name) {
              component_found = true
            }
          }

          assert(
              component_found,
              "Component \"${product_configed_component.component}\" not found" + ", please check your product configuration.")

          # Step 6: Loop OS components and check validity of product configuration.
          foreach(part_name, subsystem_parts_info.parts) {
            kernel_valid = true

            # Step 6.1: Skip component which not configured by product.
            if (part_name == product_configed_component.component) {
              # Step 6.1.1: Loop OS components adapted kernel type.

              assert(
                  kernel_valid,
                  "Invalid component configed, ${subsystem_name}:${product_configed_component.component} " + "not available for kernel: ${product_cfg.kernel_type}!")

              # Step 6.1.2: Add valid component for compiling.
              # Skip kernel target for userspace only scenario.
              ......
            }
          }
        }
      }
    }

    # Skip device target for userspace only scenario.
    if (!ohos_build_userspace_only) {
      # Step 7: Add device and product target by default.
      # liteos_m kernel organise device build targets, but not by default.
      if (product_cfg.kernel_type != "liteos_m") {
        deps += [ "${device_path}/../" ]
      }
    }
  } else {
    deps += string_split(ohos_build_target, "&&")
  }
}
```

---

## 编译构建调用过程

主要分为三部分：
- `vendor/{company}/{board}/`--产品解决方案
- `device/{company}/{board}/`--芯片开发板解决方案
- `kernel/liteos_m`--轻量系统内核

### 产品解决方案代码

### 芯片开发板解决方案代码

### 内核代码



## 移植思路总结

此部分内容直接转述自参考文章：https://developer.huawei.com/consumer/cn/blog/topic/03889745165640019

移植类型可以分为三部分：ARCH、SoC、Board
- ARCH为架构级别移植，代码位于kernel/liteos_m/arch
- SoC为对已支持的架构做芯片厂商级的移植，代码位于device/soc
- Board为对芯片下不同板子的移植，代码位于device/board
- 另，还有厂商配置相关代码，位于vendor，其主要用于编译系统、HDF配置(驱动框架的配置描述文件)：用于描述驱动程序的硬件资源、功能特性，使用Key-Value格式-由属性节点与子节点组成

移植路线：**vendor -> board -> soc -> ARCH**


## 参考文献

- [移植案例与原理 - build lite编译构建过程](https://bbs.huaweicloud.com/blogs/336721)
- [移植案例与原理 - build lite配置目录全梳理](https://bbs.huaweicloud.com/blogs/336722)
- [从零开始移植OpenHarmony轻量系统](https://developer.huawei.com/consumer/cn/blog/topic/03893164050570049)

