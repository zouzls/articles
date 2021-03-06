---
title: Raft算法笔记（一）
tags:
  - 分布式系统
  - 算法
  - Raft
categories: Distributed System
date: 2016-08-26 15:04:16
---
之前记得微博上有个博主在关于知乎上讨论「精通C++语言是一种怎样的体验」的时候，评论了一句，印象很深刻，刚刚翻出来有看了一下，原话是：“建议青年学子多把时间花在基础理论「算法，操作系统，编译原理」和具体领域「数据库，分布式系统，并发编程，机器学习」”，私以为是有几分道理的，然后最近不知怎么就对分布式感兴趣起来了。技术学习无止境，但是保持好奇心和敏锐度，总是需要的。
本文是浓缩版（虽然我觉得原论文写得更好，没有一句废话），主要是记录在阅读Raft算法过程中的理解和笔记，方便日后回头温习，如果感兴趣并且条件允许可以阅读论文原文和论文中文翻译，见文末参考资料。
<!--more-->
### 一致性问题和一致性算法
分布式系统的一个主要问题就是一致性问题，通俗的理解就是如何使一组服务器中的每个成员数据保持一致，当其中某个服务器收到客户端的一组指令时,它必须与其它服务器交流以保证所有的服务器都是以同样的顺序收到同样的指令,这样的话所有的服务器会产生一致的结果,看起来就像是一台机器一样。
为了解决这个问题而出现的解决方案就是一致性算法，一致性算法允许一组机器像一个整体一样工作，即使其中一些机器出现故障也能够继续工作下去。正因为如此，一致性算法在构建可信赖的大规模软件系统中扮演着重要的角色。
在过去的 10 年里，Paxos 算法统治着一致性算法这一领域。但是Paxos太难理解了，折磨了不少学生和开发者等等，于是有人受不了了看不下去了，决定改变这种局面，于是Raft算法这种更加具有亲和力和接地气的算法应运而生，得天时地利人和。作者是来自Stanford University的Professor：Diego Ongaro 和 John Ousterhout。
### 摘要
Raft 是一种为了管理复制日志的一致性算法。它提供了和 Paxos 算法相同的功能和性能，但是它的算法结构和 Paxos 不同，使得 Raft 算法更加容易理解并且更容易构建实际的系统。为了提升可理解性，Raft 将一致性算法分解成了几个关键模块，例如领导人选举、日志复制和安全性。同时它通过实施一个更强的一致性来减少需要考虑的状态的数量。从一个用户研究的结果可以证明，对于学生而言，Raft 算法比 Paxos 算法更加容易学习。Raft 算法还包括一个新的机制来允许集群成员的动态改变，它利用重叠的大多数来保证安全性。
### 介绍
#### 首要目标是可理解性
Raft设计的首要目标是可理解性：我们是否可以在实际系统中定义一个一致性算法，并且能够比 Paxos 算法以一种更加容易的方式来学习。此外，我们希望该算法方便系统构建者的直觉的发展。不仅一个算法能够工作很重要，而且能够显而易见的知道为什么能工作也很重要。
#### 如何提升可理解性
我们使用一些特别的技巧来提升它的可理解性，包括算法分解（Raft 主要被分成了领导人选举，日志复制和安全三个模块）和减少状态机的状态（相对于 Paxos，Raft 减少了非确定性和服务器互相处于非一致性的方式）。
#### Raft的独特特性
- 强领导者：
和其他一致性算法相比，Raft 使用一种更强的领导能力形式。比如，日志条目只从领导者发送给其他的服务器。这种方式简化了对复制日志的管理并且使得 Raft 算法更加易于理解。
- 领导选举：
Raft 算法使用一个随机计时器来选举领导者。这种方式只是在任何一致性算法都必须实现的心跳机制上增加了一点机制。在解决冲突的时候会更加简单快捷。
- 关系调整：
Raft 使用一种共同一致的方法来处理集群成员变换的问题，在这种方法中，两种不同的配置都要求的大多数机器会重叠。这就使得集群在成员变换的时候依然可以继续工作。

