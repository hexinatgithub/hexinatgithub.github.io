---
date: 2022-05-23
categories: etcd raft 6.824
tags: etcd raft 6.824
title: etcd raft 模块源码阅读（2）
---

上篇文章我们介绍了 `raft` 模块，实际用户使用 `raft` 是通过 `node` 对象，因为本质上 `raft` 是线程不安全的，`node` 对象处理了并发请求。

先从 `Node interface` 看下 `node` 对象具体提供了哪些功能。

节点状态相关，用来获取 `node` 的状态，参加竞选，每隔单位时间发送计时器事件：
    - Tick()
    - Campaign(ctx context.Context) error
    - Status() Status

消息相关，发送消息，添加日志或发送读请求：
    - Propose(ctx context.Context, data []byte) error
    - Step(ctx context.Context, msg pb.Message) error
    - ReadIndex(ctx context.Context, rctx []byte) error

事件相关，处理事件或告知处理结果：
    - Ready() <-chan Ready
    - Advance()

其他节点状态相关，汇报网络状态或快照应用情况：
    - ReportUnreachable(id uint64)
    - ReportSnapshot(id uint64, status SnapshotStatus)

我们看到 `node` 抽象和 `raft` 是比较接近的，在 `raft` 上面封装了一层便于使用。接下来看下 `rawnode` 这个对象，看下具体做了什么。

```go
// Node represents a node in a raft cluster.
type Node interface {
	// Tick increments the internal logical clock for the Node by a single tick. Election
	// timeouts and heartbeat timeouts are in units of ticks.
	Tick()
	// Campaign causes the Node to transition to candidate state and start campaigning to become leader.
	Campaign(ctx context.Context) error
	// Propose proposes that data be appended to the log. Note that proposals can be lost without
	// notice, therefore it is user's job to ensure proposal retries.
	Propose(ctx context.Context, data []byte) error
	// ...

	// Step advances the state machine using the given message. ctx.Err() will be returned, if any.
	Step(ctx context.Context, msg pb.Message) error

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
	// ...

	// ReadIndex request a read state. The read state will be set in the ready.
	// Read state has a read index. Once the application advances further than the read
	// index, any linearizable read requests issued before the read request can be
	// processed safely. The read state will have the same rctx attached.
	// Note that request can be lost without notice, therefore it is user's job
	// to ensure read index retries.
	ReadIndex(ctx context.Context, rctx []byte) error

	// Status returns the current status of the raft state machine.
	Status() Status
	// ReportUnreachable reports the given node is not reachable for the last send.
	ReportUnreachable(id uint64)
	// ReportSnapshot reports the status of the sent snapshot. The id is the raft ID of the follower
	// who is meant to receive the snapshot, and the status is SnapshotFinish or SnapshotFailure.
	// Calling ReportSnapshot with SnapshotFinish is a no-op. But, any failure in applying a
	// snapshot (for e.g., while streaming it from leader to follower), should be reported to the
	// leader with SnapshotFailure. When leader sends a snapshot to a follower, it pauses any raft
	// log probes until the follower can apply the snapshot and advance its state. If the follower
	// can't do that, for e.g., due to a crash, it could end up in a limbo, never getting any
	// updates from the leader. Therefore, it is crucial that the application ensures that any
	// failure in snapshot sending is caught and reported back to the leader; so it can resume raft
	// log probing in the follower.
	ReportSnapshot(id uint64, status SnapshotStatus)
	// Stop performs any necessary termination of the Node.
	Stop()
}
```

看下 `rawnode` 结构体和常用的几个方法，可以看到 `rawnode` 在 `raft` 上面做了一下简单的封装，实现了 `Node` 接口。

```go
// Propose proposes data be appended to the raft log.
func (rn *RawNode) Propose(data []byte) error {
	return rn.raft.Step(pb.Message{
		Type: pb.MsgProp,
		From: rn.raft.id,
		Entries: []pb.Entry{
			{Data: data},
		}})
}

// Step advances the state machine using the given message.
func (rn *RawNode) Step(m pb.Message) error {
	// ignore unexpected local messages receiving over network
	if IsLocalMsg(m.Type) {
		return ErrStepLocalMsg
	}
	if pr := rn.raft.prs.Progress[m.From]; pr != nil || !IsResponseMsg(m.Type) {
		return rn.raft.Step(m)
	}
	return ErrStepPeerNotFound
}

// Ready returns the outstanding work that the application needs to handle. This
// includes appending and applying entries or a snapshot, updating the HardState,
// and sending messages. The returned Ready() *must* be handled and subsequently
// passed back via Advance().
func (rn *RawNode) Ready() Ready {
	rd := rn.readyWithoutAccept()
	rn.acceptReady(rd)
	return rd
}

// readyWithoutAccept returns a Ready. This is a read-only operation, i.e. there
// is no obligation that the Ready must be handled.
func (rn *RawNode) readyWithoutAccept() Ready {
	return newReady(rn.raft, rn.prevSoftSt, rn.prevHardSt)
}

// Advance notifies the RawNode that the application has applied and saved progress in the
// last Ready results.
func (rn *RawNode) Advance(rd Ready) {
	if !IsEmptyHardState(rd.HardState) {
		rn.prevHardSt = rd.HardState
	}
	rn.raft.advance(rd)
}
```

