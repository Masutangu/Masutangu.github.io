---
layout: post
date: 2015-11-11T16:07:58+08:00
title: NSQ源码解读 NSQD篇
category: 源码阅读
---

接触go语言也有几个月了，从入门后一直用go写些小项目，语法和编程思维已经比较熟悉了，但是感觉很难再进阶一级，因此决定来读一读优秀的go项目源码。这篇文章名叫“解读”，其实有点言过其实。这里我主要是贴出我梳理的NSQ源码的代码架构，给出流程图，并提取其中比较精妙的代码来分析学习，算是比较粗糙。

# 为什么选择NSQ #
* 知乎上不少人推荐
* 对消息队列挺感兴趣，想了解其实现。
# 关于NSQ #
官网介绍在此：http://nsq.io/overview/quick_start.html 简单复制粘贴下NSQ包含的几个模块：
* nsqd is the daemon that receives, queues, and delivers messages to clients.
* nsqlookupd is the daemon that manages topology information. Clients query 
*  nsqlookupd to discover nsqdproducers for a specific topic and nsqd nodes broadcasts topic and channel information.
nsqadmin is a Web UI to view aggregated cluster stats in realtime and perform various administrative tasks.

阅读这篇文章前建议先看下NSQ的文档，需要对NSQ的基本概念有粗浅的了解。
# NSQD代码架构 #
我主要是读了nsq中nsqd这个模块。下图是我总结的代码架构图：
<img src="/assets/images/read-nsq-source-code/illustration-1.png" alt="图例1" title="图例1" width="800" />

下面针对上图的各个流程贴下相应的代码，方便读者查阅。
nsqd启动时开启tcp和http服务：
<img src="/assets/images/read-nsq-source-code/illustration-2.png" alt="nsqd.go: Main()" title="nsqd.go: Main()" width="800" />
tcp接收到client的请求后，创建protocol实例并调用其IOLoop()方法：
<img src="/assets/images/read-nsq-source-code/illustration-3.png" alt="tcp.go: Handle()" title="tcp.go: Handle()" width="800" />
protocol的IOLoop接收client的请求，根据命令的不同做相应处理：
<img src="/assets/images/read-nsq-source-code/illustration-4.png" alt="protocol_v2.go: IOLoop()" title="protocol_v2.go: IOLoop()" width="800" />
同时IOLoop会起一个goroutine运行messagePump()，该函数从该client订阅的channel的clientMsgChan中读取消息并发送给client：
<img src="/assets/images/read-nsq-source-code/illustration-5.png" alt="protocol_v2.go: messagePump()" title="protocol_v2.go: messagePump()" width="800" />
上面的代码看到protocol是从channel的clientMsgChan中读取消息的，那么clientMsgChan的消息是从哪里来的呢？我们来看看protocol/topic/channel的关系图，看看一条消息是如何流转的：
<img src="/assets/images/read-nsq-source-code/illustration-6.png" alt="示例6" title="示例6" width="800" />
从提交消息开始，可以通过http或者tcp的方式往一个topic发送一条消息。先看看http的方式。
注册回调函数doPUB：
<img src="/assets/images/read-nsq-source-code/illustration-7.png" alt="http.go: newHTTPServer()" title="http.go: newHTTPServer()" width="800" />
看看doPUB函数的关键代码，查询到相应的topic，创建Message实例，调用topic的PutMessage方法将该消息写入topic：
<img src="/assets/images/read-nsq-source-code/illustration-8.png" alt="http.go: doPUB()" title="http.go: doPUB()" width="800" />
再看看tcp的方式，上面提到protocol的IOLoop会根据client的不同请求做相应处理，Exec方法判断请求的参数，调用不同的方法：
<img src="/assets/images/read-nsq-source-code/illustration-9.png" alt="protocol_v2.go: Exec()" title="protocol_v2.go: Exec()" width="800" />
看看PUB()的实现，类似的，查询相应的topic，创建Message实例，调用topic的PutMessage方法将该消息写入topic：
<img src="/assets/images/read-nsq-source-code/illustration-10.png" alt="protocol_v2.go: PUB()" title="protocol_v2.go: PUB()" width="800" />
<img src="/assets/images/read-nsq-source-code/illustration-11.png" alt="protocol_v2.go: PUB()" title="protocol_v2.go: PUB()" width="800" />
topic的PutMessage函数调用了put函数，put函数将消息写入memoryMsgChan，如果memoryMsgChan写满了则写到backend队列中：
<img src="/assets/images/read-nsq-source-code/illustration-12.png" alt="topic.go: PutMessage()" title="topic.go: PutMessage()" width="800" />
<img src="/assets/images/read-nsq-source-code/illustration-13.png" alt="topic.go: PutMessage()" title="topic.go: PutMessage()" width="800" />

