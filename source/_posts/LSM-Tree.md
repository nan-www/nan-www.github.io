---
title: LSM Tree
date: 2023-12-22 17:31:11
tags:
---
# LSM-Tree
## 名词解释
### LSM-Tree
全称为 Log-Structured Merge-Tree。中文译名即为日志结构合并树。

Abase 的存储采用的[RocksDB](https://rocksdb.org/)，而RocksDB的存储结构则是采用本文要介绍的LSM-Tree。
### Append write & In-place write
中文即为追加写和原地写。两者的差异在于是否覆盖原先数据。

以下以一个K-V DB的角度看待这两者的差异。

追加写：无论是更新、删除还是新增，追加写都只会把操作写到文件的末尾。

原地写：对于更新、删除操作来说，原地写都必须要先找到原来数据所在的位置然后再对其进行更新。MySQL使用的B+Tree结构的更新、删除操作都是原地写。

站在IO的视角来看，Cost（append) <<  Cost (in-place)。原因是原地写会产生大量的随机IO。 而追加写避免随机IO的方法则是朴素的以空间换时间。

![append_only.png](LSM-Tree%2Fappend_only.png)
> 在LSM-Tree中，删除采用的是一种名为“墓碑机制”的策略。
> 
> 在K-V系统中，简单地表达对某个键值的删除可以如图上所示将value置空。
 
## 自顶向下了解LSM-Tree（以一个K-V系统为例）
以下将LSM-Tree 简写为lsm

### 内存视角
![img.png](LSM-Tree%2Fimg.png)
> PS：内存这么快，不用起来岂不是太可惜了。

这块名为active memtable的内存块是直接接受用户写请求的“存储区”。它的容量是固定的。由于内存读写快速的特性，发生在这块内存区的写请求采用的是**原地写**。
### 引发的问题
1. Active memtable满了怎么办？
2. 进程宕机了数据全没了，持久化怎么搞？
3. 上面扯了原地写和追加写，到这里都还没体现出来？

#### Active memtable满了怎么办？
满了就触发Flush将这块内存中的所有数据都写到硬盘上。
   ![flush.png](LSM-Tree%2Fflush.png)
#### 刷的前一刻宕机也还是丢数据？
学过MySQL的你肯定能想到解决办法：WAL（Write Ahead Log）
![wal.png](LSM-Tree%2Fwal.png)
#### 照这么看，Flush的时候写请求不全阻塞了？
怎么处理呢？加读写锁？NO NO NO。能用设计避免并发就不要正面硬刚。
![img_1.png](LSM-Tree%2Fimg_1.png)
当Active memtable满了的时候，LSM会在内存中**另外**创建一块Block作为新的Active memtable，并将满了的Active Memtable置为**只可读**的Block。
>PS：当Read Only的Block刷到硬盘之后该Block对应的WAL文件就可以删掉了。

### 追加写究竟在哪体现？
我需要再把上图画得完善些。
![lsm_append_write.png](LSM-Tree%2Flsm_append_write.png)
Flush的时候LSM会按照**Key的值**升序排序所有的键值之后写入到一个硬盘文件中。

这个硬盘文件在LSM中称为**SST**（Sorted String table)。这个过程就是一个简单的**追加写**操作了。
然后这个新的SST会先进入到Level-1中。在合适的情况下它必须与同在Level-1中的存在的SST进行合并操作。

正如你在图中看到的一样，每个SST可以附带一些冗余的标识来标识其自身。
由于新生成的SST中的Key的范围是**1-10**，已存在的一个SST的Key的范围是**1-5**，所以它们有可能记录了同一个Key。所以需要执行Merge操作，并在合并过程中保存对某一个Key的最新记录。
![merge.png](LSM-Tree%2Fmerge.png)

#### 有Level-1就有Level-2吧？
不错，LSM会把硬盘抽象成几个层次。每个层次的空间容量都是固定大小的，越往下一层的空间也就越大。
![level.png](LSM-Tree%2Flevel.png)
### 读请求如何处理？
处理读请求最关键的就是要保证读到最新的数据。
所以读请求的扫描是自上而下的。只要读到了对应的Key就立马返回。
![deal_read.png](LSM-Tree%2Fdeal_read.png)
#### 一些Trick
在RocksDB中，Active memtable被设计成一个跳表（skip list）来支撑范围查询和保证查询速度。
上述的机制保证了每一层的Key都是**唯一**的。所以可以在每一层设置一个**布隆过滤器**（Bloom filter）来提高查询效率。（快速判断Key是否存在该层以减少额外的IO）
可以在每一层中额外存储每个层的SST的Key的上下限来辅助定位到对应的SST（二分查找）。
## B+Tree & LSM-Tree？
从上文粗略的介绍不难看出，LSM正如其名字一样比B+Tree更加适合写多的场景。比如用其作为日志的存储结构。

## 基于RocksDB的Abase与Redis的异同？
Redis是纯内存的K-V数据库。在读取、写入速度上来看Redis是要胜于Abase的。毕竟对于LSM-Tree的读请求可能会下沉到磁盘中。而能把数据存储到磁盘中的Abase则能支撑更大的数据量。

