---
title: PID算法
date: 2023-12-05
tags:
categories:
- [无刷电机]
description: 总结个人对PID算法的理解，以及PID在实际项目开发中的应用
---


## 概述

- PID算法，即比例、积分、微分控制算法
- 其表达式为：`u(t) = Kp * e(t) + Ki * ∫ e(t) dt + Kd * de(t)/dt`
    - 其中，u(t)为输出
    - e(t)是偏差，即当前值与目标值的差值


