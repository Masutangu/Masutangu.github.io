---
layout: post
date: 2021-07-03T09:47:24+08:00
title: 一致性哈希的应用
tags: 
  - 分布式
---

### 一致性哈希

一致性哈希是业界最常用的哈希方案，通常在分布式系统中会采用一致性哈希的方式对请求进行路由。

哈希算法的好坏有四个标准：**均衡性（Balance）、单调性（Monotonicity）、分散性（Spread）**和**负载（Load）**，具体可以参考论文 [Consistent Hashing - A Distributed Caching Protocol](http://webcache.googleusercontent.com/search?q=cache:-EmlEOjK3_4J:www.birger-kuehnel.de/uploads/media/consistenthashing_01.pdf+&cd=2&hl=zh-TW&ct=clnk)。

这里重点提一下**单调性**。哈希桶数量发生变化时，已有的 key 会重新映射。单调性指 key 要么保留在原来的桶中，要么移动到新增加的桶中。如果 key 移动到原有的其他桶中，就不满足“单调性”了。

### 一致性哈希的应用

一致性哈希在实际落地中，需要注意以下两点：

#### 引入虚拟节点

引入虚拟节点，一来可以让节点分布更均匀，二来可以利用虚拟节点实现**按权重分布**（虚拟节点数 M:N，则理想情况下节点分配的请求量比近似 M:N），更重要的是**如果某个节点挂掉了，请求不会全部分流到他的相邻节点**（准确的说应该是减少发生的概率），如下图：

<img src="/assets/images/consistent-hash/illustration-1.png" width="600" />

#### 谨慎挑选哈希函数

哈希分布是否均衡**严重依赖**哈希函数的选择。而 Google 提出的 [Jump Consistent Hash Algorithm](https://arxiv.org/ftp/arxiv/papers/1406/1406.2294.pdf) 很好地解决了这个问题，其采用线性同余法产生伪随机数。伪随机数即：只要初始种子确定，那么随机序列就具有：确定和足够随机的两个特性，或者说**确定的随机性**。

### Jump Consistent Hash 的问题

但 Jump Consistent Hash 最大的问题是，节点只能在尾部进行增删。下图中，如果把 N2 删除，k2 k3 都需要顺移一个节点：

<img src="/assets/images/consistent-hash/illustration-2.png" width="600" />

原因是 Jump Consistent Hash 返回的是桶的下标，如果中间删除了一个桶，桶数组会产生偏移。因此如果 N2 节点挂掉了，不能直接删除掉，否则不满足哈希算法单调性的要求。

那么如果真的出现中间节点挂掉，如何处理？这里设计两个改造方案来解决这个问题。

### Jump Consistent Hash 改造方案

#### 方案 1

##### 流程

1. ```all_node_vec``` **保存全部节点**。如果出现节点挂掉，节点**不移除**，而是做个标记，标记节点挂掉。
2. 维护另一个**只包含有效节点**的数组，定义为 ```valid_node_vec```。节点挂掉的时候会从 ```valid_node_vec``` 中**移除**。
3. Jump Consistent Hash 先在 ```all_node_vec``` 中寻找节点，如果节点挂掉，则再从 ```valid_node_vec``` 中寻找节点。

##### 伪代码

```cpp
int FindNode(key) {
    node = JumpConsistentHash(key, all_node_vec);
    if (node.hasDown) { 
        node = JumpConsistentHash(key, valid_node_vec); // 节点挂掉了，从 valid_node_vec 中找
    }
    return node.Id;
}
```

##### 缺陷

valid_node_vec 会删掉中间节点（因为其只保存有效节点）接连挂掉节点的情况下，依然会出现下标偏移问题，不满足单调性的要求。

#### 方案 2

##### 流程

1. ```all_node_vec``` **保存全部节点**。如果出现节点挂掉，节点**不移除**，而是做个标记，标记节点挂掉。
2. Jump Consistent Hash 在 ```all_node_vec``` 中寻找节点，如果找到的节点是挂掉的，则给 key 加个偏移值，继续寻找。

##### 伪代码

```cpp
int FindNode(key) {
    node = JumpConsistentHash(key, all_node_vec);
    while (node.hasDown) { 
        key += offset
        node = JumpConsistentHash(key, all_node_vec); // 节点挂掉了，key 加个 offset 值继续寻找
    }
    return node.Id;
}
```

##### 缺陷

`while (node.hasDown)` 需要加个兜底方案。例如重试 N 次依然没找到，就分流给相邻的有效节点。

以上两个方案在落地中，需要注意服务扩缩容时，尽量在尾部进行操作。如果是中间节点出现机器故障，最好是用新机器补上，而不是直接去掉，小心规避下标偏移的问题。