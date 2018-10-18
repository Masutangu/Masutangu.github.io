---
layout: post
date: 2018-07-08T10:19:08+08:00
title: etcd-raft 源码学习笔记（PreVote）
tags: 
  - 源码阅读
  - 分布式
  - Golang
---

这篇文章介绍 etcd-raft 的 PreVote 机制，避免由于网络分区导致 candidate 的 term 不断增大。


Election timeout 之后，发送 type 为 pb.MsgHup 的请求，进入选举阶段：

```go
// tickElection is run by followers and candidates after r.electionTimeout.
func (r *raft) tickElection() {
	r.electionElapsed++

	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```

如果设置了 preVote 为 true，则先进入 prevote 阶段。调用 ```r.campaign``` 传入 type ```campaignPreElection```：

```go
func (r *raft) Step(m pb.Message) error {
    switch m.Type {
	case pb.MsgHup:
		if r.state != StateLeader {
			r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
			if r.preVote {
                // 如果 preVote 设置为 true，先发起 campaignPreElection
				r.campaign(campaignPreElection)
			} else {
				r.campaign(campaignElection)
			}
		} else {
			r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
        }
    }
    return nil
}
```

```campaign``` 方法处理选举逻辑。如果是 ```campaignPreElection```，设置节点状态为 ```StatePreCandidate```，此时不会递增节点的 Term（避免 term 增长过快）。然后向其他 peers 发送 type 为 ```pb.MsgPreVote``` 的请求：

```go
func (r *raft) campaign(t CampaignType) {
	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
		r.becomePreCandidate()  // state 设置为 StatePreCandidate
		voteMsg = pb.MsgPreVote  // msg type 设置为 preVote
		// PreVote RPCs are sent for the next term before we've incremented r.Term.
		term = r.Term + 1  // preVote 不会递增 r.Term
	} else {
		...
	}
	if r.quorum() == r.poll(r.id, voteRespMsgType(voteMsg), true) {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			r.campaign(campaignElection)  // prevote 成功，可以发起 campaignElection 了
		} else {
			... 
		}
		return
    }
    
    // 广播 pb.MsgPreVote
	for id := range r.prs {
		if id == r.id {
			continue
		}
		var ctx []byte
		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

看看 ```stepCandidate``` 如何处理 ```pb.MsgPreVote``` 请求的回包。检查选票是否达到 quorum 数量，如果已经达到，prevote 成功，可以发起真正的选举了，调用 ```r.campaign``` 传入 type ```campaignElection```：

```go
// stepCandidate is shared by StateCandidate and StatePreCandidate; the difference is
// whether they respond to MsgVoteResp or MsgPreVoteResp.
func stepCandidate(r *raft, m pb.Message) error {
	// Only handle vote responses corresponding to our candidacy (while in
	// StateCandidate, we may get stale MsgPreVoteResp messages in this term from
	// our pre-candidate state).
	var myVoteRespType pb.MessageType
	if r.state == StatePreCandidate {
		myVoteRespType = pb.MsgPreVoteResp
	} else {
		myVoteRespType = pb.MsgVoteResp
	}
	switch m.Type {
	case myVoteRespType:
		gr := r.poll(m.From, m.Type, !m.Reject)
		switch r.quorum() {
		case gr:
			if r.state == StatePreCandidate {
				r.campaign(campaignElection)  // prevote 成功，可以发起 campaignElection
			} else {
				...
			}
		case len(r.votes) - gr:  // prevote 失败（m.Reject 为 true，此时 m.Term > r.Term），转为 follower 角色
			// pb.MsgPreVoteResp contains future term of pre-candidate
			// m.Term > r.Term; reuse r.Term
			r.becomeFollower(r.Term, None)
		}
	}
	return nil
}
```

如果 campaign 类型为 ```campaignElection```，则调用 ```r.becomeCandidate```，此时设置节点状态为 ```StateCandidate```，递增节点的 Term，并向其他 peers 发送 ```pb.MsgVote``` 请求：

```go
func (r *raft) campaign(t CampaignType) {
	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
		...
	} else {
		r.becomeCandidate()  // become cancdidate，term 递增
		voteMsg = pb.MsgVote
		term = r.Term
	}
	if r.quorum() == r.poll(r.id, voteRespMsgType(voteMsg), true) {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			...
		} else {
			r.becomeLeader()  // 得到 quorum 的选票，选举 leader 成功，become leader
		}
		return
    }
    
    // 广播 pb.MsgVote，进行选举
	for id := range r.prs {
		if id == r.id {
			continue
		}
		var ctx []byte
		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

收到 ```pb.MsgVote``` 的回包后，同样检查是否选票数量是否达到 quorum，成功则该节点当选 leader：

```go
func stepCandidate(r *raft, m pb.Message) error {
	// Only handle vote responses corresponding to our candidacy (while in
	// StateCandidate, we may get stale MsgPreVoteResp messages in this term from
	// our pre-candidate state).
	var myVoteRespType pb.MessageType
	if r.state == StatePreCandidate {
		myVoteRespType = pb.MsgPreVoteResp
	} else {
		myVoteRespType = pb.MsgVoteResp
	}
	switch m.Type {
	case myVoteRespType:
		gr := r.poll(m.From, m.Type, !m.Reject)
		r.logger.Infof("%x [quorum:%d] has received %d %s votes and %d vote rejections", r.id, r.quorum(), gr, m.Type, len(r.votes)-gr)
		switch r.quorum() {
		case gr:
			if r.state == StatePreCandidate {
				...
			} else {
				r.becomeLeader() // 收到 quorum 选票，选举成功
				r.bcastAppend()  // 广播 append 消息
			}
		case len(r.votes) - gr:
			// pb.MsgPreVoteResp contains future term of pre-candidate
			// m.Term > r.Term; reuse r.Term
			r.becomeFollower(r.Term, None)
		}
	}
	return nil
}
```