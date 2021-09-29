---
layout: post
date: 2017-02-05T22:57:39+08:00
title: Linux 性能监控
tags: 
  - 工作
  - Linux
---

本文是对 [《Linux 性能监测》](https://www.vpsee.com/2009/11/linux-system-performance-monitoring-introduction/) 系列文章的读书笔记，并在此基础上丰富。

## CPU 相关
### vmstat 

```
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0 164412 143200  35916 243216    0    0    12    11   46   22  1  1 98  0  0
 0  0 164412 142960  35916 243260    0    0     0     0  583  854  2  1 97  0  0
 0  0 164412 142960  35916 243260    0    0     0     0  940 1184  2  1 97  0  0
```

参数说明：

* r: 	可运行队列的线程数，这些线程都是可运行状态，只不过 CPU 暂时不可用
* b: 	被 blocked 的进程数，正在等待 IO 请求
* in: 	被处理过的中断数
* cs: 	系统上正在做上下文切换的数目
* us: 	用户占用 CPU 的百分比
* sy: 	内核和中断占用 CPU 的百分比
* wa: 	所有可运行的线程被 blocked 以后都在等待 IO，这时候 CPU 空闲的百分比
* id: 	CPU 完全空闲的百分比

合理的参数范围：

* CPU 利用率：如果 CPU 达到 100％ 利用率，那么 us 和 sy 占比应该达到这样一个平衡：65％－70％ User Time，30％－35％ System Time，0％－5％ Idle Time
* 上下文切换：上下文切换应该和 CPU 利用率联系起来看，如果能保持上面的 CPU 利用率平衡，大量的上下文切换是可以接受的
* 可运行队列：每个处理器的可运行队列不应该超过3个线程，比如：双处理器系统的可运行队列里不应该超过6个线程

案例 1：

```
vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  0    140 2915476 341288 3951700  0    0     0     0 1057  523 19 81  0  0  0
 4  0    140 2915724 341296 3951700  0    0     0     0 1048  546 19 81  0  0  0
 4  0    140 2915848 341296 3951700  0    0     0     0 1044  514 18 82  0  0  0
 4  0    140 2915848 341296 3951700  0    0     0    24 1044  564 20 80  0  0  0
 4  0    140 2915848 341296 3951700  0    0     0     0 1060  546 18 82  0  0  0
```

上面的输出可以看出：

* system time（sy）一直保持在 80％ 以上，而且上下文切换较低（cs），说明某个进程可能一直霸占着 CPU
* run queue（r）刚好在4个

案例 2：

```
vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
14  0    140 2904316 341912 3952308  0    0     0   460 1106 9593 36 64  1  0  0
17  0    140 2903492 341912 3951780  0    0     0     0 1037 9614 35 65  1  0  0
20  0    140 2902016 341912 3952000  0    0     0     0 1046 9739 35 64  1  0  0
17  0    140 2903904 341912 3951888  0    0     0    76 1044 9879 37 63  0  0  0
16  0    140 2904580 341912 3952108  0    0     0     0 1055 9808 34 65  1  0  0
```

上面的输出可以看出：

* context switch（cs）比 interrupts（in）要高得多，说明内核不得不来回切换进程
* 进一步观察发现 system time（sy）很高而 user time（us）很低，而且加上高频度的上下文切换（cs），说明正在运行的应用程序调用了大量的系统调用
* run queue（r）在14个线程以上，按照这个测试机器的硬件配置（四核），应该保持在12个以内。

案例 3：

```
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 5  0     92 185724 735036 2736384   0    0     2    11    0    1  2  0 98  0  0
 7  0     92 184980 735044 2736376   0    0     0   124 6682 7806 39  3 58  0  0
 6  0     92 184856 735044 2737064   0    0     0  1888 6721 7601 38  5 49  8  0
 2  0     92 184732 735044 2737064   0    0     0     0 6549 7525 46  4 50  0  0
 3  0     92 183988 735044 2737904   0    0     0     0 6375 7081 45  3 52  0  0
 1  0     92 184360 735048 2738156   0    0     0     0 6764 7601 44  4 51  0  0 
 8  1     92 183368 735048 2738508   0    0     0  1384 6774 8005 42  4 54  0  0
```

cpu 利用率上不去，vmstat 输出如上，分析可能的瓶颈：

* si、so 都是 0，free 也有，说明内存足够，排除内存。
* bi、bo 都不大，所以 io 似乎不严重，排除硬盘
* cs、in 都较大，说明 CPU 频于应付上下文切换和中断，而且从 r 的数字来看正在等 CPU 的进程较多，所以猜测服务器上运行的进程较多，CPU 疲于切换进程以及应付中断，猜测瓶颈是 CPU。

### mpstat
mpstat 和 vmstat 类似，不同的是 mpstat 可以输出多个处理器的数据。

### ps
ps 用于查看某个程序、进程占用的 CPU 资源。

### 实践：定位 CPU 100% 的问题

某个进程 CPU 100% 了，如何定位？

1. 如果是多线程，使用 ```ps -eL |grep 进程id```，找出占用 CPU 最长时间的线程 id
2. 使用 ```strace -p 线程id -tt``` 观察调用的系统调用
3. 使用 ```perf top``` 看看哪个函数占用率最高
4. 或使用 ```watch pstack 线程 id``` 看看调用堆栈

之前工作中遇到过 CPU 使用率很高，通过 ```perf top``` 和 ```pstack``` 观察到写磁盘操作很多，推测是日志打印过多导致 CPU 使用率暴涨，调低了日志等级后顺利解决。

## 内存相关

kswapd daemon 用来检查 pages\_high 和 pages\_low，如果可用内存少于 pages\_low，kswapd 就开始扫描并试图释放 32个页面，并且重复扫描释放的过程直到可用内存大于 pages\_high 为止。

扫描的时候检查3件事：

* 如果页面没有修改，把页放到可用内存列表里
* 如果页面被文件系统修改，把页面内容写到磁盘上
* 如果页面被修改了，但不是被文件系统修改的，把页面写到交换空间

pdflush daemon 用来同步文件相关的内存页面，把内存页面及时同步到硬盘上。比如打开一个文件，文件被导入到内存里，对文件做了修改后并保存后，内核并不马上保存文件到硬盘，由 pdflush 决定什么时候把相应页面写入硬盘，这由一个内核参数 vm.dirty\_background\_ratio 来控制，比如下面的参数显示脏页面（dirty pages）达到所有内存页面10％的时候开始写入硬盘。

```
# /sbin/sysctl -n vm.dirty_background_ratio
10
```

### vmstat

```
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0 164412 143200  35916 243216    0    0    12    11   46   22  1  1 98  0  0
 0  0 164412 142960  35916 243260    0    0     0     0  583  854  2  1 97  0  0
 0  0 164412 142960  35916 243260    0    0     0     0  940 1184  2  1 97  0  0
```

* swpd: 已使用的 SWAP 空间大小，KB 为单位
* free: 可用的物理内存大小，KB 为单位
* buff: 物理内存用来缓存读写操作的 buffer 大小，KB 为单位
* cache: 物理内存用来缓存进程地址空间的 cache 大小，KB 为单位
* si: 数据从 SWAP 读取到 RAM（swap in）的大小，KB 为单位
* so: 数据从 RAM 写到 SWAP（swap out）的大小，KB 为单位
* bi: 从块设备读取（block in）的大小，block 为单位
* bo: 写入块设备（block out）的大小，block 为单位

```
# vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache    si    so    bi    bo   in   cs us sy id  wa st
 0  3 252696   2432    268   7148  3604  2368  3608  2372  288  288  0  0 21  78  1
 0  2 253484   2216    228   7104  5368  2976  5372  3036  930  519  0  0  0 100  0
 0  1 259252   2616    128   6148 19784 18712 19784 18712 3821 1853  0  1  3  95  1
 1  2 260008   2188    144   6824 11824  2584 12664  2584 1347 1174 14  0  0  86  0
 2  1 262140   2964    128   5852 24912 17304 24952 17304 4737 2341 86 10  0   0  4
```

上面是一个频繁读写交换区的例子，可以观察到：

* buff 稳步减少说明系统知道内存不够了，kwapd 正在从 buff 那里借用部分内存
* kswapd 持续把脏页面写到 swap 交换区（so），并且从 swapd 逐渐增加看出确实如此。根据上面讲的 kswapd 扫描时检查的三件事，如果页面被修改了，但不是被文件系统修改的，把页面写到 swap，所以这里 swapd 持续增加


### free 

```
$ free
             total       used       free     shared    buffers     cached
Mem:       1014432     884460     129972      10168      43468     248580
-/+ buffers/cache:     592412     422020
Swap:      1046524     164376     882148
```

* total: 总内存大小
* used: 已经使用的内存大小，包含 cached、buffers 和 shared 部分
* free: 空闲的内存大小
* shared: 进程间共享内存
* buffers: buffer cache
* cached: page cache

-/+ buffers/cache 看作两部分：

* -buffers/cache：正在使用的内存大小，其值等于used1 减去 buffers1 再减去 cached1
* +buffers/cache：可用的内存大小，其值等于 free1 加上 buffers1 再加上 cached

当空闲物理内存不多时，不一定表示系统运行状态很差，因为内存的 cached 及 buffers 部分可以随时被重用。swap 如果被频繁调用，bi 和 bo 长时间不为 0，才是内存资源是否紧张的依据。通过 free 看资源时，实际主要关注 -/+ buffers/cache 的值就可以知道内存到底够不够了。

### valgrind 

valgrind 是定位内存泄漏的好工具，使用 ```valgrind --lead-check=full --log-file=valgrind.log ./a.out``` 即可在进程结束运行后输出内存泄露报告。

### maps, smaps and status

参考这篇文章：[maps, smaps and Memory Stats!](https://jameshunt.us/writings/smaps.html)

### pmap 

待补充

## IO 相关

### Buffers & Cached

Linux 利用虚拟内存极大的扩展了程序地址空间，使得原来物理内存不能容下的程序也可以通过内存和硬盘之间的不断交换（把暂时不用的内存页交换到硬盘，把需要的内存页从硬盘读到内存）来赢得更多的内存，看起来就像物理内存被扩大了一样。如果数据不在内存里就引起一个缺页中断（Page Fault），然后从硬盘读取缺页，并把缺页缓存到物理内存里。缺页中断可分为主缺页中断（Major Page Fault）和次缺页中断（Minor Page Fault），要**从磁盘读取数据而产生的中断是主缺页中断**；数据已经被读入内存并被缓存起来，**从内存缓存区中而不是直接从硬盘中读取数据而产生的中断是次缺页中断**。

从内存缓存区（页高速缓存）读取页比从硬盘读取页要快得多，所以 Linux 内核希望能尽可能产生次缺页中断（从页高速缓存读），并且尽可能避免主缺页中断（从硬盘读），这样随着次缺页中断的增多，缓存区也逐步增大，直到系统只有少量可用物理内存的时候 Linux 才开始释放一些不用的页。

```
$ cat /proc/meminfo
MemTotal:        1014432 kB
MemFree:          137884 kB
Buffers:           43732 kB
Cached:           248744 kB
```

这台服务器总共有 10GB 物理内存（MemTotal），1GB 左右可用内存（MemFree），43 MB 左右用来做磁盘缓存（Buffers），248 MB 左右用来做文件缓存区（Cached）。

Linux 中内存页面有三种类型：

* Read pages: 只读页（或代码页），通过主缺页中断从硬盘读取的页面，包括不能修改的静态文件、可执行文件、库文件等。当内核需要它们的时候把它们读到内存中，当内存不足的时候，内核就释放它们到空闲列表，当程序再次需要它们的时候需要通过缺页中断再次读到内存。
* Dirty pages: 脏页，指在内存中被修改过的数据页，比如文本文件等。这些文件由 pdflush 负责同步到硬盘，内存不足的时候由 kswapd 和 pdflush 把数据写回硬盘并释放内存。
* Anonymous pages: 匿名页，属于某个进程但是又和任何文件无关联，不能被同步到硬盘上，内存不足的时候由 kswapd 负责将它们写到交换分区并释放内存。

### 顺序 IO 和 随机 IO

IO 可分为顺序 IO 和 随机 IO 两种。顺序 IO 重视每次 IO 的吞吐能力（KB per IO），而随机 IO 重视的是每秒能 IOPS 的次数，而不是每次 IO 的吞吐能力（KB per IO）。

### Swap

当系统没有足够物理内存来应付所有请求的时候就会用到 swap 设备，swap 设备可以是一个文件，也可以是一个磁盘分区。使用 swap 的代价非常大，如果系统没有物理内存可用，就会频繁 swapping。如果 swap 设备和程序正要访问的数据在同一个文件系统上，那会碰到严重的 IO 问题，最终导致整个系统迟缓，甚至崩溃。swap 设备和内存之间的 swapping 状况是判断 Linux 系统性能的重要参考，我们已经有很多工具可以用来监测 swap 和 swapping 情况，比如：top、cat /proc/meminfo、vmstat。

## Network 相关

### iptraf

两台主机之间有网线（或无线）、路由器、交换机等设备，测试两台主机之间的网络性能的一个办法就是在这两个系统之间互发数据并统计结果，看看吞吐量、延迟、速率如何。iptraf 就是一个很好的查看本机网络吞吐量的好工具。

### tcpdump

```
sudo tcpdump -i eth0 port 80  -w output.dump    // 指定端口
sudo tcpdump -i eth0 src port 80                // 指定源端口
sudo tcpdump -i eth0 dst port 80                // 指定目的端口
sudo tcpdump -i eth0 src host 192.168.1.1       // 指定源地址
sudo tcpdump -i eth0 dst host 192.168.1.1       // 指定目的地址
sudo tcpdump -i eth0 tcp                        // 协议过滤
sudo tcpdump -i eth0 udp                        // 协议过滤
```

### netstat 

使用 netstat 实时监控 udp 网络收发包情况：

```
$ watch netstat -su

IcmpMsg:
    InType0: 19
    InType3: 55
    InType8: 918828
    InType11: 5
    OutType0: 918828
    OutType3: 55
    OutType8: 20
Udp:
    101881 packets received
    51 packets to unknown port received.
    0 packet receive errors  // 关注该项 如果持续增长，说明在丢包。网卡收到了，但是应用层来不及处理
    164740 packets sent
UdpLite:
IpExt:
    InNoRoutes: 8
    OutMcastPkts: 2
    InBcastPkts: 1938486
    InOctets: 26851362935
    OutOctets: 18903227033
    OutMcastOctets: 80
    InBcastOctets: 300768203
```

## 常用工具汇总

|  		工具 	    |             		简单介绍						|
|----------------|-------------------------------------------------|
|  		top 	 |			查看进程活动状态以及一些系统状况			 |
| 	   vmstat	 | 			查看系统状态、硬件和系统信息等				  |
| 	   iostat	 |   			查看CPU 负载，硬盘状况				  |
| 		sar		 |  			综合工具，查看系统状况					 |
| 	   mpstat	 | 					查看多处理器状况				   |
| 	  netstat	 | 					查看网络状况						|
| 	   iptraf	 | 				   实时网络状况监测		   		       |
|     tcpdump	 |  			抓取网络数据包，详细分析				|
|     tcptrace	 | 					数据包分析工具  				   |
|     netperf	 | 					网络带宽工具						|
|       dstat	 |综合工具，综合了 vmstat, iostat, ifstat, netstat 等多个信息 |

## 相关资料

[《Linux Performance Measurements using vmstat》](https://www.thomas-krenn.com/en/wiki/Linux_Performance_Measurements_using_vmstat)

[《tcpdump使用技巧》](https://linuxwiki.github.io/NetTools/tcpdump.html)

[《Perf -- Linux下的系统性能调优工具》](https://www.ibm.com/developerworks/cn/linux/l-cn-perf1/)

[《通过/proc查看Linux内核态调用栈来定位问题》](https://www.51testing.com/html/56/490256-3711169.html)

[《Linux性能优化》](https://linuxtools-rst.readthedocs.io/zh_CN/latest/advance/03_optimization.html)