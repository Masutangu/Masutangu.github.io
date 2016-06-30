---
layout: post
date: 2015-08-02T16:07:58+08:00
title: 600行代码实现简单的任务分发监控器
category: 个人项目
---

写之前说些题外话，今晚认识了一位大神，聊了不到两个小时，被虐得体无完肤。觉得自己还有很长的路要走，现在做的东西还是比较小儿科。虽然这篇文章的内容比较低端，但是大神说了，要坚持写blog。所以我还是决定厚颜无耻的发出来，当作积累。

很久很久之前Go写的一个任务分发和监控管理系统，最近重构了下代码，终于稍微能看。当然这只是1.0版本，还是很低端。同时作为我的Go处女作品，还没时间去读读其他优秀的Go代码，代码风格可能不太规范。项目地址请戳：[SuperScripter](https://github.com/Masutangu/SuperScripter)。

# 背景 #
说说背景。最近在做一个模块，需要根据机器负载下发一些任务，并且监控任务的执行。如果任务执行失败需要及时拉起重跑。搜来搜去没找到合适的工具来使用。（见识浅薄，如果大家有好用的工具欢迎推荐～）于是决定自己撸一个。为什么选Go？因为对python处理进程这块不太熟悉，而且刚好开始接触Go，正好拿来练练手。

# 简单架构 #
选定了语言后，开始设计模块要怎么划分，代码要怎么组织。根据应用场景出发，画了个简单的模块图：

<img src="/assets/images/script-monitor-writen-in-600-lines-code/superscripter-version-1.png" alt="SuperScripter架构图版本1" title="SuperScripter架构图版本1" width="800" />

Worker通过控制信息通道定时发送心跳包，上报机器的状态／任务的执行情况等信息到Master。Master根据各个Worker上报的数据，分发任务到最合适（空闲）的Worker，同时重新调度Worker执行失败的任务。

因为要监控机器的状态，包括CPU／内存等指标，因此在Worker中细分了一个Monitor模块来做监控。另外任务的信息和分发的记录需要落地以便查询，所以在Master里也抽离了一个JobStore模块。

<img src="/assets/images/script-monitor-writen-in-600-lines-code/superscripter-version-2.png" alt="SuperScripter架构图版本2" title="SuperScripter架构图版本2" width="800" />

对应到我代码的目录结构：

<img src="/assets/images/script-monitor-writen-in-600-lines-code/superscripter-directory.png" alt="SuperScripter目录结构" title="SuperScripter目录结构" width="800" />

任务分发通道对应到jobqueue，实现上是用的redis。而控制信息通道采用的是rpc。

# 语言思想 #
其实这个项目和我之前写的Elric挺相似的。刚好可以对比下python和go语言在实现类似功能时思路上有什么不同。

Elric里用到了多线程。访问共享数据的时候加了线程锁，并且用了信号量来维持不同线程的同步问题。

而SuperScripter同样也有不同协程访问共享数据的场景。Go语言提供了什么样的机制做协程间的同步的呢？来看看下面这两段代码：

<img src="/assets/images/script-monitor-writen-in-600-lines-code/go-concurrence-way-1.png" alt="go处理协程同步1" title="go处理协程同步1" width="800" />

<img src="/assets/images/script-monitor-writen-in-600-lines-code/go-concurrence-way-2.png" alt="go处理协程同步2" title="go处理协程同步2" width="800" />
第86行代码起了一个goroutine来处理Worker传来的心跳包。然后开始分发各种任务。在分发任务的时候会读取Master维持的各个Worker状态列表。而在收到Worker的心跳包时会更新Worker的状态。显然，分发任务和处理心跳包会访问同一块数据。这里使用Go语言提供的Channel来确保不同goroutine之间的同步。下面通过对比流程图方便大家理解：
<img src="/assets/images/script-monitor-writen-in-600-lines-code/python-way.png" alt="pyhon实现同步" title="pyhon实现同步" width="800" />
传统的做法是为共享资源加锁来实现不同线程／进程的同步。
<img src="/assets/images/script-monitor-writen-in-600-lines-code/go-way.png" alt="go实现同步" title="go实现同步" width="800" />

而Go可以通过Channel来实现协程之间的同步。一句话总结：“ **Don’t communicate by sharing memory. sharing memory by communicating**”

# 一些细节 #
这个项目整体的思路其实很简单，但是有不少细节比较难处理。例如心跳包发送频率？一次下发任务的数量如何控制？ Worker自身状态和Master维持的状态的一致性？任务失败重试频率的控制？这个版本还处理得很粗糙。理想中的应该是：

* 心跳包发送频繁，一台Worker每轮只下发一个任务。
* 验证Worker上报的状态与Master保存的状态的一致性（例如Worker上报正在执行的任务与Master的下发记录是否一一对应），如有不一致应有相应的对策。
* 任务失败重试的等待时间应该指数增长。
* 心跳包的channel修改为buffer channel，不然一次只能处理一台Worker的心跳，效率大打折扣。

新版本规划 下个版本首先要解决上面提到的三点。另外还有下面几点规划：

* 升级控制信息通道和任务分发通道的具体实现。当前使用redis来分发任务还是略挫了些，另外Go的RPC我觉得相当的难用。可能会考虑选用消息队列来实现，把两个通道揉在一起。
* Master的高可用性，如何做到主备数据的一致性及无缝切换。
* 重新设计任务存储的数据结构，看看有没更合适的数据库。或者干脆不落地，参考redis的持久化机制。
* Web管理界面。如果数据没落地，还要新增些接口。

敬请期待！

