---
layout: post
date: 2019-12-04T12:39:25+08:00
title: 漫谈分布式：数据库的设计思想与实现
tags: 
  - 分布式
---

## 前言

这篇文章是《漫谈分布式》系列文章的第一篇，主要讨论数据库中各类数据模型，以及数据库存储引擎中索引结构的实现。

## 数据模型 Data Model

在业务研发初期，数据模型的选型及其重要。不同的数据模型支持的操作集各不相同，所适用的业务场景也不同。因此在对数据模型进行选型时，不仅要熟悉业务，基于业务场景出发，从业务发展的长远角度来做决策，同时也需要深入了解不同数据模型的优缺点及适用场景。

常见的数据模型有：**关系型（Relational Model）、文档型（Document Model）和图（Graph-Based Data Model）**。

### 关系型数据模型

关系模型的概念是由 Edgar F. Codd 在论文 [A Relational Model of Data for Large Shared Data Banks](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf) 中首次提出，奠定了关系模型的理论基础。关系模型的提出是为了解决层级/网状数据模型存在的不足。层级数据模型是以树形结构组织数据的，每个子节点只有唯一父节点。网状数据模型则允许子节点存在多个父节点，以实现“多对多”的实体对应关系。网状模型和层级模型中，**数据依赖于底层物理存储实现**。举例来说，层级数据模型和网状数据模型的数据访问路径受限制于底层的物理存储，只能基于节点的父子关系依次去遍历。

关系模型消除了三大依赖：**排序依赖（数据的展现独立于数据的存储）**、**索引依赖（新增删除索引对应用无影响）**和**访问路径依赖（不限制数据的访问路径）**，弥补了网状模型和层次模型的不足。

关系型数据库，是指以关系模型来组织数据的数据库。关系型数据库的模式（schema）固定，支持连表查询，支持事务（ACID)。

### 文档数据模型

互联网时代下涌现出大量新型业务，也催生了 NoSQL。与关系型数据库不同，NoSQL 是 schemaless 的，并不要求固定 schema，且鉴于关系型数据库维护一致性带来高昂的成本，NoSQL 一般**只提供最终一致性保证**，**支持高并发**，**可扩展性高**。

### SQL 与 NoSQL 的对比

在选择数据模型时，需要从自身业务场景出发，是否需要结构化数据，是否需要 Many-to-Many 的关系来组织数据，是否需要支持联表查询，是否需要事务支持，是否需要强一致性保证？

#### 1. Schema-on-Read vs Schema-on-Write 

Schemaless 实际上是 ```Schema-on-Read```，读取的时候才解析 schema。关系型数据库则属于 ```Schema-on-Write```，写入时检查 schema。 NoSQL 胜在 schema 灵活，关系型数据库则在结构化数据方面更有优势，并且保证了数据一致性。需要注意的是，关系型数据库中变更 schema 相当麻烦，举例来说，MySQL 的 alter table 操作需要复制整个表，可能要占用数分钟甚至数小时。

#### 2. Scalibility 不同

关系型数据库通常采用**分库分表**的方式来应对海量数据和高并发请求：

* **读写分离**：一主多从，主负责写，从允许读。主从保持数据同步。
* **垂直分库**：将不同模块的数据拆分到不同的数据库中。
* **垂直分表**：按列拆分，分离不常使用的字段。
* **水平分库**：将表的数据按规则划分到不同的数据库。
* **水平分表**：将表的数据按规则划分到多张表中。


水平分库分表后难以处理跨库 join/group by/order by 等操作，且跨库涉及的事务处理将带来高昂的成本（后续介绍分布式事务会讲解 MySQL XA 分布式事务）。

