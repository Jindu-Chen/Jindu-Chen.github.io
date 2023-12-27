---
title: OpenHarmony-V4.0-LiteOS-M开发调试记录
date: 2023-11-29 12:00:01
tags:
categories:
- [LiteOS]
description: 参考社区的STM32之Ninja编译方式的工程范例，移植搭建基于LiteOS的N32WB452工程，记录其过程中的一些编译纠错、代码编写及构建、XTS子系统测试等相关信息，以供检索回溯
---



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



