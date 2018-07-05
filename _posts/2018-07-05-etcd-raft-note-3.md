---
layout: post
date: 2018-07-05T13:46:43+08:00
title: etcd-raft 源码学习笔记（Linearizable Read 篇）
category: 源码阅读
---

这篇文章介绍 etcd-raft 如何实现 linearizable read（linearizable read 简单的说就是不返回 stale 数据，具体可以看这篇文章 [《Strong consistency models》](https://aphyr.com/posts/313-strong-consistency-models)）。

raft 论文第 8 节阐述了思路：

> Read-only operations can be handled without writing anything into the log. However, with no additional measures, this would run the risk of returning stale data, since the leader responding to the request might have been superseded by a newer leader of which it is unaware. Linearizable reads must not return stale data, and Raft needs two extra precautions to guarantee this without using the log. First, a leader must have the latest information on which entries are committed. The Leader Completeness Property guarantees that a leader has all committed entries, but at the start of its term, it may not know which those are. To find out, it needs to commit an entry from its term. Raft handles this by having each leader commit a blank no-op entry into the log at the start of its term. Second, a leader must check whether it has been deposed before processing a read-only request (its information may be stale if a more recent leader has been elected). Raft handles this by having the leader exchange heartbeat messages with a majority of the cluster before responding to read-only requests.

在收到读请求时，leader 节点保存下当前的 commit index，并往 peers 发送心跳。如果确定该节点依然是 leader，则只需要等到该 commit index 的 log entry 被 apply 到状态机时就可以返回客户端结果。

我们先通过位于 etcd/etcdserver 目录下的样例来看看应用层是如何使用 ReadIndex 来保证 linearizable read 的：

```go
// v3_server.go

type RaftKV interface {
	Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error)
	Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error)
	DeleteRange(ctx context.Context, r *pb.DeleteRangeRequest) (*pb.DeleteRangeResponse, error)
	Txn(ctx context.Context, r *pb.TxnRequest) (*pb.TxnResponse, error)
	Compact(ctx context.Context, r *pb.CompactionRequest) (*pb.CompactionResponse, error)
}

func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
	var resp *pb.RangeResponse
	var err error

	if !r.Serializable {
		err = s.linearizableReadNotify(ctx)  // 等待 linearizableReadNotify 返回 才能继续往下走
		if err != nil {
			return nil, err
		}
	}
	// 读取数据逻辑 省略..
	...
	return resp, err
}
```

在读请求 ```Range``` 执行前，调用了 ```linearizableReadNotify```：

```go
func (s *EtcdServer) linearizableReadNotify(ctx context.Context) error {
	s.readMu.RLock()
	nc := s.readNotifier
	s.readMu.RUnlock()

	// signal linearizable loop for current notify if it hasn't been already
	select {
	case s.readwaitc <- struct{}{}:
	default:
	}

	// wait for read state notification
	select {
	case <-nc.c:
		return nc.err
	case <-ctx.Done():
		return ctx.Err()
	case <-s.done:
		return ErrStopped
	}
}
```

```linearizableReadNotify``` 往 ```readwaitc``` 发送个空的结构体，并且等待 ```nc.c``` 的返回。```readwaitc``` 是在另外的 goroutine ```linearizableReadLoop``` 里监听的：

```go

func (s *EtcdServer) linearizableReadLoop() {
	var rs raft.ReadState

	for {
		ctx := make([]byte, 8)
		binary.BigEndian.PutUint64(ctx, s.reqIDGen.Next())  // ctx 即请求唯一标识 reqId

		select {
		case <-s.readwaitc:  // 监听 readwaitc
		case <-s.stopping:
			return
		}

		nextnr := newNotifier()
		nr := s.readNotifier
		s.readNotifier = nextnr

		s.r.ReadIndex(cctx, ctx)  // 调用 ReadIndex 接口，往 recvc channel 发送 type 为 pb.MsgReadIndex 的请求

		var (
			timeout bool
			done    bool
		)
		for !timeout && !done {
			select {
			case rs = <-s.r.readStateC:  // 收到 ready 对象时，会往 readStateC channel 传回来 readState，见 etcd/etcdserver/raft.go 文件的 func (r *raftNode) start(rh *raftReadyHandler)
				done = bytes.Equal(rs.RequestCtx, ctx)  // 比较下 reqId 是否一致
			case <-time.After(s.Cfg.ReqTimeout()):
				nr.notify(ErrTimeout)
				timeout = true
			case <-s.stopping:
				return
			}
		}
		if !done {
			continue
		}

        // 等待 readState 里的 index，也就是收到 pb.MsgReadIndex 请求时，leader 节点当前的 commit index 被 apply 到状态机时，此时调用 nr.notify(nil) 通知应用层可以读取状态机里的数据了，确保读到的不是 stale 数据
		if ai := s.getAppliedIndex(); ai < rs.Index {
			select {
			case <-s.applyWait.Wait(rs.Index):
			case <-s.stopping:
				return
			}
		}
		// unblock all l-reads requested at indices before rs.Index
		nr.notify(nil)
	}
}
```

在 ```linearizableReadLoop``` 调用 ```nr.notify``` 后，```linearizableReadNotify``` 从 select 阻塞中返回，此时就可以继续走 ```Range``` 的逻辑，读取数据，返回给客户端。

从上面的例子，我们了解了应用层如何使用 Node 的 ```ReadIndex``` 接口来实现 linearizable read。下面我们来介绍 ```ReadIndex``` 这个新接口：

```go
// Node represents a node in a raft cluster.
type Node interface {
	// Propose proposes that data be appended to the log.
	Propose(ctx context.Context, data []byte) error

	// Ready returns a channel that returns the current point-in-time state.
	// Users of the Node must call Advance after retrieving the state returned by Ready.
	//
	// NOTE: No committed entries from the next Ready may be applied until all committed entries
	// and snapshots from the previous one have finished.
	Ready() <-chan Ready

	// Advance notifies the Node that the application has saved progress up to the last Ready.
	// It prepares the node to return the next available Ready.
	//
	// The application should generally call Advance after it applies the entries in last Ready.
	//
	// However, as an optimization, the application may call Advance while it is applying the
	// commands. For example. when the last Ready contains a snapshot, the application might take
	// a long time to apply the snapshot data. To continue receiving Ready without blocking raft
	// progress, it can call Advance before finishing applying the last ready.
	Advance()

	// ReadIndex request a read state. The read state will be set in the ready.
	// Read state has a read index. Once the application advances further than the read
	// index, any linearizable read requests issued before the read request can be
	// processed safely. The read state will have the same rctx attached.
	ReadIndex(ctx context.Context, rctx []byte) error
}

func (n *node) ReadIndex(ctx context.Context, rctx []byte) error {
	return n.step(ctx, pb.Message{Type: pb.MsgReadIndex, Entries: []pb.Entry{{Data: rctx}}})
}
```

上篇文章 [《etcd-raft 源码学习笔记（概览篇）》](http://masutangu.com/2018/07/etcd-raft-note-2/) 提到当节点为 leader 时，```step``` 被设置为 ```stepLeader``` 。我们来看看 ```stepLeader``` 是如何处理 type 为 pb.MsgReadIndex 的 readIndexReq 的：

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	case pb.MsgReadIndex:
		// raft 5.4 safty 检查
		if r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(r.raftLog.committed)) != r.Term {
			// Reject read only request when this leader has not committed any log entry at its term.
			return nil
		}

		// thinking: use an interally defined context instead of the user given context.
		// We can express this in terms of the term and index instead of a user-supplied value.
		// This would allow multiple reads to piggyback on the same message.
		switch r.readOnly.option {
		case ReadOnlySafe:
			r.readOnly.addRequest(r.raftLog.committed, m)  // 把 readIndexReq 保存起来
			r.bcastHeartbeatWithCtx(m.Entries[0].Data)  // 广播心跳包
		}
		return nil
	}
	return nil
}

	
```

收到 readIndexReq 后，首先调用 ```r.readOnly.addRequest``` 保存下来，然后调用 ```bcastHeartbeatWithCtx``` 广播心跳包， ctx 即唯一标识 readIndexReq 的 reqId。再看看 ```stepLeader``` 如何处理心跳回包：

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	switch m.Type {
	case pb.MsgHeartbeatResp:
		pr.RecentActive = true
		pr.resume()

		if r.readOnly.option != ReadOnlySafe || len(m.Context) == 0 {
			return nil
		}

		ackCount := r.readOnly.recvAck(m)
		if ackCount < r.quorum() {  // 判断是否收到 quorum 的心跳回包
			return nil
		}

        // 收到 quorum 的心跳回包了，把 readIndexReq 依次 append r.readStates 中，返回 ready 对象时会包含 r.readStates
		rss := r.readOnly.advance(m)
		for _, rs := range rss {
			req := rs.req
			if req.From == None || req.From == r.id { // from local member
				r.readStates = append(r.readStates, ReadState{Index: rs.index, RequestCtx: req.Entries[0].Data})
			} else {
				r.send(pb.Message{To: req.From, Type: pb.MsgReadIndexResp, Index: rs.index, Entries: req.Entries})
			}
		}
	return nil
	}
	return nil
}
```

调用 ```r.readOnly.recvAck```，根据 readIndeReq 的 reqId 统计收到心跳回包的数量，如果超过 quonum 表示该节点依然是 leader，此时从 ```r.readOnly.advance``` 拿到保存的 readIndexReq，append 到 ```r.readStates``` 中。之后调用 ```newReady``` 会把 ```r.readStates``` 返回给应用层，应用层取出 readIndexReq 中的 commit index，等到其被 apply 到状态机就可以允许读操作了。

```go
func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
	rd := Ready{
		Entries:          r.raftLog.unstableEntries(),
		CommittedEntries: r.raftLog.nextEnts(),
		Messages:         r.msgs,
	}
	...
	
	if len(r.readStates) != 0 {
		rd.ReadStates = r.readStates  // 附上 r.readStates
	}
	...
	return rd
}
```