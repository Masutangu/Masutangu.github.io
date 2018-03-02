---
layout: post
date: 2018-02-26T15:32:36+08:00
title: 负载均衡
category: 工作
---

负载均衡的目标：提供最短的平均任务响应时间，能适应变化的负载，是可靠的。

负载均衡算法包含下面三个组成部分：

* 信息策略：
* 传送策略：
* 放置策略：

信息策略：
* 运行队列中的任务数
* 系统调用的频率
* CPU 上下文切换率
* 空闲 CPU 时间百分比
* 空闲存储器的大小
* 1分钟内的平均负载

放置策略：
... https://www.ibm.com/developerworks/cn/linux/cluster/linux_cluster/part9/index.html

算法实现：中心任务调度策略、梯度模型策略、发送者启动策略和接受者启动策略。



如果为了使系统信息更全面而采集了更多的参数，则往往增加了额外的开销，却得不到性能改善。

机器cpu 内存 网络流量 转换为 延时 + 业务成功率等指标，计算成 0-100 的数值。

过载保护：统计单位时间内请求的延时、成功率。

w(k) = delay_laod(k) * ok_load(k)
delay_load(k)  = delay(k) / min(delay(0), ... , delay(n))
ok_load(k) = max(ok_rate(0), ..., ok_rate(n)) / ok_rate(k)

心跳只表示进程正常

微服务

proxy 通用