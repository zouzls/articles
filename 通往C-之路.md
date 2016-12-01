---
title: 通往C++之路
date: 2016-11-25 00:47:51
tags: C++
categories: C++
---
最近在准备看看LevelDB的源码，因为它是一个写入性能高效的基于硬盘的K-V键值存储，区别于Redis这种基于内存的键值对存储数据库，仅仅知道它是以LSM-Tree数据结构实现而成并不能满足我的好奇心，果断还是想一窥LevelDB的庐山真面目，但是它毕竟又是以CPP写的，这给我这种之前长期驻扎Java为阵营的成员还是带来了不少的学习成本，但是个人觉得语言永远应该不能成为阻挡你理解计算机世界的拦路虎。
<!--more-->
思索再三，Java凭借JVM的强大带来自身编码效率和安全性的提升在包括Web的很多领域确实是一把利器，但是要想在存储或者数据库领域撬起一块砖头，还是需要CPP这把短刃的。

有过面向对象和C的基础，踏上的CPP的道路应该不会太难，这篇文章会记录在CPP学习过程中的一些觉得不错的文章资料总结或者心得。当然学习语言并不是目的，而是应该成为解决问题的工具。

查阅文章资料如下：
- [C++简易教程](http://www.runoob.com/cplusplus/cpp-tutorial.html)
- [.h和.cpp文件的区别](http://www.cnblogs.com/shelvenn/archive/2008/02/02/1062446.html)

在Ubuntu的eclipse中搭建了Leveldb的工程环境，写了一个test.cc但是怎么都运行不了，最后发现是没有更改leveldb自带的Makefile文件，没有在Makefile中加入test.cc的编译和链接指令。下文是介绍工程中Makefile的作用：
- [什么是Makefile ](http://www.cppblog.com/luqingfei/archive/2010/08/27/124946.html)
- [C/C++中各种类型int、long、double、char表示范围和位数](http://blog.csdn.net/xuexiacm/article/details/8122267)
- [大端与小端存储模式详解](http://blog.csdn.net/favory/article/details/4441361)
本文会随着学习情况逐渐更新。

-EOF-
