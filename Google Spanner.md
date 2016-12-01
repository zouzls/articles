---
title: Google Spanner
date: 2016-10-18 00:29:24
tags: [分布式系统,database]
categories: Database
---
这篇文章是由之前海量存储的论文课程报告整理而成，Google Spanner这篇2012年发表在OSDI上面的论文确实艰涩难懂，中间查阅了不少资料才稍微理解里面的关键词或者是专业术语，本文一部分内容是论文的中文翻译，作了删减，想看原文可以绕过。之所以放在blog上，为了方便日后检索该部分相关内容，同时也希望能达到一起交流的目的。
<!--more-->
### Introduction
#### What is Spanner
Spanner 是谷歌公司研发的、可扩展的、多版本、全球分布式、同步复制数据库。 它是第一个把数据分布在全球范围内的系统，并且支持外部一致性的分布式事务。
论文描述 了 Spanner 的架构、特性、不同设计决策的背后机理和一个新的时间 API，这个 API 可以暴 露时钟的不确定性。这个 API 及其实现，对于支持外部一致性和许多强大特性而言，是非常 重要的，这些强大特性包括：非阻塞的读、不采用锁机制的只读事务、原子模式变更。

#### Why is spanner
- Why not BigTable
客户反映 BigTable 无法应用到一些特定类型的应用上面，比如具备复杂可变的模式，或者对于在大范围内分布的多个副本数据具有较高的一致性要求。
- Why not Megastore
谷歌的许多应用已经选择使用Megastore，主要是因为它的半关系数据模型和对同步复制的支持，尽管Megastore 具备较差的写操作吞吐量。

Spanner 已经从一个类似 BigTable 的单一版本的键值存储，演化成为一个具有时间属性的多版本的数据库。数据被存储到模式化的、半关系的表中，数据被版本化，每个版本都会自动以提交时间作为时间戳，旧版本的数据会更容易被垃圾回收。应用可以读取旧版本的数据。Spanner支持通用的事务，提供了基于 SQL 的查询语言。

#### The interesting features of Spanner
- 第一，在数据的副本配置方面，应用可以在一个很细的粒度上进行动态控制。
应用可以详细规定，哪些数据中心包含哪些数据，数据距离用户有多远（控制用户读取数据的延迟），不同数据副本之间距离有多远（控制写操作的延迟），以及需要维护多少个副本（控制可用性和读操作性能）。数据也可以被动态和透明地在数据中心之间进行移动，从而平衡不同数据中心内资源的使用。（这个特性的实现应该是跟后面的目录设计相关的。）
下面两个属性很难在一个分布式数据库上实现：
- 第二，Spanner提供了读和写操 作的外部一致性。
- 第三，在一个时间戳下面的跨越数据库的全球一致性的读操作。

### The implemention
#### Spanner server organization
一个 Spanner 部署称为一个universe。假设Spanner 在全球范围内管理数据，那么，将会只有可数的、运行中的universe。我们当前正在运行一个测试用的 universe，一个部署/ 线上用的 universe 和一个只用于线上应用的 universe。
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%871.png)
如图上所示，一个zone是物理隔离的单元，在一个数据中心中，可能有一个或者多个 zone，例如，属于不同应用的数据可能必须被分区存储到同一个数据中心的不同服务器集合中。一个zone包括一个 zonemaster，和一百至几千个spanserver。Zonemaster把数据分配给spanserver，spanserver 把数据提供给客户端。客户端使用每个zone上面的 location proxy来定位可以为自己提供数据的 spanserver。Universe master 和 placement driver，当前都只有一个。

#### Spanserver software stack
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E6%88%AA%E5%9B%BE20161201102133.png)
如图上所示，Paxos状态机是用来实现一系列被一致性复制的映射。对于每个是领导者的副本而言，每个 spanserver 会实现一个锁表来实现并发控制。对于每个扮演领导者角色的副本，每个 spanserver也会实施一个事务管理器来支持分布式事务。这个事务管理器被用来实现一个 participant leader，该组内的其他副本则是作为 participant slaves。如果一个事务只包含一个Paxos组（对于许多事务而言都是如此），它就可以绕过事务管理器，因为锁表和 Paxos二者一起可以保证事务性。如果一个事务包含了多 于一个Paxos组，那些组的领导者之间会彼此协调合作完成两阶段提交。其中一个参与者组， 会被选为协调者，该组的 participant leader 被称为 coordinator leader，该组的 participant slaves 被称为 coordinator slaves。每个事务管理器的状态，会被保存到底层的 Paxos 组。

