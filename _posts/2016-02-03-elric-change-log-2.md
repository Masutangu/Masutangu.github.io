---
layout: post
date: 2016-02-03T16:07:58+08:00
title: Elric Change Log II
category: 优化&重构
---

最近有些时间，于是对分布式框架Elric做了些优化，同时新增了些新特性，在这里分享给大家。

# 优化Worker拉取任务逻辑 #
之前的逻辑是从任务队列里拉取任务后就塞给进程池，会导致worker不断从任务队列里取任务，然后在进程池里等待执行。这样的话，Worker不是按需取任务，而是揽一大堆活然后一直积压在手里做不完，而后续拉起空闲的Worker则取不到任务。

<img src="/assets/images/elric-change-log-2/illustration-1.png" alt="示例1" title="示例1" width="800" />

之前的逻辑，拉取任务后直接塞给进程池这里我对取任务的逻辑做了优化，使用Queue来统计正在执行中的任务。初始化一个Queue，其最大长度等于Worker的进程池大小。拉取任务前，往Queue里put一个控制符。任务执行完时，从Queue里get一个控制符。当进程池的进程都在执行任务的时候，此时Queue是满的，put操作会阻塞，因此Worker阻塞到Queue有空间的时候（有进程完成任务了，从Queue里get走了一个控制符，也意味着有空闲的进程了），才会到任务队列里拉取任务。

<img src="/assets/images/elric-change-log-2/illustration-2.png" alt="示例2" title="示例2" width="800" />

*拉取任务前，尝试往queue里put一个控制符，如果queue已满（没有空闲进程），则一直阻塞*

<img src="/assets/images/elric-change-log-2/illustration-3.png" alt="示例3" title="示例3" width="800" />

*任务执行完，从queue里get一个控制符*

commit：https://github.com/Masutangu/Elric/commit/e84d359b2082e97f4aa8b400f2b8e1651506fae3

# 限制任务队列的长度 #
之前Master会一直往任务队列里提交任务，并不关心任务队列积压的任务数。这样如果Worker挂掉一段时间后再拉起的时候，就会一直执行积压的过期任务。

于是我希望给Master提供一个控制队列长度的能力。首先给任务队列新增一个接口is_full()，返回True表示任务队列已满。在Master提交任务到任务队列前，先检查任务队列是否已满。如果队列已经满了，则把该任务写到Buffer Queue里，在另外的线程里去做定期重试写入任务队列的逻辑。

这里其实是参考了nsq的做法。只不过我把Python的Queue当成Golang的channel来使用。

<img src="/assets/images/elric-change-log-2/illustration-4.png" alt="示例4" title="示例4" width="800" />
*新增is_full()接口*

<img src="/assets/images/elric-change-log-2/illustration-5.png" alt="示例5" title="示例5" width="800" />
*提交任务前检查下任务队列是否已满，如果是，则写到buffer_queue里*

<img src="/assets/images/elric-change-log-2/illustration-6.png" alt="示例6" title="示例6" width="800" />
*start_process_buffer_job线程处理buffer_queue的任务，定期尝试写入到任务队列里，如果任务队列已满，则再次放回buffer_queue。如果超过一定时间都没有提交成功，则将任务丢弃*

commit：https://github.com/Masutangu/Elric/commit/592e6756725bf2e138d2e1f1de1c9f7d579a4324

# 提供任务存储的mongodb支持 #
之前的任务存储是基于内存，为了使master有更高的可用性，这里我新增了mongodb的支持。

同时为了更好的监控任务的执行情况，需要有存储来记录每个任务执行是否成功，失败则记录Exception的信息。如果是循环任务，则只需要记录近N次的执行结果。

Mongodb的Array类型提供了限制大小的能力，非常符合我仅记录近N次执行结果的需求。

<img src="/assets/images/elric-change-log-2/illustration-7.png" alt="示例7" title="示例7" width="800" />
*slice用与限制array的大小*

commit：https://github.com/Masutangu/Elric/commit/c87fd9c227359ca5ce31f19c2be05154c96a45f0