接下来主要看下 `node` 这个对象，`node` 对象统一处理了所有的并发请求，使得访问 `raft` 变得线程安全。

从 `node` 的结构体我们可以看出，这个对象是通过 `channel` 来使并发请求线程安全的。

```go
// node is the canonical implementation of the Node interface
type node struct {
	propc      chan msgWithResult
	recvc      chan pb.Message
	confc      chan pb.ConfChangeV2
	confstatec chan pb.ConfState
	readyc     chan Ready
	advancec   chan struct{}
	tickc      chan struct{}
	done       chan struct{}
	stop       chan struct{}
	status     chan chan Status

	rn *RawNode
}
```

从下面代码可以看出来，`node` 节点是使用了 `channel` 加 `select` 这个 `go` 的特性，使得所有的并发请求按照一定的顺序依次进行处理，
然后在通过 `rawnode` 对象调用 `raft` 的 API，保证了 `raft` 状态线程的安全。

`node` 没有使用 `lock` 而是使用 `channel` 使得处理并发请求的代码变得非常的干净清晰，而且统一放在了一个函数中，假设使用 `lock` 的情况下，
则所有需要使用共享内存的地方都需要考虑加锁的情况，使得处理并发请求的代码太过分散，也非常容易出错。

效率方面，由于使用 `channel` 的 `size` 为 0 ，`go` 内部其实是调用了 `yield` 函数，把 `cpu` 的使用权全交给别的 `goroutine` ，
随后 `run goroutine` 被 `select` 唤醒，然后进行处理，效率上面和 `lock` 应该是一样的，甚至还更有效率，因为锁内部是有个 `count` 计数器的，
会导致其他 `CPU` `count` 的缓存频繁 `invalid`，每个 `CPU` 需要频繁去访问内存获取 `count` 的值，有很大上下文的切换和读内存的开销，
`lock` vs `channel` 以后有时间开一个系列，详细了解一下内部的实现。

```go
func (n *node) run() {
	// ...

	for {
		// ...

		select {
		// TODO: maybe buffer the config propose if there exists one (the way
		// described in raft dissertation)
		// Currently it is dropped in Step silently.
		case pm := <-propc:
			m := pm.m
			m.From = r.id
			err := r.Step(m)
			if pm.result != nil {
				pm.result <- err
				close(pm.result)
			}
		case m := <-n.recvc:
			// filter out response message from unknown From.
			if pr := r.prs.Progress[m.From]; pr != nil || !IsResponseMsg(m.Type) {
				r.Step(m)
			}
		case cc := <-n.confc:
			// ...
		case <-n.tickc:
			n.rn.Tick()
		case readyc <- rd:
			n.rn.acceptReady(rd)
			advancec = n.advancec
		case <-advancec:
			n.rn.Advance(rd)
			rd = Ready{}
			advancec = nil
		case c := <-n.status:
			c <- getStatus(r)
		case <-n.stop:
			close(n.done)
			return
		}
	}
}
```

发送请求的实现就很清晰了，往 `channel` 里面发送消息就好了。

看一下 `step` 的实现，通过 `ch <- pm` 发送消息给 `run` 方法，有必要 `wait` 的话，`run` 方法处理完消息后，
把结果放入 `pm.result` 中，`step` 方法再把结果返回。

`step` ------- pm -------> `run` \
$~~~~$|  <------- result ----------  |

```go
// Step advances the state machine using msgs. The ctx.Err() will be returned,
// if any.
func (n *node) stepWithWaitOption(ctx context.Context, m pb.Message, wait bool) error {
	// ...

	// 接受消息的 channel，n.propc
	ch := n.propc
	pm := msgWithResult{m: m}
	if wait {
		pm.result = make(chan error, 1)
	}
	select {
	// 发送消息给 channel，之后会在 run 方法中获取到消息进行处理
	case ch <- pm:
		if !wait {
			return nil
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	select {
	// 等待消息的结果，并返回
	case err := <-pm.result:
		if err != nil {
			return err
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	return nil
}
```

思考：在统一的一个 `goroutine` 里面处理所有的事件，`node` 这个模型和 `Linux` 中总线设备驱动模型非常类似，我们都知道 `CPU` 需要处理 `I/O` 事件，
所有事件都是硬件通过 `Bus` 发通知给 `CPU` ，例如 disk，network，DMA 等，`CPU` 收到 `signal` 或 `exception` 后进行处理，
一个 `core` 一次只能处理一个事件。

通过 `etcd raft` 模块的源码可以看到，大神有非常深厚的计算机知识底蕴，如果想变成大神，除了通过源码不断和大神修炼外功外，也需要不断加深计算机基础，修炼内功心法。
