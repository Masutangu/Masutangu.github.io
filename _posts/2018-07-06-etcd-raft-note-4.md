---
layout: post
date: 2018-07-06T08:46:43+08:00
title: etcd-raft 源码学习笔记（Linearizable Read 之 Lease）
tags: 
  - 源码阅读
  - 分布式
  - Golang
---

这篇文章介绍 etcd-raft 如何实现 linearizable read（linearizable read 简单的说就是不返回 stale 数据，具体可以看这篇文章 [《Strong consistency models》](https://aphyr.com/posts/313-strong-consistency-models)）。

除了基于 [ReadIndex](http://masutangu.com/2018/07/etcd-raft-note-3/) 之外，raft 论文第 8 节还阐述了另一种基于 heartbeat 的 lease 思路：

> Alternatively, the leader could rely on the heartbeat mechanism to provide a form of lease, but this would rely on timing for safety (it
assumes bounded clock skew).

raft 中，follower 至少会在 election timeout 之后才重新进行选举。leader 定期发送 heartbeat，在收到 quonum 节点的回包后的 election timeout 这段时间间隔内，不会有新一轮的选举（因为各个机器的 cpu 时钟有误差，所以这个方案有风险）。

lease 模式对应用层提供的接口还是 ```ReadIndex```，应用层处理的方式也和基于 ReadIndex 模式相同。只是 raft 内部逻辑不同。

如果指定了 leaseBase 的模式，那要求 ```CheckQuorum``` 为 true，```validate``` 方法做了这个检查：

```go
func (c *Config) validate() error {
	...
	if c.ReadOnlyOption == ReadOnlyLeaseBased && !c.CheckQuorum {
		return errors.New("CheckQuorum must be enabled when ReadOnlyOption is ReadOnlyLeaseBased")
	}

	return nil
}
```

指定了 ```checkQuorum``` 为 true 之后，每次 tick 都会看是否应该检查 Quorum（间隔 electionTimeout），通过发送 pb.MsgCheckQuorum 类型的请求：

```go
// tickHeartbeat is run by leaders to send a MsgBeat after r.heartbeatTimeout.
func (r *raft) tickHeartbeat() {
	r.heartbeatElapsed++
	r.electionElapsed++

	if r.electionElapsed >= r.electionTimeout {  // 每隔 electionTimeout 检查一次
		r.electionElapsed = 0
		if r.checkQuorum {
			r.Step(pb.Message{From: r.id, Type: pb.MsgCheckQuorum})  // 检查 Quorum
		}
		// If current leader cannot transfer leadership in electionTimeout, it becomes leader again.
		if r.state == StateLeader && r.leadTransferee != None {
			r.abortLeaderTransfer()
		}
	}

	if r.state != StateLeader {
		return
	}

	if r.heartbeatElapsed >= r.heartbeatTimeout {
		r.heartbeatElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgBeat})
	}
}
```


```stepLeader``` 收到 pb.MsgCheckQuorum 后调用 ```checkQuorumActive``` 进行检查，如果返回 false，此时把节点变更为 follower：

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	case pb.MsgCheckQuorum:
		if !r.checkQuorumActive() {  
			r.becomeFollower(r.Term, None)  // checkQuorumActive 失败，变成 follower
		}
		return nil
	}
	return nil
}
```


```checkQuorumActive``` 即统计 active 的 peers 数量是否超过 quonum：

```go
// checkQuorumActive also resets all RecentActive to false.
func (r *raft) checkQuorumActive() bool {
	var act int

	r.forEachProgress(func(id uint64, pr *Progress) {
		if id == r.id { // self is always active
			act++
			return
		}

		if pr.RecentActive && !pr.IsLearner {
			act++
		}

		pr.RecentActive = false
	})

	return act >= r.quorum()
}
```

```RecentActive``` 是 leader 收到 peers 的心跳回包或者 appendEntriesReq 的回包时设置的：

```go
func stepLeader(r *raft, m pb.Message) error {
	// All other message types require a progress for m.From (pr).
	pr := r.getProgress(m.From)
	
	switch m.Type {
	case pb.MsgAppResp:
		pr.RecentActive = true
		...
	case pb.MsgHeartbeatResp:
		pr.RecentActive = true
		...
	}
}
```

文章开头提到， leaseBase 模式下 etcd-raft 对应用层暴露的也是 ```ReadIndex``` 接口。在收到 pb.MsgReadIndex 类型的请求时，由于 CheckQuonum 保证了我们 leader 有效，就可以直接 append 到 r.readStates 中。

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	case pb.MsgReadIndex:
		// 5.4 safety
		if r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(r.raftLog.committed)) != r.Term {
			// Reject read only request when this leader has not committed any log entry at its term.
			return nil
		}

		// thinking: use an interally defined context instead of the user given context.
		// We can express this in terms of the term and index instead of a user-supplied value.
		// This would allow multiple reads to piggyback on the same message.
		switch r.readOnly.option {
		case ReadOnlyLeaseBased: // leaseBase 模式
			ri := r.raftLog.committed
			if m.From == None || m.From == r.id { // from local member
				r.readStates = append(r.readStates, ReadState{Index: r.raftLog.committed, RequestCtx: m.Entries[0].Data})
			} else {
				r.send(pb.Message{To: m.From, Type: pb.MsgReadIndexResp, Index: ri, Entries: m.Entries})
			}
		}

		return nil
	}
	return nil
}
```