NoSQL 只保证最终一致性，扩展性强。NoSQL 和关系型数据库**本质上是 BASE 模型和 ACID 模型的区别**。BASE 即**基本可用（Basically Available）**、**软状态（Soft State）**、**最终一致性（Eventually Consistency）**，由 Dan Pritchett 在 [BASE: An Acid Alternative](https://dl.acm.org/citation.cfm?id=1394128)中提出，其核心思想是：

> Trading some consistency for availability can lead to dramatic improvements in scalability.

即**牺牲部分一致性可以显著提升系统的扩展性**，感兴趣可以阅读本文结尾的附录章节。

**CAP 定理**指出，分布式计算机系统**不可能同时满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition Tolerance）。**

<img src="/assets/images/distributed-system-1/illustration-1.png" width="400" />

这也是系统设计中的 **trade-off**，没有银弹也没有优劣之分，只能根据业务的场景，结合组件所提供的能力做出取舍。例如，Riak 采用最终一致性，牺牲强一致性以换取可用性的大幅提升；关系型数据库为提供 ACID 保证（通常使用二阶段提交 Two-Phase Commit）舍弃了可用性；内存数据库例如 Redis、Memcache 为了提升写性能，舍弃了持久性（数据保存在内存，定期刷磁盘）。在选型前先思考，业务场景是否可以容忍数据丢失（舍弃持久性）？是否可以接受数据的不一致（舍弃强一致性）？还是宁愿服务不可用也要保证一致性（舍弃高可用）？

## 存储引擎

介绍完数据模型后，来讨论下数据库存储引擎的实现。**存储引擎即对数据的一种存取机制**，其中最关键的概念是索引，**索引是帮助数据库高效获取数据的数据结构**。常见的索引有基于 **Hash Table Index**、**Log-Structured** 和 **Page-Oriented Storage（B-Tree）** 。不同数据结构实现的索引各有各自的优缺点，不同的存储引擎中实现索引所采用的数据结构也不相同。

### **Hash Table Index**

Hash Table Index 适用于 Key-Value 数据库，Riak 的 bitcask 存储引擎就使用了 Hash Table Index。写操作将记录追加在磁盘文件，并把其所在的偏移值记录在内存的哈希表中。读操作根据哈希表中 key 对应的偏移值从磁盘文件相应的位置读取。 

<img src="/assets/images/distributed-system-1/illustration-2.png" width="600" />

Hash Table Index 具备了如下优点：

* 读写效率高。每次写操作都是顺序写，并且只需更新内存数据结构，读操作只需要查询一次内存哈希表中的偏移值再加上一次硬盘随机读。
* 模型简单，易于理解。

实践中使用 Hash Table Index 需要注意下面几个细节：

* 重启后通过遍历磁盘数据文件记录来重建哈希表会比较耗时，bitcask 采用索引文件来加速重启后重建哈希表。
* 使用 checksum 来检测未完整写入的数据。
* 注意并发写的问题，写操作可以通过采用单一写线程的方式来避免竞争。因为写入后记录是不可修改的，因此读操作允许多线程。

Hash Table Index 局限在于：

* 内存需要足以装下整个哈希表。
* 范围查询（Range Query）很低效，因为哈希表中的 key 是无序的，范围查询等同于遍历哈希表中所有的 key。 

### SSTables 和 LSM-Trees

上一小节提到 Hash Table Index 存储引擎的局限性，一是哈希表必须放得下在内存中，二是对范围查询不友好。该如何优化呢？

如果磁盘文件是按 key 来排序的，那哈希表就可以只存部分 key（稀疏索引），减小了哈希表占用的内存，如下图：

<img src="/assets/images/distributed-system-1/illustration-3.png" width="600" />

当需要查找 key 为 "at" 的记录时，只需要找到 key "apple" 和 "beer" 的偏移值，扫描磁盘文件中这段区间，就可以找到 key 为 "at" 的记录（或者该记录不存在）。

如果哈希表也是按 key 来排序的，那也天然支持了范围查询。

因此如果能保证哈希表和磁盘文件都是按 key 排序的，就顺利解决了上述提到的 Hash Table Index 的局限性。算法简单流程如下：

* 在内存中使用红黑树或 AVL 树（通常称为 memtable）组织 key 映射，且按 key 排序。
* 当 memtable 达到一定阈值时，以 SSTable 的格式**顺序写**到磁盘中。由于 memtable 是有序的，持久化生成的 SSTable 自然也是有序的。
* SSTable 会定期进行合并，因为 SSTable 都是有序的，采用 mergesort 算法即可高效合并。
* 每次写操作都会记录操作流水，防止因 memtable 来不及持久化到磁盘导致数据丢失。

LevelDB 和 RocksDB 就采用了上述的算法。这种基于日志合并的思想称之为 **LSM-Tree（Log-Structured Merge-Tree）**，最开始是由 Patrick O'Neil1 在论文 [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf) 中提出的。SSTable 和 memtable 的概念则出自 Google 的 [Bigtable: A Distributed Storage System for Structured Data](https://ai.google/research/pubs/pub27898) 。Lucene 也采用了类似的方法来存储项字典（term dictionary）。

由于 LSM-Tree 的读性能较差，需要依次检索 memtable、从新到旧的 segment files。使用 bloom filter 优化，先判断 key 是否存在，不存在的 key 则不需要再次遍历检索，可以减少大量无效查询。

合并 SSTable 有两种不同的策略：**STCS（Size-Tiered Compaction Strategy）**和 **LCS（Leveled Compaction Strategy）**。LevelDB 和 RocksDB 采用的是 LCS，HBase 则是 STCS。STCS 会在类似大小的 SSTable 数量达到阈值时进行合并。LCS 则是在每一层的 SSTable 的总数达到阈值时，与下一层的 SSTable 进行合并。

<img src="/assets/images/distributed-system-1/illustration-4.png" width="600" />

LCS 解决了 STCS 空间放大和读放大问题，但却引入了较严重的写放大问题，在这两篇文章：[Scylla’s Compaction Strategies Series: Write Amplification in Leveled Compaction](https://www.scylladb.com/2018/01/31/compaction-series-leveled-compaction/) 和 [Scylla’s Compaction Strategies Series: Space Amplification in Size-Tiered Compaction](https://www.scylladb.com/2018/01/17/compaction-series-space-amplification/)有详细的介绍。简单的说，STCS 在 compaction 的时候，将 X size 的数据合并为 X size，而 LCS compaction 的时候，是将 X size 的 level i 数据和约 10 * X 的 level i+1 数据合并到 level + i 去的，写放大因子接近 10 倍。因此 LCS 适合读（特别是随机读）多写少的场景，STCS 适合写多读少的场景。

> 为什么 LCS 合并的时候，level i + 1 的数据量是 level i 的 10 倍？举例来说，假设因子是 10，L1 有 10 个 SSTable，则 L1 中每个 SSTable 覆盖的数据大概为整个数据集的十分之一。L2 中 SSTable 数量为 L1 的十倍，总共 100 个，每个 SSTable 覆盖的数据约为整个数据集的百分之一。那么 L1 中一个 SSTable 将会和 L2 中大约 10 个 SSTable 的数据有交集。在合并单个 L1 SSTable 需要读取 L2 中与其有数据交集的 SSTable，从前面分析可知大约需要读取 L2 中 10 个 SSTable，因此写放大是 10 倍。

### B-Trees

数据库中最常用的索引结构是 B-Tree。基本上所有的关系型数据库的索引都是基于 B-Tree 实现的。B-Tree 的设计哲学与 Log-Structured 不同，Log-Structured 将数据以 segments 来切分（mb级别），并总是以顺序写的方式写入。B-Tree 则是将数据以大小固定的 page 来切分（通常是 4kb），每次读写以 page 为单位。在对磁盘的使用上，两者有本质上的不同：**LSM 是 transfer 型，取决于磁盘的传输速率；B-Tree 是 seek 型，取决于磁盘的查找速率**，每次访问通常需要 log(N) 次磁盘查找。

关于 B-Tree 的发展历程可以读 [A Short History of the BTree](https://www.perforce.com/blog/vcs/short-history-btree)。

<img src="/assets/images/distributed-system-1/illustration-5.png" width="600" />

分支因子（branching factor）是指 B-Tree page 中 ref 的数量。4层深、4 KB 大小的 page size 且分支因子为 500 的 B-tree 可以存储 256 TB 的数据量。

考虑到写操作可能会涉及多个 page 的更新，为了避免进程在异常终止的情况下出现部分更新，B-Tree 中写操作都需要先记录 write-ahead log（WAL） ，在进程重启时可以根据 WAL 来重建 B-Tree。[LMDB](https://static.sched.com/hosted_files/buildstuff14/96/20141120-BuildStuff-Lightning.pdf) 则采用 Copy-on-Write 的机制来实现，如下图。进程异常终止的情况下也能保证数据结构的完整性，因此无需 WAL 也无需在进程重启后执行重建恢复。

<img src="/assets/images/distributed-system-1/illustration-6.png" width="600" title="copy-on-write" />

Copy-on-Write 的 B-Tree 也称为 Append-Only B-Tree，可以参考 [how the append-only btree works](https://www.bzero.se/ldapd/btree.html) 这篇文章了解其实现。

### LSM-Tree 与 B-Tree 的对比

一般来说，LSM-Tree 适合多写少读的场景，B-Tree 适合多读少写的场景。LSM-Tree 读操作需要检查 memtable、多个 segement files，效率较低。但范围查询的部分场景下，LSM-Tree 会优于 B-Tree，因为 B-Tree 下执行范围查询所得出多个 page 在物理磁盘上可能是非连续存储的，每一页数据都需要走一遍磁盘寻址。而 LSM-Tree 会定期执行 merge，一系列按 key 排好序的数据会依次写到磁盘上。

#### 读写放大对比

详细可读：[B-Tree vs LSM-Tree](https://tikv.org/docs/deep-dive/key-value-engine/b-tree-vs-lsm/)
及论文：[A Comparison of Fractal Trees to Log-Structured Merge (LSM) Trees](https://www.pandademo.com/wp-content/uploads/2017/12/A-Comparison-of-Fractal-Trees-to-Log-Structured-Merge-LSM-Trees.pdf)。

LSM-Tree 相比起 B-Tree 写放大（write amplification）更低，但读放大更高。

以字节为单位，假设 B-Tree 的 page size 为 B，键值、指针和记录的大小都为常量字节数，每个中间节点包含 O(B) 个子节点，每个叶子节点包含的记录数为 O(B)，可知 B-Tree 的高度为 O(logB N/B)（N 为整个数据库记录数据的总大小）。在最差情况下，每次写操作都需要将 page 刷回磁盘，写放大为 O(B)。最差情况下，读放大为 B-Tree 的高度，为 O(logB N/B)。

假设采用 LCS 策略的 LSM-Tree 每层数据量逐层以 k 倍增长，数据总量为 N，sstable 文件大小为 B，则 LSM-Tree 的高度为 O(logk N/B)。每一层的数据平均被上一层合并 k/2 次，因此写放大为 O(k * logk N/B)。读操作需要在每一层执行二分查找，最后一层的读次数为 log N/B，倒数第二层的读次数为 log N/(k * B) = log N/B - log k，倒数第三层为 log N/(k^2 * B) = log N/B - 2*log k，以此类推。第一层为 1，因此总的读放大为 O((log ^ 2 N/B)/logk)。

采用 STCS 策略的 LSM-Tree，每层数据只会被写到下一层一次，因此写放大为 O(logk N/B)，但读操作需要读取每一层的数据，因此读放大为 O(k(log ^ 2 N/B)/logk)。

对于写操作频繁的业务场景来说，写放大越大，会占用更多的磁盘带宽，因此影响到读请求的处理速度。如果采用 LCS 策略的 LSM-Tree，还会影响到 compaction 的执行，导致 L0 的 sstable 越来越多，进而带来更多的空间放大和读放大。

相比起 B-Tree，LSM 的写放大相对较小，而且基本是顺序写。B-Tree 则是随机写，在机械硬盘上顺序写比随机写速度更快，不过有些 SSD 会把随机写转化为顺序写，因此随机写还是顺序写带来的差别不大。另外，LSM 周期性的 compaction 操作会带来性能抖动。

#### 空间利用率对比

LSM-Tree 可以更好的进行压缩，占用的磁盘空间更小，并且会定期 compaction 消除冗余数据减少磁盘占用。B-Tree 存在碎片的问题，尤其是数据更新修改频繁的业务场景。碎片分为**内部碎片**和**外部碎片**。外部碎片指数据存储的物理顺序和逻辑顺序不一致。逻辑顺序是由索引键定义的，物理顺序是在物理磁盘中数据页的存储顺序。当更新索引键导致原来的 page 容量不够需要拆分时，数据的逻辑顺序和物理顺序不再一致，产生外部碎片。内部碎片则指在索引页内部存在没有被使用的空间，部分空间被闲置，存在空间浪费。

## 附录

### BASE

应对高并发，应用通常有两种扩展方式：**水平扩展**和**垂直扩展**。垂直扩展即将应用部署在更高性能的机器上，但始终受限于单台机器的性能且成本较大。水平扩展更加灵活，其一般分为两个维度：一是把不同模块的数据划分到不同的数据库，称之为 functional scaling；另一种是相同模块的数据切分到不同的数据库，称之为 sharding。如下图：

<img src="/assets/images/distributed-system-1/illustration-8.png" width="600" />

#### Functional Partitioning

良好的数据库设计，通常会按不同模块将数据切分到不同的表。关系型数据库还会提供外键给使用者以维护不同表之间的数据一致性，但这也限制了数据库的部署策略，外键约束的表必须位于相同的数据库中，意味着需要部署在同一台机器上。因此当需要对数据库进行水平扩展，将不同模块的表部署到不同的机器上，就无法依赖外键来维持一致性了，此时维护一致性的要求就转移到应用侧来实现。

#### ACID Solution

目前实现跨数据库实例的 ACID 保证最常用的方法是 2PC。但 2PC 降低了整个系统的可用性。

#### An ACID Alternative

ACID 为数据库提供了一致性的保证，BASE 则是为了提升系统的可用性而提出。BASE 即**基本可用（basically available）、软状态（soft state）又称柔性事务、最终一致性（eventually consistent）。**

以一个买卖交易场景为例子，下面我们一步步进行从 ACID 到 BASE 的转换。定义表 user 记录每个用户的买卖总金额，定义表 transaction 记录每次交易的买卖双方和交易金额：

<img src="/assets/images/distributed-system-1/illustration-9.png" width="600" />

如果是 ACID 的方式来实现，SQL 语句大致如下：

```
Begin transaction
 Insert into transaction(xid, seller_id, buyer_id, amount);
 Update user set amt_sold=amt_sold+$amount where id=$seller_id;
 Update user set amt_bought=amount_bought+$amount where id=$buyer_id;
End transaction
```

如果我们需要对数据进行切分，user 表和 transaction 表位于不同的数据库实例：

<img src="/assets/images/distributed-system-1/illustration-10.png" width="600" />

这时就无法用单个事务来保证 ACID 了。如果换个角度，把 user 表当做 transaction 表的一个缓存，允许短暂的不一致性，那么可以如下修改 SQL 语句：

```
Begin transaction
 Insert into transaction(id, seller_id, buyer_id, amount);
End transaction
Begin transaction
 Update user set amt_sold=amt_sold+$amount where id=$seller_id;
 Update user set amt_bought=amount_bought+$amount
where id=$buyer_id;
End transaction
```

但如果 user 表的事务执行失败，数据就会永远处于不一致的状态，引入一个支持持久化的消息队列来解决这个问题：

```
Begin transaction
 Insert into transaction(id, seller_id, buyer_id, amount);
 Queue message “update user(“seller”, seller_id, amount)”;
 Queue message “update user(“buyer”, buyer_id, amount)”;
End transaction
For each message in queue
 Begin transaction
 Dequeue message
 If message.balance == “seller”
 Update user set amt_sold=amt_sold + message.amount
 where id=message.id;
 Else
 Update user set amt_bought=amt_bought + message.amount
 where id=message.id;
 End if
 End transaction
End for
```

前面提到说，跨节点需要 2PC 来保证事务的 ACID。如果我们把消息队列部署在 transaction 表所在的机器，则第一个事务（insert + queue）可以避免使用 2PC 来保证 ACID，但第二个事务（dequeue + update）则需要 2PC 来保证，因为 user 表和消息队列部署在不同的机器上。因此这个方案为了保证数据的一致性，还是得采用 2PC。

如何能避免 2PC 又可以维持数据的最终一致性呢？一个方案是取出消息后确保处理成功后才删除。如果中间崩溃了，进程恢复后消息并没有丢失，还可以继续处理。但假设消息处理成功了，但还没删除的时候进程崩溃了，那么进程恢复之后该消息会被再次处理。考虑到这点，我们需要把消息处理逻辑设计为**幂等性**的，允许多次重复处理，就可以采用上述的方案。 

新增 applied 表，记录 trans 消息是否已经被处理过：

<img src="/assets/images/distributed-system-1/illustration-11.png" width="600" />

修改 SQL 语句如下：

```
Begin transaction
  Insert into transaction(id, seller_id, buyer_id, amount);
  Queue message “update user(“seller”, seller_id, amount)”;
  Queue message “update user(“buyer”, buyer_id, amount)”;
End transaction
For each message in queue
  Peek message
  Begin transaction
  Select count(*) as processed from applied where xid=message.trans_id and amount=message.balance and user_id=message.user_id
   
  If processed == 0
    If message.balance == “seller”
      Update user set amt_sold=amt_sold + message.amount where id=message.id;
    Else
      Update user set amt_bought=amt_bought + message.amount where id=message.id;
    End if
    Insert into applied(message.trans_id, message.user_id, message.balance);
  End if
  End transaction
  
  If transaction successful
    Remove message from queue
  End if
End for
```

通过这样的设计，我们支持应用的水平扩展，实现了更高的可用性，且保证了最终一致性。

#### EDA 事件驱动架构

[BASE: An Acid Alternative](https://dl.acm.org/citation.cfm?id=1394128) 中提到了 EDA（Event-Driven Architecture），这里也简单聊一聊。EDA 的思想在于**事件的生成和分发处理**。优点在于事件的生产者和消费者互相独立，部署灵活，组件高度解耦。

EDA 通常有两种不同的拓扑结构：**中介（mediator）**和**代理（broker）**。Mediator 用于当需要在一个事件中通过中介来协调多个步骤，broker 则适用于去中心化，事件链式处理的场景。

Mediator 模式下组件包括 mediator 和事件处理器，事件分为初始事件（initial event）和待处理事件（processing  event）。Mediator 收到的原始事件为初始事件，根据原始事件和业务处理逻辑，mediator 会生成相应的待处理事件分发给后端事件处理器，如下图所示：

<img src="/assets/images/distributed-system-1/illustration-12.png" width="600" />

需要注意的是，mediator 虽然了解处理初始事件的每一个步骤，**但它并不需要亲自执行业务逻辑，而是切分为各个待处理事件，分发给事件处理器去做具体的业务逻辑**。另外**每个事件处理器都是相互独立的**，彼此不依赖。

Broker 模式下组件包括 broker 和事件处理器。Broker 模式下没有初始事件的概念，每个事件处理器都以“接收一个事件->处理->产生一个新事件”的模式运作，一环扣一环，就好比接力赛跑：

<img src="/assets/images/distributed-system-1/illustration-13.png" width="600" />

更多细节请参考 [Software Architecture Patterns-Chapter 2. Event-Driven Architecture](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch02.html)。


### B-Tree 的 page size 取值

B-Tree 的 page size 取值一般为 4 KB 或 16 KB。写操作频繁的业务场景建议设置小一些的 page size 值，读操作频繁的业务场景建议设置相对较大的 page size 值。因为 page size 越大，写放大越严重，但树的高度也越浅。另外，如果 page size 设置过小，不能很好的利用机械硬盘的带宽。假设 page size 为 4 KB，磁盘传输速率为 100 MB/s，磁盘寻道时间为 5ms，传输 4 KB 的耗时为 0.04 ms，读/写一个 4 KB 的 page 总共耗费 5.04 ms，那么算下来有效传输速率仅为 794 KB /s，远远没有达到 100 MB/s。

### LSM 读写放大优化

参考资料：

* [Optimizing Space Amplification in RocksDB](https://cidrdb.org/cidr2017/papers/p82-dong-cidr17.pdf)
* [WiscKey: Separating Keys from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)

### WAL 的实现

待补充。