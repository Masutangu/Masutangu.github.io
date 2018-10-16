---
layout: post
date: 2016-02-29T16:07:58+08:00
title: UDF + Trigger 实时监控 Mysql 数据更新
tags: 数据库
---

最近在做缓存相关的事情，需要在mysql的上层架一层缓存，以缓解mysql的压力，简单的架构图如下：

<img src="/assets/images/using-udf-plus-trigger/illustration-1.png" alt="示例1" title="示例1" width="800" />

大家都知道，缓存带来性能上的提高，然而却有数据不一致的可能。比方说修改了mysql的数据，但是用户读取到的缓存数据还未更新，这时就会有不一致的问题。

这样就需要一种机制来监控mysql中数据的变化以更新缓存：
<img src="/assets/images/using-udf-plus-trigger/illustration-2.png" alt="示例2" title="示例2" width="800" />

# 方案 #
有两种办法可以实时监控mysql，一是利用mysql的binlog，二是利用mysql的trigger。这篇文章主要介绍trigger的方式。

trigger可以在特定事件发生时触发指定的操作，因此可以用trigger来监听insert/update/delete操作。再利用udf，我们就能将这些事件通知到同步server，再由同步server更新缓存中已经过时的数据。

> 关于udf的介绍：
> http://dev.mysql.com/doc/refman/5.7/en/adding-udf.html

> 如何编写udf：http://blog.loftdigital.com/blog/how-to-write-mysql-functions-in-c

这里我使用了mysql-udf-http，它是开源的UDF，提供了利用HTTP协议进行REST操作的能力。

安装完成后，创建trigger如下：

<img src="/assets/images/using-udf-plus-trigger/illustration-3.png" alt="示例3" title="示例3" width="800" />

创建trigger成功后，如果在posts表有insert操作，就会发一个http put请求到指定ip和端口上；如果有delete操作，就会发送一个http delete请求到指定的ip和端口上。

这样我们只需要搭建一个http同步server，等待接收触发trigger调用udf发送的http请求，并更新相应的缓存就可以了。

如果你担心trigger是比较昂贵的操作，你可以在mysql的从库上创建trigger，该从库不对外服务，只用于监控数据更新。

# 不足 #
使用trigger＋udf虽然方便，不过需要手动为监听的每张表创建trigger。另外处理trigger事件的同步server只能使用单进程模型，不然无法保证同步的顺序。但是单进程的效率又太低了。

# 一点联想 #
实现监控mysql数据更新这个功能的过程中，我对设计数据库表结构也有一点启发：在设计表的时候，需要做到**动静分离**。把静态的（不常更新的）数据放在一张表，而把动态的（经常更新的）数据放在另外的表。

比如说：我们把专辑的信息放一张表，把专辑的订阅数／粉丝数／播放量放另外的表。这样我们只需要监控专辑信息表，及时更新缓存就可以了。

相反的，如果我们把订阅数等和专辑信息表放在一起，由于订阅数经常会变，我们要么得写一个复杂的trigger监听某几个字段的更新（不清楚能否实现），要么需要在同步server做逻辑判断，非常的繁琐。




