---
layout: post
date: 2016-07-07T15:02:11+08:00
title: LevelDB 源码阅读
category: 源码阅读
---

# LevelDB 文档

## Compaction

### 文件

* Log files

    log file (*.log) 存储最近的更新操作记录。当 log file 达到一定大小时，其将被转换成 sorted table，并重新创建一份 log file。

    同时 leveldb 会在内存拷贝一份 log file 的内容，存储在 memtable。所有的毒操作将会查询 memtable。 

* Sorted tables

    sorted table (*.sst) 存储根据 key 来排序的记录。每一条记录要么是 key 对应的值，要么是 key 的删除标记。

    sorted table 通过 level 来组织。从 log file 生成的 sorted table 为 young level（也称为 level 0）。当 young level 下的文件数超过一定阈值时，young level 下的所有文件会与 level 1 下与其有重叠的文件合并 ，并生成新的 level 1 下的文件。

    level 0 下的不同文件可能包含重叠的 key 值。但其他 level 下的文件，key range 都是不重叠的。当 level L 下的文件总大小超过 10^L MB时，level L 下的一个文件就会与 level L+1 下的所有与它有重叠的 key 的文件合并，生成一系列 level L+1 下的新文件。
    
* Manifest
    Manifest 文件列出构成各个 level 的 sorted table，以及相应的 key ranges。每次数据库重新打开都会创建一个新的 manifest 文件。

* Current
    Current 用于记录当前最新的 Manifest 文件名。

* Info logs 
    记录一些信息。


### Level 0
当 log file 增大到一定大小时，会做如下操作：
* 创建新的 memtable 和 log file 用与记录后续的更新操作。
* 后台执行：
    * 将之前 memtable 的内容写到 sstable
    * 丢弃 memtable
    * 删除旧的 log file 和旧的 memtable
    * 将新创建的 sstable 添加到 young level

### Compactions

当 level L 的大小超过自身的限制时，后台线程会进行 compact 操作。Compaction 会从 level L 选择一个文件，以及 level L+1 中与该文件 key 重叠的文件。

Compaction 将选中的文件进行合并，生成一系列新的 level L+1 文件。当当前输出文件大小达到 2 MB 时，或当当前输出文件的 key range 覆盖超过十个 level L+2 的文件时，就会输出一个新的 level L+1 文件。

Compaction 是以 rotate 的方式展开的，每次记住上次 compaction 的最后一个key，下次 compaction 将从该 key 后的第一个文件开始，如果没有则从头开始。

Compaction 会丢弃掉被覆盖的 value 值。如果更高 level 下的文件的 key range 不包含当前 key，该 key 的删除标记也将被丢弃掉。

# 代码架构

## include/leveldb


# 细节

## Comparator

    * FindShortestSeparator

    * 单例
        ```C++
        const Comparator* BytewiseComparator() {
            port::InitOnce(&once, InitModule);
            return bytewise;
        }
        ```

