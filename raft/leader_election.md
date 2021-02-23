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

### RPC ——请求投票 requestVote

