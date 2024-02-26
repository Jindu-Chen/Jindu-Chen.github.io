---
title: Git使用小贴士
date: 2023-12-21
tags:
categories:
- [Git]
description: 主要记录Git的常用命令、一些非典型操作及其意义解析，也包括在 Git 应用中所遇到的一些问题及解决方案，便于检索与回顾
---


## 常用命令表

| 命令名称 | 命令 | 备注 |
|---|---|---|
| 克隆工程 | git clone xxx | xxx为克隆链接 |
| 暂存文件修改、删除 | git add -u | 暂存文件修改、删除 |
| 暂存文件修改、新建 | git add . | 暂存文件修改、新建 |
| 暂存文件修改、删除、新建 | git add -A | 暂存文件修改、删除、新建 |
| 暂存文件 | git add xxx | xxx可为目录、文件，为.时暂存所有 |
| 取消所有已暂存文件 | git reset HEAD . | 放弃所有暂存，不会取消修改 |
| 取消指定文件的暂存 | git reset HEAD filepathname | 放弃指定文件的暂存 |
| 放弃未暂存的指定文件的修改 | git checkout -- filepathname | 放弃未暂存的文件修改 |
| 放弃未暂存的文件修改 | git checkout . | 不会删除新建的，未被git跟踪的文件 |
| 显示将要被删除的文件 | git clean -n | 显示将要被删除的文件 |
| 删除工作区新增的文件、文件夹 | git clean -df | 删除新增的文件和新增的文件夹 |
| 删除工作区新增的文件 | git clean -f | 删除当前目录下所有没被track过的文件，.gitignore指定的文件和文件夹不会被删除 |
| 提交 | git commit -m "xxxxx" | xxx为备注信息 |
| 查看修改的地方 | git diff xxx | 查看修改的地方 |
| 查看工作区、暂存区 | git status | 查看工作区、暂存区 |
| 拉取 | git pull | 在主目录下操作 |
| 合并分支 | git merge dev | 把dev分支合并到当前分支 |
| 合并目标分支的指定文件 | git merge dev --no-ff filepathname | 合并目标分支的指定文件 |
| 推送 | git push | 推送 |
| 强制推送 | git push -f origin master | 适用于本地回退版本后修改推送，master为分支名 |
| 更新远程分支列表 | git remote update origin --prune | 与远程保持同步 |
| 更新子模块 | git submodule update --init | 首次后可不带--init |
| 切换到某一本地分支 | git checkout dev | dev为分支名 |
| 查看本地分支 | git branch | 查看本地分支 |
| 切换到本地某个分支 | git checkout dev | 切换到本地某个分支 |
| 查看本地分支与远程分支的关联情况 | git branch -vv | 查看本地分支与远程分支的关联情况 |
| 查看远程分支 | git branch -a | 查看远程分支 |
| 创建本地分支并切换 | git checkout -b V1.1 | V1.1为分支名 |
| 创建远程分支 | git push --set-upstream origin V1.1 | 建立本地分支与新建远程分支的关联 |
| 新建本地分支并跟踪远程分支 | git branch dev origin/dev | dev为分支名称 |
| 新建本地分支并跟踪远程分支且切换 | git checkout -b dev origin/dev | 本地新建分支 dev ，跟踪远程的同名分支 dev |
| 本地新建分支 dev ，跟踪远程的同名分支 dev | git checkout --track origin/dev | 本地新建分支 dev ，跟踪远程的同名分支 dev |
| 获取远程地址及通信方式 | git remote -v | 以git开头为ssh协议 |
| 更新本地仓库的remote地址 | git remote set-url origin xxx | xxx为远程地址 |
| 回退到上一个版本 | git reset --hard HEAD^ | 删除工作区的改动代码，撤销commit、add |
| 回退到上X个版本 | git reset --hard HEAD~3 | ~4，删除期间的所有改动，撤销commit、add |
| 撤销提交（commit） | git reset --soft HEAD^ | 不删除工作区的改动代码，不撤销add |
| 查看代码提交记录 | git log | 查看代码提交记录 |
| 删除本地分支 | git branch -d Chapater6 | Chapater6为分支名 |
| 删除远程分支 | git push origin --delete Chapater6 | |
| 删除本地标签 | git tag -d rel1 | rel1为标签名 |
| 查看所有本地标签 | git tag -l | |
| 删除远程标签 | git push --delete origin prod1.0 | |
| 查看远程仓库信息 | git remote show origin | 查看远程仓库信息 |


