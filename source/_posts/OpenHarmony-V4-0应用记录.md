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

### debug.config配置

debug.config文件位于`//vendor/${company}/${company_product}/kernel_configs`目录下，该文件配置由用户在`//kernel/liteos_m/`目录下`make menuconfig`生成，当然用户也可以直接修改debug.config进行工程的宏定义配置
<br>




## 子系统添加和构建

参考OpenHarmony-v4.0源码工程下的`//vendor/talkweb/niobe407`、`//device/board/talkweb`、`//device/soc/st/stm32f4xx`，此部分是基于stm32的厂商工程范例
<br>

此处重复讲述几个与子系统相关的概念：以便在添加子系统时清楚所需要添加和编译链接的子系统
OpenHarmony轻量系统下的子系统主要分为三部分：基础子系统、功能子系统、应用子系统
- 基础子系统
  即`//foundation`，包括bootstrap（系统启动和关闭）、hal_sysparam（负责管理系统参数）、systemabilitymgr（负责管理系统服务）、commonlibrary（提供通用库）等，当然内核子系统是必须的
  
<br>

- 功能子系统
  HiViewDFX ：为图形框架应用子系统
  HiViewDFX ：为图形框架应用子系统
  hdf：驱动层框架

<br>

- 应用子系统
  shell：命令行接口
  

## 轻量系统编程开发

### 系统初始化函数

**LOS_KernelInit()函数**
<br>

**OHOS_SystemInit()函数**
- 主要为隐式执行各种初始化函数，包含用户声明的自定义初始化函数
- 如执行通过`SYS_RUN(OHOS_Main);`声明的OHOS_Main函数
```
void OHOS_SystemInit(void)
{
    MODULE_INIT(bsp);
    MODULE_INIT(device);
    MODULE_INIT(core);
    SYS_INIT(service);
    SYS_INIT(feature);
    MODULE_INIT(run);
    SAMGR_Bootstrap();
    LiteParamService();
}
```
<br>


**LOS_Start()函数**
<br>


## 编译纠错

- 编译成功，但生成的可执行bin文件明显偏小，查看Kconfig配置文件是否有选项没写对，导致的没编译链接

- 提示以下报错，考虑为 g++编译时 extern C {}的嵌套引用的编译错误，根据编译报错提示，一步步查找，其嵌套包含的头文件其里的`#ifdef __cplusplus } #endif`是否一一匹配
- 此处最终发现为国民技术n32g45x.h包含的n32g45x_conf.h包含的n32g45x_dvp.h缺少了ifdef cplusplus 的右大括号
```
[OHOS ERROR] [13/365] gcc CXX obj/base/hiviewdfx/hilog_lite/frameworks/featured/libhilog_static.hilog.o
[OHOS ERROR] FAILED: obj/base/hiviewdfx/hilog_lite/frameworks/featured/libhilog_static.hilog.o 
[OHOS ERROR] ccache arm-none-eabi-g++ -D_XOPEN_SOURCE=700 -DOHOS_DEBUG -D__LITEOS__ -D__LITEOS_M__ -DSECUREC_IN_KERNEL=0 -I../../../device/soc/n32/n32g452/liteos_m -I../../../device/soc/n32/CMSIS/core -I../../../device/soc/n32/CMSIS/device -I../../../device/soc/n32/n32g452/N32G45x_Driver/inc -I../../../device/soc/n32/n32g452 -I../../../utils/native/lite/include -I../../../commonlibrary/utils_lite/include -I../../../kernel/liteos_m/components/cpup -I../../../kernel/liteos_m/components/exchook -I../../../foundation/systemabilitymgr/samgr_lite/interfaces/kits/samgr -I../../../drivers/hdf_core/framework/core/common/include/manager -I../../../base/hiviewdfx/hilog_lite/interfaces/native/innerkits/hilog -I../../../base/hiviewdfx/hilog_lite/interfaces/native/innerkits -I../../../third_party/bounds_checking_function/include -I../../../kernel/liteos_m/arch/arm/common -I../../../kernel/liteos_m/arch/arm/cortex-m4/gcc -I../../../kernel/liteos_m/arch/arm/include -I../../../kernel/liteos_m/arch/include -I../../../kernel/liteos_m/kernel/include -I../../../third_party/cmsis/CMSIS/Core/Include -I../../../third_party/cmsis/CMSIS/RTOS2/Include -I../../../third_party/cmsis/CMSIS/DSP/Include -I../../../third_party/cmsis/CMSIS/DSP/PrivateInclude -I../../../kernel/liteos_m/kal/cmsis -I../../../kernel/liteos_m/kal/libc/newlib/porting/include -I../../../kernel/liteos_m/kal/posix/include -I../../../kernel/liteos_m/components/shell/include -I../../../kernel/liteos_m/components/signal -I../../../kernel/liteos_m/utils -I../../../device/board/breo/n32g452_breo/liteos_m -I../../../base/hiviewdfx/hiview_lite -I../../../base/hiviewdfx/hilog_lite/frameworks/mini -I../../../base/hiviewdfx/hilog_lite/interfaces/native/kits/hilog_lite -I../../../device/board/breo/n32g452_breo/Application -I../../../device/soc/n32/n32g452/N32G45x_Driver/include -Os -Og -Wall -fdata-sections -ffunction-sections -DUSE_STDPERIPH_DRIVER -DN32G452 -D__LITEOS_M__ -mcpu=cortex-m4 -mthumb -Og -Wall -fdata-sections -ffunction-sections -DUSE_STDPERIPH_DRIVER -DN32G452 -D__LITEOS_M__ -mcpu=cortex-m4 -mthumb -mcpu=cortex-m4 -fno-common -fno-builtin -fno-strict-aliasing -Wall -fstack-protector-all -std=c++11 -c ../../../base/hiviewdfx/hilog_lite/frameworks/featured/hilog.cpp -o obj/base/hiviewdfx/hilog_lite/frameworks/featured/libhilog_static.hilog.o
[OHOS ERROR] ../../../base/hiviewdfx/hilog_lite/frameworks/featured/hilog.cpp:66:1: error: expected '}' at end of input
[OHOS ERROR]    66 | } // namespace OHOS
[OHOS ERROR]       | ^
```

