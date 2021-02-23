## Raft 第一部分——领导者选举

在 Raft 协议中，服务器被分为了三种角色，分别对应不同的功能。

1. Leader：领导者，负责接受客户端请求，并主动将日志同步给 Follower 跟随者（心跳包）。
2. Follower：跟随者，负责进行投票选举出 Leader 以及维护本地副本。
3. Candidate：候选者，当 Follower 因为一些原因无法与领导者沟通时，进入 Candidate 状态，向周围服务器发出投票请求。

### Follower 职责

Follower 职责比较简单，在领导者选举部分中，它只需要检查一件事。

> 是否收到来自 Leader 的心跳包？
>
> 是 -> 继续等待 heartbeatTimeout 
>
> 否 -> 转入 Candidate 进行选举

```go
case STATE_FOLLOWER:
			select {
			case <-rf.heartbeatChan:
            /* Heart is beating do nothing */
			case <-time.After(rf.heartbeatTimeout):
				rf.to(STATE_CANDITATE)
			}
```

### Candidate 职责

当 Follower 转入 Candidate 后，它进行下面的事。

> 将自己的任期增加 `rf.currentTerm+1`
>
> 投票给自己 `votedFor=rf.me`
>
> **同时** 向所有其他服务器发起投票请求 `requestVote`

```go
case STATE_CANDITATE:
			rf.mu.Lock()
			rf.currentTerm++
			rf.votedFor = rf.me
			rf.mu.Unlock()
			go rf.broadcastRequestVote()
```

之后 Candidate 进入等待状态，直到下面的其中一个事件发生。

1. 收到来自其它 Leader 的 **有效** 心跳包 -> 转为 Follower
2. 收到投票结果 -> 转为 Leader
3. 收到超时信号 -> 进入重新开始选举，重新递增任期，投票给自己，发起投票请求

```go
select {
			case <-rf.heartbeatChan:
			/* Other new leader */
			case voteResult := <-rf.voteResultChan:
			/* Successful Elected */
				if voteResult {
					rf.to(STATE_LEADER)
					go rf.broadcastHeartbeat()
				}
			case <-time.After(rf.electionDuration):
			/* Restart Election */
}
```

### Leader 职责

Leader 只进行一个操作，按周期发出心跳包维护/同步日志

```go
case STATE_LEADER:
			select {
			case <-rf.heartbeatChan:
			/* Other new leader */
			case <-time.After(rf.heartbeatDuration):
			/* Send heartbeat */
				go rf.broadcastHeartbeat()
			}
```

### RPC 处理通则

下面的规则对所有 RPC 回复/请求 有效，即对任何状态下的服务器，都会做下面两件事

1. 丢弃 **过期请求/回复** `req.Term < self.Term`
2. 收到 **未来请求/回复** `req.Term > self.Term`，退回 Follower 模式，放弃选举/心跳包

### RPC —— 发起投票请求 broadcastRequestVote

在领导者选举阶段，投票请求的 RPC 仅需要两个参数。

```go
type RequestVoteArgs struct {
	/* Candidate's Term */
	Term        int
	/* Candidate's ID */
	CandidateId int
}
```

这决定了收到投票请求的服务器是否会为该候选人投票。

在收到回复时，下面是一些 **非常重要** 的点。

1. 丢弃 Term 小于自身的回复。
2. 当回复的 Term > 自身的 Term，说明已经有服务器进入更新的任期，放弃选举，退回 Follower 状态。
3. 只计入与自身 Term 相同的投票。

> 1，2 点对所有 RPC 回复/请求 有效，即对任何状态下的服务器，都会做下面两件事
>
> 1. 丢弃 **过期请求/回复** `req.Term < self.Term`
> 2. 收到 **未来请求/回复** `req.Term > self.Term`，退回 Follower 模式，放弃选举/心跳包

```go
select {
		/* Async Process Reply */
		case reply = <-replyChan:
			rf.mu.Lock()
			if reply.Term > rf.currentTerm {
			/* Fallback to Follower */
				rf.currentTerm = reply.Term
				rf.votedFor = -1
				rf.state = STATE_FOLLOWER
				rf.mu.Unlock()
				rf.voteResultChan <- false
				return
			}
			if reply.Term == rf.currentTerm && reply.VoteGranted {
			/* Count Vote */
				voteCnt++
			}
			if voteCnt > len(rf.peers)/2 {
			/* Once collect sufficient vote, become a leader */
				rf.mu.Unlock()
				rf.voteResultChan <- true
				return
			}
			rf.mu.Unlock()
		case <-time.After(100 * time.Millisecond):
		/* RPC Fail */
			continue
		}
```

### RPC —— 处理投票请求 RequestVote

投票请求返回的结构体如下所示

```go
type RequestVoteReply struct {
	/* Sync term for outdated candidate */
	Term        int
	/* Whether voted for candidate */
	VoteGranted bool
}
```

在处理投票请求时，除了关注 **处理通则** 中的部分，只需要关注一件事，即 `votedFor` 的处理。

在 Raft 标准中，一个服务器在一个 Term 中只能为一个候选者投票。

那么在收到请求投票的请求时，只有当 `votedFor == req.CandidateId` 或者 `votedFor == -1` 即当投票者投的是该候选人或者投票者尚未投票（通常出现在落后的 Follower 中）。

```go
if args.Term > rf.currentTerm {
/* General rule of RPC, update follower's term */
		rf.currentTerm = args.Term
		/* New term unvoted yet */
		rf.votedFor = -1
		rf.state = STATE_FOLLOWER
	}

	if args.Term == rf.currentTerm && (rf.votedFor == -1 || rf.votedFor == args.CandidateId) {
		reply.VoteGranted = true
		reply.Term = args.Term
		rf.votedFor = args.CandidateId
	} else {
		reply.VoteGranted = false
		reply.Term = args.Term
	}
```

### RPC —— 发出心跳包 broadcastAppendEntries

在领导者选举阶段我们只需要实现最基础发送并按照 **处理通则** 处理回复即可。

```go
select {
	case reply = <-replyChan:
		rf.mu.Lock()
		if reply.Term > rf.currentTerm {
            /* Outdated leader, goes to follower */
			rf.currentTerm = reply.Term
			rf.votedFor = -1
			rf.state = STATE_FOLLOWER
			rf.mu.Unlock()
			break
		}
		rf.mu.Unlock()
	case <-time.After(100 * time.Millisecond):
		continue
}
```

### RPC —— 处理心跳包 AppendEntries

与发出类似，都比较简单。

```go
if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.votedFor = -1
		rf.state = STATE_FOLLOWER
	}

	if args.Term == rf.currentTerm {
		rf.state = STATE_FOLLOWER
		/* Notify main loop */
		rf.heartbeatChan <- *args
	}
```

### 完整实现

完整实现可以在 [这里](https://github.com/JohnWestonNull/MIT6.824-Raft-KV) 找到