## 注意项解答

### 执行 git pull 会覆盖本地的修改吗？
  
没有冲突的情况下，远端会直接更新至本地上，但不会改变本地未提交的变动；如果本地修改已提交，则会执行一个远端分支和本地分支的合并

### git fetch 和 git pull 的区别与联系

`git fetch`用于从远程仓库获取最新的提交，保存到本地的远程跟踪分支中（`FETCH_HEAD`），可以通过查看此分支了解远程仓库的更新情况
- `git diff FETCH_HEAD`比较查看该分支和当前工作分支的内容

<br>

`git pull`会自动获取远程仓库的更新，并且合并到当前分支上，相当于`git fetch` + `git merge FETCH_HEAD`
- 将远程仓库中指定分支的最新提交 ID 保存到本地的 FETCH_HEAD 分支中
- 将 FETCH_HEAD 分支合并到当前工作分支中


## 基础非典型操作

### 本地git配置
<br>

**配置本地与远端的SSH密钥连接流程：**
- 本地生成SSH公钥和私钥(如果没有的话，另，linux下公钥通常存放于`~/.ssh/*.pub`)
  - `ssh-keygen -t rsa -b 4096 -C xxx@xxx.com`
- 复制公钥，添加至远端平台的SSH设置上
<br>

**查看本地配置：**
- `git config --list`查看当前项目的所有配置
- `git config --global --list`查看全局配置
<br>

**修改用户名(全局/当前项目)**

此用户名即提交日志上所展示的用户名称

  - 修改全局用户名：`git config --global user.name "xxx"`，影响用户的所有仓库
  - 修改当前路径项目的用户名：`git config user.name "xxx"`
  - 查看全局用户名：`git config user.name`
<br>

**初始化本地工程并与远端已有仓库的main分支关联：**
- 进入工程根目录，`git init`初始化本地仓库
- 添加远程仓库：`git remote add origin <远程仓库地址>`
- `git branch -M main`将当前分支重命名为`main`，M即`--move --force`的缩写。（可以分别输入`git add --all`，`git commit -m "first commit"`完成对本地分支的首次提交）
- 使用`git pull origin main`，将远程仓库的main分支拉取到本地，或者`git push -u origin main -f`将本地的xxx分支强制推送到远端main分支，其中-u是`--set-upstream`的缩写，后续会保持这个跟踪关系



### 标签相关操作

**克隆远程仓库上指定标签对应的工程：**
- 模板：`git clone -b v1.0.0 https://github.com/example/my_repo`--克隆仓库 https://github.com/example/my_repo 中标签 v1.0.0 对应的源码到当前目录
- 如：`git clone -b OpenHarmony-v4.0-Release https://gitee.com/openharmony/kernel_liteos_m.git`


### 本地仓库同时推送远端的多个平台（推送gitee、github）

**方法一：为远程仓库指定多个地址**
- 在本地仓库的根目录有`.git`隐藏目录，找到`.git/config`文件，打开编辑，如下所示
- 其中代码行`url = git@gitee.com:Jindu-Chen/jindu-chen.git`为新增（首先需要在远程创建/拥有此仓库，并且本地用户已经配置好与gitee、github的连接）
- 保存退出，此时修改本地代码并提交，会同时推送到两个远端
```
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = git@gitee.com:Jindu-Chen/jindu-chen.git       # 新添加行
        url = git@github.com:Jindu-Chen/Jindu-Chen.github.io.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
[user]
        name = Jindu Chen
```