- 提示以下信息：进入`kernel/liteos_m`，然后make menuconfig，选中兼容项，失能musl库，改为new lib。
```
[OHOS ERROR] arm-none-eabi-gcc: fatal error: /home/jd_chen/Downloads/gcc-arm-none-eabi-10-2020-q4-major/bin/../lib/gcc/arm-none-eabi/10.2.1/../../../../arm-none-eabi/lib/nano.specs: attempt to rename spec 'link' to already defined spec 'nano_link'
[OHOS ERROR] compilation terminated.
```

- 提示以下编译报错，其中fsync函数用于将文件的所有修改同步到磁盘，而此处提示缺少相关定义，首先检查轻量系统有没有配置fs文件系统，另外本人另辟蹊径，自己定义了一个空实现，使得编译成功，如下：
```
int fsync(int fd)
{
  return 0;
}
```
报错信息如下：
```
[OHOS ERROR] [34/56] LINK obj/kernel/liteos_m/bin/liteos
[OHOS ERROR] FAILED: obj/kernel/liteos_m/bin/liteos obj/kernel/liteos_m/unstripped/bin/liteos 
[OHOS ERROR] ccache arm-none-eabi-gcc -static -Wl,--gc-sections -Wl,-Map=OHOS_Image.map --specs=nano.specs -Wl,-u_printf_float -Wl,--wrap=_calloc_r -Wl,--wrap=_malloc_r -Wl,--wrap=_realloc_r -Wl,
......
libs/libhal_sysparam.a libs/libhal_sys_param.a libs/libhilog_static.a libs/libparam_client_lite.a libs/libinit_utils.a libs/libinit_log.a libs/libsamgr.a libs/libsamgr_adapter.a libs/libsamgr_source.a libs/libbroadcast.a libs/libnative_file.a libs/libstatic_hal_file.a libs/libhilog_lite_static.a libs/libhiview_lite_static.a libs/libhievent_lite_static.a libs/libhuks_3.0_sdk.a -lc -lm -lnosys -Wl,--end-group -o obj/kernel/liteos_m/unstripped/bin/liteos  && ccache arm-none-eabi-strip --strip-unneeded "obj/kernel/liteos_m/unstripped/bin/liteos" -o "obj/kernel/liteos_m/bin/liteos"
[OHOS ERROR] /home/jd_chen/Downloads/gcc-arm-none-eabi-10-2020-q4-major/bin/../lib/gcc/arm-none-eabi/10.2.1/../../../../arm-none-eabi/bin/ld: libs/libhiview_lite_static.a(libhiview_lite_static.hiview_util.o):(.data.hiview_fsync+0x0): undefined reference to `fsync'
[OHOS ERROR] collect2: error: ld returned 1 exit status
```

- 提示找不到相应的函数实现或者相应的宏定义，可全局搜索其对应的实现，考虑在合适位置添加源文件路径或者头文件路径，从而使得编译成功


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

## 程序运行错误排查

### 程序运行至`hilog will init.`

**串口日志打印信息如下：**指示为内存访问错误，最终确认为结构体变量都设置为一字节对齐，导致的结构体访问指针变量 触发 非法内存访问错误
```
entering kernel init...
hilog will init.

