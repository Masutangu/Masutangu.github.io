---
layout: post
date: 2019-12-20T18:40:22+08:00
title: 漫谈分布式：事务和隔离性级别
tags: 
  - 分布式
---

## 前言

这篇文章是《漫谈分布式》系列文章的第四篇，重点讨论**单点数据库**中事务的 ACID 特性中的隔离性、所提供的不同隔离级别：**提交读（Read Committed）**、**可重复读（Repeatable Read）**、**快照隔离（Snapshot Isolation）**、**可串行化（Serializability）**以及不同隔离级别的实现和开销，关于分布式数据库的事务将在下篇文章讨论。

## 事务

**事务是一系列操作的集合，打包成一个逻辑单元提供给应用层的抽象**。应用层可以认为事务是单一原子性操作，事务执行要么成功（全部操作都执行成功），要么失败（全部操作都没有执行）。不同事务并行执行不会互相影响。

### ACID 特性

事务具备**原子性（Atomicity）**、**一致性（Consistency）**、**隔离性（Isolation）**、**持久性（Durability）**，简称 ACID 四大特性。
除了 ACID，在之前[《漫谈分布式：数据库的设计思想与实现》](https://masutangu.com/2019/12/04/distributed-system-1/) 这篇文章还提到过另外一种模型：BASE。

#### 原子性  

原子性指不可分割的执行单元。事务的原子性指事务的所有操作要么全部成功执行，要么全部不执行，不存在部分执行的情况。

#### 一致性  

一致性，这个词在不同的语境表示不同的含义：

* 在主从复制的场景中，一致性表示主从数据是否一致。
* 在 CAP 理论中，一致性是指线性一致性（Linearizability）。
* 在 ACID 的语境下，一致性表示数据库处于正确的状态。

数据库处于正确的状态，意味着事务从开始到结束需要遵守不变量，保证完整性约束不被破坏。**不变量其实是应用层层面的特性，需要应用层去保证的约束。**而 A（atomicity）、 I（isolation） 和 D（durability）是数据库层面的特性，应用层通过数据库提供的 A（atomicity）I（isolation）D（durability） 来保证 C（consistency）。

#### 隔离性 

隔离性又称可串行性，指并行执行事务时事务彼此是互相隔离的，就如同串行执行事务的效果一样。在生产环境中，由于性能问题，并不是所有数据库都会支持最高级别可串行性的隔离级别，例如 Oracle 中实现的是快照隔离的隔离级别，比可串行性隔离级别稍弱一些。

#### 持久性

指数据库一旦修改成功就是永久的，即使系统故障也不会丢失。

### 单一对象/多对象事务

#### 单一对象写操作

即使是只涉及单一对象的写，也需要原子性和隔离性的保证。比如考虑以下场景：

* 写入未完成时网络断开或断电或机器宕机，导致残留不完整数据。 
* 更新数据时断电，新旧数据混杂。
* 写入未完成时，其他读操作读取到不完整数据。

因此单一对象的写操作也需要保证原子性和隔离性。原子性可以通过 REDO/UNDO 日志来实现，隔离性可以通过加锁来保证。

#### 多对象事务

许多分布式数据库并不支持多对象事务。首先跨分区事务很难实现，且对系统的高可用性和性能也有影响。但某些场景下多对象事务是无法避免的，比如：

* 关系型数据库中的外键约束会涉及到多个对象的写操作。
* 如果存在二级索引，更新数据的同时也需要更新二级索引。

### 数据库隔离级别

多个并行执行的事务如果涉及到同一份数据的读写就容易出现 bug。**并行执行事务的正确性关键在于提供事务之间的隔离性。**最高级别的隔离性为可串行化，其保证了并行事务的执行和串行化执行的结果一致，但这同时也带来性能上的额外开销。

接下来主要讨论不同隔离级别各自解决了什么问题、存在什么样的不足以及简单讨论其实现机制。

#### 提交读

提交读提供了两个保证：

* **无脏读（No dirty reads）**：只能读到已提交的数据。
* **无脏写（No dirty writes）**：只能写覆盖已经提交的数据。 

##### 无脏读

<img src="/assets/images/distributed-system-4/illustration-1.png" width="600" />

事务 2 只有在事务 1 提交之后才能读到被更新的 x 的最新值。

##### 无脏写

脏写指覆盖了未提交的写数据，以下图为例：

<img src="/assets/images/distributed-system-4/illustration-2.png" width="600" />

事务 1 覆盖了事务 2 未提交的对 x 的写操作，事务 2 覆盖了事务 1 未提交的 y 写操作。

##### 实现细节

避免脏写一般通过 row-level 锁来实现。当事务需要修改某个对象时需要先获取该对象的锁，并且持有该锁直到事务提交或者终止。避免脏读也可以通过相同的机制来实现，但在实践中性能很低，读事务会被一个长时间执行的写事务所阻塞。因此许多数据库采用了另外的机制来避免脏读：**记录修改且未提交的旧值，在该事务未提交前，其他事务的读请求返回该旧值。**

需要注意的是提交读隔离级别无法避免如下读-修改-写带来的写丢失问题：

<img src="/assets/images/distributed-system-4/illustration-3.png" width="600" />

两个递增事务都返回成功，实际上只成功了一个。

#### 可重复读和快照隔离

以银行转账为例，提交读存在下面的问题：

<img src="/assets/images/distributed-system-4/illustration-4.png" width="600" />

事务 2 两次查询账号返回的总额多了 50，事务隔离的情况下，两次查询结果要么返回（100，100），要么返回（50，150）。出现这种现象称为**不可重复读或读偏差（Read Skew）**。

虽然这种情况只是读取时出现了短暂不一致，但在有些场景下这种短暂不一致会带来严重的后果：

* 备份

    备份的时候如果 dump 下来的数据不一致，之后从备份恢复数据库时就无法修复了。

* 完整性检查分析

    有时我们会定期对数据库做完整性检查的扫描。如果数据库会出现短暂的不一致性，这样的扫描检查就没太多意义（存在误报）。

**快照隔离**是解决读偏差最常见的解决方案。每个事务都从一个数据处于一致状态的数据库快照中读取，事务只能读取到在其开始执行时已经提交了的数据。很多数据库例如 PostgreSQL、MySQL（采用 InnoDB 存储引擎）、Oracle、SQL Server 都支持快照隔离。

快照隔离和可重复读这两个隔离级别很类似，stackoverflow 上有对比过这两种隔离级别的细微差别：

> "Snapshot" guarantees that all queries within the transaction will see the data as it was at the start of the transaction. "Repeatable read" guarantees only that if multiple queries within the transaction read the same rows, then they will see the same data each time. (So, different tables might get snapshotted at different times, depending on when the transaction first queries them.)

##### 实现细节

快照隔离通常实现上使用锁来避免脏写。**读操作则不需要获取任何锁。快照隔离的准则是读操作不阻塞写，写操作不阻塞读。**最常见的实现方法是**多版本并发控制（MVCC）**，每个对象记录多个版本的值，如下图。前面提到提交读隔离级别中实现避免脏读是通过记录修改且未提交的旧值来实现，而 **MVCC 是该方案的泛化版本。**提交读隔离级别的实现中，每个对象只需要保存两个版本：已经提交的和被覆盖但尚未提交的。快照隔离级别的实现中每个对象则需要保存多个版本：

<img src="/assets/images/distributed-system-4/illustration-5.png" width="600" />

##### 索引管理

多个数据版本的话，如何管理索引？一个方案是索引指向所有版本的数据，然后查询的时候再做过滤。PostgreSQL 管理索引做的优化可参考 [Mvcc Unmasked](https://momjian.us/main/writings/pgsql/mvcc.pdf)。CouchDB、Datomic 和 LMDB 采用的是另外的方案：append-only/copy-on-write B-trees。更新的时候不修改原来的树，而是把修改的页复制一份，然后逐层往上，从父节点到根节点，都拷贝并更新指向新的子节点，可参考[《漫谈分布式：数据库的设计思想与实现》](https://masutangu.com/2019/12/04/distributed-system-1/) 文中的说明。COW B-tree 实现下，每次写事务都会创建一个新的 B-tree 根节点，每个根节点是一个处于一致状态的数据库快照（因为不会修改原来的树），因此也无需根据事务 id 做过滤。

#### 避免更新丢失

前面介绍的提交读和可重复读隔离级别是并行读-写事务的隔离性保证，接下来讨论并行写-写事务会带来的竞争问题。**更新丢失（Lost Update）**指并行写事务在写冲突时出现写丢失，回顾之前小节提到的读-修改-写场景：

<img src="/assets/images/distributed-system-4/illustration-3.png" width="600" />

两个递增事务实际上只成功了一个，存在写丢失的情况。避免更新丢失有下列几种常见的解决方案：

##### 原子更新

有些数据库会提供原子更新操作，例如下面语句：

```
UPDATE counters SET value = value + 1 WHERE key = 'x';
```

在绝大多数关系型数据库中该语句可以被安全地并行执行，事务会在读取对象时使用**排他锁（Exclusive Lock）**上锁，其他事务需要等到其更新完成锁释放后才能获取到锁，这种隔离级别称为**游标稳定性隔离（Cursor Stability）**。由于游标稳定性隔离只会锁住访问和更新过的行，可能会出现不可重复读和幻读（见下文），但修改过的行会被上锁直到事务提交或回滚，保证不会出现脏读。

实现原子操作另外一种做法是在单一线程里执行原子操作。

##### 显式锁

还可以通过在应用层显式加锁的方式来避免写丢失。应用层先显式锁住待更新的对象，事务执行完提交之后，才释放锁。如下：

```
BEGIN TRANSACTION;
SELECT * FROM xxx
WHERE xxx
FOR UPDATE;
...
COMMIT;
```

SELECT FOR UPDATE 会对查询返回的所有行数据都添加排他锁，其他事务的更新操作会被阻塞。 

##### 检测更新丢失

相比起前面提到的提前上锁的预防机制，还可以有另外的思路：先放任事务并行执行，如果事务管理器检查到存在更新丢失，再终止该事务。在 [PostgreSQL 官方文档](https://www.interdb.jp/pg/pgsql05.html)讲解了如何检测更新丢失，执行 Update 操作时会调用 ExecUpdate 函数，其伪代码如下：

```
(1)  FOR each row that will be updated by this UPDATE command
(2)       WHILE true

               /* The First Block 目标行正在被其他事务更新 */
(3)            IF the target row is being updated THEN
(4)	              WAIT for the termination of the transaction that updated the target row

(5)	              IF (the status of the terminated transaction is COMMITTED)
   	                   AND (the isolation level of this transaction is REPEATABLE READ or SERIALIZABLE) THEN
(6)	                       ABORT this transaction  /* First-Updater-Win */
	              ELSE 
(7)                           GOTO step (2)
	              END IF

               /* The Second Block 目标行已经被其他事务更新 */
(8)            ELSE IF the target row has been updated by another concurrent transaction THEN
(9)	              IF (the isolation level of this transaction is READ COMMITTED THEN
(10)	                       UPDATE the target row
	              ELSE
(11)	                       ABORT this transaction  /* First-Updater-Win */
	              END IF

               /* The Third Block 目标行没有被其他事务更新或待更新 */
                ELSE  /* The target row is not yet modified or has been updated by a terminated transaction. */
(12)	              UPDATE the target row
                END IF
           END WHILE 
      END FOR 
```

从伪代码可以看出，Update 语句的执行有三个分支，对应三份代码块：**1. 目标行正在被其他事务更新；2. 目标行已经被其他事务更新；3. 目标行没有被其他事务更新或待更新**，从官方文档中提供的时序图可以更清晰的看出代码逻辑：

<img src="/assets/images/distributed-system-4/illustration-6.png" width="600" />


当事务 B 发现目标行正在被事务 A 更新时，阻塞直到事务 A 提交或终止。如果事务 A 成功执行且隔离级别是可重复读或者可串行化，则终止事务 B；如果事务 A 被终止，则重新进入 while 循环。当事务 B 发现目标行已经被事务 A 更新时，如果隔离级别为可重复读或可串行化，则终止事务 B，否则执行 update 语句。

##### CAS Compare-and-set

有些数据库支持 CAS 的原子操作，**但如果 where 语句是从老的快照中读取，那 CAS 不能避免更新丢失！**

##### 分布式系统下的更新丢失

通过锁或 CAS 避免更新丢失的保护机制只能针对单一一份数据的情况。而在分布式数据中，数据不只在一个节点上，同一份数据可能在多个节点上被同时修改，具体阅读[《漫谈分布式：数据复制》](https://masutangu.com/2019/12/13/distributed-system-2/) 了解多节点下的写冲突如何解决。

#### 幻读与写偏差

考虑预定房间的场景，两个并行事务同时执行下面语句：

```
BEGIN TRANSACTION;
    -- 检查房间 1 在该时间段是否有预定记录
SELECT COUNT(*) FROM bookings WHERE room_id = 1 AND
end_time > '2019-01-01 09:00' AND start_time < '2019-01-01 11:00';
    -- 如果没有被预定，则插入预定记录
INSERT INTO bookings
(room_id, start_time, end_time, user_id)
VALUES (1, '2019-01-01 09:00', '2019-01-01 11:00', user_id);
COMMIT;
```

两个事务有可能都成功插入了预定记录，导致房间预定冲突。这种现象称之为**写偏差（write skew）**，属于更新丢失的泛化版本。当两个事务读取相同的对象集，并且更新了其中的部分对象集时，就可能出现写偏差（如果更新了同样的对象，属于脏写或者更新丢失）。

出现写偏差现象可以总结出如下模式：

1. 执行 SELECT 语句查询是否符合某个条件。
2. 基于 SELECT 语句查询的条件，执行写操作或更新操作。
3. 执行的写操作或更新操作打破了步骤 1 中 SELECT 语句查询的条件。导致基于该 SELECT 查询条件的其他并行事务出现了写偏差。

**一个事务的写操作改变了另一个事务的查询结果，这种现象称之为幻读（phantom）**，快照隔离无法避免幻读。在更新丢失小节提到过显示加锁的方法，通过 SELECT FOR UPDATE 为查询结果加上排他锁，是可以避免写偏差。但上述例子中 SELECT 查询结果返回是空数据集，这种情况下就没有办法通过 SELECT FOR UPDATE 来加锁。幻读可以通过**物化冲突（Materializing Conflicts）**的方式来解决，以预定房间的例子来说，**创建一张\<房号-时间段\>表，把查询条件-“房号和时间段”进行物化**，通过 SELECT FOR UPDATE 该表来加锁就可以避免幻读。但这种方法会导致业务代码和锁机制耦合在一起，不太建议使用。避免幻读最好的办法是提供可串行化的隔离级别。

#### 可串行化隔离级别

前面提到的几种隔离级别虽然解决了不少问题，但依然存在弊端：

* 概念难以理解，而且不同数据库对不同隔离级别的实现和支持各不相同。
* 难以判断应用层业务代码在哪个隔离级别上运行是安全的。
* 没有合适的工具检测竞争条件。

可串行化隔离级别是最高的隔离级别，保证事务并行执行的结果与串行执行是一致的。实现可串行化通常有下面三种方法：

* **串行执行事务**
* **二阶段锁**
* **乐观并发控制技术**

##### 串行执行事务

最简单的并发控制方案是避免并发，在单个线程上串行的执行事务。VoltDB/H-Store、Redis 和 Datomic 都采用了这种方案。串行执行有时会比并发执行更有效率，因为执行过程中不需要加锁。串行方式虽然简单，但吞吐量受限于 CPU 单核的速度。

单线程串行执行，出于吞吐量的考虑不允许交互性的事务，应该以**存储过程（Stored Procedure）**的形式来封装事务。

串行执行事务需要注意以下几点：

* 每个事务必须小而快，慢事务阻塞会拖慢后续事务的执行。
* 内存需要装得下事务执行所需的数据集，如果需要磁盘访问则会非常慢。

##### 二阶段锁 Two-Phase Locking (2PL)

目前实现串行化最广泛的算法是二阶段锁：

* 如果事务 A 读取了一个对象，事务 B 想对该对象执行写操作前需要等 A 已经提交或者终止。
* 如果事务 A 写入了一个对象，事务 B 想读取该对象，必须等 A 已经提交或者终止（不允许读取旧版本的值）。

**二阶段锁的核心思想在于：读写互相阻塞。**快照隔离则是读写互不阻塞。MySQL (InnoDB) 和 SQL Server 采用 2PL 来提供可串行化隔离级别，2PL 的读写互相阻塞通过**共享锁**和**独占锁**来实现：

* 事务读之前需要先获取共享锁，共享锁允许被多个事务同时获取，但如果有其他事务已经持有该对象的独占锁，则读事务需要等待。
* 事务写之前需要先获取独占锁，如果该对象已经存在锁（共享或者独占），则写事务需要等待。
* 如果事务先读后写，则需要将共享锁升级为独占锁。
* 事务获取锁之后，需要等待事务提交或终止时才能释放锁。这也是命名 two-phase 的原因：**第一阶段-事务执行时获取锁，第二阶段-事务提交或终止时释放锁。**

2PL 下难免会出现死锁，检测死锁最简单的方式是超时机制，当事务执行超过一定时长，就将其回滚。还可以通过有向等待图的方式检测死锁：

<img src="/assets/images/distributed-system-4/illustration-7.png" width="400" />

当事务需要等待其他事务的锁时，就生成一条有向边指向持有锁的事务。如果出现环路证明有死锁。

前面小节提到了幻读的现象：一个事务的更新导致改变了另外一个事务的查询结果。解决幻读需要采用**谓词锁（Predicate Lock）**，谓词锁和其他锁的区别在于，它不属于某个特定的数据对象，而是属于某个匹配查询条件。

* 如果事务 A 要读某个匹配查询条件的对象，例如 SELECT 查询，需要先获取查询条件上的共享谓词锁。如果此时另外一个事务 B 持有任何满足这个查询条件的对象的独占锁，A 必须等待直到 B 释放锁。

* 如果事务 A 想要插入、更新或者删除某个对象，需要先检查对象的新值和旧值是否与当前已经存在的谓词锁匹配，如果事务 B 持有该匹配的谓词锁，则 A 必须等待 B 释放。

支持谓词锁的 2PL 可以避免所有竞争，包括写偏差，因此提供了可串行化级别的隔离。不过当存在很多谓词锁的情况下，检查匹配的锁是一个非常耗时的操作。因此很多数据库使用**索引区间锁（Index-Range Locking）**进行优化。索引区间锁类似于谓词锁，但没有谓词锁那么精确，索引区间锁锁住的范围更大，因此查找匹配的开销更低。

举例来说，预定下午 1-3 pm 时间段房号 1 的房间，若索引为房号，将索引区间锁锁在房号 1 索引，这种方式将锁定房间 1 的所有时间段；若索引为时间，把锁加在 1-3 pm 的索引范围，这种方式将锁住下午 1-3 pm 时间段的所有房间。如果没有合适的索引，则可以把锁加在整张表上。索引区间锁相比谓词锁锁住了更大范围，虽然会影响并发的性能，但这种做法保证了可串行级别的隔离。索引区间锁在 PostgreSQL 的实现可以参考 [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf)。

##### 乐观并发控制策略 

**可串行快照隔离（Serializable Snapshot Isolation 简称 SSI）**是在 2008 年首次提出，可串行快照隔离保证了可串行的隔离级别且只带来很小的性能开销，在单点数据库（PostgreSQL 在 9.1 版本之后使用 SSI 实现可串行快照隔离）和分布式数据库（FoundationDB 使用了类似的算法）都有应用。


2PL 是所谓的**悲观并发控制机制**，类似互斥锁；而 SSI 为**乐观并发控制机制**：事务执行中不阻塞，提交时再检查隔离规则。如果并行事务存在很多竞争，则不适用乐观并发控制机制，因为会引发大量事务需要重试；如果事务之间存在少量竞争，乐观机制比悲观更适用。

从命名可以看出 SSI 的实现基于快照隔离，在其基础上采用 MVSG 算法来检测冲突。具体可读论文 [Serializable Isolation for Snapshot Databases](https://www.cs.nyu.edu/courses/fall12/CSCI-GA.2434-001/p729-cahill.pdf) 和 Michael Cahill 的博士论文 [Serializable Isolation for Snapshot Databases](https://cahill.net.au/wp-content/uploads/2010/02/cahill-thesis.pdf)。


## 附录

### 隔离级别相关论文

* [HAT, not CAP: Towards Highly Available Transactions](https://www.bailis.org/papers/hat-hotos2013.pdf)
* [Hermitage: Testing the “I” in ACID](https://martin.kleppmann.com/2014/11/25/hermitage-testing-the-i-in-acid.html)
* [Making Snapshot Isolation Serializable](https://www.cse.iitb.ac.in/infolab/Data/Courses/CS632/2009/Papers/p492-fekete.pdf)
*  [Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions](https://pmg.csail.mit.edu/papers/adya-phd.pdf) 从理论角度分析弱隔离级别
* [Isolation in DB2 (Repeatable Read, Read Stability, Cursor Stability, Uncommitted Read) with Examples](https://mframes.blogspot.com/2013/07/isolation-in-cursor.html)
* [Cursor Stability (CS) – IBM DB2 Community](https://www.toadworld.com/platforms/ibmdb2/w/wiki/6661.cursor-stability-cs.aspx)
* [Architecture of a Database System](https://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf) 