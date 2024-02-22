---
title: OpenHarmony轻量系统--XTS子系统编译适配及测试
date: 2024-02-02
tags:
categories:
- [LiteOS]
- [OpenHarmony]
description: 介绍基于国民技术N32G452芯片(Cortex-M4内核)的OpenHarmony-V4.0轻量系统移植工程，进行XTS子系统编译适配及应用兼容性测试分析的开发过程，以供参考
---

<br>

本文内容为**本人在移植国民技术N32G452适配OpenHarmony轻量系统的过程中，以添油方式仓促写就，其中或有不少相互矛盾或有失偏颇之处，但对于初入此道之开发者应能有一定的启迪作用。**

---

## 轻量系统应用兼容性测试适配

- 前提：基于OpenHarmony-V4.0-Release全量代码仓
- 在vendor目录下产品的config.json文件中，添加XTS子系统组件定义


### 如何添加XTS子系统

执行编译XTS组件的指令不唯一，取决于gn构建文件内的私有工程设置
- 通常为：`hb build -f -b debug --gn-args build_xts=true`，build_xts是一个变量，如果使能了该变量，则会增加相关的编译部分。另外在用户私有的BUILD.gn文件里，也有该变量的相关依赖部分。`--gn-args`用于传递构建参数至GN构建系统
- `hb build -b debug`，执行此条指令生成调试版本镜像，与 XTS 编译无关


就本人目前理解而言，要添加XTS子系统，前期需要实现的几点工作如下：
- 首先完成基于私有开发板的 OpenHarmony 工程移植适配工作，使得基于内核子系统的最简工程可以运行
- 而后建议在`target_config.h`添加启动shell功能配置，同时完成串口外设调试终端的移植适配，`hb build`，并烧录须测试验证ok。当然期间内必存在各种各样的编译报错，包括依赖关系不全、代码工程的不适配等
- 添加各项子系统并使其构建成功，如启动恢复子系统、DFX子系统，等等，
- 最后添加XTS子系统配置及其依赖，当然测试套件的静态库文件相当大，可以分批测试并保存测试结果，每次链接一部分库进行测试
  - 一般在`//device/board/xxx/xxx/BUILD.gn`文件里的 `if(build_xts=true)`选项卡内添加相关的链接选项，其中`hctest、sysparam、bootstrap`等静态库是必须的，在添加XTS子系统之前，应完成相关部分部件的编写编译构建与测试，此部分是测试框架基石

<br>

**添加编译XTS子系需要执行的步骤：**
- 在工程配置`target_config.h`宏定义中使能Test测试项
  - 使得系统启动初始化时会进行相关函数调用
- 在`//kernel/liteos_m`中make menuconfig中，使能测试相关配置 -- 实际上为配置`//vendor/xxx/kernel_configs/debug.config`选项
- 在`//vendor/xxx/config.json`添加合适的子系统配置
  - 补全文件末尾的`third_patry`和`product_adapt_dir`
- 在`//device/board/xxx/liteos_m/config.gni`添加适配的头文件路径
  - 链接选项中增添合适的静态库链接，以实现链接测试用例相关库，其中测试静态库文件生成于`//out/${company}//${product}//libs`
  - 还须添加源文件路径？
- 在`//device/board/xxx/xxx.ld`链接脚本中添加OpenHarmony特有的测试段



### XTS子系统编译流程
- 通过hb工具读取系统配置文件(即vendor目录下产品的config.json)，获取参与编译的子系统以及部件信息
- 通过gn工具会读取XTS子系统BUILD.gn--all_features会读取轻量系统全部acts测试套件
  - 判断是否debug版本`if (ohos_build_type == "debug" && ohos_test_args != "notest") `，非debug版本XTS不参与编译
- 将轻量系统全部acts测试套件与产品的config.json子系统组件对比，存在的子系统部件参与最终编译
- 最终生成静态库.a归档路径：`out/.../.../libs`
  - XTS子系统: libmodule_xxx.a，测试框架：libhctest.a
- .a库链接操作


## XTS子系统兼容性测评要求

### 规范编码以通过兼容性测试

- 遵循 OpenHarmony 的开发规范。OpenHarmony 提供了完善的开发规范，包括代码风格、命名规范、注释规范等

- 使用 OpenHarmony 的标准 API


### XTS子系统测试项

轻量系统的全部测试项包含有：应用管理(Ability和Bundle)、网络通信(LwIP)、文件系统、系统参数、Wi-Fi IOT、分布式数据管理、安全、日志、事件、分布式调度、系统更新、系统引导



## OpenHarmony轻量系统认证流程

准备烧录镜像、相关指导文档、寄送样品及工具



通常情况下，由于芯片内存大小有限，无法一次性将所有的 XTS 测试用例库链接进程序中

因此，主要采用分批烧录、验证串口打印信息的方式，每一个测试模块都对应一个烧录镜像文件

另外，需要提供相关烧录工具以及指导文件
- 如果是通过烧录口直接烧录的方式，那么寄送的样品可以把线路板暴露出来
- 当然，也可以通过 IAP 在应用编程方式，或者空中升级方式，只要能够达到目标即可


## XTS子系统源码分析

### XTS子系统软件工作流程

```
LITE_TEST_CASE(IntTestSuite, TestCase001, Level0) 
{  
  //do something 
};
```



## XTS子系统的疑惑解答

### 相对于不同架构的工程，其XTS子系统源码是否也要更改？
### 相对新移植的工程，是直接复制现有的XTS源码，还是要开发者基于移植工程、按照规范编写相关程序？
### 在更换不同版本如 3.2 -> 4.0 系统时，需要在哪些地方更改？更改哪些内容？



## 参考站点

- [OpenHarmony L0级设备XTS适配经验分享](https://blog.csdn.net/Via6666/article/details/132093613)
- [移植案例与原理 - XTS子系统之应用兼容性测试套件](https://bbs.huaweicloud.com/blogs/336717)
- [轻量系统STM32F407芯片移植案例](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/porting/porting-stm32f407-on-minisystem-eth.md)