topic的messagePump函数会不断从memoryMsgChan/backend队列中读消息，并将消息每个复制一遍，发送给topic下的所有channel（因为channel会修改消息里的字段，因此需要将每个消息都复制一遍）：
<img src="/assets/images/read-nsq-source-code/illustration-14.png" alt="topic.go: messagePump()" title="topic.go: messagePump()" width="800" />
<img src="/assets/images/read-nsq-source-code/illustration-15.png" alt="topic.go: messagePump()" title="topic.go: messagePump()" width="800" />
channel也非常类似，PutMessage方法调用put方法，把消息写入memoryMsgChan，如果memoryMsgChan写满了则写到backend队列中：
<img src="/assets/images/read-nsq-source-code/illustration-16.png" alt="channel.go: PutMessage()" title="channel.go: PutMessage()" width="800" />

channel的messagePump函数会不断从memoryMsgChan／backend队列中读消息，并把消息发送到clientMsgChan中：
<img src="/assets/images/read-nsq-source-code/illustration-17.png" alt="channel.go: messagePump()" title="channel.go: messagePump()" width="800" />
最后，protocol的messagePump方法会从client订阅的topic下的所有channel中的clientMsgChan中读取消息，并发送给client（详见上图）。这就是整个消息流转的流程。

# NSQD细节代码分享 #
1. channel和select  
channel和select的结合使用非常精妙。例如下面这段代码：
<img src="/assets/images/read-nsq-source-code/illustration-18.png" alt="channel.go: put()" title="channel.go: put()" width="800" />
NSQ Channel类包含了MemoryQueue和BackendQueue。在往NSQ Channel塞数据时（调用put()方法），首先先写MemoryQueue，如果MemoryQueue写满了，则先写到BackendQueue里缓存起来。上面这段代码用select非常简洁的实现里这个功能。select的case1尝试往memoryMsgChan里写数据，如果memoryMsgChan已经写满，这时会进入到default代码端，将msg写到BackendQueue里（writeMessageToBackend()方法）。
下面这段代码也非常巧妙，go在select的时候会自动跳过channel为nil的case。由于clientMsgChan有可能在另一个goroutine里被close掉。因此在检测到clientMsgChan已经关闭时（ok为false），将clientMsgChan设为nil。下次进入select语句时，case _, ok := <-clientMsgChan这个分支将被跳过。
<img src="/assets/images/read-nsq-source-code/illustration-19.png" alt="channel.go: Empty()" title="channel.go: Empty()" width="800" />
对于channel的处理，为什么clientMsgChan使用range，而memoryMsgChan使用select？区别是？
<img src="/assets/images/read-nsq-source-code/illustration-20.png" alt="channel.go: flush()" title="channel.go: flush()" width="800" />
由于在退出时，clientMsgChan会被close。当channel已经关闭时，for…range循环会自动结束。而对于select来说，“A closed channel is always available for read so its case in select is always readable.”。因此在处理memoryMsgChan时，需要加上default分支，因为即使memoryMsgChan被关闭了，select还是会阻塞，除非把该channel设为nil。

2. 解读DiskQueue  
DiskQueue类里有两个变量：readPos和nextReadPos。看起来有点重复，实际上表示了不同的含义。readPos记录当前readFileNum指向的文件已经读取并发送出去的文件位置。nextReadPos则记录当前readFileNum指向的文件已经读取但是还没发送出去的文件位置。瞧一瞧下面这段代码：
<img src="/assets/images/read-nsq-source-code/illustration-21.png" alt="diskqueue.go: ioLoop()" title="diskqueue.go: ioLoop()" width="800" />
首先注意到，当nextReadPos等于readPos的时候，才会调用readOne()读取一块数据。这是因为只有当nextReadPos（已发送的文件位置）与readPos（已读取未发送的文件位置）相等，才表示之前读取的数据都已经发送出去了。当从readOne()读取一块新数据并发送出去的时候（r<-dataRead)，调用MoveForward()函数，我们看看MoveForward()做了什么事情：
<img src="/assets/images/read-nsq-source-code/illustration-22.png" alt="diskqueue.go: moveForward()" title="diskqueue.go: moveForward()" width="800" />
很简单，他把readPos设置为nextReadPos的值。这样下次进入循环的时候，nextReadPos就会等于readPos，也就会再次调用readOne()读取一块数据。

