---
layout: post
date: 2015-08-02T16:07:58+08:00
title: 谈谈我对 Golang 的理解
tags: 
  - 编程语言
  - Golang
---

谈谈我对Golang的理解 一直想了解下go这门新的语言。前段时间看了Google IO上Rob Pike对Go语言的介绍，又过了下[A Tour of Go](https://tour.golang.org/welcome/1)。然后趁热打铁花了两天时间写了个比较有趣的脚本管理器[Scripter](https://github.com/Masutangu/SuperScripter)。当然对Go语言的语法规范和代码组织还不了解，所以这代码写的有些凌乱。

<img src="/assets/images/talk-about-golang/golang.jpeg" alt="golang吉祥物" title="golang吉祥物" width="800" />

回到主题，从开始看Google IO，到用Go写完了Scripter，加起来花了差不多五天。不是我天赋过人，而是Go的语法实在是太简洁了。简单来说，Go是一门写起来像动态语言，有着动态语言开发效率的静态语言。

听起来很爽，赶紧学起来。今天就给大家介绍下Go语言，希望通过这篇文章能帮助大家快速掌握Go语言的特点，并用它写出各种有趣的程序。（注：文章举例／说明都是参考Google IO上的材料，因为我也是从那里了解的^ ^）

# Go语言的起源 #
Go语言是系统级编程语言。当初设计的初衷是用于开发Web Server。后来发布之后，发现它不仅仅适用于开发Web Server，还很适合来写各种脚本程序。

Go语言特点:
* Go是object-oriented语言，不是type-oriented。（不理解type-oriented的概念）

* Go虽然是object-oriented，但继承并不是它的主要特性。不象C++和JAVA有父类／子类的概念。事实上，Go里面并没有明确的类的概念。

* Go中变量的类型由编译器自动推导（implicit），无需显式指定（explicit）。最重要的一点，Go中的类型通过实现接口(Interface)定义的方法来实现接口，没有显式声明的必要（对比JAVA需要显式声明该类型Implements该接口）。

* Go语言是concurrent，并非parallel。关于concurrent和parallel的区别，请看这篇文章Concurrency is not parallelism。Rob Pike在Google IO上提到这两者的区别：

	- Concurrency is the composition of independently executing computations.

	- Concurrency is a way to structure program.

	- Parallelism is about performance. Concurrency is about program design.

我的理解是Concurrency是组织程序的一种方式，而Parallelism则是同一时间并行执行程序。Concurrency指的是程序的structure，Parallelism则是程序的execution。

**注**：关于Concurrency和Parallelism的区别，[这篇文章](https://blog.golang.org/concurrency-is-not-parallelism)解释得更为清楚：并发是指程序的逻辑结构。非并发的程序就是一根竹竿捅到底，只有一个逻辑控制流，也就是顺序执行的(Sequential)程序，在任何时刻，程序只会处在这个逻辑控制流的某个位置。而如果某个程序有多个独立的逻辑控制流，也就是可以同时处理(deal)多件事情，我们就说这个程序是并发的。这里的“同时”，并不一定要是真正在时钟的某一时刻(那是运行状态而不是逻辑结构)，而是指：如果把这些逻辑控制流画成时序流程图，它们在时间线上是可以重叠的。

并行是指程序的运行状态。如果一个程序在某一时刻被多个CPU流水线同时进行处理，那么我们就说这个程序是以并行的形式在运行。（严格意义上讲，我们不能说某程序是“并行”的，因为“并行”不是描述程序本身，而是描述程序的运行状态，但这篇小文里就不那么咬文嚼字，以下说到“并行”的时候，就是指代“以并行的形式运行”）显然，并行一定是需要硬件支持的。）

# Interface #
 上一节提到了Go语言两个与众不同的特点：Interface和Concurrency。接下来我们就来谈谈这两个特性。

首先是Go语言的Interface，我的第一感觉是：清爽，简洁。Go编程规范推荐每个Interface只提供一到两个的方法。这样使得每个接口的目的非常清晰。另外Go的隐式推导也使得我们组织程序架构的时候更加灵活。在写JAVA／C++程序的时候，我们一开始就需要把父类／子类／接口设计好，因为一旦后面有变更，修改起来会非常痛苦。而Go不一样，当你在实现的过程中发现某些方法可以抽象成接口的时候，你直接定义好这个接口就OK了，其他代码不需要做任何修改，编译器的自动推导会帮你做好一切。

看起来有点绕？下面给出一个例子（参考自Google IO）：

假设我们要实现zlib压缩。以下是JAVA的代码：
<img src="/assets/images/talk-about-golang/golang-example-1.png" alt="golang示例1" title="golang示例1" width="800" />

我们定义了一个ZlibCompressor的类，实现了一个compress方法用以做压缩。OK，看起来很简单。但是，如果后来我们希望扩展更多的压缩方式，我们就需要把compress这个接口给抽象出来：
<img src="/assets/images/talk-about-golang/golang-example-2.png" alt="golang示例2" title="golang示例2" width="800" />


这里的改动就非常大了，首先你需要定义一个AbstractCompressor的接口，并且需要修改ZlibCompressor类的声明，显示指定它extends了AbstractCompressor接口。如果说ZlibCompressor是标准库提供的话，那这是不可能做到的（修改不了标准库的代码）。

同样的目的，用Go实现起来就非常简单。同样的，我们先定义一个ZlibCompressor类：
<img src="/assets/images/talk-about-golang/golang-example-3.png" alt="golang示例3" title="golang示例3" width="800" />


后续的扩展非常简单，定义一个Compressor接口，完全不需要修改ZlibCompressor的代码，编译器会自动推导出ZlibCompressor实现了Compressor接口：

<img src="/assets/images/talk-about-golang/golang-example-4.png" alt="golang示例4" title="golang示例4" width="800" />


是不是非常的灵活？通过Go语言提供的能力，我们不需要过早的去思考如何组织我们的代码，而是在实现过程中不断优化整个程序的架构，避免过早设计和过度设计。

# Concurrency #
Go语言的Concurrency中有两个重要的概念：goroutine和channel。

goroutine可以理解为轻量级的线程，拥有自己的调用堆栈。goroutine非常的轻便，一个程序能够同时运行着成千上万个goroutine。

channel用于goroutine之间的通信。我们知道，多进程／线程相互之间的通信是个很头疼的问题，为了解决这个问题，引入了锁，共享内存等机制。但是Go采用了完全不一样的思路。Google IO上Rob Pike用一句话总结Go处理并发的思想：“**Don’t communicate by sharing memory. sharing memory by communicating**”。我觉得非常的到位。Go并不是通过访问共享内存来处理进程间的通信，而是通过Communicate来实现goroutine之间共享的数据。说起来又是有些绕，看看例子吧（参考自Google IO）：
<img src="/assets/images/talk-about-golang/golang-example-5.png" alt="golang示例5" title="golang示例5" width="800" />


这个例子有点类似于生产者／消费者的模型。boring函数的角色是生产者，main函数的角色是消费者。按照我们以前的做法，会申请一个全局变量，生产者往里面丢东西，消费者从里面取东西。生产者和消费者通过信号量来保持同步。

而Go语言的处理方式则简单得多，声明一个channel对象（代码中的c := make(chan string)），生产者往channel对象里面塞数据（c<-fmt.Sprintf(“%s, %d”, msg, i)），消费者往channel对象里面取数据（fmt.Println(<-joe)）。如果channel里面的数据未被消费，此时输入操作会阻塞，如果channel里面暂无数据，此时取出操作会阻塞。使用channel，简洁方便，远离锁／信号量／共享内存带来的麻烦事。是不是非常方便呢？

# 更多有趣的小特性 #
Go语言还提供了许多特性（这个词可能用得不准确），包括switch/defer。这里就不再多说，如有兴趣可以到Go官网看提供的教程。

# 结束语 #
我也是一个Go初学者。Go语言给我的印象非常好。语法上非常简洁，没有多余的规则。写起来非常像Python，开发效率极高。至于性能方面，我还没有相关的经验，这里也不好多加评价。

希望通过这篇文章能帮助大家快速上手Go语言，喜欢上Go语言。如果有什么问题，请随时发邮件给我，一起交流学习。如果想用Go语言开发什么好玩的工具，欢迎和我分享，一起来写Go语言！

