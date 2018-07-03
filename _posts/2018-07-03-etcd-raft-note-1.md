---
layout: post
date: 2018-07-03T13:21:23+08:00
title: etcd-raft 源码学习笔记（示例篇）
category: 源码阅读
---

本系列文章为 [etcd-raft](https://github.com/coreos/etcd/tree/master/raft) 源码阅读笔记，采用自顶向下的方式。这篇是开篇，首先来看看 etcd 提供的基于 raft 库实现的 kv store 示例，代码目录位于 contrib/raftexample。

从 main 函数开始读起：

```go
func main() {
    ...
    proposeC := make(chan string)
    defer close(proposeC)
    
	var kvs *kvstore
	getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }
	commitC, errorC, snapshotterReady := newRaftNode(*id, strings.Split(*cluster, ","), *join, getSnapshot, proposeC, confChangeC)

    kvs = newKVStore(<-snapshotterReady, proposeC, commitC, errorC)
    ...
}
```

```getSnapshot``` 为应用层 kv 提供的 snapshot 方法，在 raft 中调用该方法进行 snapshot。```proposeC``` 是应用层 kv 向 raftNode 发送请求的 channel，```commitC``` 为 raftNode 通知应用层 kv 已经提交的请求的 channel。

先看看 ```newKVStore``` 的实现：

```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *string, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	// replay log into key-value map
	s.readCommits(commitC, errorC)
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
}
```

```readCommits``` 方法从 ```commitC``` 中读取已经提交的请求进行处理：

```go
func (s *kvstore) readCommits(commitC <-chan *string, errorC <-chan error) {
	for data := range commitC {
		var dataKv kv
		dec := gob.NewDecoder(bytes.NewBufferString(*data))  // decode 
		s.mu.Lock()
		s.kvStore[dataKv.Key] = dataKv.Val  // 更新 kv
		s.mu.Unlock()
	}
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```

再看看 ```newRaftNode``` ，其会调用 ```startRaft``` 启动底层 raft：

```go
func (rc *raftNode) startRaft() {
	oldwal := wal.Exist(rc.waldir)
	rc.wal = rc.replayWAL()

	rpeers := make([]raft.Peer, len(rc.peers))
	for i := range rpeers {
		rpeers[i] = raft.Peer{ID: uint64(i + 1)}
	}
	c := &raft.Config{
		ID:              uint64(rc.id),
		ElectionTick:    10,
		HeartbeatTick:   1,
		Storage:         rc.raftStorage,
		MaxSizePerMsg:   1024 * 1024,
		MaxInflightMsgs: 256,
	}

	if oldwal {
		rc.node = raft.RestartNode(c)
	} else {
		startPeers := rpeers
		if rc.join {
			startPeers = nil
		}
		rc.node = raft.StartNode(c, startPeers)
	}

	go rc.serveRaft()  // 监听 http 
	go rc.serveChannels()  // 监听 proposeC channel，读取应用层请求 进行处理
}
```

```serveChannels``` 就做了两个事，1. 另起一个 goroutine，接收 proposeC 里发送自应用层的请求，通过 ```Propose``` 方法交给底层 raft 处理；2. 调用 ```Ready``` 方法，接收发送自 raft 的 ready 对象，调用 ```publishEntries``` 将已经提交的 entries 发送到 ```commitC``` channel，交由应用层处理，再调用 ```Advance``` 方法通知底层 raft 准备好接收下一个 ready 对象了。

```go
func (rc *raftNode) serveChannels() {
	defer rc.wal.Close()

	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	// send proposals over raft
	go func() {
		var confChangeCount uint64 = 0

		for rc.proposeC != nil && rc.confChangeC != nil {
			select {
			case prop, ok := <-rc.proposeC:
				if !ok {
					rc.proposeC = nil
				} else {
					// blocks until accepted by raft state machine
					rc.node.Propose(context.TODO(), []byte(prop))  // 调用 Propose 发送给 raft 请求
				}
			}
		}
		// client closed channel; shutdown raft if not already
		close(rc.stopc)
	}()

	// event loop on raft state machine updates
	for {
		select {
		case <-ticker.C:
			rc.node.Tick()

		// store raft entries to wal, then publish over commit channel
		case rd := <-rc.node.Ready():  // 应用层调用 Ready() 获取 ready 对象
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
			}
			rc.node.Advance()  // 应用层调用 Advance() 通知 raft 已经处理完 ready 对象 

		case <-rc.stopc:
			rc.stop()
			return
		}
	}
}
```

```publishEntries``` 将 ready 对象里的 ```CommittedEntries``` 发送到 ```commitC```，由应用层 kv 处理：

```go
// publishEntries writes committed log entries to commit channel and returns
// whether all entries could be published.
func (rc *raftNode) publishEntries(ents []raftpb.Entry) bool {
	for i := range ents {
		switch ents[i].Type {
		case raftpb.EntryNormal:
			if len(ents[i].Data) == 0 {
				// ignore empty messages
				break
			}
			s := string(ents[i].Data)
			select {
			case rc.commitC <- &s:  // 发送到 commitC channel
			case <-rc.stopc:
				return false
			}
		}

		// after commit, update appliedIndex
		rc.appliedIndex = ents[i].Index

		// special nil commit to signal replay has finished
		if ents[i].Index == rc.lastIndex {
			select {
			case rc.commitC <- nil:
			case <-rc.stopc:
				return false
			}
		}
	}
	return true
}
```

整体架构如下，RaftNode 的角色为应用层和底层 raft 的桥梁：

<img src="/assets/images/etcd-raft-node-1/illustration-1.png" width="800"/>


可以看出，应用层主要用到 raft.Node 的 ```Propose```、```Ready```、```Advance```三个接口：

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
}
```

