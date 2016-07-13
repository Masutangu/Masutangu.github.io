---
layout: post
date: 2016-07-13T19:27:03+08:00
draft: true
title: 使用go-sql-driver出现TIME_WAIT
---

dial tcp xxx.xxx.xxx.xxx:xxxx: cannot assign requested address

netstat 看到很多time_wait 

导致端口不够用，因此新链接没法绑定端口，报 cannot assign requested address 的错误。

解决方法：
设置连接池的上限 SetMaxOpenConns