---
layout: post
date: 2018-07-06T22:56:37+08:00
title: etcd-raft 源码学习笔记（Leader Transfer）
category: 源码阅读
---

这篇文章介绍 etcd-raft 如何实现 leadership transfer，把 leader 身份转移给某个 follower。

应用层调用 ```TransferLeadership``` 方法，发送一个 type 为 pb.MsgTransferLeader 的请求给 raft 处理。

```go
func (n *node) TransferLeadership(ctx context.Context, lead, transferee uint64) {
	select {
	// manually set 'from' and 'to', so that leader can voluntarily transfers its leadership
	case n.recvc <- pb.Message{Type: pb.MsgTransferLeader, From: transferee, To: lead}:
	case <-n.done:
	case <-ctx.Done():
	}
}
```

```stepLeader``` 收到 pb.MsgTransferLeader 后，检查下是否有正在进行的 leader transfer，并检查 tranferee 的 log 是否是最新的，如果是，调用 ```sendTimeoutNow```，如果不是最新日志，则发送 appendEntriesReq，收到 MsgAppResp 后，如果条件符合，再调用 ```sendTimeoutNow```：

```go
func stepLeader(r *raft, m pb.Message) error {
	// All other message types require a progress for m.From (pr).
	pr := r.getProgress(m.From)
	// These message types do not require any progress for m.From.
	switch m.Type {
		case pb.MsgTransferLeader:
		leadTransferee := m.From
		lastLeadTransferee := r.leadTransferee
		if lastLeadTransferee != None {
			if lastLeadTransferee == leadTransferee {
				return nil
			}
			// 取消之前的
			r.abortLeaderTransfer()
		}
		if leadTransferee == r.id {
			// leadTransferee 已经是 leader 了
			return nil
		}
		// Transfer leadership should be finished in one electionTimeout, so reset r.electionElapsed.
		r.electionElapsed = 0
		r.leadTransferee = leadTransferee
		// 如果 leadTransferee 的 log 已经是最新的了 则马上调用 sendTimeoutNow，开始 transfer
		if pr.Match == r.raftLog.lastIndex() {
			r.sendTimeoutNow(leadTransferee)
		} else {
			// 否则先往 leadTransferee append 日志
			r.sendAppend(leadTransferee)
		}

		...
		// 收到 append 回包后，检查是不是有 in progress 的 leader transfer，并且 log 也是最新了的话，则调用 sendTimeoutNow
		case pb.MsgAppResp:
		pr.RecentActive = true

		...
		// Transfer leadership is in progress.
		if m.From == r.leadTransferee && pr.Match == r.raftLog.lastIndex() {
			r.sendTimeoutNow(m.From)
		}
		...
	}
	return nil
}
```

Leader transfer 过程中不处理 pb.MsgProp 类型的请求：

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	case pb.MsgProp:
		...
		if r.leadTransferee != None {
			// 正在 leader tranfer，不处理 Propose 请求
			return ErrProposalDropped
		}
	}
	return nil
}
```

```sendTimeoutNow``` 发送 pb.MsgTimeoutNow 的请求，看看 follower 如何处理：

```go
func stepFollower(r *raft, m pb.Message) error {
	switch m.Type {
		case pb.MsgTimeoutNow:
		if r.promotable() {
			// Leadership transfers never use pre-vote even if r.preVote is true; we
			// know we are not recovering from a partition so there is no need for the
			// extra round trip.
			r.campaign(campaignTransfer)
		}
	}
	return nil
}
```

```campain``` 会发送 voteMsg 给 peers 进行选举：

```go

func (r *raft) campaign(t CampaignType) {
	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
		r.becomePreCandidate()
		voteMsg = pb.MsgPreVote
		// PreVote RPCs are sent for the next term before we've incremented r.Term.
		term = r.Term + 1
	} else {
		r.becomeCandidate()  // 变成 Candidate term + 1，此时该节点 term 最大，所以该节点将成为新的 leader
		voteMsg = pb.MsgVote
		term = r.Term
	}
	if r.quorum() == r.poll(r.id, voteRespMsgType(voteMsg), true) {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			r.campaign(campaignElection)
		} else {
			r.becomeLeader()
		}
		return
	}
	
	// 发送 voteMsg
	for id := range r.prs {
		if id == r.id {
			continue
		}
		var ctx []byte
		if t == campaignTransfer {
			ctx = []byte(t)
		}
		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

当前 leader 变成 follower 之后，会调用 ```reset```，```reset``` 将调用 ```abortLeaderTransfer``` 把 ```r.leadTransferee``` 设置为 None，leader transfer 完成。

```go
func (r *raft) reset(term uint64) {
	if r.Term != term {
		r.Term = term
		r.Vote = None
	}
	r.lead = None

	r.electionElapsed = 0
	r.heartbeatElapsed = 0
	r.resetRandomizedElectionTimeout()
	r.abortLeaderTransfer()
	...
}
```