---
date: 2022-05-20
categories: etcd raft 6.824
tags: etcd raft 6.824
title: etcd raft 模块源码阅读（1）
---

# 摘要

[raft](http://nil.csail.mit.edu/6.824/2022/papers/raft-extended.pdf) 是一个用来复制日志的一致性算法，相比于其他一致性算法，例如 Paxos，raft 更加易于理解和学习，并且还提供了例如快照，集群配置信息变更等功能。

本系列主要是阅读并理解学习 etcd(version: release-3.5) raft 模块的源码，主要关注点在 leader 的选举，log 的复制，其他功能会忽略，有兴趣的小伙伴可以继续深入阅读。在阅读源码之前，建议大家先看下 raft 的[论文](http://nil.csail.mit.edu/6.824/2022/papers/raft-extended.pdf)和[视频](https://youtu.be/R2-9bsKmEbo)，这样阅读起来会更加的高效。

## 概览

本篇文章主要是对 raft 模块的整体结构进行一个梳理，raft 算法本身涉及到很多点，需要一一梳理才能全部理解，相应的代码量也比较多，在我们对 raft 的主干有了清晰的认识后，可以通过对照论文，再详细的去阅读代码中的细节实现。

## 目录结构

开始阅读代码之前，先梳理一下 raft 的目录结构，了解一下各个模块的具体功能。

```console
/workspaces/etcd/raft $ ls -l | grep '^d'
drwxrwxrwx+ 3 codespace root        4096 May 16 09:51 confchange
drwxrwxrwx+ 3 codespace root        4096 May 16 09:51 quorum
drwxrwxrwx+ 2 codespace root        4096 May 16 14:39 raftpb
drwxrwxrwx+ 2 codespace root        4096 May 16 14:39 rafttest
drwxrwxrwx+ 2 codespace root        4096 May 16 14:39 testdata
drwxrwxrwx+ 2 codespace root        4096 May 16 09:51 tracker
```

`confchange` 目录主要是存储 raft 配置变更相关的代码，这部分我们不进行讨论。

`quorum` 目录存储的是 vote，log committed 相关的代码，用来计算是否赢选举，计算最新的 committed index 用来决定哪些 log 已经复制到大多数的 peer 中。

`raftpb` 目录消息格式，用来在 peer 之间进行通信。

`rafttest` 目录包含了大量的单元测试代码，这块内容很值得学习，在工作学习过程中接触到的很多代码会和第三方库耦合在一起，需要网络数据库等环境才能进行测试，没法进行单元测试，这样随着业务的变更代码的质量会迅速的下降，开发工作会变得非常的繁琐和辛苦，如果能把大量通用的逻辑全部抽出来放到一块，和其他模块进行解耦的话，能保证这块的代码质量，开发工作也会减轻不少。如果在开发的过程中，能将测试考虑进来，在编写代码的过程中应该会有不一样的思考吧，这块有时间在深入学习和分享给大家，这里先不深入讨论了。

`testdata` 目录给单元测试代码提供测试数据的，忽略。

`tracker` 目录主要用于给 leader 跟踪每个 follower 的状态，包括每个 follower ack 的 log index，follower 的状态，vote 情况等。

介绍了几个子模块，接下来我们看下几个主要文件:

1. `storage.go` 主要是抽象了 Storage 的 interface，raft 没有像论文中一样集成了存储日志，生成快照的功能，而是把存储的功能解耦出去，留给开发者去实现。
2. `log.go` 主要就是包含了 raft 的 log 相关的代码，包括追加日志，获取日志，判断 log 是否比 peer 的日志新，日志是否和 peer 的有冲突等功能
3. `raft.go` `rawnode.go` `node.go` 是 raft 包的主干代码，本质是维护了一个状态机，通过事件（消息）驱动的形式更新内部状态，提供获取内部状态等功能，接下来我们主要讲解这三个文件。

## 功能实现

***Note: `// ... ` 表示这部分代码忽略，不影响阅读。***

我们先从 `rarf.go` 这个文件切入，了解 raft 状态机具体的实现。

我们从论文中了解到，为了避免不必要的选举，需要每一个 raft peer 设置一个随机的选举超时时间 electionTimeout，这里实现了全局的加锁的 rand ，用来生成随机选举超时时间。

```go
// lockedRand is a small wrapper around rand.Rand to provide
// synchronization among multiple raft groups. Only the methods needed
// by the code are exposed (e.g. Intn).
type lockedRand struct {
	mu   sync.Mutex
	rand *rand.Rand
}

func (r *lockedRand) Intn(n int) int {
	r.mu.Lock()
	v := r.rand.Intn(n)
	r.mu.Unlock()
	return v
}

var globalRand = &lockedRand{
	rand: rand.New(rand.NewSource(time.Now().UnixNano())),
}
```

现在来看看生成 raft 对象的时候需要的一些主要配置参数

```go
// Config contains the parameters to start a raft.
type Config struct {
	// ID is the identity of the local raft. ID cannot be 0.
	// ID 是本 raft 的全局唯一标识，用来和其他 peer 进行区分
	ID uint64
	
	// ElectionTick 和 HeartbeatTick 是以 Node.Tick 为单位的，这样使用者可以自己定义
	// 时间单位
	
	// ElectionTick is the number of Node.Tick invocations that must pass between
	// elections. That is, if a follower does not receive any message from the
	// leader of current term before ElectionTick has elapsed, it will become
	// candidate and start an election. ElectionTick must be greater than
	// HeartbeatTick. We suggest ElectionTick = 10 * HeartbeatTick to avoid
	// unnecessary leader switching.
	// ElectionTick 是选举超时时间，建议设置为 10 * HeartbeatTick ，避免不必要的选举
	ElectionTick int
	// HeartbeatTick is the number of Node.Tick invocations that must pass between
	// heartbeats. That is, a leader sends heartbeat messages to maintain its
	// leadership every HeartbeatTick ticks.
	// HeartbeatTick 是心跳间隔时间，每 HeartbeatTick 发送一次心跳给 follower 用来维护领导权
	HeartbeatTick int
	
	// Storage is the storage for raft. raft generates entries and states to be
	// stored in storage. raft reads the persisted entries and states out of
	// Storage when it needs. raft reads out the previous state and configuration
	// out of storage when restarting.
	// Storage 是用来存储快照和日志的，并提供获取日志快照的功能
	Storage Storage
	// Applied is the last applied index. It should only be set when restarting
	// raft. raft will not return entries to the application smaller or equal to
	// Applied. If Applied is unset when restarting, raft might return previous
	// applied entries. This is a very application dependent configuration.
	// Applied 是已经应用的 log index，已经应用的日志保证会存储在 Storage 里
	Applied uint64
	
	// MaxSizePerMsg, MaxCommittedSizePerReady, MaxUncommittedEntriesSize, MaxInflightMsgs
	// 是用来限制每个消息的大小，已经提交并还未应用的日志大小，未提交的日志大小和 
	// raft 发送而没有回复的消息数量，主要用来控制内存和网络使用量，避免过大的网络请求和内存使用
	// MaxSizePerMsg limits the max byte size of each append message. Smaller
	// value lowers the raft recovery cost(initial probing and message lost
	// during normal operation). On the other side, it might affect the
	// throughput during normal replication. Note: math.MaxUint64 for unlimited,
	// 0 for at most one entry per message.
	MaxSizePerMsg uint64
	// MaxCommittedSizePerReady limits the size of the committed entries which
	// can be applied.
	MaxCommittedSizePerReady uint64
	// MaxUncommittedEntriesSize limits the aggregate byte size of the
	// uncommitted entries that may be appended to a leader's log. Once this
	// limit is exceeded, proposals will begin to return ErrProposalDropped
	// errors. Note: 0 for no limit.
	MaxUncommittedEntriesSize uint64
	// MaxInflightMsgs limits the max number of in-flight append messages during
	// optimistic replication phase. The application transportation layer usually
	// has its own sending buffer over TCP/UDP. Setting MaxInflightMsgs to avoid
	// overflowing that sending buffer. TODO (xiangli): feedback to application to
	// limit the proposal rate?
	MaxInflightMsgs int
	
	// ...
	
	// ReadOnlyOption specifies how the read only request is processed.
	//
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	//
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	// CheckQuorum MUST be enabled if ReadOnlyOption is ReadOnlyLeaseBased.
	// ReadOnlyOption 主要是 read 操作的时候是否需要获取 quorum，
	// 即是否需要发送给每一个 follower，来保证强一致性
	ReadOnlyOption ReadOnlyOption
	
	// ...
}
```

`raft` 的一些主要属性

```go
type raft struct {
	// id 是 raft 的唯一标识
	id uint64

	// Term 是 logic clock，用来表示当前 leader 的任期
	Term uint64
	// Vote 表示投票给了谁
	Vote uint64

	// readStates 一开始理解这个属性的时候有点困惑，
	// 其实这个是 follower 把 read 请求发给了 leader 后,
	// 所返回的结果, raft 是强一致性，所以每次读操作都需要发给 leader ，
	// 等待 leader 返回结果后，在发给 client，readStates 是读操作结果的缓存。
	// 这样做读操作不需要 block ，等结果返回后在发送给 client。
	readStates []ReadState

	// the log
	// raftlog 是存储日志的对象，这里包括已经存储到硬盘和内存中的日志
	raftLog *raftLog

	// maxMsgSize, maxUncommittedSize, uncommittedSize 用来限制内存和网络的使用
	maxMsgSize         uint64
	maxUncommittedSize uint64
	// an estimate of the size of the uncommitted tail of the Raft log. Used to
	// prevent unbounded log growth. Only maintained by the leader. Reset on
	// term changes.
	uncommittedSize uint64
	// TODO(tbg): rename to trk.
	// prs 是 leader 用来跟踪每一个 follower 的状态的，包括日志状态，投票情况
	prs tracker.ProgressTracker

	// state 用来记录当前状态机的状态
	state StateType

	// ...

	// msgs 存储需要处理的一些消息，比如接受到的还未存储并应用的消息
	msgs []pb.Message

	// the leader id
	// lead 是指当前的 leader id
	lead uint64
	// ...

	readOnly *readOnly

	// number of ticks since it reached last electionTimeout when it is leader
	// or candidate.
	// number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	electionElapsed int

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int

	checkQuorum bool
	// ...

	heartbeatTimeout int
	electionTimeout  int
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int
	// ...

	// tick 是每隔一段心跳时间用来发送心跳或者选举时间过了发送选举请求的函数
	tick func()
	// step 函数是用来处理接受到的消息，包括本地的消息和来至 peer 的消息
	// 这个函数是整个事件驱动的关键，所有的事件都要通过这个函数进行处理，
	// 驱动整个状态机内部状态的更新
	step stepFunc

	// ...

	// pendingReadIndexMessages is used to store messages of type MsgReadIndex
	// that can't be answered as new leader didn't committed any log in
	// current term. Those will be handled as fast as first log is committed in
	// current term.
	// pendingReadIndexMessages 是用来缓存所有的读操作请求的，因为 leader 并没有应用任何本
	// 任期内的日志，这些请求先缓存起来等 leader commmit 第一个日志后进行处理
	pendingReadIndexMessages []pb.Message
}
```

对 `raft` 的一些主要方法进行分类整理一下：

1. 消息相关，包括了发送消息，广播心跳，处理接受到的消息
   - send(m pb.Message)
   - sendAppend(to uint64)
   - maybeSendAppend(to uint64, sendIfEmpty bool) bool
   - sendHeartbeat(to uint64, ctx []byte)
   - bcastAppend()
   - bcastHeartbeat()
   - bcastHeartbeatWithCtx(ctx []byte)
   - appendEntry(es ...pb.Entry) (accepted bool)
   - Step(m pb.Message) error
   - stepLeader(r *raft, m pb.Message) error
   - stepCandidate(r *raft, m pb.Message) error
   - stepFollower(r *raft, m pb.Message) error
   - handleAppendEntries(m pb.Message)
   - handleAppendEntries(m pb.Message)
   - handleSnapshot(m pb.Message)
   - responseToReadIndexReq(req pb.Message, readIndex uint64) pb.Message
2. 告知消息处理结果，比如应用，存储日志，应用快照等，主要是应用层和 `raft` 层进行信息同步，让 `raft` 更新内部的状态
   - advance(rd Ready)
3. 定时器相关的函数，例如心跳计时器，选举超时计时器超时时，触发函数调用
   - tickElection()
   - tickHeartbeat()
4. 状态相关，用于变更 `raft` 状态机的状态
   - becomeFollower(term uint64, lead uint64)
   - becomeCandidate()
   - becomeLeader()
   - hup(t CampaignType)
   - campaign(t CampaignType)
   - poll(id uint64, t pb.MessageType, v bool) (granted int, rejected int, result quorum.VoteResult) ，poll 函数放在这一类主要是决定是否更新状态为 leader
5. 其他函数和配置变更，设置状态相关，还有一些辅助函数，有些会忽略，有些遇到了在详细解释。

对函数进行分类以后，对 `raft` 的功能就有了比较清晰的了解了，首先 `raft` 本身是可以接受发送消息请求（eg: send(m pb.Message)），也可以接受消息（eg: Step(m pb.Message)）。

`raft` 接受消息的时候，会根据消息维护自身的状态，例如保持 follower 状态，更新 `raft.*raftLog.committedIndex` ，创建拒绝投票或者准许投票发送请求等。

`raft` 接受发送消息请求的时候，本身并不会去发送网络请求，而是根据相应的请求维护好状态机后，把消息存储起来，通过 Ready 对象封装需要处理的消息和快照等信息，交给应用层去进行处理，应用层存储完日志或者发送完网络请求后，把结果通过 `advance(rd Ready)` 告知 `raft` 对象，`raft` 对象在进行状态的更新，例如删除日志，更新 `raft.*raftLog` 状态等。

其他定时器相关的功能，也是一样的结构，通过应用层告知 `raft` 定时器超时了，`raft` 处理相关的事件，例如发送心跳，参加选举，然后再改变状态机的状态。

### 源码解析

对 `raft` 有了具体的了解以后，接下来我们具体来看下代码的实现。

先从发送消息进行切入，可以看到发送消息和心跳只是简单的把消息放入消息队列中，并不做任何的处理。

```go
// send schedules persisting state to a stable storage and AFTER that
// sending the message (as part of next Ready message processing).
func (r *raft) send(m pb.Message) {
	if m.From == None {
		m.From = r.id
	}
	// ...
	r.msgs = append(r.msgs, m)
}

// sendHeartbeat sends a heartbeat RPC to the given peer.
func (r *raft) sendHeartbeat(to uint64, ctx []byte) {
	commit := min(r.prs.Progress[to].Match, r.raftLog.committed)
	m := pb.Message{
		To:      to,
		Type:    pb.MsgHeartbeat,
		Commit:  commit,
		Context: ctx,
	}

	r.send(m)
}

// sendAppend sends an append RPC with new entries (if any) and the
// current commit index to the given peer.
func (r *raft) sendAppend(to uint64) {
	r.maybeSendAppend(to, true)
}

// maybeSendAppend sends an append RPC with new entries to the given peer,
// if necessary. Returns true if a message was sent. The sendIfEmpty
// argument controls whether messages with no entries will be sent
// ("empty" messages are useful to convey updated Commit indexes, but
// are undesirable when we're sending multiple messages in a batch).
func (r *raft) maybeSendAppend(to uint64, sendIfEmpty bool) bool {
	// 获取该 to follower 的进度信息
	pr := r.prs.Progress[to]
	if pr.IsPaused() {
		return false
	}
	m := pb.Message{}
	m.To = to

	// 前一个日志的 Term
	term, errt := r.raftLog.term(pr.Next - 1)
	// 获取需要发送给该 follower 的日志
	ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)
	if len(ents) == 0 && !sendIfEmpty {
		return false
	}

	if errt != nil || erre != nil { // send snapshot if we failed to get term or entries
		// snapshot 相关的处理忽略
		// ...
	} else {
		// 根据 follower 的状态，设置 pb.Message 的属性
		m.Type = pb.MsgApp
		m.Index = pr.Next - 1
		m.LogTerm = term
		m.Entries = ents
		m.Commit = r.raftLog.committed
		if n := len(m.Entries); n != 0 {
			switch pr.State {
			// optimistically increase the next when in StateReplicate
			case tracker.StateReplicate:
				// 这块主要是更新发送并且没有回复的消息的数量，用来控制网络和内存大小
				last := m.Entries[n-1].Index
				pr.OptimisticUpdate(last)
				pr.Inflights.Add(last)
			case tracker.StateProbe:
				pr.ProbeSent = true
			default:
				r.logger.Panicf("%x is sending append in unhandled state %s", r.id, pr.State)
			}
		}
	}
	r.send(m)
	return true
}

// bcastAppend sends RPC, with entries to all peers that are not up-to-date
// according to the progress recorded in r.prs.
func (r *raft) bcastAppend() {
	// prs 跟踪了每个 follower 的状态，给每一个 follower 发送消息
	r.prs.Visit(func(id uint64, _ *tracker.Progress) {
		if id == r.id {
			return
		}
		r.sendAppend(id)
	})
}

// bcastHeartbeat sends RPC, without entries to all the peers.
func (r *raft) bcastHeartbeat() {
	lastCtx := r.readOnly.lastPendingRequestCtx()
	if len(lastCtx) == 0 {
		r.bcastHeartbeatWithCtx(nil)
	} else {
		r.bcastHeartbeatWithCtx([]byte(lastCtx))
	}
}
```

接下来我们来看一下收到消息的处理过程，应用层接受到消息通过调用 `raft.Step(m pb.Message)` 通知 `raft`， `raft.Step(m pb.Message)` 函数首先会忽略掉一些（`m.Term < r. Term`）过期的消息，如果 `m.Term > r.Term` 则会变成 `follower` 状态。

可以看到本地消息请求，也是通过消息请求的方式来进行处理的，可以这样理解，消息看做是命令，该命令可以是从远程传过来的，也可以是本地应用层给的。

如果是本地参加选举消息（`pb.MsgHup`）或者是远程投票请求（`pb.MsgVote`）都会进行投票相关的处理，否则都进入到 `r.step(m)` 进行处理。

```go
func (r *raft) Step(m pb.Message) error {
	// Handle the message term, which may result in our stepping down to a follower.
	switch {
	case m.Term == 0:
		// local message
	case m.Term > r.Term:
		// ...
		switch {
			// ...
		default:
			// ...
			// m.Term 大于 r.Term，更改为 follower 状态
			if m.Type == pb.MsgApp || m.Type == pb.MsgHeartbeat || m.Type == pb.MsgSnap {
				r.becomeFollower(m.Term, m.From)
			} else {
				r.becomeFollower(m.Term, None)
			}
		}

	case m.Term < r.Term:
		// m.Term 小于当前 r.Term ，则需要忽略该条消息
		// ...
		return nil
	}

	switch m.Type {
	// 自身开始预选举或者选举
	case pb.MsgHup:
	// ...
	// 处理预选举或者选举请求
	case pb.MsgVote, pb.MsgPreVote:
		// ...
	default:
		err := r.step(r, m)
		// ...
	}
	return nil
}
```

`r.step(m)` 函数根据 `raft` 的不同状态，进行不同的处理。

当 `raft` 为 `leader` 状态的时候，先处理本地请求，例如发送心跳，发送消息，读操作请求，如果不是本地的请求，例如 `Append` 消息的 `ACK` 包，`HeartBeat` 的 `ACK` 包等，则根据消息体更新状态机或者产生新的消息。

当 `raft` 为 `Candidate` 状态的时候，则调用 `stepCandidate(r *raft, m pb.Message) error` ，同理，当  `raft` 为 `Follower` 状态的时候，则调用 `stepFollower(r *raft, m pb.Message) error`。

```go
func stepLeader(r *raft, m pb.Message) error {
	// These message types do not require any progress for m.From.
	// 这些都是些本地请求，不需要 m.From 属性
	switch m.Type {
	case pb.MsgBeat:
		// ...
		// 发送心跳
		return nil
	case pb.MsgCheckQuorum:
		// ...
	case pb.MsgProp:
		// ...
		// 处理发送请求
		return nil
	case pb.MsgReadIndex:
		// ...
		// 处理读操作
		return nil
	}

	// ...
	pr := r.prs.Progress[m.From]
	// ...
	switch m.Type {
	case pb.MsgAppResp:
		// ...
		if m.Reject {
			// 当请求拒绝的时候
			// ...
		} else {
			// 接受了 Append 请求，更新内部状态
			// ...
		}
	case pb.MsgHeartbeatResp:
		// 处理心跳消息的回复
		// ...
	case pb.MsgSnapStatus:
		// ...
	case pb.MsgUnreachable:
		// ...
	}
	return nil
}
```

现在我们对 `raft` 发送接受消息的机制有了一定了解后，我们深入到 `raft` 内部状态去看看。

之前所说，应用层通过 `advance(rd Ready)` 来和 `raft` 进行状态的同步，那为了了解 `raft` 内部状态，我们看一下和 `Ready` 相关的代码。

```go
// Ready encapsulates the entries and messages that are ready to read,
// be saved to stable storage, committed or sent to other peers.
// All fields in Ready are read-only.
type Ready struct {
	// The current volatile state of a Node.
	// SoftState will be nil if there is no update.
	// It is not required to consume or store SoftState.
	*SoftState

	// The current state of a Node to be saved to stable storage BEFORE
	// Messages are sent.
	// HardState will be equal to empty state if there is no update.
	pb.HardState

	// ReadStates can be used for node to serve linearizable read requests locally
	// when its applied index is greater than the index in ReadState.
	// Note that the readState will be returned when raft receives msgReadIndex.
	// The returned is only valid for the request that requested to read.
	ReadStates []ReadState

	// Entries specifies entries to be saved to stable storage BEFORE
	// Messages are sent.
	Entries []pb.Entry

	// Snapshot specifies the snapshot to be saved to stable storage.
	Snapshot pb.Snapshot

	// CommittedEntries specifies entries to be committed to a
	// store/state-machine. These have previously been committed to stable
	// store.
	CommittedEntries []pb.Entry

	// Messages specifies outbound messages to be sent AFTER Entries are
	// committed to stable storage.
	// If it contains a MsgSnap message, the application MUST report back to raft
	// when the snapshot has been received or has failed by calling ReportSnapshot.
	Messages []pb.Message

	// MustSync indicates whether the HardState and Entries must be synchronously
	// written to disk or if an asynchronous write is permissible.
	MustSync bool
}

func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
	rd := Ready{
		Entries:          r.raftLog.unstableEntries(),
		CommittedEntries: r.raftLog.nextEnts(),
		Messages:         r.msgs,
	}
	if softSt := r.softState(); !softSt.equal(prevSoftSt) {
		rd.SoftState = softSt
	}
	if hardSt := r.hardState(); !isHardStateEqual(hardSt, prevHardSt) {
		rd.HardState = hardSt
	}
	// ...
	if len(r.readStates) != 0 {
		rd.ReadStates = r.readStates
	}
	rd.MustSync = MustSync(r.hardState(), prevHardSt, len(rd.Entries))
	return rd
}
```

从这段代码看出，`raft` 里面主要有两个属性，`raft.raftLog` 和 `raft.readStates`，`raft.raftLog` 主要在内存中存储了 committed 和 uncommitted 的日志，`raft.readStates` 主要存储了之前 `MsgReadIndex` 读操作请求返回的结果，`raft.hardState()` 函数返回了在消息发送之前必须要保存的 `raft` 状态，这些都是应用层的责任去管理保存这些状态。

`Ready` 对象可以认为是 `raft` 对象的一个快照，是当前时间点的状态，所以当应用层遵循规范处理完这些消息后，只需要把原 `Ready` 对象传回给 `raft` 就能更新内部状态。

```go
func (r *raft) advance(rd Ready) {
	r.reduceUncommittedSize(rd.CommittedEntries)

	if newApplied := rd.appliedCursor(); newApplied > 0 {
		oldApplied := r.raftLog.applied
		r.raftLog.appliedTo(newApplied)
		// ...
	}

	if len(rd.Entries) > 0 {
		e := rd.Entries[len(rd.Entries)-1]
		r.raftLog.stableTo(e.Index, e.Term)
	}
	// ...
}
```

可以看到 `advance(rd Ready)` 只是把内部的状态进行了更新，例如：删除了一些内存中的日志。

当我们了解 `raft` 的具体的结构，`raft` 计时器相关和状态变更的代码就非常好理解了，这里就不介绍了。

看到这里可能会有疑问，消息的发送和接收，还有定时器等事件是并发的，`raft.go` 并没有使用锁，其实 `raft` 本身就是线程不安全的，下篇文章我们分析下 `node.go` 和 `rawnode.go` 的代码，看下 `node` 是如何处理多线程并发请求的。
