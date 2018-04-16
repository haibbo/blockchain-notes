## Orderer 创建通道

### 客户端

命令:

```shell
peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx 
```

### 流程

创建通道只和Orderer交互, 和Peer无交互.

- Orderer收到创建通道的请求后,会检查是否是配置交易(CONFIG_UPDATE)
- 检查通道是否已经存在, 不存在就创建
- 修改类型为CONFIG, 并加入从系统通道获取Orderer和Consortium等信息
- Orderer签名创建Proposal, 类型为ORDERER_TRANSACTION, 并发到系统通道中.完成通道创建
- 调用消息过滤器newSystemChainFilter,  返回systemChainCommiter
- 调用Commiter->newChain 完成往系统通道账本写入用户通道的区块, 并注册

客户端会来要用户通道的创世区块, 这里的分析不包含这部分.



### 接收Envelop消息, 获取channel Header

客户端发过来的CONFIG_UPDATE由orderer的Broadcast handler来处理.

```go
// orderer/common/broadcast/broadcast.go 的Handle方法
msg, err := srv.Recv() //接收Envelope消息
payload, err := utils.UnmarshalPayload(msg.Payload) // 反序列化获取Payload
chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader) // 获取channel Header
if chdr.Type == int32(cb.HeaderType_CONFIG_UPDATE) {
    msg, err = bh.sm.Process(msg) //configupdate.Processor.Process
}
```

### 创建新channel config Envelope

msg, err = bh.sm.Process(msg)代码如下：

```go
func (p *Processor) Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error) {
    channelID, err := channelID(envConfigUpdate) //获取ChannelHeader.ChannelId
    //multichain.Manager.GetChain方法，获取chainSupport，以及chain是否存在
    support, ok := p.manager.GetChain(channelID)
    if ok {
        //已存在的channel配置，调取multichain.Manager.ProposeConfigUpdate方法 更新配置
        return p.existingChannelConfig(envConfigUpdate, channelID, support)
    }
    //新channel配置，调取multichain.Manager.NewChannelConfig方法
    return p.newChannelConfig(channelID, envConfigUpdate)
}
// 代码在orderer/configupdate/configupdate.go
```

p.newChannelConfig(channelID, envConfigUpdate) 代码如下:

```go
ctxm, err := p.manager.NewChannelConfig(envConfigUpdate) //返回configManager结构
newChannelConfigEnv, err := ctxm.ProposeConfigUpdate(envConfigUpdate) // 得到ConfigEnvelope
newChannelEnvConfig, err := utils.CreateSignedEnvelope(cb.HeaderType_CONFIG, channelID, p.signer, newChannelConfigEnv, msgVersion, epoch) //封装成CONFIG类型的Envelope,并签名
return p.proposeNewChannelToSystemChannel(newChannelEnvConfig) //封装成ORDERER_TRANSACTION的Envelop, 需要放到系统通道里创建新通道
//代码在orderer/configupdate/configupdate.go
```

## Filters Apply and commit

回到 Handle 方法:

```go
_, filterErr := support.Filters().Apply(msg)
if !support.Enqueue(msg) {
    return srv.Send(&ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE})
}
```

Filters 是由createSystemChainFilters 创建的, 其中有一个newSystemChainFilter的rule, 它的Apply:

```go
    //判断是否已经到达最大channel上限
    maxChannels := scf.support.SharedConfig().MaxChannelsCount()
    if maxChannels > 0 {
        if uint64(scf.cc.channelsCount()) > maxChannels {
        }
    }  
……….
    //验证是否有权限操作，主要涉及签名的验证等
    err = scf.authorizeAndInspect(configTx)
    if err != nil {
        return filter.Reject, nil
    }
    //返回成功
    return filter.Accept, &systemChainCommitter{
        filter:   scf,
        configTx: configTx,
    }
//代码orderer/multichain/systemchain.go
```

## newChain 

上步返回的Commiter的Commit函数会调用newChain, 它会生成新的chain, 在multiledger上注册, 启动对该chain的监听协程, 当收到这个链上的交易请求后, 会分发给这个链来处理.  

```go
    ledgerResources := ml.newLedgerResources(configtx)
    ledgerResources.ledger.Append(ledger.CreateNextBlock(ledgerResources.ledger, []*cb.Envelope{configtx})) // 把applicaton channel的创世区块存入系统通道账本

    newChains := make(map[string]*chainSupport)
    for key, value := range ml.chains {
        newChains[key] = value
    }
    cs := newChainSupport(createStandardFilters(ledgerResources), ledgerResources, ml.consenters, ml.signer) //创建chainSupport
    chainID := ledgerResources.ChainID()

    newChains[string(chainID)] = cs
    cs.start() //创建goroutine,从orderer读消息传给kafaka或是solo, 必要时写BlockCutter
//代码 orderer/multichain/manage.go
```

chainSupport结构:

```go
type chainSupport struct {
	*ledgerResources
	chain         Chain
	cutter        blockcutter.Receiver
	filters       *filter.RuleSet
	signer        crypto.LocalSigner
	lastConfig    uint64
	lastConfigSeq uint64
}
```

结构chainSupport实现了接口ChainSupport

```go
type ChainSupport interface {
	PolicyManager() policies.Manager //策略管理
	Reader() ledger.Reader
	Errored() <-chan struct{}
	broadcast.Support
	ConsenterSupport //嵌入ConsenterSupport接口
	Sequence() uint64
	//支持通道更新
	ProposeConfigUpdate(env *cb.Envelope) (*cb.ConfigEnvelope, error)
}
type ConsenterSupport interface {
	crypto.LocalSigner
	BlockCutter() blockcutter.Receiver
	SharedConfig() config.Orderer
	CreateNextBlock(messages []*cb.Envelope) *cb.Block
	WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block
	ChainID() string
	Height() uint64
}

type Consenter interface { //定义支持排序机制
	HandleChain(support ConsenterSupport, metadata *cb.Metadata) (Chain, error)
}

// orderer/solo 或是 orderer/kafaka 实现了这个接口
type Chain interface {
	Enqueue(env *cb.Envelope) bool//接受消息
	Errored() <-chan struct{}
	Start() //开始
	Halt() //挂起
}
// 代码在orderer/multichain/chainsupport.go
```

