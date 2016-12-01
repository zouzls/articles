---
title: LevelDB之LSM-Tree
date: 2016-11-23 15:56:27
tags: [leveldb,lsm-tree,database]
categories: Storage
---
一直觉得数据库内部的实现是个黑匣子，总感觉很神秘，借着前段时间「海量存储」这门课程课堂报告的机会，读了一些Paper，虽然没能全部仔细读完，但是从每篇Paper的Abstract里面还是能窥见当前领域的近来的一些学术动态，很多东西没能记住，但是，印象唯独深刻的是，几乎大部分paper都在讨论：如何使得存储系统的I/O性能更快更有效率，如何使得同样的物理存储能够存储更多的数据内容？
<!--more-->
### 前话
在这一点上，其实是让我开始觉得计算机领域无非就是「存储+计算」，从冯-诺依曼时代就是如此。最后我选择了[Spanner](https://www.usenix.org/node/170855)这篇论文作为了自己的课程报告，原因是它的Abstract太吸引我了。
>Spanner is Google’s scalable, multi-version, globally distributed, and synchronously-replicated database. **It is the first system to distribute data at global scale and support externally-consistent distributed transactions. **

虽然当时是对里面很对的术语multi-version、synchronously-replicated、externally-consistent、 distributed transactions不太清楚，出于对「分布式数据库」中分布式和数据库两个领域的好奇心，还是读下去了，当然现在很多专业术语现在还是并不能完全透彻明白，但是已经open in my mind，对于数据库里面核心的概念「事务」、「并发控制」的理解又加深了一层。这里并不打算讨论spanner相关内容，感兴趣的同学可以自己了解。

随着对Mysql数据库了解加深，知道Mysql的存储引擎主要是Innodb和MyISAM，前者一般是默认存储引擎，主要是B+Tree的数据结构来组织数据。
知道这一点的时候是我第一次能将之前学习的「数据结构」和具体应用领域结合起来，当时感觉是兴奋的，不再觉得学习这些有什么用。
我发现很多知识，当第一次出现在你的眼前的时候，往往不会引起你的注意，也根本引起不了你的好奇心，但是接二连三出现在视线时，探索的欲望就来了，比如LSM Tree。本文以LevelDB为例介绍LSM在存储引擎中的实现。

### LSM的背景
LSM Tree的全名叫做Log Structured-Merge Tree。
LSM是当前被用在许多产品的文件结构策略：HBase, Cassandra, LevelDB, SQLite,甚至在mangodb3.0中也带了一个可选的LSM引擎（Wired Tiger 实现的）。
简单的说，LSM被设计来提供比传统的B+树或者ISAM更好的写操作吞吐量，通过消去随机的本地更新操作来达到这个目标。

那么为什么这是一个好的方法呢？这个问题的本质还是磁盘随机操作慢，顺序读写快的老问题。这二种操作存在巨大的差距，无论是磁盘还是SSD。
想象一下在「操作系统」里面关于磁盘寻址就能明白，随机读写的效率是非常低的，因为要涉及到频繁的磁道寻址。

### LevelDB中LSM引擎实现
#### LSM的基本思想
lsm tree 是针对写入速度瓶颈问题而提出的. mysql 这种数据库的存储引擎使用了 B+ 树来持久化数据, B+ 树是一个索引树, 可以说是同时考虑了读写均衡, 其结构上对树高进行了优化, 搜索耗时相比 AVL 树降下来. 然而问题依然是前面我们谈到的 "对一个随机读优化的排序结构执行随机写是有很大开销的", 所以对那些需要高频写操作的系统来讲, B+ 树作为存储结构可能并不合适.
这时 lsm tree 可能是个更好的选择, 它是一种类似日志的数据结构, 将随机写变为顺序写, 核心思想是:
- 对变更进行批量 & 延时处理
- 通过归并排序将更新迁移到硬盘上

自从上世纪某年代一篇关于日志结构文件系统的论文发表之后, 有人基于这个想法提出了解决写瓶颈的问题: 使用顺序写(追加)替代随机写. 为什么要变为顺序写呢? 因为顺序写不需要多次寻址, 速度能达到硬盘理论传输速度, 而随机写则受限于硬盘寻址速度.
leveldb 在持久化上借鉴了 lsm tree 的设计. 具体实现就是 memtable + sstable.

#### 结构规划
![](https://cloud.githubusercontent.com/assets/3054411/19015433/dbc1b240-8836-11e6-88e4-67d6b23a3d24.png)
根据此图先初步说一下它的读写流程:
- write: 写入 log 文件一条操作(数据更新)记录, 再往 memtable 里写入 k-v 对.
- read: 读 memtable (没找到)→ 读 immutable memtable (没找到)→ 读 sstable (没找到)→ 空

#### memtable

memtable 是个内存数据结构, 也是读写操作的入口.作为一个存储引擎, 读无非就是读若干个 key, 然后返回对应的 value; 而写就是存下这些 key-value 对.
所以 memtable 中存的每条数据也都是一个键值对.
关键点是 memtable 中的数据是按 key 的字母表顺序排序的.
然而对一个具备随机读能力的排序结构执行插入操作往往是有开销的, 通常写瓶颈也都是集中在这里. 但是也有很多数据结构提供了加速随机写, 比如链表, AVL 树, B 树, skiplist. memtable 的核心结构就是用了跳表.

结构简单, 概率上近似 O(log N) 读写复杂度的跳表在很多存储系统里得到应用, 比如 elasticsearch 的高速搜索就是基于跳表,位图以及倒排索引实现的. memtable 采用这种结构初步实现快速读写.

接下来, 既然是存储引擎, 那么肯定能持久化数据, 而 leveldb 核心技术就是围绕持久化过程而构建的.

#### sstable
sstable 全名 sort-string table, bigtable 使用的存储技术. 顾名思义, sstable 中的数据都是有序的.
除了日志之外, leveldb 的数据统统存储在 sstable 中. sstable主要以block来区分，每个sstable主要包括data block（若干）、index block、footer ，逻辑视图如下所示。
![](https://cloud.githubusercontent.com/assets/3054411/19015445/fd244ab0-8836-11e6-961b-6404474941d2.png)
下面分别介绍不同种类的block作用：
- data block
存储的是所有的data（k-v）。更具体来说, 是 data 域的 record 段存储的. record 里的 k-v 对全部按 key 有序排列。
- index block
index block 中的每条记录会对某个 data block 建立索引.
- footer
footer 是整个 sstable 的末尾 block, 记录了 indexblock 的起始偏移量和大小, 很容易看出其存在意义就是为了方便读取 index block.

#### sstable与lsm tree之间的关系
leveldb 将 sstable 划分为了不同层次(level). level i+1 层的 sstables 由 level i 层的 sstables merge 得来. 而最上层称为 level 0.

那么第 0 层的 sstables 是怎么来的呢?

memtable + sstable = lsm tree

- memtable → immutable memtable (内存中)
- immutable memtable → (level 0) sstable (内存 → 外存)

还记得最开始提到的 memtable? 那个读写入口数据结构. leveldb 给了他一个阈值, 但凡 memtable 里的数据大小达到了阈值, 后台任务就将 memtable 标记为只读, 也就是变成了 immutable memtable.
然后创建一个新的 memtable 用于接下来的读写.

immutable memtable 经过一段时间会被迁移到硬盘上, 成为 sstable, 这一过程称之为 compaction.
因为 memtable 的结构是有序的, 因此 sstable 也是有序存储的.

#### Compaction
compaction 是执行 lsm-tree 中 merge 的过程.
对于不同应用情况, 分为 minor compaction 和 major compaction.

- minor compaction

minor compaction 用于内存到外存的迁移过程. 就是简单的遍历跳表, 依次写入新的 sstable record, 最后建立 index block 并完善一些其他的重要元信息. 这也就是从 immutable memtable 到 level 0 sstable 的迁移.

- major compaction

major compaction 用于 level 之间的迁移.
当某个 level 的 sstables 数量超过一个给定的阈值, 就会触发 major compaction.
这里有两点差异:

对 level > 0 的 sstables, 选择其中一个 sstable 与 下一层 sstables 做合并.
对 level = 0 的 sstables, 在选择一个 sstable 后, 还需要找出所有与这个 sstable 有 key 范围重叠的 sstables, 最后统统与level 1 的 sstables 做合并.
不知之前是否注意到, level 0 的 sstables 可能有键范围的重合. (因为 level 0 的 sstable 是直接由 memtable 变过来的, 而不同的 memtable 之间并没有约束 key 必须独立.)

#### merge 策略

compaction 过程中有提到选择 sstable, 如何选择? 这就看 leveldb 的 merge 策略了:

- level i 按顺序选择.
- level i+1 选择所有与 level i 所选 sstable(s) 有 key 范围重叠的 sstables.
- 将这些 sstables 做 K 路归并排序. 对于相同的 key, 只保留最新的(上层 sstable, 同层中新的 sstable).
- 清除参与此次 merge 的所有 sstables, 保留新的 sstable.
有了前面提到的 compaction 第二个约束和 merge 策略, 间接解释了 level 0 和其他 level 不同的原因: 在level 0 的 compaction后, level 1 产生的 sstable是没有 key范围重叠的, 因此向高层 level 的 compaction 也不会有同一个 level 下 key 范围重叠的 sstable 产生.

到这里, leveldb 如何实现的 lsm-tree 就比较明朗了: 内存结构中写数据, 每次写只考虑当前 memtable, 不涉及其他结构的更新, 内存数据顺序迁移到外存中, 形成一个日志结构, 这就是 leveldb 顺序写的思路.
### LevelDB的写入和读取效率
#### 高效的写入效率
到此为止, leveldb 的写入策略就介绍完了, 这里做个总结:

- 内存 + 跳表
- 外存中追加, 使用顺序写

#### 读取效率如何?

说完了写, 那么并不具备读优势的 leveldb 读取是怎样设计的呢?

毕竟凡事很难达到两全, 顺序写的这种结构就注定了丧失读效率(需要多次遍历寻址).
然而 leveldb 仍然费尽心思在代码层面而非算法层面做了优化.

查询过程中优化的顺序如下:

- 内存 + skiplist
- sstables B-search (通过 manifest 文件)
- 页缓存
- bloom filter(周期性 compaction 做辅助)

在查询 level 0 sstables 时, 因为不同 sstables 的 key 范围可能有重叠, 所以只能把但凡包含 key 的 sstables 找出来, 排序取最新的 (manifest 文件记录了当前所有sstable 的信息, 包括文件名, 所处 level, key 的范围等等). 因为 sstable 是有序的, 所以在 level > 0 寻找 sstable 这一过程可以应用二分搜索完成. 找到 sstable 后, 读出其 index block, 加载到内存作为 table cache, 再根据索引去查具体的 data block. 而 data block 也包含了一组 k-v, 为了提高定位 key 的效率, leveldb 又引入了布隆过滤器这一数据结构.

### 引用
[存储引擎技术架构与内幕](https://github.com/abbshr/abbshr.github.io/issues/58)
[Log Structured Merge Trees(LSM) 原理](http://www.open-open.com/lib/view/open1424916275249.html)

### 最后
本文对结合LevelDB存储引擎对于LSM Tree原理进行了总结，大部分内容出自于引用，有时间希望能对存储引擎有多更多的了解。