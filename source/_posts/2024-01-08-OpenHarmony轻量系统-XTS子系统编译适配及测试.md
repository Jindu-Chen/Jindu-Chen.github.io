---
title: OpenHarmony轻量系统--XTS子系统编译适配及测试
date: 2024-01-08 03:38:03
tags:
categories:
- [LiteOS]
- [OpenHarmony]
description: 介绍基于国民技术N32G452芯片(Cortex-M4内核)的OpenHarmony-V4.0轻量系统移植工程，进行XTS子系统编译适配及应用兼容性测试分析的开发过程，以供参考
---



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


### 如何添加XTS子系统

- `hb build -b debug`

- 目前已知在//out/xxx/xxx/libs有.a库文件
  - 如何将其链接进bin文件，并使其得以运行呢
  - 在何处添加链接选项，添加怎样的链接选项呢？官方的链接选项会报错
  - 考虑在config.gni添加合适的链接选项，同时是否需要考虑在何处添加函数调用？

- 在工程配置`target_config.h`宏定义中使能Test测试项
  - 使能系统启动会进行相关函数调用
- 在`//kernel/liteos_m`中make menuconfig中，使能测试相关配置
- 在`//vendor/xxx/config.json`添加合适的子系统配置
- 在`//device/board/xxx/liteos_m/config.gni`添加适配的头文件路径
  - 链接选项中增添合适的静态库链接，以实现链接测试用例相关库，其中测试静态库位于`//out/${company}//${product}//libs`
  - 还须添加源文件路径？
- 在`//device/board/xxx/xxx.ld`链接脚本中添加OpenHarmony特有的测试段


### XTS是如何测试的


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



## 参考站点

- [OpenHarmony L0级设备XTS适配经验分享](https://blog.csdn.net/Via6666/article/details/132093613)





