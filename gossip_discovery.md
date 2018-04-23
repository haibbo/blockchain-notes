## Gossip - 成员发现和管理

Gossip的成员管理主要由它的discovery模块完成, 代码在gossip/discovery/下.

### 数据结构

```go
type gossipDiscoveryImpl struct {
	incTime         uint64      // 节点的启动时间
	seqNum          uint64		// 单调递增的计数器 每发送一个消息 +1	
	self            NetworkMember 
	deadLastTS      map[string]*timestamp     // 最后一次检查到的故障节点的时间
	aliveLastTS     map[string]*timestamp     // 最后一次检查到的存活节点的时间
	id2Member       map[string]*NetworkMember // 所有已知的节点
	aliveMembership *util.MembershipStore  // 存活节点列表
	deadMembership  *util.MembershipStore  // 故障节点列表

	msgStore *aliveMsgStore   // 保存所有收到的aliveMessage的store

	comm  CommService
	crypt CryptoService
	lock  *sync.RWMutex

	toDieChan        chan struct{}
	toDieFlag        int32
	port             int
	logger           *logging.Logger
	disclosurePolicy DisclosurePolicy
	pubsub           *util.PubSub
}
// 代码在gossip/discovery/discovery_impl.go
```

根据上面的结构, 可以看出Gossip的发现模块维护着所有节点的信息，包括存活节点和故障节点，包括最近一次检测到它们存活或者掉线的时间. 这里所有节点信息都是以PKI-ID为标识符的.

### 发现模块的goroutines

在函数NewDiscoveryService中:

```go
	d.validateSelfConfig()
	d.msgStore = newAliveMsgStore(d)

	go d.periodicalSendAlive()
	go d.periodicalCheckAlive()
	go d.handleMessages()
	go d.periodicalReconnectToDead()
	go d.handlePresumedDeadPeers()
// 代码在gossip/discovery/discovery_impl.go
```

#### periodicalSendAlive

周期性发送alive消息, 公告自己的存活性

```go
func (d *gossipDiscoveryImpl) periodicalSendAlive() {

	for !d.toDie() {
		time.Sleep(getAliveTimeInterval()) // 间隔时间
		msg, err := d.createAliveMessage(true)
		d.comm.Gossip(msg)
	}
}
// 代码在gossip/discovery/discovery_impl.go
```

#### periodicalCheckAlive

周期性的检查aliveLastTS中的时间戳，判断结点是死是活，然后相应调整deadLastTS，aliveLastTS，aliveMembership，deadMembership中的信息.

```go
func (d *gossipDiscoveryImpl) periodicalCheckAlive() {
	for !d.toDie() {
		time.Sleep(getAliveExpirationCheckInterval())
		dead := d.getDeadMembers() //查询aliveLastTS, 找到超过ExpirationTimeout的节点, 返回他们的PKI-ID
		if len(dead) > 0 {
			d.logger.Debugf("Got %v dead members: %v", len(dead), dead)
			d.expireDeadMembers(dead)  //  删除aliveLastTS中的信息, 并添加到deadLastTS, 根据PKI-ID获取Member信息, 并把它从aliveMemship中删除, 加入到DeadMemberShip中
		}
	}
}
```

#### periodicalReconnectToDead

周期性的查新尝试连接死掉的结点，通过尝试向死掉的结点发送GossipMessage_MemReq类型的消息来实现.

```go
func (d *gossipDiscoveryImpl) periodicalReconnectToDead() {
	for !d.toDie() {
		wg := &sync.WaitGroup{}
		for _, member := range d.copyLastSeen(d.deadLastTS) {
			wg.Add(1) // 为每一个dead节点建立一个gorintine, 并行check
			go func(member NetworkMember) {
				defer wg.Done()
				if d.comm.Ping(&member) {
					d.sendMembershipRequest(&member, true) // Ping 成功后发送GossipMessage_MemReq, 有回应handleMessages会处理, 更新发现模块里的数据
				} else {
					d.logger.Debug(member, "is still dead")
				}
			}(member)
		}
		wg.Wait()
		time.Sleep(getReconnectInterval()) // 周期性的尝试连接
	}
}
```

#### handlePresumedDeadPeers

循环处理来自discoveryAdapter成员presumedDead频道的消息，presumedDead频道在newDiscoveryAdapter()中初始化时即被赋值为gossip实例的成员g.presumedDead，而g.presumedDead是gossip专用于处理被认为是死掉的peer结点的频道.

#### handleMessages

循环处理来自discoveryAdapter成员in Chan频道的消息，这个通道只支持成员消息:

- GossipMessage_AliveMsg
- GossipMessage_MemReq
- GossipMessage_MemRes

里面任何一个消息都可以作为ALiveMsg调用handleAliveMessage来更新存活节点的相关配置, 但是除AliveMsg外其它Msg不会放到msgStore里面. 下面分析GossipMessage_MemReq和GossipMessage_MemRes的处理过程, 忽略调用handleAliveMessage的部分.

**GossipMessage_MemReq**

收到MemReq, 节点会把自己已知的Membership回给发送方.下面是创建MemRes的函数:

```go

func (d *gossipDiscoveryImpl) createMembershipResponse(aliveMsg *proto.SignedGossipMessage, targetMember *NetworkMember) *proto.MembershipResponse {	
    shouldBeDisclosed, omitConcealedFields := d.disclosurePolicy(targetMember)

	for _, dm := range d.deadMembership.ToSlice() {
		if !shouldBeDisclosed(dm) {
			continue
		}
		deadPeers = append(deadPeers, omitConcealedFields(dm))
	}

	for _, am := range d.aliveMembership.ToSlice() {
		if !shouldBeDisclosed(am) {
			continue
		}
		aliveSnapshot = append(aliveSnapshot, omitConcealedFields(am))
	}
	return &proto.MembershipResponse{
		Alive: append(aliveSnapshot, omitConcealedFields(aliveMsg)),
		Dead:  deadPeers,
	}
}
```

**GossipMessage_MemRes**

```go
if memResp := m.GetMemRes(); memResp != nil {
        // 通知Subscriber 我已经接受到了对应的MemRes, 你不需要继续发送了MemReq了.
		d.pubsub.Publish(fmt.Sprintf("%d", m.Nonce), m.Nonce)
		for _, env := range memResp.Alive {
			am, err := env.ToGossipMessage()

			if d.msgStore.CheckValid(am) {
				d.handleAliveMessage(am)
			}
		}
		for _, env := range memResp.Dead {
			dm, err := env.ToGossipMessage()
		    newDeadMembers := []*proto.SignedGossipMessage{}
		   // 找出不在已知列表中的节点, 加入newDeadMember
		   if _, known := d.id2Member[string(dm.GetAliveMsg().Membership.PkiId)]; !known {
				newDeadMembers = append(newDeadMembers, dm)
			}
            // 把新得到的DeabMember加入到deadMembership, 并且创建新的时间戳,加入到deadLastTS
			d.learnNewMembers([]*proto.SignedGossipMessage{}, newDeadMembers)
		}
	}
```

