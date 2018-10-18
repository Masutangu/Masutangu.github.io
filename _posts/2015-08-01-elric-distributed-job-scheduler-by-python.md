---
layout: post
date: 2015-08-01T16:07:58+08:00
title: Elric：Python 实现的分布式任务调度系统
tags: 
  - 个人项目
  - Python
---

五月中的时候，我花了两周的业余时间，用python写了第一个github项目：[Elric](https://github.com/Masutangu/Elric)。Elric是一个简单的分布式任务调度系统。这里想分享下写这个项目的缘由，思路以及Elric架构演变。

# 名字来源 #
<img src="/assets/images/elric-distributed-job-scheduler-by-python/full_metal_alchemist.jpeg" alt="钢之炼金术师" title="钢之炼金术师" width="800" />

这个项目的名字Elric，取自我非常喜欢的动漫：钢之炼金术师 里面主角的姓。强烈建议没看过的朋友一定要看看，这部动漫给我带来很大的影响。

# 背景 #
 说起爬虫，相信很多人都会第一时间提起Scrapy。我第一次写爬虫的时候，也是用了Scrapy。但是Scrapy是单机部署，如果要抓取的网站比较大，只靠一台机器来抓取得花上比较长的时间。刚好之前工作上有个任务是要抓取一批网站，并且需要做到抓取效率快（例如每十五分钟抓取一次，更新记录）。于是我开始寻找分布式爬虫的方案。

Google一把，发现已经有人基于Redis实现了一个scrapy插件：scrapy-redis。研究了一把，发现它其实是一个伪分布式的scrapy插件。只是在抓取的起始url采用redis分发一下，后续的页面解析出来的url还是由单机的scrapy进行抓取。当时由于时间紧迫，我就直接在这个插件上进行修改，新增了接口把页面解析得到的url塞回redis，通过redis再分发给不同机器上的scrapy程序抓取，算是完成了任务。

# 思考 #
 虽然完成了任务。但是这并不是完美的解决方案。修改接口带来的后果是，同一个url有两种抛出的形式，一种是塞回redis，一种是直接交给自己的scheduler调度，自己处理（scrapy的原始模式）。这里在代码里必须显示指定要采用那种方式处理url，写起来就会比较凌乱。并且现在抓取过的url就有了两个存储的地方，一个是我新增的，一个是scrapy本身实现的。管理起来也非常不便。于是我寻思着，要不自己造个轮子（刚好年初给自己定的kpi是写一个开源项目，一直苦于不知道写什么）？其实分布式爬虫的思想在上面已经提到了：把解析到的url放到一个公共的地方做去重逻辑，再统一分发给不同的worker就可以了。

# 架构 #
 既然有了目标，那就开始动手。第一步把这个目标做分解，要实现一个分布式爬虫，本质上需要的是一个分布式任务调度器，Master-Worker架构。master负责分发任务给worker，worker负责处理和提交任务。
 <img src="/assets/images/elric-distributed-job-scheduler-by-python/elric_version_1.png" alt="Elric架构Version 1" title="Elric架构Version 1" width="800" />

 从上面的架构图可以看出，Elric有四个基本的模块：Master，Worker，Message Queue和RPC。Master的职责在于接受worker提交的任务，对任务做逻辑处理（去重等），并把任务push到Message Queue中。Worker则通过从Message Queue中pull任务回来处理，并把处理任务过程中产生的新任务（例如爬虫中解析一个页面产生的子链接）重新提交给Master。Messsage Queue主要负责存放序列化后的任务，而RPC主要用与Master和Worker之间的通信。

简单的架构搭起来后，接下来的问题是：Worker如何并行处理任务？为了效率最大化，肯定需要一个进程池／线程池负责处理任务，再另开一个进程／线程从Message Queue中拉取新任务。

<img src="/assets/images/elric-distributed-job-scheduler-by-python/elric_version_2.png" alt="Elric架构Version 2" title="Elric架构Version 2" width="800" />
这里我使用了Python的 [concurrent.futures](https://docs.python.org/3.2/library/concurrent.futures.html#module-concurrent.futures)库提供的进程池用于执行Worker的任务。Worker每次从Message Queue中获取到新任务，就直接丢给进程池进行处理，避免任务执行时被阻塞。

好了，接下来，我们希望有些任务可以定时执行。例如我要每十五分钟取抓取某网站，我可不想写个crontab每隔十五分钟向Master提交任务。我希望能把任务的周期信息也记录在任务里。刚好之前做另外一个需求的时候接触了一个Python实现的定时任务的库[Apscheduler](https://apscheduler.readthedocs.org/en/latest/)，并且当初由于好奇，看过了一遍它的源码。因此打算参照它的代码，把这个能力集成到Elric中。
<img src="/assets/images/elric-distributed-job-scheduler-by-python/elric_version_3.png" alt="Elric架构Version 3" title="Elric架构Version 3" width="800" />
Apscheduler的原理很简单。文字描述起来比较麻烦，伪代码如下：
<pre><code>
while( True) {
    next_run_time = getNextJobRunTime() //拿到所有任务下次执行的时间
    min_wait_time = now()-max(next_run_time) //计算出等待的最短时间
    wait(min_wait_time) //sleep一把，避免cpu空转
	job = getJobFromJobStore(now()) //取出需要执行的任务
	push_job_to_message_queue(job) //将任务塞入Message Queue中
｝
</code></pre>

搞定，这样Elric不仅能处理即时任务，也能处理周期任务。越来越强大了^ ^

最后，我们要为爬虫定制一个特性：去重。我们希望能够指定哪些url只需要抓取一次（不会更新的页面）。这样能大大节省我们机器的开销。去重其实非常简单，最简单的做法就是用一个集合保存需要去重的任务，Master在接收到Worker提交的任务后，首先判断这个任务是否需要去重，如果需要的话，判断该任务是否已经存在于集合中，如果已经存在，则直接丢弃。下图为最终的架构图:

<img src="/assets/images/elric-distributed-job-scheduler-by-python/elric_version_4.png" alt="Elric架构Version 3" title="Elric架构Version 3" width="800" />

Done! 一个非常简单的分布式任务调度器就完成了。

# 结束语 #
 实现了Elric之后，我马上把之前的爬虫代码迁移到这上面，现在的接口清晰了很多，代码写起来也非常的清爽，感觉很Cool。在经过自己测试了没有问题之后，我开始鼓动同事也采用Elric来写爬虫，借此机会来发现代码中的不足和缺陷（例如其中有一个接口比较难理解，我打算在下次update 的时候改掉）。

总的来说，这是我第一次尝试自己写一个工具，收获非常大。如果你有什么建议或者困惑，欢迎发邮件给我。如果喜欢这个项目，欢迎给我一颗Star鼓励鼓励^ ^

后续阅读：

[Elric Change Log 1](http://masutangu.com/2015/10/elric-change-log-1/)

[Elric Change Log 2](http://masutangu.com/2016/02/elric-change-log-2/)
