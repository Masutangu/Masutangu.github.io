---
layout: post
date: 2016-03-08T16:07:58+08:00
title: 使用 binlog 实时监控 Mysql 数据更新
tags: 数据库
---

上一篇文章《[UDF+Trigger实时监控Mysql数据更新](https://masutangu.github.io/blog/2016/02/29/udftrigger%E5%AE%9E%E6%97%B6%E7%9B%91%E6%8E%A7mysql%E6%95%B0%E6%8D%AE%E6%9B%B4%E6%96%B0/)》介绍了用UDF+Trigger的方式来监控Mysql数据的更新，这次介绍下使用binlog监控数据更新的方法。

# binlog 简介 #

>The binary log is a set of log files that contain information about data modifications made to a MySQL server instance.

其主要有以下两个用途：

* 主从同步

* 数据恢复

mysql主从同步原理 利用binlog来监控mysql数据的更新，以更新缓存。原理类似于mysql的主从同步。先简单介绍下mysql主从同步的原理：

>* Whenever the master’s database is modified, the change is written to a file, the so-called binary log, or binlog. This is done by the client thread that executed the query that modified the database.
>* The master has a thread, called the dump thread, that continuously reads the master’s binlog and sends it to the slave.
>* The slave has a thread, called the IO thread, that receives the binlog that the master’s dump thread sent, and writes it to a file: the relay log.
>* The slave has another thread, called the SQL thread, that continuously reads the relay log and applies the changes to the slave server.

# 利用binlog监听mysql更新 #
我们的目的是监听mysql数据变化，及时更新缓存以保证缓存数据不过期。如果我们模拟Mysql Slave的交互协议，伪装自己为Mysql Slave，向Master发送dump协议，Master收到dump请求，就会推送binary log给我们的伪Slave。接下来解析binlog，把相应的更新同步到缓存就可以了。

# Python-Mysql-Replication # 
伪装Mysql Slave，解析binlog都需要对mysql有更深入的了解。为了把精力关注在我们的目标上，这里我选择了一个现有的python库：Python-Mysql-Replication。

官方给的demo很简单：
<img src="/assets/images/using-binlog/illustration-1.png" alt="示例1" title="示例1" width="800" />

dump方法就会把各类event的详细信息都打印出来。我们需要关心的event类型有：DeleteRowsEvent / UpdateRowsEvent / WriteRowsEvent / RotateEvent。

前三个事件分别对应删除操作／更新操作／新增操作。看看UpdateRowsEvent包含了哪些信息：

<img src="/assets/images/using-binlog/illustration-2.png" alt="示例2" title="示例2" width="800" />

我们需要关注的信息有date/log position/table/values。log position表示这个event在binlog文件的offset。

RotateEvent则给出当前使用的binlog的文件名（binlog文件超过指定size时会rotate，这时通过监听RotateEvent就能拿到最新的binlog文件名）：

<img src="/assets/images/using-binlog/illustration-3.png" alt="示例3" title="示例3" width="800" />

通过RotateEvent提供的binlog文件名，和DeleteRowsEvent / UpdateRowsEvent / WriteRowsEvent等Event附带的log position信息，我们就能记录当前已经处理的binlog的偏移。实际上，Mysql Slave就是通过这种方式来做增量更新的，Slave通过将Relay_Master_Log_File和Exec_Master_Log_Pos这两个字段记录在relay-log.info文件来存储同步的进度。

# 和udf+trigger方式的比较 #
使用udf＋trigger的方式的优点是简单，不足主要如下：

* 运维成本高

* 有一定开销

* 难以监控trigger的成功率

使用binlog的难点主要是解析比较麻烦，不过现有的各种库很好的帮我们处理了。优点主要如下：

* 运维无成本

* 对mysql没有什么开销

由于binlog是全库的log，如果需求是监听一两张表的数据更新，建议采用trigger＋udf的方式。而如果监听的表数量较多，那么建议使用binlog的方式。

