---
title: "Goleveldb源码分析（六）sstable"
date: 2022-04-11T21:52:56+08:00
categories: [goleveldb]
tags: [go, 读点源码]
draft: true
---

Leveldb是典型的LSM树（Log Structured-Merge Tree）实现，即一次leveldb的写入过程并不是直接将数据持久化到磁盘文件中，而是将写操作首先写入日志文件中，其次将写此操作应用再memtable上。

当leveldb达到checkpoint（memtable中的数据量超过了阈值），会将当前memtable冻结成一个不可更改的内存数据库（immutable memory db），并创建一个新的memtable供系统继续使用。

immutable memory db会在后台进行一次minor compaction（这里不展开讲，下一篇会详述），即将内存数据库中的数据持久化到磁盘文件中。

Leveldb或者说LSM属设计Minor Compaction的目的是：

1.   有效地降低内存使用率；
2.   避免日志文件过大，系统恢复时间过长。

当memory db的数据被持久化到文件中时，leveldb将以一定的规则进行文件组织，这种文件格式即为sstable。本文将详细的介绍sstable的文件格式以及相关的读写操作。

## SStable文件格式

### 物理结构

为了提高整体的读写效率，一个sstable文件按照固定大小进行块划分，默认每个块的大小为4KiB。每个Block中，除了存储数据以外，还会存储两个额外的辅助字段：

1.  压缩类型
2.  CRC校验码

压缩类型说明了Block中存储的数据是否进行了数据压缩，若是，采用了哪种算法进行压缩。leveldb中默认采用[Snappy算法](https://github.com/google/snappy)进行压缩。

CRC校验码是循环冗余校验校验码，校验范围包括数据以及压缩类型。

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/sstable_physic.jpeg)

### 逻辑结构

在逻辑上，根据功能不同，leveldb在逻辑上又将sstable分为：

1.  **data block**: 用来存储key value数据对；
2.  **filter block**: 用来存储一些过滤器相关的数据（布隆过滤器），但是若用户不指定leveldb使用过滤器，leveldb在该block中不会存储任何内容；
3.  **meta Index block**: 用来存储filter block的索引信息（索引信息指在该sstable文件中的偏移量以及数据长度）；
4.  **index block**：index block中用来存储每个data block的索引信息；
5.  **footer**: 用来存储meta index block及index block的索引信息；

![img](https://cdn.jsdelivr.net/gh/wuliuqii/pic@master/img/sstable_logic.jpeg)

注意，1-4类型的区块，其物理结构都是如上一节所示，每个区块都会有自己的压缩信息以及CRC校验码信息。
