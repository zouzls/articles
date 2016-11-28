---
title: git常用命令总结
date: 2016-11-17 11:29:27
tags: [git,常用命令]
categories: Git
---
因为最近项目用到git较多，一些常用的命令经常忘记，好记性不如烂笔头，索性总结一下。
<!--more-->
### 使用场景
目前的使用场景就是在gitlab或者github开了一个repository远程仓库，而且存在两个branch分支，分别是master和dev分支，其中master用作发布，dev用作开发，经常提交更新是在dev分支。
需要在多台机子上进行代码同步，虽然开发工具eclipse中集成了Git插件，但是很多时候插件并不够友好，还是需要用Git命令来搞定。
### Git命令总结
#### 1.Git配置
```bash
# 显示当前的Git配置,包括远程仓库、分支追踪关系等等
$ git config --list

# 设置提交代码时的用户信息，注意：加--global，表示全局，否则为当前项目
$ git config [--global] user.name "name"
$ git config [--global] user.email "email address"
```

#### 2.增加文件
```bash
# 添加当前目录的所有文件到暂存区（☆）
$ git add .

# 添加指定文件到暂存区
$ git add [file1] [file2] ...

# 添加指定目录到暂存区，包括子目录
$ git add [dir]

```
#### 3.提交文件
```bash
# 提交暂存区到仓库区
$ git commit -m [message]

# 使用一次新的commit，替代上一次提交
# 如果代码没有任何新变化，则用来改写上一次commit的提交信息
$ git commit --amend -m [message]

# 还有可以修改上次提交author和committor信息
```
#### 4.增加远程仓库
```bash
# 显示所有远程仓库
$ git remote -v

# 增加一个新的远程仓库，并命名，比如：
# git remote add origin ssh://xxx.xx.xx/xx.git
$ git remote add [shortname] [url]

```
如果没有添加远程仓库，eclipse中每次push都需要手动添加远程仓库url，会很繁琐。

#### 5.追踪关系
在某些场合，Git会自动在本地分支与远程分支之间，建立一种追踪关系（tracking）。比如，在git clone的时候，所有本地分支默认与远程主机的同名分支，建立追踪关系，也就是说，本地的master分支自动"追踪"origin/master分支，否则远程同步需要手动指定分支，而且体现在eclipse中就是另外一台机子eclipse拉取不到更新。
Git也允许手动建立追踪关系。
```bash
# 下面命令指定本地master分支追踪origin/dev分支。
$ git branch --set-upstream master origin/dev

# 下面的命令可以看到当前分支追踪的远程分支
$ git branch -vv
```
指定追踪关系之后，我重新再eclipse中pull就能更新代码了。

#### 6.远程同步
```bash
# =========================git push==============================
# 注意，分支推送顺序的写法是<来源地>:<目的地>，所以git pull是<远程分支>:<本地分支>，而git push是<本地分支>:<远程分支>。
$ git push <远程主机名> <本地分支名>:<远程分支名>


# 如果省略远程分支名，则表示将本地分支推送与之存在"追踪关系"的远程分支（通常两者同名），如果该远程分支不存在，则会被新建。
$ git push origin master
# 上面命令表示，将本地的master分支推送到origin主机的master分支。如果后者不存在，则会被新建。


# 如果当前分支与远程分支之间存在追踪关系，则本地分支和远程分支都可以省略。
$ git push origin
# 上面命令表示，将当前分支推送到origin主机的对应分支。


# 如果当前分支只有一个追踪分支，那么主机名都可以省略。
$ git push

# =========================git pull==============================
# git pull命令的作用是，取回远程主机某个分支的更新，再与本地的指定分支合并。它的完整格式稍稍有点复杂。
$ git pull <远程主机名> <远程分支名>:<本地分支名>

# 如果远程分支是与当前分支合并，则冒号后面的部分可以省略。
$ git pull origin dev

# 如果当前分支与远程分支存在追踪关系，git pull就可以省略远程分支名。
$ git pull origin
# 上面命令表示，本地的当前分支自动与对应的origin主机"追踪分支"（remote-tracking branch）进行合并。

# 如果当前分支只有一个追踪分支，连远程主机名都可以省略。
$ git pull
# 上面命令表示，当前分支自动与唯一一个追踪分支进行合并。
```

### 参考资料
[常用 Git 命令清单](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)
[Git远程操作详解](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)

### 最后
本文会随Git的使用时常更新。


-EOF-