#### DataModel
- Schematized semi-relational tables
由此可以看出 Spanner 的数据模型的一个显著特点是弱化了列的概念，即 BigTable 是通过一对（行坐标、 列坐标）结合一个时间戳（可选）来索引一个数据 cell；而 Spanner 采用行坐标结合时间戳的方式索引一个数据行（与关系型数据的方式类似）。
Spanner 会把下面的数据特性集合暴露给应用：基于模式化的半关系表的数据模型，查 询语言和通用事务。支持这些特性的动机，是受到许多因素驱动的。需要支持模式化的半关系表是由 Megastore的普及来支持的。
Spanner 采用的数据模型不是纯粹关系型的，因为在Spanner中每行必须包含一个名称。因而，Spanner 更像是一个映射主键列与非主键列的 key-value 存储。
- query language like sql
从下图可以看出，查询语言风格跟sql还是很类似的：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%872.png)

#### Directories and Placement 
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%873.png)
每个 Spanner 数据库必须被客户端分割成一个或多个表的层次结构。客户端应用会使用 INTERLEAVE IN 语句在数据库模式中声明这个层次结构。目录表中 的每行都具有键 K，和子孙表中的所有以 K 开始（以字典顺序排序）的行一起，构成了一个目录。
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%875.png)
如图上所示，一个目录是数据放置的基本单元。属于一个目录的所有数据，都具有相同的副本配置。当数据在不同的 Paxos 组之间进行移动时，会一个目录一个目录地转移。Spanner 可能会移动一个目录从而减轻一个 Paxos 组的负担。也可能会把那些被频繁地一起访问的目录都放置到同一个组中。或者会把一个目录转移到距离访问者更近的地方。

### TrueTime
#### question
**What is MVCC？**
Mvcc主要是目前数据库中常用的一种处理并发的机制，比如mysql的innodb引擎、postgre、oracle，主要是对一个数据维持多个版本，实现了无锁方式的读写、快照读。这样读不会阻塞写、写不会阻塞读，而且达到了可重读和读已提交的隔离级别。因为系统会给每个事务分配时间戳作为事务ID，每个数据会维持几个版本，版本号就是时间戳，比如创建时间和删除时间，用来表示是哪个事务创建和删除的。每次修改数据创建一个新版本数据，并不在原来的基础上修改，所以读事务并不影响写，但是如果遇到写的时候正好进来一个读的事务，也可以读到上一个版本的快照，不至于读到中间状态，这样写不影响读，同时就保证了外部一致性，也实现了无锁读，MVCC的这种思想解决了读写、写读、读读效率的巨大提升。
**如果是单点写？**
如果是单机数据库（单点写），事务提交的时候获取时间戳作为事务ID，谁先提交谁的时间戳就小。Oracle就是这么干的。
如果是一个分布式数据库呢，由于1.0版本之前的OceanBase实际上是单点写，也就是说事务ID是在同一个节点上获得，和第1点类似。
**如果是多点写？**
既然多点写，那么机器和机器之间必然存在时钟误差。事务时间戳从不同的机器上获得，那么必然会存在在wall time（就是现实时间、实际时间，这么理解）上先提交的事务最后得到的事务ID(时间戳)反而更小这种情况。
**If Lamport clock？If Vector clock？**
不够高效，都存在通讯开销。

#### TrueTimeAPI
TrueTime API 是考虑时钟不确定性的时间 API。在 TrueTime API 中， 时间被表示成一段时间区间 TTinterval。提供下面的三个API接口：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%876.png)
#### TrueTime Architecture
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%877.png)
如图上所示，TrueTime 是由每个数据中心上许多的 time master 机器和每台机器上的 timeslave daemon 共同实现。所有 master 的时间参考值会进行彼 此校对；每个 master 也会交叉检查时间参考值和本地时间，如果两者差别太大则 master 会 将自己驱逐。
每个 daemon（client）会从不同的 master 中收集时间参考值， master 的选取包括 GPS 和原子钟，同时涵盖本地的数据中心和远处的数据中心。在同步时， daemon 会考虑一个ε作为时间的不确定性。
### Concurrency Control
本部分内容描述 TrueTime 如何可以用来保证并发控制的正确性，以及这些属性如何用 来实现一些关键特性，比如外部一致性的事务、无锁机制的只读事务、针对历史数据的非阻 塞读。这些特性可以保证，在时间戳为 t 的时刻的数据库读操作，一定只能看到在 t 时刻之 前已经提交的事务。

