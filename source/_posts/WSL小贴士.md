---
title: WSL小贴士
date: 2023-11-28 11:20:51
updated: 2023-12-04
tags:
categories:
- [WSL]
description: 日常使用WSL开发的一些操作总结
---


#### 在windows下访问WSL的目录
- 在wwindows的资源管理器输入`\\wsl$`即可
- 直接访问WSL的根路径
  - `C:\Users\xxx\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc\LocalState\rootfs`
  - 其中`xxx`为个人用户名
  - 如果过程中没有找到对应的文件夹，请勾选`资源管理器->查看->隐藏的项目`
<br>

#### 重启WSL

- 打开PowerShell
  - 输入命令`Get-Service LxssManager | Restart-Service`，重启LxssManager服务
  - 停止LxssManager服务/关机：`net stop LxssManager`
  - 启动LxssManager服务/开机：`net start LxssManager`


#### WSL下通过命令执行Windows下的bat脚本
- `wsl.exe -d Ubuntu-18.04 cmd.exe /c "C:\Users\program.bat"`
  - 系统发行版本与脚本路径 替换为实际的即可


#### 恢复误删文件



#### 开启WSL的NFS服务
	实现u-boot通过nfs服务挂载WSL下的根文件系统


#### 删除WSL中的一个Linux发行版

WSL可以同时安装多个Linux发行版，打开PowerShell：
- 输入`wsl --list`列出所有已安装的发行版
- 输入`wsl --unregister Ubuntu-18.04`表示删除其中名为`Ubuntu-18.04`的发行版，而后静静等待命令执行完毕即可


#### 参考站点

