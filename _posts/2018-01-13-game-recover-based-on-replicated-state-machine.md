---
layout: post
date: 2018-01-13T11:30:24+08:00
title: 基于 Replicated State Machine 实现游戏进程恢复
category: 工作
---

## Introduction

游戏服务器实现的业务逻辑普遍比较复杂，且大部分是带有状态的。如果进程重启或意外崩溃，会导致该服务器上的玩家断线，丢失进行中的游戏数据，带来极差的游戏体验。为了避免这种情况出现，一般游戏服务器都会持久化玩家数据以实现进程恢复，当重启或进程意外崩溃时，重新拉起进程后可以恢复到之前的状态。**常用的做法是将玩家的状态信息保存在共享内存中，重启时加载共享内存进行恢复。**

共享内存虽然方便，但会有许多限制。比如 C++ 涉及到**多态（虚函数表）、STL容器（heap分配）**，都不能直接映射到共享内存中：

<img src="/assets/images/game-recover-based-on-replicated-state-machine/illustration-1.png" width="800"/>


这篇文章提供了新的思路，提出一个实现游戏进程恢复更简洁的做法。


## Replicated State Machine

在分布式系统中，**replicated State Machine 是实现 fault tolerance 的一个重要方式**，通常由复制日志来实现。每一台服务器保存一份日志，日志中包含一系列的命令，状态机会按顺序执行这些命令。因为每一台计算机的状态机都是确定的（deterministic state machine），执行的命令相同，最后输出的结果也相同。

## Combine

如果我们能**以确定状态机（deterministic state machine）来实现游戏逻辑**，就可以运用 replicated State Machine 的思想。只要我们把所有触发状态机状态变更的 event 都保存下来，重启时直接重放一遍，就可以回到重启前的状态了。

假设实现一个回合制 pvp 的对战游戏逻辑。RoomSvr 上有若干个房间。我们**以 state machine 来实现业务逻辑，每个房间通过 state machine 维护自己的状态信息，将玩家的请求和定时器超时都转化为 event，通过 event distributer 分发给相应房间处理，并且将 event 通过 logging module 序列化保存到本地**：

<img src="/assets/images/game-recover-based-on-replicated-state-machine/illustration-2.png" width="800"/>

进程重启时直接加载日志，读取 event 丢给房间的 state machine 进行重放，就能将每个房间恢复到进程重启/挂掉之前的状态了。

## How to Snapshot

日积月累，序列化的日志会越来越多，如何能清理到不再需要的日志，提高重启时加载的速度呢？由于游戏逻辑不是简单的 kv 存储，无法直接做 snapshot，也无法参考 leveldb LSM-Tree 的做法，需要换一种方式来减少日志的堆积。

当这一局已经结束时，这局的相关 event 就可以全部删掉了。如果将 event 序列化到日志中，要删除会比较麻烦。所以考虑利用共享内存来实现。

<img src="/assets/images/game-recover-based-on-replicated-state-machine/illustration-3.png" width="800"/>

实现 ShmStore 来管理共享内存的存储，替换掉上图的 Logging Module。```ShmArr``` 为映射到共享内存的 ```RoomInfo```数组，```RoomInfo``` 是 C struct 结构，记录了房间id、房间的状态（是否有效）和房间的所有 event。  ```room_idx_map_``` 维护着房间 id 到 ```ShmArr``` 下标的关系，```free_idx_``` 保存着 ```ShmArr``` 中空闲的下标。当创建新房间时，从 ```free_idx_``` 中取一个空闲下标，并把房间 id 到该下标的映射关系保存于 ```room_idx_map_``` 中，将 ```RoomInfo``` 的 status 置为有效。之后该房间的所有 event 就保存在对应的 ```RoomInfo``` 结构里。当房间销毁时，将对应的 ```RoomInfo``` 结构清空，同时从```room_idx_map_```删除对应的映射关系，并把该 ```RoomInfo``` 的下标添加回 ```free_idx_```。

当读取共享内存重建房间状态时，只加载 status 为有效的 ```RoomInfo``` 结构。

## Conclusion

最终模块图如下：

<img src="/assets/images/game-recover-based-on-replicated-state-machine/illustration-4.png" width="800"/>


