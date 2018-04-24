## Gossip - Leader选举

主节点是一个组织内作为代表连接排序服务的节点,  Leader选举的算法有如下特性:

- Peers通过比较ID打破对等性

- Peer要么是leader要么是follower, 在同一个view里, 只有一个leader

- 如果出现网络分区, 有几个分区就有几个leader, 分区修复后, 最终只能有一个leader

- 主节点选举的消息:

  1）一种是参与主节点选举的消息proposal，它在竞争主节点的过程中广播给所有节点的消息。

  2）一种声明成为主节点的消息declaration，它通过比较，可申请为主节点的消息。

节点刚启动时, leaderExists为0, 代表当前Leader并不存在. 节点new一个goroutine接受leadership的消息. 等待网络稳定再开始参与主节点选举.

### 主要入口:

```go
func (le *leaderElectionSvcImpl) start() {
	le.stopWG.Add(2)
	go le.handleMessages() // 接收leadership, 如果收到declaration 设置leaderExists
	le.waitForMembershipStabilization(getStartupGracePeriod()) // 等待网络稳定
	go le.run() // 开始主节点选举或角色的切换
}
```

### 消息处理:

```go
func (le *leaderElectionSvcImpl) handleMessage(msg Msg) {

	if msg.IsProposal() {
		le.proposals.Add(string(msg.SenderID())) // 以ID为Key保存所有接收到的proposal
	} else if msg.IsDeclaration() {
        //收到declaration 设置leaderExists
		atomic.StoreInt32(&le.leaderExists, int32(1))
		if le.sleeping && len(le.interruptChan) == 0 {
            // 通知, 唤醒调用waitForInterrupt的goroutine
			le.interruptChan <- struct{}{}
		}
        // 如果收到的declaration ID 比我们的小, 并且当前我们是leader, 设置isLeader为0
        // 停止delivery service
		if bytes.Compare(msg.SenderID(), le.id) < 0 && le.IsLeader() {
			le.stopBeingLeader()
		}
	} 
}
```

### 等待网络稳定:

网络稳定的等待时间是15秒，如果在等待时间内节点数不再变化或者已经出现了主节点，就是稳定了。如果在等待过程中没有产生主节点，则参与主节点的选举。

```go

func (le *leaderElectionSvcImpl) waitForMembershipStabilization(timeLimit time.Duration) {
	endTime := time.Now().Add(timeLimit)
	viewSize := len(le.adapter.Peers())
	for !le.shouldStop() {
		time.Sleep(getMembershipSampleInterval()) // 检查成员稳定性的采样间隔, 默认1s 
		newSize := len(le.adapter.Peers())
        // 1, 节点数量不发生变化 2, 超时 3, 已经有人声明为主节点
		if newSize == viewSize || time.Now().After(endTime) || le.isLeaderExists() {
			return
		}
		viewSize = newSize
	}
}
```

### 主干:

```go
func (le *leaderElectionSvcImpl) run() {
	for !le.shouldStop() {
		if !le.isLeaderExists() {
            // 如果没有leader存在, 开始参选leader
			le.leaderElection()
		}
		// 如果已经有主节点, 停止Yielding
		if le.isLeaderExists() && le.isYielding() {
			le.stopYielding()
		}
		if le.shouldStop() {
			return
		}
		if le.IsLeader() {
            // 发送declaration 声明自己为leader, 调用waitForInterrupt, 等leaderAliveThreshold/2 或是从interruptChan中收到消息.
			le.leader()
		} else {
            // 一次主节点选举的有效时间leaderAliveThreshold是10s，没有成为主节点的节点清除收到的proposal消息，然后等待有效时间结束，进入下一轮主节点选举的过程。
			le.follower()
		}
	}
}
```

#### Leader选举:

```go
func (le *leaderElectionSvcImpl) leaderElection() {
	// 如果当前是放弃选举的状态, 直接退出
	if le.isYielding() {
		return
	}
	// 提案自己作为一个Leader
	le.propose()
	// 等5s  等待proposal消息在组内扩散, 并持续收集其他的proposal   或是出现了leader
	le.waitForInterrupt(getLeaderElectionDuration())
	// 如果在时间超时（5秒）之前有其他节点声明为主节点，则接受它为主节点
	if le.isLeaderExists() {
		le.logger.Debug(le.id, ": Some peer is already a leader")
		return
	}
	// 若超时时间到了，还没有出现主节点，则按PKI-ID的字母顺序排序，最靠前的节点（也可以采用其他的算法）标识“isLeader”为true，声明自己为主节点，在组织内广播一个declaration消息。
    // 在这里我们只要发现有人比我们的ID小, 就退出选举
	for _, o := range le.proposals.ToArray() {
		id := o.(string)
		if bytes.Compare(peerID(id), le.id) < 0 {
			return
		}
	}
	// 我们是最佳的leader选择, 成为Leader, 开始delivery service
	le.beLeader()
	atomic.StoreInt32(&le.leaderExists, int32(1))
}
```

### 成为Leader/放弃Leader

beLeader和stopBeingLeader都会调用callback, callback对应的实现:

```go
func(isLeader bool) {
	if isLeader {
		yield := func() {
			g.lock.RLock()
			le := g.leaderElection[chainID]
			g.lock.RUnlock()
			le.Yield()
		}
		logger.Info("Elected as a leader, starting delivery service for channel", chainID)
		g.deliveryService.StartDeliverForChannel(chainID, committer, yield); 
	} else {
		logger.Info("Renounced leadership, stopping delivery service for channel", chainID)
		g.deliveryService.StopDeliverForChannel(chainID)
	}
```

yield作为最后一个参数传入StartDeliverForChannel, 在Deliver Block的过程中遇到问题退出后会调用yield. yield 会使节点放弃下次成为主节点或是超时60s.

### 选举算法伪代码

代码中有一段伪代码, 比较清晰的说明了Leader选举算法的流程:

```
// variables:
// 	leaderKnown = false
//
// Invariant:
//	Peer listens for messages from remote peers
//	and whenever it receives a leadership declaration,
//	leaderKnown is set to true
//
// Startup():
// 	wait for membership view to stabilize, or for a leadership declaration is received
//      or the startup timeout expires.
//	goto SteadyState()
//
// SteadyState():
// 	while true:
//		If leaderKnown is false:
// 			LeaderElection()
//		If you are the leader:
//			Broadcast leadership declaration
//			If a leadership declaration was received from
// 			a peer with a lower ID,
//			become a follower
//		Else, you're a follower:
//			If haven't received a leadership declaration within
// 			a time threshold:
//				set leaderKnown to false
//
// LeaderElection():
// 	Gossip leadership proposal message
//	Collect messages from other peers sent within a time period
//	If received a leadership declaration:
//		return
//	Iterate over all proposal messages collected.
// 	If a proposal message from a peer with an ID lower
// 	than yourself was received, return.
//	Else, declare yourself a leader

```

