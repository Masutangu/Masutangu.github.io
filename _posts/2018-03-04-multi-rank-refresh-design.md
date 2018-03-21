---
layout: post
date: 2018-03-04T09:00:42+08:00
title: 多排行榜数据刷新方案
category: 工作
---

## 一. 背景
最近工作遇到一个棘手的问题：**多个不同的排行榜的玩家信息如何保持一致**。简单描述下场景，以小游戏跳一跳为例子，一开始游戏只有一个好友排行榜，好友排行榜以玩家的最高分数进行排序，这样好处理，搭一个关系链svr，该 svr 上缓存玩家好友的信息（避免每次去 DB 查询），并使用玩家信息中的最高分数进行排序。客户端请求时下方相应的排名和玩家信息，包括最高分数信息（客户端需要展示）即可。但如果我们要新增一个全国排行榜，全国排行榜以玩家的最高步数＋耗时进行排行。这时需要搭一个全国排行榜svr，该svr上同样缓存进入全国排行榜的玩家的信息，使用玩家信息中的最高分数＋耗时进行排序。同样的在客户端请求时下发相应的排名和玩家信息。

*方案一：各个排行榜有自己的玩家缓存：*
<img src="/assets/images/multi-rank-refresh-design/illustration-1.png" width="800"/>

## 二. 问题
但问题来了，**如何保证出现在两个榜单的同一玩家数据是一致的？**（不一致是因为两个排行 svr 分别缓存了玩家的信息，每个svr的排行数据缓存刷新周期也不一致）。

这时候就需要使用一个全局的 data svr 来缓存玩家的信息，**保证不同排行 svr 取到的玩家数据是一致的，同时两个排行svr刷新缓存的周期需要保持一致**。

*方案二：每个排行榜都从data svr 拉取玩家数据：*
<img src="/assets/images/multi-rank-refresh-design/illustration-2.png" width="800"/>

数据源都从 data svr 拉取这个很简单，关键在于**如何让不同排行榜的刷新周期保持一致**。本文提出一个不成熟有待考验的方案解决这个问题，并给出简单的协议例子说明。

## 三. 方案

### 1. 协议

CSProto 表示从客户端到服务器的协议，SSProto 表示服务器之间的协议。Common 为公用协议。定义排行榜相关协议如下：

*公用协议：*

```
package Common;

message PlayerInfo {
    int32 uid              = 1;  // 玩家 uid
    int64 udpate_timestamp = 2;  // 玩家信息的更新时间
    // 玩家信息 包括例如最高分 耗时 具体字段略过略
}
```

*客户端和服务器之间的协议：*

```
import "common.proto";

package CSProto;

message RankReq {
    int32 rank_type = 1; // 排行榜类型 例如好友排行榜 全国排行榜
}

message RankInfo {
    Common.PlayerInfo info = 1; // 玩家信息 客户端展示用
    int32 rank_idx         = 2; // 玩家排名
}

message RankRes {
    int32 rank_type             = 1; // 排行榜类型 例如好友排行榜 全国排行榜
    repeated RankInfo rank_list = 2; // 排行数据
    int64 update_timestamp      = 3; // 排行榜更新时间
}
```

*服务器和服务器之间的协议：*

```
package SSProto;

message GetPlayerInfoReq {
    repeated int32 uid_list = 1;
} 

message GetPlayerInfoRes {
    repeated Common.PlayerInfo info_list = 1;
}
```

*消息流如下：*
<img src="/assets/images/multi-rank-refresh-design/illustration-3.png" width="800"/>

### 2. 保证排行榜 svr 刷新周期一致

排行榜 svr 从 Data svr 中拉取玩家的信息进行排序，而 Data svr 会定期去更新玩家的信息，可以推导出：

**排行榜的刷新时间等于max(排行榜上榜的玩家数据的更新时间戳)**

因此回包给客户端的 RankRes 中的 update_timestamp 取值的伪代码如下：

```
rank_res.update_timestamp =  max([rankinfo.info.udpate_timestamp for rankinfo in rank_res.rank_list])
```

客户端使用 map 来管理每个排行榜的更新时间：

```
# rank_map
<ranktype1, udpate_timestamp1>
<ranktype2, update_timestamp2>
...
```

并定义所有排行榜的最新更新时间 rank_max_update_timestamp，取值伪代码如下：

```
rank_max_update_timestamp = max([update_timestamp for _, update_timestamp in rank_map]) 
```

当玩家点击某个排行榜，客户端发现该排行榜的 update_timestamp 小于 rank_max_update_timestamp，就能判定该排行榜上存在过时的玩家数据，这时就应该向后台发起 RankReq 获取排行榜请求。**通过及时请求过时排行榜数据，保证每个排行榜的 update_timestamp 一致，就能保证排行榜上玩家信息的一致，也就保证了在多个排行榜上玩家信息的一致。**

### 3. 优化

上面提到，每次 update_timestamp 小于 rank_max_update_timestamp，客户端都会重新请求一次排行榜，后台会返回最新的排行榜数据，这里其实可以做下优化。

<img src="/assets/images/multi-rank-refresh-design/illustration-4.png" width="800"/>

客户端先请求了 ranktype1 的排行榜，可以看出 ranktype1 的 update_timestamp 为 t1。之后又请求了 rank_type2 的排行榜，返回 ranktype2 的 update_timestamp 为 t2。由于 t1 < t2，客户端会发现需要更新 ranktype1。但从图上可以看出其实后台不需要再返回一次 ranktype1 的排行数据了（ranktype1 榜上的玩家 uid1 uid2 uid3 的数据并没有变化）。因此我们在 RankReq 里加上客户端本地该排行榜的 update_timestamp，如果后台发现客户端的 update_timestamp 和后台的是一致的，就返回特定的错误码告诉客户端排行榜依然有效。

*新增客户端本地该排行榜的 update_timestamp：*

```
message RankReq {
    int32 rank_type               = 1; // 排行榜类型 例如好友排行榜 全国排行榜
    int64 client_update_timestamp = 2; // 客户端本地该排行榜的 update_timestamp
}
```

采用这种方式的话，每次点击该排行榜客户端还是需要发起一次（无效）的后台请求，如何做优化呢？这里用了一个简单的方案，如果客户端收到排行榜依然有效的错误码，就把本地该排行榜 update_timestamp 更新为 rank_max_update_timestamp：

<img src="/assets/images/multi-rank-refresh-design/illustration-5.png" width="800"/>

每次客户端请求，rank svr 都需要到 data svr 查询玩家信息。可以在 rank svr 上缓存玩家的信息，到 data svr 查询时如果玩家数据无变化，则返回特定错误码，rank svr 继续使用本地的玩家信息缓存。

## 三. 总结

本文提出了一个有待考验的多排行榜数据刷新方案，为解决多个排行榜数据不一致的问题。该方案还有一些细节有待考量，欢迎大家有任何想法或有更好的方案邮件我一起讨论。