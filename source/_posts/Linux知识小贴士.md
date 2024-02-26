---
title: Linux知识小贴士
date: 2023-11-28
tags:
categories:
- [Linux]
description: 日常使用Linux的一些操作总结，不限定范围，涉及运维、辅助开发等相关应用点
---


#### 内网但非同一网段环境下开启NFS服务

- 在Linux主机和Windows主机之间开启NFS服务 之 Windows10通过NFS挂载Linux目录
  - 在Linux主机上配置NFS服务器
    - 输入命令`sudo apt-get install nfs-kernel-server rpcbind`安装NFS服务器和PRCBIND
    - 创建共享目录：`sudo mkdir /home/jd_chen/nfs_share`，其中/home/jd_chen/nfs_share为想要共享的目录
      - `chmod -R 777 /home/jd_chen/nfs_share`将所有权限递归应用于目标目录的所有文件和子目录
    - 配置NFS服务：`sudo vim /etc/exports`
      - 在文件中添加`/home/jd_chen/nfs_share *(rw,sync,no_root_squash)`
      - 其中*表示允许所有的网络段访问，rw表示访问者具有可读写权限，sync表示将缓存写入设备中，表示同步缓存，no_root_squash表示访问者具有root权限
    - 启动NFS服务：`sudo service nfs-kernel-server start`或者`systemctl start nfs-kernel-server`
      - `systemctl status nfs-kernel-server`验证NFS服务是否已启动
      - `systemctl enable nfs-kernel-server`配置在系统启动时自动启动NFS服务
      - `showmount -e`查看NFS共享目录
<br>

  - 在Windows下配置NFS客户湍服务
    - 打开`控制面板\程序\程序和功能`，点击`启用或关闭Windows功能`，找到`NFS服务`，勾选`NFS客户端`和`管理工具`
    - 重启电脑
<br>

  - 在Windows下挂载NFS目录 
    - 按下`Win+R`组合键打开运行对话框
    - 输入`mount 192.168.112.240:/home/jd_chen/nfs_share X:`
      - 其中192.168.112.240为Linux主机的ip地址，/home/jd_chen/nfs_share为需要共享的目录，X:为映射到Windows本地的盘符
  - 将命令mount改为umount即可卸载挂载，或者右键盘符->断开连接

---
- 在Linux主机使用挂载Linux服务器的NFS共享目录
  - 客户端执行`yum install -y nfs-utils`
  - 执行`mount <服务器IP>:<share_path> <挂载点>`
    - 如：`mount 192.168.1.1:/mnt/nfs /mnt/share`
      - 表示将服务器 192.168.1.1 上的 /mnt/nfs 目录挂载到客户端 /mnt/share 目录
    - 如：`mount -t nfs -o nolock,nfsvers=3 192.168.1.1:/mnt/nfs /mnt/share`
      - 其中-t nfs指定文件系统类型为NFS，-o nolock表示禁用文件锁定功能，nfsvers=3指定使用NFS版本3
  - 执行`df -h`验证NFS共享是否可用

---
- 嵌入式Linux开发中，配置NFS挂载根文件系统

---


#### 远程端口流量通过SSH映射到本地

在本地 Windows 打开 PowerShell，输入：`ssh -L 4000:localhost:4000 jd_chen@192.168.112.240`
- 其中，4000是要转发的端口以及映射到本地的端口，jd_chen为远程主机用户名
- 另外，如果为VSCode + SSH开发环境，可以在VSCode的终端界面`PORTS`直接添加转发/预览



#### Linux给根目录扩容


### 软件资源获取、下载、安装相关

**知道*.tar.gz文件的URL，则可以使用`wget`命令直接从网站上下载该文件**
- 如想下载的文件的URL是http://example.com/file.tar.gz
- 则可以`wget http://example.com/file.tar.gz`在当前目录下下载该文件

<br>

- 下载了*.tar.gz文件后，键入`tar -xzvf file.tar.gz`命令，解压文件到当前目录
- 而后进入其目录`cd file`，输入如下命令安装，当然以下只是方式之一。或者通常目录下或有安装脚本，使用sudo权限执行其脚本即可
- 安装完毕后，可以通过运行其可执行程序，如：xxx --version，检查是否安装成功，提示缺少依赖，则`sudo apt-get install xxx`安装对应库即可，或者`sudo apt --fix-broken install`安装缺失的软件包依赖

**通过源代码编译安装的方式：**
```
./configure
make
sudo make install
```

**基于.deb文件安装：**
- 使用dpkg命令，如：`sudo dpkg -i file.deb`
- 解决安装过程中的依赖：暂略，`sudo apt --fix-broken install`



### 命令行操作相关

在shell脚本中，`#!/bin/bash`通常放在第一行，当然也可以放在其它行，也可以去掉该行，或者也可以在该行之前放置其它如注释部分
- `#!/bin/bash`用于告诉系统使用何种shell解释器来解释此脚本，若不指定，则会使用默认shell解释器
- 但通常系统默认的解释器即为bash shell


#### ./xxx.sh和sh xxx.sh

直接使用./xx.sh会在当前shell中执行shell脚本，而使用sh xx.sh会在一个新的子shell中执行shell脚本。
- 在当前shell中执行shell脚本时，shell脚本中的变量和环境会影响当前shell
- 在子shell中执行shell脚本时，shell脚本中的变量和环境不会影响当前shell
<br>

以下复述自Brad：
- 如果shell脚本中定义了一个变量，但没有使用export命令将其导出到环境中，则该变量只在shell脚本中有效，在shell脚本执行完毕后，该变量就会被删除。
  > 当使用export命令将变量导出到环境中时，该变量会被添加到环境变量列表中。环境变量列表是所有shell共享的，包括父shell和子shell
- 如果shell脚本中定义了一个变量，并使用export命令将其导出到环境中，则该变量会一直存在，直到shell退出。
- 如果shell脚本中使用unset命令删除了一个变量，则该变量会立即被删除，无论该变量是否被导出到环境中
  > 使用unset命令删除一个变量，只会在当前shell中删除该变量。如果该变量被导出到环境中，则该变量会从环境变量列表中删除，但不会从其他shell中删除
- 如果shell使用exit命令退出，则shell脚本中的所有变量和环境都会被删除，无论该变量是否被导出到环境中


---

### Linux常用命令

#### 解压缩相关

将 /home/user/test.tar.gz 解压到 /home/user/ 目录：`tar -zxvf /home/user/test.tar.gz -C /home/user/`
