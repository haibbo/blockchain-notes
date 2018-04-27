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
	pubsub           *util.PubSub // PubSub Struct 完成世界的订阅和分发
}
// 代码在gossip/discovery/discovery_impl.go
```

根据上面的结构, 可以看出Gossip的发现模块维护着所有节点的信息，包括存活节点和故障节点，包括最近一次检测到它们存活或者掉线的时间. 这里所有节点信息都是以PKI-ID为标识符的.

### 发现模块的goroutines

在函数NewDiscoveryService中:

```go
	d.validateSelfConfig()
	d.msgStore = newAliveMsgStore(d) // MsgStore 参考下面对它的介绍

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

### 模块PubSub

订阅是异步通讯的一种方式, 接收方（receiver）会来订阅发送方（sender）的消息，发送方会把相关的消息或数据放到接收方所订阅的队列中，而接收方会从队列中获取数据。

```go
// PubSub 可以被用来publish事件给多个订阅者和订阅事件
type PubSub struct {
	sync.RWMutex
	subscriptions map[string]*Set
}
type Subscription interface {
    // Listen 会阻塞,直到对应的事件发生(publish), 或是TTL到期
	Listen() (interface{}, error)
}

type subscription struct {
	top string
	ttl time.Duration // 生成时间, 当时间到达后对应的subscription会被清除掉
    c   chan interface{} // 接受事件的通道, 试用interface{}来接收任意的事件
}

```

它支持的3个主要方法:

```go
// 等到超时或事件被Publish
func (s *subscription) Listen() (interface{}, error) {
	select {
	case <-time.After(s.ttl):
		return nil, errors.New("timed out")
	case item := <-s.c:
		return item, nil
	}
}
// Public 事件给这个主题上所有的订阅者
func (ps *PubSub) Publish(topic string, item interface{}) error {
	s, subscribed := ps.subscriptions[topic]
	for _, sub := range s.ToArray() {
        // 获取这个topic上的所用订阅者, 依次把事件写给它们的channel.
		c := sub.(*subscription).c
		c <- item
	}
	return nil
}
// Subscribe 返回一个订阅者
func (ps *PubSub) Subscribe(topic string, ttl time.Duration) Subscription {
	sub := &subscription{
		top: topic,
		ttl: ttl,
		c:   make(chan interface{}, subscriptionBuffSize),
	}
	s, exists := ps.subscriptions[topic]
	// 如果针对这个topic没有对应的订阅集, 创建一个
	if !exists {
		s = NewSet()
		ps.subscriptions[topic] = s
	}
	// 向某一个主题的订阅者集加入一个订阅者
	s.Add(sub)
	// 超时后执行unSubscribe, 取消订阅
	time.AfterFunc(ttl, func() {
		ps.unSubscribe(sub)
	})
	return sub
}
```

在Discovery中, Publish使用在收到Membership Response后:

```go
d.pubsub.Publish(fmt.Sprintf("%d", m.Nonce), m.Nonce)
```

Subscribe和Listen用在发送完Membership Request,等待Response的函数sendUntilAcked中:

```go
func (d *gossipDiscoveryImpl) sendUntilAcked(peer *NetworkMember, message *proto.SignedGossipMessage) {
	nonce := message.Nonce // 获取Nonce
	for i := 0; i < maxConnectionAttempts && !d.toDie(); i++ {
        // 订阅Topic为Nonce, TTL为5的事件
		sub := d.pubsub.Subscribe(fmt.Sprintf("%d", nonce), time.Second*5)
		d.comm.SendToPeer(peer, message)
        // 开始监听直到收到或超时
		if _, timeoutErr := sub.Listen(); timeoutErr == nil {
			return
		}
		time.Sleep(getReconnectInterval())
	}
}
```

### 模块MsgStore

MsgStore用于存储消息, 并且这里的消息是有有效期的. 并且提供了接口来根据一些规则更新里面的消息.

```go
func newAliveMsgStore(d *gossipDiscoveryImpl) *aliveMsgStore {
    policy := proto.NewGossipMessageComparator(0) // 消息的替换规则参考(invalidationPolicy)
	trigger := func(m interface{}) {} // 替换消息后执行的函数
	aliveMsgTTL := getAliveExpirationTimeout() * msgExpirationFactor // 25 * 20 = 500s
	externalLock := func() { d.lock.Lock() }
	externalUnlock := func() { d.lock.Unlock() }
	callback := func(m interface{}) { // 消息超时后执行的清除函数 
		msg := m.(*proto.SignedGossipMessage)
		if !msg.IsAliveMsg() {
			return
		}
		id := msg.GetAliveMsg().Membership.PkiId
        // 某个ID的AliveMessage过期后, 会删除关于它所以的信息.
		d.aliveMembership.Remove(id)
		d.deadMembership.Remove(id)
		delete(d.id2Member, string(id))
		delete(d.deadLastTS, string(id))
		delete(d.aliveLastTS, string(id))
	}

	s := &aliveMsgStore{
		MessageStore: msgstore.NewMessageStoreExpirable(policy, trigger, aliveMsgTTL, externalLock, externalUnlock, callback),
	}
	return s
}
```

上面的pol 

包含一个通用的消息验证策略函数：

type MessageReplacingPolicy func(this interface{}, that interface{})InvalidationResult

其中，this和that指代的是当前消息和原有消息的比较，并判断当前消息的有效性。比较的结果InvalidationResult有3种可能。

1）MESSAGE_INVALIDATES：当前消息是有效的，原有消息是无效的，用当前消息替换原有消息；

2）MESSAGE_INVALIDATED：当前消息是无效的，原有消息是有效的，丢弃当前的消息；

3）MESSAGE_NO_ACTION：两个消息可能是不同类型的，两个消息不进行比较，两个消息都是有效的。MessageReplacingPolicy可以根据实际存储的消息定义不同的比较策略。

```go
// 根据包类型的不同有不一样的规则.
func (mc *msgComparator) invalidationPolicy(this interface{}, that interface{}) common.InvalidationResult {
	thisMsg := this.(*SignedGossipMessage)
	thatMsg := that.(*SignedGossipMessage)

	if thisMsg.IsAliveMsg() && thatMsg.IsAliveMsg() {
		return aliveInvalidationPolicy(thisMsg.GetAliveMsg(), thatMsg.GetAliveMsg())
	}

	if thisMsg.IsDataMsg() && thatMsg.IsDataMsg() {
		return mc.dataInvalidationPolicy(thisMsg.GetDataMsg(), thatMsg.GetDataMsg())
	}

	if thisMsg.IsStateInfoMsg() && thatMsg.IsStateInfoMsg() {
		return mc.stateInvalidationPolicy(thisMsg.GetStateInfo(), thatMsg.GetStateInfo())
	}

	if thisMsg.IsIdentityMsg() && thatMsg.IsIdentityMsg() {
		return mc.identityInvalidationPolicy(thisMsg.GetPeerIdentity(), thatMsg.GetPeerIdentity())
	}

	if thisMsg.IsLeadershipMsg() && thatMsg.IsLeadershipMsg() {
		return leaderInvalidationPolicy(thisMsg.GetLeadershipMsg(), thatMsg.GetLeadershipMsg())
	}

	return common.MessageNoAction
}
```

替换的规则基本上,包括

1. 时间戳的比较
2. 消息序列号的比较
3. 节点PKI-ID的比较

![](_images/msgstore_verify.png)