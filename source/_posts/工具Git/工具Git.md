---
title: 工具Git
date: 2023-10-30 00:00:00
updated: 2023-10-30 00:00:00
description: Git是一种十分方便的项目管理工具，最常用的是Github托管代码和Vscode管理差异
categories:
- 技术
- 项目管理工具
- Git
tags:
- Git 
aside: true
top_img: img/a2.png
cover: img/a2.png
---

# 工具Git

## Git工作流程与原理

### 工作流程

一般工作流程如下：

- 克隆 Git 资源作为工作目录。
- 在克隆的资源上添加或修改文件。
- 如果其他人修改了，你可以更新资源。
- 在提交前查看修改。
- 提交修改。
- 在修改完成后，如果发现错误，可以撤回提交并再次修改并提交。

<img src="\asset\git-process.png" style="zoom:67%;" />

### 工作区、暂存区和版本库

Git 工作区、暂存区和版本库概念：

- **工作区：**就是你在电脑里能看到的目录。
- **暂存区：**英文叫 stage 或 index。一般存放在 **.git** 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
- **版本库：**工作区有一个隐藏目录 **.git**，这个不算工作区，而是 Git 的版本库。

<img src="\asset\git-workspaace.jpg" style="zoom: 67%;" />

- 图中左侧为工作区，右侧为版本库。在版本库中标记为 "index" 的区域是暂存区（stage/index），标记为 "master" 的是 master 分支所代表的目录树。
- 图中我们可以看出此时 "HEAD" 实际是指向 master 分支的一个"游标"。所以图示的命令中出现 HEAD 的地方可以用 master 来替换。
- 图中的 objects 标识的区域为 Git 的对象库，实际位于 ".git/objects" 目录下，里面包含了创建的各种对象及内容。
- 当对工作区修改（或新增）的文件执行 **git add** 命令时，暂存区的目录树被更新，同时工作区修改（或新增）的文件内容被写入到对象库中的一个新的对象中，而该对象的ID被记录在暂存区的文件索引中。
- 当执行提交操作 **git commit** 时，暂存区的目录树写到版本库（对象库）中，master 分支会做相应的更新。即 master 指向的目录树就是提交时暂存区的目录树。
- 当执行 **git reset HEAD** 命令时，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。
- 当执行 **git rm --cached <file>** 命令时，会直接从暂存区删除文件，工作区则不做出改变。
- 当执行 **git checkout .** 或者 **git checkout -- <file>** 命令时，会用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区中的改动。
- 当执行 **git checkout HEAD .** 或者 **git checkout HEAD <file>** 命令时，会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

### 分支管理

<img src="\asset\git-brance.svg" style="zoom: 50%;" />

​	一个分支代表一条独立的开发线。使用分支意味着你可以从开发主线上分离开来，然后在不影响主线的同时继续工作。Git 分支实际上是指向更改快照的指针。

## Git使用指南

<img src="\asset\git-command.jpg"  />

Git 常用的是以下 6 个命令：**git clone**、**git push**、**git add** 、**git commit**、**git checkout**、**git pull**

**说明：**

- workspace：工作区
- staging area：暂存区/缓存区
- local repository：版本库或本地仓库
- remote repository：远程仓库

### 最常用命令

```shell
# 以下是最常用的命令组合

# 克隆远端仓库 不写分支则克隆默认分支
git clone -b <branch> [href]

# 本地操作
## 初始化新的本地仓库
git init
## 本地提交组合
git add .
git commit -m "Message"

# 远端操作
## 查看远端信息
git remote -v
## 推送至远端
git push [remote] [branch]
## 拉取从远端
git pull [remote] [branch]

# 切换用户，同时还要删除Windows的凭证
git config --global user.name "VVolfBite"
git config --global user.email 1327256274@qq.com

# 查看状态
git status 

# 添加到ignore
路径用/ 通配符可以使用\*


```

### Git用户配置

```shell
#查看全部配置
git config --list 
#配置用户信息
git config --global user.name "VVolfBite"
git config --global user.email 1327256274@qq.com
```

​	如果用了 **--global** 选项，那么更改的配置文件就是位于你用户主目录下的那个，以后你所有的项目都会默认使用这里配置的用户信息。

​	如果要在某个特定的项目中使用其他名字或者电邮，只要去掉 --global 选项重新配置即可，新的设定保存在当前项目的 .git/config 文件里。

### Git创建仓库

```shell
# 创建仓库，若不加路径就是当前目录创建
git init [path]
	--path为创建的路径
# 拷贝仓库 会下载一个远端仓库
 git clone [url]
 	-- url为拷贝的路径
```

### Git本地操作

```shell
# 添加文件到暂存区
git add [file | dir]
	-- 一般做法是 git add . 即添加当前目录
	
# 提交暂存区到仓库,若不加file就是提交暂存区到仓库区
git commit  [file | dir] -m [message]
	-- 一般做法是 git commit -m "message" 
	
# 将文件从暂存区和工作区中删除
git rm <file>
# 把文件从暂存区域移除，但仍然希望保留在当前工作目录中
git rm --cached <file>

# 移动或重命名一个文件、目录或软连接，感觉没啥用
git mv [file] [newfile]

# 查看分支
git branch -a
# 创建分支
git branch <branchname>
# 删除分支
git branch -d (branchname)
# 切换分支
git switch <branch-name>
# 创建并切换分支
git switch -c <new-branch-name>
# 合并分支命令
git merge 
```

### Git远程操作

```shell
# 列出当前仓库中已配置的远程仓库
git remote
# 列出当前仓库中已配置的远程仓库，并显示它们的 URL
git remote -v
# 添加一个新的远程仓库。指定一个远程仓库的名称和 URL，将其添加到当前仓库中。
git remote add origin https://github.com/user/repo.git
# 将已配置的远程仓库重命名。
git remote rename origin new-origin
# 从当前仓库中删除指定的远程仓库。
git remote remove new-origin
# 修改指定远程仓库的 URL。
git remote set-url origin https://github.com/user/new-repo.git
# 显示指定远程仓库的详细信息，包括 URL 和跟踪分支。
git remote show origin

# 从将本地的分支版本上传到远程并合并
git push <远程主机名> <本地分支名>:<远程分支名>
# 如果本地分支名与远程分支名相同，则可以省略冒号
git push <远程主机名> <本地分支名>
# 如果本地版本与远程版本有差异，但又要强制推送可以使用 --force 参数
git push --force origin master

# 提取更新的数据，你可以首先执行
git fetch [alias]
# 合并分支到你的当前分支；fetch完成之后要尝试合并merge
git merge [alias]/[branch]

# 从远程获取代码并合并本地的版本。git pull 其实就是 git fetch 和 git merge FETCH_HEAD 的简写。
git pull <远程主机名> <远程分支名>:<本地分支名>
```

