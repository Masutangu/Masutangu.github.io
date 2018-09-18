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