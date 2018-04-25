## Gossip - 状态更新
代码实现主要功能如下:
- 处理 DataMsg 去 commit 区块和广播区块给其他节点
- 发送/处理 StateReq 和 StateReponse
- 通过反熵来获取缺失的数据

代码在 gossip/state/下.

### Payloads Buffer

在分析主要功能前, 先介绍下 PayloadsBuffer. 它用于暂存收到的区块信息, 为什么要暂存? 因为区块信息有可能不是按照顺序到达节点的, 而我们 commit 区块是有顺序的, 如果丢弃它们, 之后再用别的途径获取会增加网络负担和节点同步的时间.

PayloadsBuffer 会在内部保存一个索引, 记录等待提交账本的下一个区块的序列号 next. 如果接收到的区块序列号小于next, 说明是过期的区块, 直接丢弃. 如果序列号大于 next, PayloadsBuffer 把收到的数据放到序列号对应的缓冲区数组里. 只要收到的数据和已提交区块序列号连续了, 就会把连续的数据区块提交到账本里, 然后删除缓冲区中已提交的数据区块, 同时更新索引 next. 

```go
type PayloadsBuffer interface {
	// 添加区块到buffer
	Push(payload *proto.Payload) error
	// 返回预期的下一个区块
	Next() uint64
	// 返回下一个区块, 并把它从buffer中删除
	Pop() *proto.Payload
	// 获取当前的buffer大小
	Size() int
    // 用于指示 加入的区块等于预期的区块, 可以commit了
	Ready() chan struct{}
	Close()
}
```

### 主干:

```go
    // 监听连接, 收取数据包(取DataMsg 处理StateReq和StateReponse)
	go s.listen()
	// Commit 区块并且广播区块给其它节点
	go s.deliverPayloads()
    // 通过反熵更新缺失的数据
	go s.antiEntropy()
    // 处理StateRequest 消息
	go s.processStateRequests()
```

### 收包:

把DataMsg放到PayloadsBuffer, 把StateReq和StateResp放到相应的channel中去.

```go
select {
	case msg := <-s.gossipChan:
		logger.Debug("Received new message via gossip channel")
    // 把收到的DataMsg的Payload放入Payload_Buffer里, 当seq == NEXT, bReady = 1
		go s.queueNewMessage(msg) 
	case msg := <-s.commChan:
		logger.Debug("Direct message ", msg)
    	// 把StateReq消息放到channel stateRequestCh, 这个缓存channel的max len是100
    	// 把StateResp消息放到channel stateResponseCh
		go s.directMessage(msg) 
}
```

### 处理DataMsg:

当收到了区块匹配预期的下一个区块, 开始把连续的区块依次提交,广播.

```go
func (s *GossipStateProviderImpl) deliverPayloads() {
    for {
		select {
		// 当收到的next的block
		case <-s.payloads.Ready():
			logger.Debugf("Ready to transfer payloads to the ledger, next sequence number is = [%d]", s.payloads.Next())
			// 处理所有连续的Block
			for payload := s.payloads.Pop(); payload != nil; payload = s.payloads.Pop() {
				rawBlock := &common.Block{}
				pb.Unmarshal(payload.Data, rawBlock)
				// commit 并且把这个block广播给其他节点
				s.commitBlock(rawBlock)
    }
}
```

### 反熵

反熵 (Anti-entropy) 是指每个节点周期性地和邻居节点交换保存的数据, 然后对比本地数据和邻居节点所保存的数据, 检查是否有缺失或者过期的数据, 然后更新本地节点的数据为最新的数据. 

```go
func (s *GossipStateProviderImpl) antiEntropy() {

	for {
		select {
        // 每10s做一次
		case <-time.After(defAntiEntropyInterval):
            // 获取本地当前区块height
			current, err := s.committer.LedgerHeight()
            // 获取当前channel中节点最大的区块height
			max := s.maxAvailableLedgerHeight()
			// 向节点发送请求获取 current-max的区块
			s.requestBlocksInRange(uint64(current), uint64(max))
		}
	}
}
func (s *GossipStateProviderImpl) requestBlocksInRange(start uint64, end uint64) {

	for prev := start; prev <= end; {
        // 每次要求的区块个数不能超过10
		next := min(end, prev+defAntiEntropyBatchSize)
        // 创建State_Req消息
		gossipMsg := s.stateRequestMessage(prev, next)
		responseReceived := false
		tryCounts := 0

		for !responseReceived {
            // 发送请求 等待响应, 如果3次尝试都没有得到响应 退出
			if tryCounts > defAntiEntropyMaxRetries {
				return
			}
			// 选择节点要区块, 1, 找到有height >= next的节点门, 2 随机返回一个
			peer, err := s.selectPeerToRequestFrom(next)
			// 发送请求
			s.gossip.Send(gossipMsg, peer)
			tryCounts++

			// 等待, 直到超时或收到state Response
			select {
			case msg := <-s.stateResponseCh:
                // 是否是本次请求的回复
				if msg.GetGossipMessage().Nonce != gossipMsg.Nonce {
					continue
				}
				// 对数据进行校验 并加到Payload_Buffer里
				index, err := s.handleStateResponse(msg)
				prev = index + 1
				responseReceived = true
                // 超时重试(3s)
			case <-time.After(defAntiEntropyStateResponseTimeout):
		}
	}
}
```

### 处理State Request消息

对请求的区块范围做合法性检测, 把本地又的区块依次取出, 放到Response并发送给请求者. 

```go

func (s *GossipStateProviderImpl) handleStateRequest(msg proto.ReceivedMessage) {
	request := msg.GetGossipMessage().GetStateRequest()

	batchSize := request.EndSeqNum - request.StartSeqNum
    // 请求的区块不能超过10
	if batchSize > defAntiEntropyBatchSize {
		return
	}
	// 开始的区块编号要小于结束区块编号
	if request.StartSeqNum > request.EndSeqNum {
		return
	}
	// 即使要求的最大区块比我们当前的大. 不报错, 能传回多少传多少
	currentHeight, err := s.committer.LedgerHeight()
	endSeqNum := min(currentHeight, request.EndSeqNum)
	// 依次获取本地区块, Marshal后加到Payloads里
	response := &proto.RemoteStateResponse{Payloads: make([]*proto.Payload, 0)}
	for seqNum := request.StartSeqNum; seqNum <= endSeqNum; seqNum++ {
		
		blocks := s.committer.GetBlocks([]uint64{seqNum})
		blockBytes, err := pb.Marshal(blocks[0])
		response.Payloads = append(response.Payloads, &proto.Payload{
			SeqNum: seqNum,
			Data:   blockBytes,
		})
	}
	// 把区块通过State Response发回给请求端.
	msg.Respond(&proto.GossipMessage{
		// 从请求中得到的NOnce
		Nonce:   msg.GetGossipMessage().Nonce,
		Tag:     proto.GossipMessage_CHAN_OR_ORG,
		Channel: []byte(s.chainID),
		Content: &proto.GossipMessage_StateResponse{response},
	})
}
```