3. GC优化  
<img src="/assets/images/read-nsq-source-code/illustration-23.png" alt="protocol_v2.go: IOLoop()" title="protocol_v2.go: IOLoop()" width="800" />
使用ReadSlice的原因如下：
To reduce socket IO syscalls, client net.Conn are wrapped with bufio.Reader and bufio.Writer. The Readerexposes ReadSlice(), which reuses its internal buffer. This nearly eliminates allocations while reading off the socket, greatly reducing GC pressure. This is possible because the data associated with most commands does not escape (in the edge cases where this is not true, the data is explicitly copied).

4. Goroutine管理  
NSQ的官方文档提到了管理goroutine的一些方法。通常会用一个exitchan来向goroutine发送退出信号，如有必要会使用sync.WaitGroup来等待goroutine的退出。
另外对于NSQ来说，处理退出时的信息同步非常重要，官网给出的建议如下：
	- Ideally the goroutine responsible for sending on a go-chan should also be responsible for closing it.
	- If messages cannot be lost, ensure that pertinent go-chans are emptied (especially unbuffered ones!) to guarantee senders can make progress.
	- Alternatively, if a message is no longer relevant, sends on a single go-chan should be converted to a select with the addition of an exit signal (as discussed above) to guarantee progress.
	-	The general order should be:
		-	Stop accepting new connections (close listeners)
		-	Signal exit to child goroutines (see above)
		-	Wait on WaitGroup for goroutine exit (see above)
		-	Recover buffered data-
		-	Flush anything left to disk
	
	第一点很浅显易懂，一般是由生产者来关闭go-chan。第二点没能理解，在源码中我找不到例子来解释。第三点直接看代码：
<img src="/assets/images/read-nsq-source-code/illustration-24.png" alt="channel.go: messagePump()" title="channel.go: messagePump()" width="800" />
第四点NSQ总结了一个通用的顺序，我们先拿topic的代码来解释下。
步骤1：［第319行］发送退出信号给child goroutine（topic没有维护和client的连接，因此无需关闭connection）
步骤2：［第322行］等待child goroutine退出
步骤3：［第347行］将数据写到缓冲区（backend）
步骤4：［第348行］将缓冲区（backend）的数据写到磁盘
<img src="/assets/images/read-nsq-source-code/illustration-25.png" alt="topic.go: exit()" title="topic.go: exit()" width="800" />
再看看channel的例子。
步骤1：［第173行］关闭client的连接
步骤2：［第177行］发送退出信号给child goroutine
步骤3：［第186行］将数据写到缓冲区（backend）
步骤4：［第187行］将缓冲区（backend）的数据写到磁盘
<img src="/assets/images/read-nsq-source-code/illustration-26.png" alt="channel.go: exit()" title="channel.go: exit()" width="800" />
对比了channel和topic，会发现channel的exit()并没有等待child goroutine退出。这里为什么topic需要等待child goroutine结束呢？我猜测是因为child goroutinue有可能会写消息到topic下的channel，而可以看到在topic的exit()函数有关闭channel的操作，因此需要确保child goroutinue退出，不再有消息写入channel后，才执行关闭channel的操作。
# 心得体会 #
这次读NSQ的代码，最大的困难在于之前我对NSQ一点都不了解，没有使用过的经验。读源码之前看过官方的文档，但是文档的说明其实比较少。另外国内也几乎没有相关的资料可以参考。所以一开始读起来有点吃力。
尝试过自顶（nsqd，protocol类）向下（topic，channel类）的顺序来读，也尝试过从自底向上的顺序来读。最后我个人感觉阅读的顺序应该是先从底部（topic，channel类）读起，这部分需要读的细致些。再往上一层层读（protocol，tcp，http，最后到nsqd），梳理好整个的架构。最后再自顶向下一层层画出架构图。
另外一个非常大的感受是，要把自己的所得写成文章真是一件很不容易的事情。尤其是梳理架构／流程这一部分。想把自己理解的东西通俗易懂的表达出来是一门大学问，希望读者多多提出意见和批评，好让我不断学习和进步。
结束语
nsqd主要的精华在于golang的channel，读完能够更深入的掌握channel的用法和使用时机。之后如果有时间，我会继续阅读nsqlookup这个模块的代码，感兴趣的同学到时再一起探讨吧^_^

