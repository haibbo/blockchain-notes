### 配置交易(Configuration TX)

由于区块链系统是的分布式的，对配置进行更新是一件很有挑战的任务, 易出现不同节点之间配置不一致的问题，导致整个网络功能异常. 

在Fabric网络中，通过采用配置交易（ConfigurationTransaction，ConfigTX）来实现对通道相关配置的更新。配置更新操作如果被执行，也会像交易一样经过网络中节点的共识确认。简单说就是把配置更新,像交易一样处理.

生成配置交易的命令:
```shell
configtxgen -profile  TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME
```
对应的log:
```shell
#################################################################
### Generating channel configuration transaction 'channel.tx' ###
#################################################################
2018-03-19 15:34:18.274 CST [common/configtx/tool] main -> INFO 001 Loading configuration
2018-03-19 15:34:18.279 CST [common/configtx/tool] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-03-19 15:34:18.279 CST [common/configtx/tool] doOutputChannelCreateTx -> INFO 003 Writing new channel tx
```

配置交易里面又有什么?
使用下面命令显示通道配置交易信息:
```shell
configtxgen -profile TwoOrgsChannel -inspectChannelCreateTx channel-artifacts/channel.tx
```
 ![channel_tx](./_images/channel_tx.png)

它为应用通道指定了Application、Consortium信息：   

- Consortium：该应用通道所关联联盟的名称。
- Application：通道的组织信息, 策略等 
  - > 图中的例子是, Admins的策略类型是IMPLICIT_META, 规则是MAJORITY, 需要这个联盟里面大多数的管理员签名,认可.

### 与创世区块区别

- 创世区块是一个区块, orderer启动时加载它用来配置系统链, 它是整个区块链网络的第一个区块.
- channel的配置交易, 它一个交易, 通过inspectChannelCreateTx, 可以看到它里面的内容是读写集, 它用来之后创建channel. 
- Channel的配置交易, 只包含这个channel对应的联盟, 应用通道的策略等. 
- 系统链, 只有一个, 不可以动态创建删除. 
- 应用通道可以增加, 删除, 更新, 用配置交易做统一的处理, 之后会单独放到一个块里. 


