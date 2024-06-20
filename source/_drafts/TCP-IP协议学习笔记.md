---
title: TCP/IP协议学习笔记
date:
tags:
categories: 
description: 
---


## 前言

首先，为什么要学习TCP/IP协议呢？

本人在此前的嵌入式MCU项目开发中，基本上没接触过网络应用，但众所周知，TCP/IP协议是网络编程的基础，因此学习了解TCP/IP协议很有必要，另外在学习之前，也有一些前置疑问。

1. 为什么要深入学习TCP/IP协议呢？它对网络应用层编程有什么具体帮助呢？应用开发不是调API就行了吗？
2. 深入学习TCP/IP协议，需要哪些基础知识呢？能够解决哪些在应用开发中遇到的问题呢？举例？
3. 熟悉三次握手和四次挥手，对应用开发有什么帮助呢？举例？
4. 在嵌入式MCU、移动设备开发包括4G、Wi-Fi中，TCP/IP协议应用场景有哪些？具体的开发步骤有哪些？
5. 在嵌入式设备的无线通信中，无线网络通信通道是如何构建起来的？
6. http协议是什么？html是什么？与TCP/IP协议有什么关系？HTTP API和Socket API有什么区别？分别用于何种场景？HTTP API是基于Socket的吗？
7. TCP/IP 与 UDP 协议的联系与区别？
8. Socket API 是什么？有什么参数？Socket与网络编程的联系？网络编程最终底层都是基于Socket的开发？
9. Wi-Fi协议栈与TCP/IP协议栈的联系？4G/5G协议栈与Wi-FI协议栈的联系？蜂窝网络与Wi-Fi协议的联系？


## 概述TCP/IP协议

Socket（套接字） 是一个编程接口，是操作系统为网络通信提供的抽象层。它允许程序通过网络发送或接收数据。Socket本身并不直接等于TCP，而是可以基于不同的传输层协议，包括TCP和UDP（User Datagram Protocol）。当提到“TCP Socket”时，通常是指使用TCP作为传输层协议的Socket编程接口。

在使用TCP时，Socket API帮助开发者实现TCP的三次握手建立连接、数据传输、四次挥手断开连接等过程。


esp32


### ESP32之网络编程

ESP32完成TCP连接是通过Socket编程实现的。以下是ESP32建立TCP连接的基本步骤，这些步骤正是基于Socket API的操作：

初始化Wi-Fi连接：首先，ESP32需要连接到一个可用的Wi-Fi网络。这通常涉及到使用ESP-IDF框架中的Wi-Fi库来设置SSID（网络名称）和密码，然后调用相应的函数连接到指定的Wi-Fi网络。

创建Socket：一旦Wi-Fi连接成功，接下来在ESP32上创建一个Socket。这通常通过调用socket()函数实现，需要指定地址族（如AF_INET表示IPv4）、套接字类型（如SOCK_STREAM用于TCP）和协议（通常为0，让系统选择默认的TCP协议）。

配置服务器地址：使用struct sockaddr_in结构体设置服务器的IP地址和端口号，然后通过connect()函数尝试与服务器建立TCP连接。这一步会触发TCP的三次握手过程。

数据收发：连接建立后，可以使用send()或write()函数发送数据到服务器，使用recv()或read()函数接收服务器响应的数据。在ESP32中，为了提高效率和响应性，通常会将数据收发放在不同的任务（threads）中执行。

关闭连接：数据传输完成后，可以通过调用close()函数来关闭Socket，这会触发TCP的四次挥手过程，释放网络资源。

整个过程是基于ESP-IDF框架提供的Socket API完成的


## 参考站点

- [一文讲透TCP/IP协议 | 图解+秒懂+史上最全](https://blog.csdn.net/sunyctf/article/details/128975665?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-128975665-blog-118386484.235^v43^pc_blog_bottom_relevance_base6&spm=1001.2101.3001.4242.1&utm_relevant_index=3)