### .gitignore忽略规则

- 在git项目根目录下创建.gitignore文件，然后添加需要忽略跟踪的文件选项
  - .gitignore文件仅对当前目录及其子目录起作用
- 此前已经被跟踪的文件，添加.gitignore无效
    >执行`git rm --cached /path/to/remove`取消跟踪，然后提交推送

- 强行跟踪被.gitignore忽略的某个指定文件
`git add -f /path/to/add`或者 在.gitignore文件后添加`!path/to/track`(`!`表示覆盖前面的忽略规则)
如：`git add -f hello.bin` or 添加`!hello.bin`

__附上一个自用的.gitignore文件：__
```
  .vscode
  .history
  build
  Listings
  Objects

  # Listing Files
  *.i
  *.lst
  *.map

  # Object Files
  *.axf
  *.b[0-2][0-9]
  *.b3[0-1]
  *.bak
  *.build_log.htm
  *.crf
  *.d
  *.dep
  *.elf
  *.htm
  *.iex
  *.lnp
  *.o
  *.obj
  *.sbr

  # Firmware Files
  *.bin
  *.h86
  *.hex

  # Debugger Files
  .ini
```


### 版本发布

## 进阶操作

### 钩子配置(Git Hooks)

- Hook 即在执行某个事件之前或之后进行一些其他额外的操作，如下
  - 自动部署代码
  - 自动进行代码审查
  - 安全审查
  - 日志记录
  - 通知用户
- Git有许多的事件（commit、push 等等），每个事件也应了有不同的钩子（如 commit 前，commit 后）
- 使用：暂略


### 子模块

- 对于需要流水线自动化作业编译的项目，可以将其工具链单独存放，作为子模块被调用

- 删除项目内已有的子模块配置，将其转化为本项目内容
```    
cd ./path/to/submodule
rm -rf .git

# at project root directory
git rm --cached ./path/to/submodule
git add --all
git commit -m "xxx"
git push
```

- 克隆包含子模块的仓库后，子模块的拉取
    `git submodule update --init --recursive`


## 非典型问题解决方案

### 同一Linux主机-多github账户的权限报错问题

明明已经配置好本地与 Github 云端的 SSH 密钥，但依旧报错，内容如下：
```
ERROR: Permission to xxx.git denied to xxx.
fatal: 无法读取远程仓库。
请确认您有正确的访问权限并且仓库存在。
```
<br>

解决步骤如下：
- 如果需要使用两个不同的 Github 账户，则需要在本地主机上生成两个相应的 SSH Key，将公钥分别添加到不同 Github 账户上
- 由于主机有两个 SSH Key，导致 Github 无法识别对应关系，所以会报错权限问题
- 这时，需要用户手动配置对应映射关系，给另一个 Github 账户的站点**起一个别名**，起到区分作用即可
- 进入 .ssh 文件夹 -- `cd ~/.ssh`，创建配置文件 -- `vim config`，添加内容如下：
```
  #Default GitHub
  Host github.com             # 选定一个默认账户，别名不用更改
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa    # 对应其生成的密钥

  #new github
  Host github-alias           # 另一个账户的别名
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa1   # 另一个账户所对应的密钥
```
假设现在有一个仓库，是属于另一个 Github 账户的，那么需要更改其远程仓库链接为别名即可：

比如，将`git@github.com:Jindu-Chen/Jindu-Chen.github.io.git`改为`github-alias:Jindu-Chen/Jindu-Chen.github.io.git`



## 参考站点

- [What does the -M mean in git branch -M main?](https://stackoverflow.com/questions/68277661/what-does-the-m-mean-in-git-branch-m-main)
- [Github remote permission denied](https://stackoverflow.com/questions/47465644/github-remote-permission-denied)

