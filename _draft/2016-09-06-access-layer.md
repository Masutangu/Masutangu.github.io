---
layout: post
date: 2016-09-03T00:44:24+08:00
title: 接入层
category: 后台开发
---


SYN 攻击是一种常见的 DoS 技术，原理为
1、过滤网关防护
这里，过滤网关主要指明防火墙，当然路由器也能成为过滤网关。防火墙部署在不同网络之间，防范外来非法攻击和防止保密信息外泄，它处于客户端和服务器之间，利用它来防护SYN攻击能起到很好的效果。过滤网关防护主要包括超时设置，SYN网关和SYN代理三种。
■网关超时设置：防火墙设置SYN转发超时参数（状态检测的防火墙可在状态表里面设置），该参数远小于服务器的timeout时间。当客户端发送完SYN包，服务端发送确认包后（SYN+ACK），防火墙如果在计数器到期时还未收到客户端的确认包（ACK），则往服务器发送RST包，以使服务器从队列中删去该半连接。值得注意的是，网关超时参数设置不宜过小也不宜过大，超时参数设置过小会影响正常的通讯，设置太大，又会影响防范SYN攻击的效果，必须根据所处的网络应用环境来设置此参数。
■SYN网关：SYN网关收到客户端的SYN包时，直接转发给服务器；SYN网关收到服务器的SYN/ACK包后，将该包转发给客户端，同时以客户端的名义给服务器发ACK确认包。此时服务器由半连接状态进入连接状态。当客户端确认包到达时，如果有数据则转发，否则丢弃。事实上，服务器除了维持半连接队列外，还要有一个连接队列，如果发生SYN攻击时，将使连接队列数目增加，但一般服务器所能承受的连接数量比半连接数量大得多，所以这种方法能有效地减轻对服务器的攻击。
■SYN代理：当客户端SYN包到达过滤网关时，SYN代理并不转发SYN包，而是以服务器的名义主动回复SYN/ACK包给客户，如果收到客户的ACK包，表明这是正常的访问，此时防火墙向服务器发送ACK包并完成三次握手。SYN代理事实上代替了服务器去处理SYN攻击，此时要求过滤网关自身具有很强的防范SYN攻击能力。

 2、加固tcp/ip协议栈
防范SYN攻击的另一项主要技术是调整tcp/ip协议栈，修改tcp协议实现。主要方法有SynAttackProtect保护机制、SYN cookies技术、增加最大半连接和缩短超时时间等。tcp/ip协议栈的调整可能会引起某些功能的受限，管理员应该在进行充分了解和测试的前提下进行此项工作。
■SynAttackProtect机制
为防范SYN攻击，win2000系统的tcp/ip协议栈内嵌了SynAttackProtect机制，Win2003系统也采用此机制。SynAttackProtect机制是通过关闭某些socket选项，增加额外的连接指示和减少超时时间，使系统能处理更多的SYN连接，以达到防范SYN攻击的目的。默认情况下，Win2000操作系统并不支持SynAttackProtect保护机制，需要在注册表以下位置增加SynAttackProtect键值：
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
当SynAttackProtect值（如无特别说明，本文提到的注册表键值都为十六进制）为0或不设置时，系统不受SynAttackProtect保护。
当SynAttackProtect值为1时，系统通过减少重传次数和延迟未连接时路由缓冲项（route cache entry）防范SYN攻击。
当SynAttackProtect值为2时（Microsoft推荐使用此值），系统不仅使用backlog队列，还使用附加的半连接指示，以此来处理更多的SYN连接，使用此键值时，tcp/ip的TCPInitialRTT、window size和可滑动窗囗将被禁止。
我们应该知道，平时，系统是不启用SynAttackProtect机制的，仅在检测到SYN攻击时，才启用，并调整tcp/ip协议栈。那么系统是如何检测SYN攻击发生的呢？事实上，系统根据TcpMaxHalfOpen,TcpMaxHalfOpenRetried 和TcpMaxPortsExhausted三个参数判断是否遭受SYN攻击。
TcpMaxHalfOpen 表示能同时处理的最大半连接数，如果超过此值，系统认为正处于SYN攻击中。Win2000　server默认值为100，Win2000　Advanced server为500。
TcpMaxHalfOpenRetried定义了保存在backlog队列且重传过的半连接数，如果超过此值，系统自动启动SynAttackProtect机制。Win2000　server默认值为80，Win2000 Advanced server为400。
TcpMaxPortsExhausted　是指系统拒绝的SYN请求包的数量，默认是5。
如果想调整以上参数的默认值，可以在注册表里修改（位置与SynAttackProtect相同）
■ SYN cookies技术
我们知道，TCP协议开辟了一个比较大的内存空间backlog队列来存储半连接条目，当SYN请求不断增加，并这个空间，致使系统丢弃SYN连接。为使半连接队列被塞满的情况下，服务器仍能处理新到的SYN请求，SYN cookies技术被设计出来。
SYN cookies应用于linux、FreeBSD等操作系统，当半连接队列满时，SYN　cookies并不丢弃SYN请求，而是通过加密技术来标识半连接状态。
在TCP实现中，当收到客户端的SYN请求时，服务器需要回复SYN+ACK包给客户端，客户端也要发送确认包给服务器。通常，服务器的初始序列号由服务器按照一定的规律计算得到或采用随机数，但在SYN cookies中，服务器的初始序列号是通过对客户端IP地址、客户端端囗、服务器IP地址和服务器端囗以及其他一些安全数值等要素进行hash运算，加密得到的，称之为cookie。当服务器遭受SYN攻击使得backlog队列满时，服务器并不拒绝新的SYN请求，而是回复cookie（回复包的SYN序列号）给客户端， 如果收到客户端的ACK包，服务器将客户端的ACK序列号减去1得到cookie比较值，并将上述要素进行一次hash运算，看看是否等于此cookie。如果相等，直接完成三次握手（注意：此时并不用查看此连接是否属于backlog队列）。
在RedHat linux中，启用SYN cookies是通过在启动环境中设置以下命令来完成：
# echo 1 > /proc/sys/net/ipv4/tcp_syncookies
■ 增加最大半连接数
大量的SYN请求导致未连接队列被塞满，使正常的TCP连接无法顺利完成三次握手，通过增大未连接队列空间可以缓解这种压力。当然backlog队列需要占用大量的内存资源，不能被无限的扩大。

。。。http://baike.baidu.com/subview/14041717/14616901.htm