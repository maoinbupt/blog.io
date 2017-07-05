---
layout: post
title: Git常用命令
date: 2017-7-5
categories: blog
tags: [知识管理]
description: Git常用命令
---

## Git简介 ##
>Git 是 Linus开发的一个开源的的版本控制系统。它采用了分布式版本库的方式，因此无需服务器支持也可以在本地方便地建立版本库。

获取地址：

- [Git windows版](https://git-scm.com/download/win)
- [Git MAC 版](https://git-scm.com/download/mac)
- [Git Linux版](https://git-scm.com/download/linux)


## Git命令简介 ##

### 配置 ###
打开git bash，配置username 和email，这应与你在gitlab或者github的信息相对于

`git config --global user.name "username"`
`git config --global user.email "username@xxx.com"`

git bash 配置ssh以访问远程仓库

`ssh-keygen -t rsa`

将`C:\Users\username\.ssh`下生成的id_rsa.pub内容传给git服务器即可

### 初始化仓库 ###

#### 1.初始化本地仓库 ####



在本地仓库的目录下

`git init`

将filename文件添加到暂存区

`git add <filename>`

将全部可索引文件加入到暂存区

`git add .`

提交暂存区的文件

`git commit -m "My First Commit"`


查看提交记录便可以看到刚才的提交：

`git log`



介绍几个概念：工作区即为本地工作路径，stage为版本库中的‘暂存区’，init后仓库自动有了master分支，HEAD可以当做指向当前分支的头部的指针。add操作就是将本地文件加入到版本库的暂存区，commit即是将暂存区的文件都提交到当前分支。
![引用自廖雪峰的git教程](https://raw.githubusercontent.com/maoinbupt/blog.io/master/img/git-pic1.jpg)

如果想忽略某些文件，如build生成的临时文件等：进入目录，新建 .gitignore,编辑忽略的文件如：

`/bin`

`/gen`

为本地仓库建立远程仓库origin为默认远程仓库名：

`git remote add origin git@git.example.com:path/TestGit.git`

将刚才的commit提交到远程仓库：

`git push origin master`

#### 2.从远处仓库初始化到本地 ####

`git clone https://github.com/example/test.git`


#### 3.从远处仓库拉取代码 ####

将远程版本库的修改同步到本地,2种方法

- fetch:从远程版本库获取最新代码同步到本地，存放在origin/master分支上
 
`git fetch origin master`

查看远程版本库修改了哪些

`git diff origin/master`

将origin/master分支修改的内容合并到本地master分支上

`git merge origin/master`

- pull:相当于将fetch和merge一起执行

`git pull origin master`


### 分支 ###

分支可以想象为一个仓库下的不同文件夹，初始化仓库默认会建master分支，但是在开发过程中，一般需要多个分支，如developer分支来供主项目开发，developer_1.1同步进行1.1功能的开发，有可能还有bugfix分支，master可以作为正式版本的记录

查看当前都有哪些分支

`git branch -a`

创建一个分支：
`git branch developer`

切换到别的分支

`git checkout developer`

或创建切换合起来：

`git checkout -b developer`

将developer上的修改合并到master

`git checkout master`

`git merge developer`

删除分支,注意仅仅删除了本地分支

`git branch -D developer`

推送本地分支到远程

`git push origin local_branch:remote_branch`

切换到远程分支，并在本地建立本地分支，建立track remote关系,建立追溯关系后push就会自动push到目标远程分支

`git checkout -b developer origin/developer`

设定本地分支的远程父分支,建立track remote关系

`git branch --set-upstream-to=origin/developer developer`

### TAG ###

TAG相对分支最大的不同就是不可以在上面提交新的commit，相当于当前代码的‘快照’，但是可以基于tag拉分支，所以tag一般作为正式版本代码的一个标记。

在当前分支下创建一个TAG:

`git tag -a 0.1.3 -m "Release version 0.1.3"`

切换到标签：

`git checkout tagname`

推送到远程

`git push origin  tagname`

将本地所有tag都推送

`git push origin --tags`

获取远程tag：

`git fetch origin tag tagname`

给指定的commit打标签：

`git tag -a tagname commit_id`

删除本地标签：

`git tag -d tagname`

删除远程标签：

`git push origin --delete tag tagname`

或推送一个空tag到远程tag：

`git tag -d tagname`

`git push origin :refs/tags/tagname`



### 其它 ###

查看当前状态：

`git status`

查看修改内容：

`git diff`

`git diff src/com/example/providertest/MainActivity.java`

撤销修改

- 修改后直接撤销：

`git checkout src/com/example/providertest/MainActivity.java`

- add后撤销：

`git reset HEAD src/com/example/providertest/MainActivity.java`


强制回退到某个commit，不保存本地修改

`git reset --hard commit-id`

将其它分支的某个commit移到当前分支

`git cherry-pick commit_id`

为命令设置别名

`git config --global alias.co checkout  // checkout别名co`

`git config --git global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --date=relative"  //设置美化log，别名git lg`



好了，以上就是一些常用的GIT命令，希望对你有帮助。














