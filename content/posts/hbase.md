+++ 
date = 2021-02-17T11:14:59+08:00
title = "Apache HBase"
description = "Apache HBase"
tags = ["infra"]
+++

# 背景

[Migrating Messenger storage to optimize performance - Facebook Engineering](https://engineering.fb.com/2018/06/26/core-data/migrating-messenger-storage-to-optimize-performance/)

HBASE是什么

> Apache HBase™ is the Hadoop database, a distributed, scalable, big data store. 

> Use Apache HBase™ when you need random, realtime read/write access to your Big Data. This project's goal is the hosting of very large tables -- billions of rows X millions of columns -- atop clusters of commodity hardware. Apache HBase is an open-source, distributed, versioned, non-relational database modeled after Google's Bigtable: A Distributed Storage System for Structured Data by Chang et al. Just as Bigtable leverages the distributed data storage provided by the Google File System, Apache HBase provides Bigtable-like capabilities on top of Hadoop and HDFS.

其它有意思的信息

- 代码行数规模
    - commit 25e3633; Top 5 language

        [github.com/AlDanial/cloc](http://github.com/AlDanial/cloc) v 1.88 T=34.49 s (143.8 files/s, 38891.5 lines/s)

        **Language files blank comment code**

        * Java                                   4350         128075         226689         786391
        * AsciiDoc                                 62           7136            1747          25015
        * Python                                   18           3984           3380          23084
        * Ruby                                    219           3145           5311          15698
        * C++                                       5           2637            120          15089

- 起始年份
    - 2008.03.28 发布开源版本
    - 距今 2021.02.10  13年 生命力旺盛
- 起源
    - Google BigTable 一个开源实现

# 解决的问题

解决海量数据结构化存储、查询

# 提供的核心能力

- 空间上可水平扩展
- 时间上在约束范围内

# 系统架构

- 系统组件
    - 列簇 Column Family
        - 本质上就是一棵LSM tree (log-structured merge-tree)

- 组件关系

- 数据结构
    - SkipList
        - 查找、删除、插入的复杂度都是O(logN)
    - LSM tree
        - 特点
            - 对写入请求更友好（HDFS擅长顺序写，不支持随机写）
        - 按空间分为两部分
            - 内存
                - 有序数据集合
                    - 可选数据结构：平衡二叉树、红黑树、跳跃表（SkipList）
                    - HBASE的选择 跳跃表 Skiplist
                        - 理由：更优秀？相比于平衡二叉树、红黑树 TBD
            - 磁盘
                - 多个独立KeyValue有序文件
                - 布隆过滤器 Bloom Filter
                    - 理由：加速读，减少没必要的读
                - cache
                    - 理由：加速读，减少磁盘IO
        - 索引
            - 内存部分
            - 磁盘部分
        - LSM trees are used in data stores such as Bigtable, HBase, LevelDB, SQLite4, Tarantool RocksDB, WiredTiger Apache Cassandra, InfluxDB[8] and ScyllaDB
    - RowKey
        - 组成部分
            - Key部分
                - keyLen：占用4字节，用来存储KeyValue结构中Key所占用的字节长度。
                - 其它
            - Value部分
        - RowKey大小比较
            - 按照 RowKeyBytes、familyBytes、QualifierBytes、timestamp（越大排序越靠前）、type顺序
- 算法
    - 核心过程
        - LSM K路多路归并
            - 解决的问题：解决N个有序文件合并为一个有序文件

refs:

- HBase原理与实践