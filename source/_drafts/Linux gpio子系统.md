---
title: Linux gpio子系统
date: 2023-12-19 10:21:09
tags:
categories:
- [Linux]
- [I.MX6ULL]
description: gpio子系统对上层提供了用于设置GPIO方向、电平、速度等参数的API函数，使得开发者只需在设备树添加gpio相关信息后、使用其API函数即可简便地使用GPIO。本文从个人角度出发，讲述对gpio子系统的理解和部分源码剖析。
---


## gpio子系统基本概念简述

- 首先，需要区分pinctrl和gpio子系统
  - pinctrl子系统是设置引脚(PIN/PAD)的复用属性和电气属性，一个引脚如果被复用为GPIO属性，那么就需要用到gpio子系统
  - gpio子系统，提供设置和使用GPIO的系列API，如：配置输入输出方向、设置输出电平、读取输入电平、设置输出速度、注册GPIO中断服务函数等等