### 复制状态机
#### 一致性算法的提出背景
一致性算法是从复制状态机的背景下提出的。在这种方法中，一组服务器上的状态机产生相同状态的副本，并且在一些机器宕掉的情况下也可以继续运行。复制状态机在分布式系统中被用于解决很多容错的问题。例如，大规模的系统中通常都有一个集群领导者，像 GFS、HDFS 和 RAMCloud，十分典型的使用一个单独的复制状态机去管理领导选举和存储配置信息并且在领导人宕机的情况下也要存活下来。比如 Chubby 和 ZooKeeper。
#### 基于复制日志实现
复制状态机通常都是基于复制日志实现的。每一个服务器存储一个包含一系列指令的日志，并且按照日志的顺序进行执行。每一个日志都按照相同的顺序包含相同的指令，所以每一个服务器都执行相同的指令序列。因为每个状态机都是确定的，每一次执行操作都产生相同的状态和同样的序列。
#### 保证复制日志相同
保证复制日志相同就是一致性算法的工作了。在一台服务器上，一致性模块接收客户端发送来的指令然后增加到自己的日志中去。它和其他服务器上的一致性模块进行通信来保证每一个服务器上的日志最终都以相同的顺序包含相同的请求，尽管有些服务器会宕机。一旦指令被正确的复制，每一个服务器的状态机按照日志顺序处理他们，然后输出结果被返回给客户端。因此，服务器集群看起来形成一个高可靠的状态机。

### 为了可理解性的设计
#### 设计Raft的几个初衷：
- 它必须提供一个完整的实际的系统实现基础，这样才能大大减少开发者的工作；
- 它必须在任何情况下都是安全的并且在大多数的情况下都是可用的；
- 并且它的大部分操作必须是高效的。
- 但是我们最重要也是最大的挑战是可理解性。它必须保证对于普遍的人群都可以十分容易的去理解。
- 另外，它必须能够让人形成直观的认识，这样系统的构建者才能够在现实中进行必然的扩展。

#### 解决可理解性的问题
我们意识到对这种可理解性分析上具有高度的主观性；尽管如此，我们使用了两种通常适用的技术来解决这个问题。
- 第一个技术就是众所周知的问题分解：
只要有可能，我们就将问题分解成几个相对独立的，可被解决的、可解释的和可理解的子问题。例如，Raft 算法被我们分成领导人选举，日志复制，安全性和角色改变几个部分。
- 第二个方法是通过减少状态的数量：
来简化需要考虑的状态空间，使得系统更加连贯并且在可能的时候消除不确定性。尤其是，随机化方法增加了不确定性，但是他们有利于减少状态空间数量，通过处理所有可能选择时使用相似的方法。我们使用随机化去简化 Raft 中领导人选举算法。

### Raft一致性算法
Raft 通过选举一个高贵的领导人，然后给予他全部的管理复制日志的责任来实现一致性。领导人从客户端接收日志条目，把日志条目复制到其他服务器上，并且当保证安全性的时候告诉其他的服务器应用日志条目到他们的状态机中。拥有一个领导人大大简化了对复制日志的管理。例如，领导人可以决定新的日志条目需要放在日志中的什么位置而不需要和其他服务器商议，并且数据都从领导人流向其他服务器。一个领导人可以宕机，可以和其他服务器失去连接，这时一个新的领导人会被选举出来。
通过领导人的方式，Raft 将一致性问题分解成了三个相对独立的子问题，这些问题会在接下来的子章节中进行讨论：
- 领导选举：
一个新的领导人需要被选举出来，当先存的领导人宕机的时候。
- 日志复制：
领导人必须从客户端接收日志然后复制到集群中的其他节点，并且强制要求其他节点的日志保持和自己相同。
- 安全性：
在 Raft 中安全性的关键是状态机安全：如果有任何的服务器节点已经应用了一个确定的日志条目到它的状态机中，那么其他服务器节点不能在同一个日志索引位置应用一个不同的指令。

这篇先到这儿，后面再具体分析每个部分，领导人选举、日志复制和安全性，在这之前可以看看[Raft协议的动画演示](http://thesecretlivesofdata.com/raft/)，会对接下来的理解起到帮助作用。

### 参考
[Raft论文英文](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)
[Raft论文中文翻译](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)


-EOF-
