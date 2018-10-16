---
layout: post
date: 2015-10-27T16:07:58+08:00
title: Elric Change Log I
tags: 优化&重构
---

自从把Elric放在线上跑，到现在已经跑来小半年。也发现了些接口设计上的不合理和小坑。今天花时间改进了下，分享下改进的地方。

# 接口简化 #
之前的worker在往某个queue提交任务之前，需要调用add_queue()方法来先在master创建一个queue。同时worker也需要注册自己监听哪些queue，这里worker和master就有两个queue的交互，一条queue用于提交任务，一条queue用于拉取任务来执行。但对于Elric的使用者，并不需要关心这些细节，因此我把add_queue()接口给去掉了：

<img src="/assets/images/elric-change-log-1/change-1.png" alt="代码修改1" title="代码修改1" width="800" />

<img src="/assets/images/elric-change-log-1/change-2.png" alt="代码修改2" title="代码修改2" width="800" />
把worker和master的add_queue接口去掉。在worker提交任务的时候，检查该queue是否存在，如果不存在（抛KeyError）则创建之。

# Bug fix #
之前遇到过奇怪的bug。在获取到执行时间点的任务时，有些任务没有被取出来，代码逻辑如下：
<img src="/assets/images/elric-change-log-1/bug-1.png" alt="bug代码1" title="bug代码1" width="800" />

问题就出在get_due_jobs()上，看下我修改前后的代码：
<img src="/assets/images/elric-change-log-1/bug-fix-1.png" alt="bug fix代码1" title="bug fix代码1" width="800" />

之前采用的是yield，现在改成把到期的任务放到list后再返回整个list。这里的yield是坑，因为yield会记住上次执行的位置，而我们的代码逻辑如下：

<img src="/assets/images/elric-change-log-1/illustration-1.png" alt="样例1" title="样例1" width="800" />

假设当前时间是8点，get_due_jobs()第一次for循环取出job1，并更新了job1的next_run_time为9点（假设job1的intervals为1小时），可以看到job_run_time列表中，job2现在排在第一位，job1排在第二位。而get_due_jobs()的下一次for循环将从job_run_time列表的第二位开始（因为使用了yield，会从代码上次执行的位置后继续执行），因此导致job2没有被取出来。因此这里我把yield的逻辑改成列表。

# 爬虫扩展 #
很早之前我就在Elric的基础上实现了分布式爬虫，只是代码一直没提交。这里也顺带提一下。相信看过这篇[文章](blog/2015/08/01/elricpython实现的分布式任务调度系统/)的同学都知道，Elric最初就是为分布式爬虫而设计的。那在他基础上封装一个爬虫也是非常简单的事情。有兴趣的同学直接看这个文件即可。简单来说就是在原有基础上添加了爬取去重逻辑，master在提交任务前先判断该任务是否已经存在，如果已存在则直接过滤。

# 任务优化 #
之前在任务分发下去之后，master就不管任务成功与否（master也没办法拿到这部分信息）。而我们有时需要对失败的任务做些记录或者处理，因此我给Elric新增了一个rpc接口：finish_job()。任务成功执行后，会向master上报：
<img src="/assets/images/elric-change-log-1/optimization-1.png" alt="优化1" title="优化1" width="800" />

这里如果要任务执行失败也上报也是ok的。至于master的finish_job接口要如何设计，则完全取决于你了。在我扩展实现的爬虫master中，任务执行成功后我会把该任务的过滤信息记录下来，以此实现爬虫的去重逻辑：
<img src="/assets/images/elric-change-log-1/optimization-2.png" alt="优化2" title="优化2" width="800" />

# 总结心得 #
自己写的工具，有机会还是要推动大家去使用。写demo容易，写一个成熟的工具很困难。只有大家去使用自己的工具，发现坑或者发现不合理的地方，才能促进自己去思考优化改进。

