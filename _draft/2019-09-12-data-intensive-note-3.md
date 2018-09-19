---
layout: post
date: 2018-09-12T22:03:19+08:00
title: Designing Data-Intensive Applications 读书笔记（三）
category: 读书笔记
---

# Chapter 6. Partitioning

For very large datasets, or very high query throughput, that is not sufficient: we need to break the data up into **partitions**, also known as **sharding**.

The main reason for wanting to partition data is **scalability**. Different partitions can be placed on different nodes in a shared-nothing cluster.

In this chapter we will first look at different approaches for partitioning large datasets and observe **how the indexing of data interacts with partitioning**. We’ll then talk about **rebalancing**, which is necessary if you want to add or remove nodes in your cluster. Finally, we’ll get an overview of **how databases route requests to the right partitions and execute queries**.

## Partitioning and Replication

Partitioning is usually combined with replication so that copies of each partition are stored on multiple nodes. **Even though each record belongs to exactly one partition, it may still be stored on several different nodes for fault tolerance.**

A node may store more than one partition. If a leader–follower replication model is used, the combination of partitioning and replication can look like bellow figure:

<img src="/assets/images/data-intensive-note-3/illustration-1.png" width="800"/>

## Partitioning of Key-Value Data

If the partitioning is unfair, so that some partitions have more data or queries than others, we call it skewed. 

### Partitioning by Key Range

<img src="/assets/images/data-intensive-note-3/illustration-2.png" width="800"/>

Within each partition, we can keep keys in sorted order. This has the advantage that range scans are easy, and you can treat the key as a concatenated index in order to fetch several related records in one query.

### Partitioning by Hash of Key

Because of this risk of skew and hot spots, many distributed datastores use a hash function to determine the partition for a given key.

Once you have a suitable hash function for keys, you can assign each partition a range of hashes (rather than a range of keys), and every key whose hash falls within a partition’s range will be stored in that partition. 


<img src="/assets/images/data-intensive-note-3/illustration-3.png" width="800"/>

Unfortunately however, by using the hash of the key for partitioning we lose a nice property of key-range partitioning: the ability to do efficient range queries. Keys that were once adjacent are now scattered across all the partitions. In MongoDB, if you have enabled hash-based sharding mode, any range query has to be sent to all partitions.

Cassandra achieves a compromise between the two partitioning strategies. A table in Cassandra can be declared with a compound primary key consisting of several columns. Only the first part of that key is hashed to determine the partition, but the other columns are used as a concatenated index for sorting the data in Cassandra’s SSTables. A query therefore cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it can perform an efficient range scan over the other columns of the key.

### Skewed Workloads and Relieving Hot Spots

As discussed, hashing a key to determine its partition can help reduce hot spots. However, it can’t avoid them entirely: in the extreme case where all reads and writes are for the same key, you still end up with all requests being routed to the same partition.