#### Assigning Timestamps to RW Transactions
外部一致性
即两个事务，一个事务结束后另一个事务才开始。
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%878.png)
如图上所示，利用TrueTime API，Spanner可以保证给事务标记的时间戳介于事务开始的真实时间和事务结束的真实时间之间。假如事务开始时TrueTime API返回的时间是{t1, ε}，此时真实时间在t1-ε到t1+ε之间；事务结束时TrueTime API返回的时间是{t2, ε}，此时真实时间在t2-ε到t2+ε之间。Spanner会在t1+ε和t2-ε之间选择一个时间点作为事务的时间戳，但这需要保证t1+ε小于t2-ε，为了保证这点，Spanner会在事务执行过程中等待直到t2-ε大于t1+ε的时候才提交事务。可以推导出，Spanner中一个事务至少需要2ε的时间(平均8毫秒)才能完成。
#### RW Transactions
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%879.png)
2PL：两段锁协议。就是事务的时候加锁，事务结束释放锁。当所有的锁都已经获得以后，在任何锁被释放之前，就可以给事务分配时间戳。
- A single leader replica
在为写操作分配时间戳时，当事务只涉及单个 leader 副本，则可以使用单调增加的方式分配时间戳。
- Multiple leader replica
在2PC（两阶段提交）过程中， 非协调者 leader 会将选取的准备时间戳通知协调者 leader，协调者leader会根据所有收到准备时间戳为整个事务选取一个时间戳，并通知所有非协调者 leader，所有参与的 leader会在同一时间戳进行提交并释放锁。

#### Snapshot Read
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%8711.png)
一个快照读操作，是针对历史数据的读取，执行过程中，不需要锁机制。一个客户端可 以为快照读确定一个时间戳，或者提供一个时间范围让 Spanner 来自动选择时间戳。不管是 哪种情况，快照读操作都可以在任何具有足够新的副本上执行。
#### Read Transaction
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/%E5%9B%BE%E7%89%8710.png)
一个只读事务分成两个阶段执行：分配一个时间戳 read，然后当成 sread时刻的快照读 来执行事务读操作。
一个只读事务具备快照隔离的性能优势。在一个只读事务中的读操作，在 执行时会采用一个系统选择的时间戳，不包含锁机制，因此，后面到达的写操作不会被阻塞。 
在一个只读事务中的读操作，可以到任何足够新的副本上去执行。

因为只读操作时候，读的是历史版本，为什么不会阻塞写是因为每次写都会创建一个新的版本。

### Reference
[Spanner](https://www.usenix.org/node/170855)
[Google Spanner (中文版)_厦门大学数据库实验室](http://dblab.xmu.edu.cn/post/google-spanner/)
[事务与分布式事务原理与实现-沈询](http://v.youku.com/v_show/id_XMTMyMzE5MTg2MA==.html?f=23272406&from=y1.2-3.2)
[阿里技术沙龙](http://club.alibabatech.org/article_detail.htm?articleId=22)
[百万节点数据库扩展之道(4): Google Spanner](http://www.jianshu.com/p/7ac3f09203b7)
[分布式数据库中为什么要使用 Vector Clock？](https://www.zhihu.com/question/19994133)
[分布式事务实现-Spanner - Eamon13](http://www.cnblogs.com/zhangeamon/p/5586862.html)
[我自然 | Google Spanner原理- 全球级的分布式数据库](http://www.yankay.com/google-spanner%E5%8E%9F%E7%90%86-%E5%85%A8%E7%90%83%E7%BA%A7%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E6%95%B0%E6%8D%AE%E5%BA%93/)

++++++++++++++++++++++++

附件：[Storage-System-Presentation课程报告PPT](http://oaewlsdmg.bkt.clouddn.com/Storage%20System%20Presentation.ppt)


-EOF-

