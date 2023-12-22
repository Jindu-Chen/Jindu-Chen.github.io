---
title: 嵌入式MCU的bootloader设计
date: 2023-12-22 02:30:03
tags:
categories:
description: 讲述在MCU开发中bootloader的设计，其主要工作流程为芯片上电->执行bootloader代码->初始化时钟和配置必要外设->检测是否需要进行固件更新，是-则将新固件从存储区拷贝替换至应用程序运行区，然后跳转至应用程序入口；否-则直接跳转至应用程序入口开始执行业务。
---


