---
title: LevelDB之源码探析小结
date: 2016-12-06 10:05:39
tags: [leveldb,调试,编译，database]
categories: Storage
---
趁着前一篇LevelDB之LSM-Tree的文章中对LevelDB的技术好奇的热情尚未退却，决定一览LevelDB内部的实现，好歹对此类K-V键值存储系统有个代码级别的初步认识，果然还是有所收获的，比如对C++的学习、对数据结构的认识、Skiplist（跳表）、变长整数编码、memtable中的内存管理等等。网上源码分析贴很多，文章质量各有所长。本文并不打算如对代码进行一一分析，决定只做资料的搬运工，完成对所看过资源的一些索引，方便日后回头查看，就酱，另外，对于编译调试环境的搭建网上资源并不多，会多花一些文字来介绍。
<!--more-->
### 编译与调试环境搭建
#### windows8.1+vs2013
尝试过在windows环境下用VS2013进行编译调试，结果报一堆错误，经过各种修改，最终还是显示缺少源文件port/port_win.h，弃。期间参考了下面两篇帖子：
>http://www.cnblogs.com/b-gao/p/leveldb_windows_vs2013_cpp.html
>http://blog.csdn.net/flyfish1986/article/details/46806893

#### ubuntu16.04+eclipse
网上很多直接采用gdb的方式来调试，大神可以直接绕过。如果不满意于gdb的不友好，习惯IDE的，下面的方法也许有帮助。
用VMware安装ubuntu16.04，再在Ubuntu上安装了eclipse，网上教程很多，两行命令可以搞定：
```bash
$ sudo apt-get install eclipse #安装eclipse
$ sudo apt-get install eclipse-cdt #安装插件支持c/c++开发的cdt插件
```
安装完成后，尝试写一个helloWord（参考[试着建一个C语言项目](http://blog.csdn.net/xlgen157387/article/details/41245309)）,如下：
```cpp
    #include <stdio.h>
    #include <stdlib.h>
    int main(){
        printf("hello word");
        return 0;
    }
```
上述测试结束后，就可以开始levelDB的环境搭建了。
1.下载levelDB，可以直接用git的方式。
```bash
$ git clone https://github.com/google/leveldb.git
```
2.File -> New -> Makefile Project with Existing Code
3.选择源码目录，语言选择C语言。如果是 Linux 环境，则选中 Linux GCC，Mac 环境则选择 MacOSX GCC，Windows 下用 Eclipse 则可以选择 MinGW GCC。
4.编写test.cc文件调用levelDB的库，毕竟levelDB提供的就是一个库，将test.cc放到导入的工程里面，暂且放到db文件目录下。demo如下：
```cpp
#include <iostream>
#include "leveldb/db.h"

using namespace std;
using namespace leveldb;

int main() {
    DB *db ;
    Options op;
    op.create_if_missing = true;
    Status s = DB::Open(op,"/tmp/testdb",&db); ///tmp/testdb就是数据库文件

    if(s.ok()){
        cout << "创建成功11111" << endl;
        s = db->Put(WriteOptions(),"abcd","1234");
        if(s.ok()){
            cout << "插入数据成功1111111" << endl;
            string value;
            s = db->Get(ReadOptions(),"abcd",&value);
            if(s.ok()){
                cout << "获取数据成功11111111,value:" << value << endl;
            }
            else{
                cout << "获取数据失败" << endl;
            }
        }
        else{
            cout << "插入数据失败" << endl;
        }
    }
    else{
        cout << "创建数据库失败" << endl;
    }
    delete db;
    cout<<"1212121212"<<endl;
    return 0;
}
```
5.习惯了写java或者c的以为可以直接这样debug运行了，然后发现根本不行。从第二步我们知道该工程是用Makefile构建的，关于Makefile的作用可以参考[什么是Makefile ](http://www.cppblog.com/luqingfei/archive/2010/08/27/124946.html)，所以需要对test.cc进行预编译的过程才行。方法是修改Makfile：
```
# 修改编译参数
OPT ?= -g2

# 添加TESTS=
db/test \

# 添加下列到相应的地方
$(STATIC_OUTDIR)/test:db/test.cc $(STATIC_LIBOBJECTS) $(TESTHARNESS)
    $(CXX) $(LDFLAGS) $(CXXFLAGS) db/test.cc $(STATIC_LIBOBJECTS) $(TESTHARNESS) -o $@ $(LIBS)
```

6. 项目levelDB ->右键build configuration ->build all,这个过程实际就是执行make的命令。
7. 项目levelDB ->右键debug/run as ->local c/c++ application,bin选择test。
8. 不出意外会输出test.cc中的数据。
```
创建成功11111
插入数据成功1111111
获取数据成功11111111,value:1234
1212121212
```

这样就OK了。

### LevelDB工程目录简介
在看源码之前最好对LevelDB的原理有一个基本的认识，参看[LevelDB之LSM-Tree](http://zouzls.github.io/2016/11/23/LevelDB%E4%B9%8BLSM-Tree/).
#### docs
源码文档，建议从docs/index.html开始，里面介绍了LevelDB的一些重要组件，比如Snapshot、Slice、Comparators等，以及为了实现高性能所采取的一些方案实现，如Compression、Cache、Filters等等。
尤其是table_format.txt比较详细的解释了sstable的组织格式，在看说说他病了之前建议先看这个官方文档。网上的文章基本都是对该文件的解读。其他文档如impl.html、log_format.txt没有做仔细研读，就不说了。

#### db
memtable.h/cc :实现内存中的mentable，注意观察内部成员变量
skiplist.h    :负责组织mentable中的kv键值对数据，跳表接口

#### table
block_builder.h/cc : 构造block，准备写入
block.h/cc         :通过ReadBlock读取

table_builder.cc : 构造Table，Add()、Flush()、WriteBlock
table.cc         : 定义了操作table的方法，InternalGet

#### util
arena.h/cc       : 内存分配器，比较有意思，可以看看
coding.h/cc      : 变长整数编码的实现，有意思
status.cc        : 错误管理，返回状态之类的

#### include
slice.h       : leveldb自己的封装字符串的类

其他目录文件可以参考[源文件](http://mingxinglai.com/cn/2015/08/gdb-leveldb/).

### LevelDB组件源码
#### 基础类
1. Slice
其实就是对字符串的封装，Slice是贯穿整个源码的类，只有两个私有的成员变量，如下：
```cpp
    class Slice {
        ...
     private:
      const char* data_;
      size_t size_;
    };
```

2. Key
3. VarInt
作者为了将整数存储的空间进一步压缩，想法子采用变长整数编码的技术，因为如果不这样，一个int整数无论大小如何都会占用4B的空间，采用变长编码之后会根据实际数字的大小采用不同的字节数。
参考：[讨论coding.h/coding.cc](http://brg-liuwei.github.io/tech/2014/10/20/leveldb-4.html)
4. Arena
在理解下面的mentable的实现时，必须要先知道memtable是如何管理内存，因为数据首先会以一定的格式存到mentable中，那么每次add（k,v）的过程中，是必然要分配内存空间的,那么Arena其实就相当于memtable的内存大管家，来管理内存的分配和释放。Arena的几个私有变量如下：
```cpp
    // 当前指针指向的位置
  char* alloc_ptr_;
    //分配的blocks里面还有多少空间没用完
  size_t alloc_bytes_remaining_;
    // 用来存放blocks的指针
  std::vector<char*> blocks_;
  // 用了多少内存空间
  port::AtomicPointer memory_usage_;
```
详情参考：[Arena内存管理](http://mingxinglai.com/cn/2013/01/leveldb-arena/)
5. Skiplist
关于跳表，网上的资源很多,跳跃表（skiplist）是一种随机化的数据，由 William Pugh 在论文《Skip lists: a probabilistic alternative to balanced trees》中提出，效率和平衡树媲美——查找、删除、添加等操作都可以在对数期望时间（O(lgn)）下完成，并且比起平衡树来说，跳跃表的实现要简单直观得多。
![](http://www.spongeliu.com/wp-content/uploads/2010/09/3.png)

#### memtable
先看看memtable.h中的几个比较重要的私有成员变量
```cpp
  typedef SkipList<const char*, KeyComparator> Table;
  //key比较器
  KeyComparator comparator_;
    //引用数目
  int refs_;
    //内存大管家
  Arena arena_;
    //skiplist数据结构
  Table table_;
```
再来看看Add方法的源码是如何实现的
```cpp
//传参中，key、value均是Slice类型
void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  // 一个kv条目的组成格式
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;//注意internal_key和key（user key）的区别
  const size_t encoded_len =//一个条目的总长度计算，待会要用
      VarintLength(internal_key_size) + internal_key_size +
      VarintLength(val_size) + val_size;
  char* buf = arena_.Allocate(encoded_len);//分配encoded_len的内存
  char* p = EncodeVarint32(buf, internal_key_size);//1、放key的size
  memcpy(p, key.data(), key_size);                 //2、放key的内容
  p += key_size;                                   //3、p指针后移                 
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  p = EncodeVarint32(p, val_size);                 //4、放value的size
  memcpy(p, value.data(), val_size);               //5、放value的内容
  assert((p + val_size) - buf == encoded_len);
  table_.Insert(buf);//插入到skiplist中
}
```

其他详情参考：[MemTable](http://mingxinglai.com/cn/2013/01/leveldb-memtable/)

#### sstable
1. 逻辑结构
如果你想看原汁原味的作者写的文档，可以打开/doc/table_format.txt,作者在该文档中大致描述了sstable的轮廓，不过看完总是还有些许地方比较疑惑，比如block data中的restartpoint（重启点）概念，从整体来看，下面这篇参考博文对sstable的解释更清楚易懂：

>也许你会以为在划分好block的数据存储区域以后那么就是一个一个的KV对（如图中的Record）了，但是其实不是，leveldb为了降低数据的存储量和快速的查找引入了一个重启点（restartpoint）的概念。出自参考1：[SSTable之逻辑结构](http://www.cnblogs.com/KevinT/p/3815764.html)

参考2：[SSTable的文件组织](http://blog.csdn.net/sparkliang/article/details/8635821)
2. 



