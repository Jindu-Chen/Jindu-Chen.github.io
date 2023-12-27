---
title: OpenHarmony-V4.0-LiteOS-M鸿蒙轻内核开发调试记录
date: 2023-11-29 12:00:01
tags:
categories:
- [LiteOS]
description: 参考社区的STM32之Ninja编译方式的工程范例，移植搭建基于LiteOS的N32WB452工程，记录其过程中的一些编译纠错、代码功能编写及构建、XTS子系统测试等相关信息，以供检索回溯
---


## 目标工程的编译构建流程



## 编译纠错


- 提示以下信息，
    ```
    ../../../kernel/liteos_m/kernel/src/los_init.c:54:10: fatal error: los_exc_info.h: No such file or directory
    54 | #include "los_exc_info.h"
        |          ^~~~~~~~~~~~~~~~
    compilation terminated.
    [43/265] gcc cross compiler obj/kernel/liteos_m/kal/posix/src/posix.pthread_cond.o
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

## 添加系统引导、更新升级功能



## 轻量系统应用兼容性测试适配

- 在vendor目录下产品的config.json文件中，添加XTS子系统组件定义

### XTS子系统编译流程
- 通过hb工具读取系统配置文件(即vendor目录下产品的config.json)，获取参与编译的子系统以及部件信息
- 通过gn工具会读取XTS子系统BUILD.gn--all_features会读取轻量系统全部acts测试套件
  - 判断是否debug版本`if (ohos_build_type == "debug" && ohos_test_args != "notest") `，非debug版本XTS不参与编译
- 将轻量系统全部acts测试套件与产品的config.json子系统组件对比，存在的子系统部件参与最终编译
- 最终生成静态库.a归档路径：`out/.../.../libs`
  - XTS子系统: libmodule_xxx.a，测试框架：libhctest.a
- .a库链接操作

### XTS子系统测试项

轻量系统的全部测试项包含有：应用管理(Ability和Bundle)、网络通信(LwIP)、文件系统、系统参数、Wi-Fi IOT、分布式数据管理、安全、日志、事件、分布式调度、系统更新、系统引导
```
#//test/xts/acts/build_lite/BUILD.gn
    if (ohos_kernel_type == "liteos_m")
      all_features += [
        "//test/xts/acts/ability_lite/ability_hal:ActsAbilityMgrTest",
        "//test/xts/acts/appexecfwk_lite/appexecfwk_hal:ActsBundleMgrTest",
        "//test/xts/acts/communication_lite/lwip_hal:ActsLwipTest",

        #"//test/xts/acts/communication_lite/wifiservice_hal:ActsWifiServiceTest",
        "//test/xts/acts/commonlibrary_lite/file_hal:ActsUtilsFileTest",
        "//test/xts/acts/startup_lite/syspara_hal:ActsParameterTest",
        "//test/xts/acts/iothardware_lite/peripheral_hal:ActsWifiIotTest",
        "//test/xts/acts/distributeddatamgr_lite/kv_store_hal:ActsKvStoreTest",
        "//test/xts/acts/security_lite/huks/liteos_m_adapter:ActsHuksHalFunctionTest",
        "//test/xts/acts/hiviewdfx_lite/hilog_hal:ActsDfxFuncTest",
        "//test/xts/acts/hiviewdfx_lite/hievent_hal:ActsHieventLiteTest",
        "//test/xts/acts/distributed_schedule_lite/system_ability_manager_hal:ActsSamgrTest",
        "//test/xts/acts/update_lite/dupdate_hal:ActsUpdaterFuncTest",
        "//test/xts/acts/startup_lite/bootstrap_hal:ActsBootstrapTest",
      ]
```
- ActsAbilityMgrTest: 测试Ability管理器相关功能，包括Ability的启动、停止、生命周期管理等。<br>
- ActsBundleMgrTest: 测试Bundle管理器相关功能，包括Bundle的安装、卸载、查询等。<br>
- ActsLwipTest: 测试LwIP网络协议栈相关功能，包括TCP/IP连接、数据传输等。<br>
- ActsUtilsFileTest: 测试文件系统相关功能，包括文件的创建、读取、写入、删除等。<br>
- ActsParameterTest: 测试系统参数读取和设置功能。<br>
- ActsWifiIotTest: 测试Wi-Fi IOT相关功能，包括AP模式、Station模式、Socket连接等。<br>
- ActsKvStoreTest: 测试分布式数据管理相关功能，包括KV存储的读写、订阅等。<br>
- ActsHuksHalFunctionTest: 测试HUKS安全框架的HAL层功能，包括密钥管理、加解密等。<br>
- ActsDfxFuncTest: 测试HiLog日志框架功能。<br>
- ActsHieventLiteTest: 测试HiEvent事件框架功能。<br>
- ActsSamgrTest: 测试分布式调度功能，包括Ability间通信、服务发布和订阅等。<br>
- ActsUpdaterFuncTest: 测试系统更新功能。<br>
- 隐藏测试项: ActsWifiServiceTest: 测试Wi-Fi服务相关功能，但被注释掉（以#开头）。<br>
- ActsBootstrapTest: 测试系统引导功能。<br>


