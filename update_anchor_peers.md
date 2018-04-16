## 更新通道锚节点信息

锚节点负责代表组织与其他组织中的节点进行数据交换(Gossip通信).

### 客户端

客户端使用如下指令更新锚节点信息:

```shell
peer channel update -o orderer.example.com:7050 -c $CHANNEL_NAME -f ./channel-artifacts/${CORE_PEER_LOCALMSPID}anchors.tx
```

发送给排序服务的交易请求类型为HeaderType_CONFIG_UPDATE.

### 排序节点

排序节点收到后会重新创建一个类型为HeadType_CONFIG的信封. 这部分可参考创建channel. 每个链都有一些Filter rules, 若检测到类型是HeadType_CONFIG, 会交给configManager来处理. 流程如下:

![](/Users/haibbo/code/blockchain-notes/_images/configManagerUpdate.png)

## PreApply

```go
func (cm *configManager) prepareApply(configEnv *cb.ConfigEnvelope) (*configResult, error) {
	// 检查配置序列号
	if configEnv.Config.Sequence != cm.current.sequence+1 {
		return nil, fmt.Errorf("Config at sequence %d, cannot prepare to update to %d", cm.current.sequence, configEnv.Config.Sequence)
	}
	// 解析出ConfigUpdateEnvelope
	configUpdateEnv, err := envelopeToConfigUpdate(configEnv.LastUpdate)
	// 校验ConfigUpdateEnvelope和填充
	configMap, err := cm.authorizeUpdate(configUpdateEnv)

	channelGroup, err := configMapToConfig(configMap)
	// reflect.Equal will not work here, because it considers nil and empty maps as different
	proto.Equal(channelGroup, configEnv.Config.ChannelGroup)

	result, err := cm.processConfig(channelGroup)
	if err != nil {
		return nil, err
	}

	return result, nil
}
//代码在common/configtx/manager.go
```



```go
func (cm *configManager) authorizeUpdate(configUpdateEnv *cb.ConfigUpdateEnvelope) (map[string]comparable, error) {

	// 解析出ReadSet
	readSet, err := MapConfig(configUpdate.ReadSet)
    // 校验ReadSet
	err = cm.current.verifyReadSet(readSet)
	// 解析出writeSet
	writeSet, err := MapConfig(configUpdate.WriteSet)

	// 根据ReadSet和WirteSet计算配置更新
	deltaSet := ComputeDeltaSet(readSet, writeSet)
	signedData, err := configUpdateEnv.AsSignedData()
    // 校验配置更新
	err = cm.verifyDeltaSet(deltaSet, signedData)
	// 根据配置更新和当前的配置更新为最新的配置
	fullProposedConfig := cm.computeUpdateResult(deltaSet)
    //校验是否所有的writeSet都在fullProposedConfig 出现
	err := verifyFullProposedConfig(writeSet, fullProposedConfig)

	return fullProposedConfig, nil
}
//代码在common/configtx/manager.go
```



### 广播到所有节点

Gossip



