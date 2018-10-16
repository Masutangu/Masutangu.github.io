---
layout: post
date: 2016-01-12T16:07:58+08:00
title: 设计模式之控制反转
tags: 设计模式
---

今天重新设计了[Elric](https://github.com/Masutangu/Elric)的logging模块，接触到了Inversion of Control设计模式，记录下分享给大家。*注：本文定义介绍均取自wiki，代码样例取自[Elric](https://github.com/Masutangu/Elric)*

# 定义 #
**控制反转**（Inversion of Control，缩写为IoC），是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。”

# 实现方法 #
实现控制反转主要有两种方式：**依赖注入**和**依赖查找**。

两者的区别在于，前者是被动的接收对象，在类A的实例创建过程中即创建了依赖的B对象，通过类型或名称来判断将不同的对象注入到不同的属性中，而后者是主动索取响应名称的对象，获得依赖对象的时间也可以在代码中自由控制。

# Demo #
这里用一个简单的例子来解释依赖注入的实现方式。以Elric为例，其包括了Master, Worker, Executor, JobQueue, JobStore等好几个模块。而这几个模块有相互依赖，比如说JobStore作为Master的一个成员变量，JobStore依赖到Master创建的logger对象来打印log。

一开始我的做法是，将Master的logger对象传给JobStore：
<img src="/assets/images/inversion-of-control/illustration-1.png" alt="示例1" title="示例1" width="800" />

JobStore就可以将传进来的logger对象保存到自己的成员变量self.log中，之后就可以通过该成员变量打印log了。
<img src="/assets/images/inversion-of-control/illustration-2.png" alt="示例2" title="示例2" width="800" />

那如果JobStore还需要依赖到Master其他的成员变量呢？也要一个一个通过构造函数传进来吗？又或者说JobStore还依赖Master的成员方法，这时应该怎么处理？有没有更好的方案呢？

当然有，很简单，把整个Master对象传进来就好啦！
<img src="/assets/images/inversion-of-control/illustration-3.png" alt="示例3" title="示例3" width="800" />

相应的修改JobStore的构造函数：

<img src="/assets/images/inversion-of-control/illustration-4.png" alt="示例4" title="示例4" width="800" />

Done！之后JobStore如果需要用到Master的其他成员变量或者方法，就可以通过self.context来调用啦！

# 总结 #
在之前看nsq的代码，就看到过这种把A对象传到B对象的构造函数的做法，当初并不了解其中缘由。今天重新设计了Elric的logging模块，使用到IoC时，才恍然大悟，也算是有所收获～