*************Exception Information**************
Type      = 11
ThrdPid   = 25
Phase     = exc in task
FaultAddr = 0xabababab
Current task info:
Task name = (null)
Task ID   = 25
Task SP   = (nil)
Task ST   = 0x0
Task SS   = 0x0
Exception reg dump:
PC        = 0x80111e2
LR        = 0x8010b81
SP        = 0x20023fb0
R0        = 0x0
R1        = 0x1
R2        = 0x20011b14
R3        = 0x400
R4        = 0x8018a64
```

**原因分析：**

确认程序出错于函数`OHOS_SystemInit()`中，而后**在hiview_log.c找到如下代码：**
- 增加节点信息打印，发现函数未能执行到如下的**print1处**，说明`InitCoreLogOutput()`函数执行跑飞
```
static void HiLogInit(void)
{
    HIVIEW_UartPrint("hilog will init.\n");
//print0
    InitCoreLogOutput();

//print1
    /* The module that is not registered cannot print the log. */
    if (HiLogRegisterModule(HILOG_MODULE_HIVIEW, "HIVIEW") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_SAMGR, "SAMGR") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_UPDATE, "UPDATE") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_ACE, "ACE") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_AAFWK, "AAFWK") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_APP, "APP") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_GRAPHIC, "GRAPHIC") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_MEDIA, "MEDIA") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_DMS, "DMS") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_SEN, "SEN") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_SCY, "SCY") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_SOFTBUS, "SOFTBUS") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_POWERMGR, "POWERMGR") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_UIKIT, "UIKIT") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_GLOBAL, "GLOBAL") == FALSE ||
        HiLogRegisterModule(HILOG_MODULE_DATAMGR, "DATAMGR") == FALSE) {
        return;
    }
//print2
    HiviewRegisterInitFunc(HIVIEW_CMP_TYPE_LOG, InitLogOutput);
    HiviewRegisterInitFunc(HIVIEW_CMP_TYPE_LOG_LIMIT, InitLogLimit);
    HILOG_DEBUG(HILOG_MODULE_HIVIEW, "hilog init success.");
}
CORE_INIT_PRI(HiLogInit, 0);
```

**问题解决：**
在`target_config.h`添加失能`LOSCFG_PLATFORM_EXC`宏定义，或者在 debug.config 配置不使用内存访问错误，当然还可在源码中注释掉 此部分的使能，代码节选如下：
```
// kernel/liteos_m/arch/arm/cortex-m4/gcc/los_interrupt.c

    /* Enable USGFAULT, BUSFAULT, MEMFAULT */
    *(volatile UINT32 *)OS_NVIC_SHCSR |= (USGFAULT | BUSFAULT | MEMFAULT);

    /* Enable DIV 0 and unaligned exception */
#ifdef LOSCFG_ARCH_UNALIGNED_EXC
    *(volatile UINT32 *)OS_NVIC_CCR |= (DIV0FAULT | UNALIGNFAULT);
#else
    *(volatile UINT32 *)OS_NVIC_CCR |= (DIV0FAULT);
#endif
```

将上述代码中的`MEMFAULT`或者`UNALIGNFAULT`去掉 即可失能 内存非对齐访问错误



## 参考站点

- [Gitee-OpenHarmony基于命令行入门](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/quick-start/Readme-CN.md#/openharmony/docs/blob/master/zh-cn/device-dev/quick-start/quickstart-pkg-sourcecode.md)
- [轻量系统STM32F407芯片移植案例](https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/porting/porting-stm32f407-on-minisystem-eth.md)

