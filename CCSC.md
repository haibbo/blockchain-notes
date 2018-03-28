## 配置管理系统链码

Configuration SystemChaincode，其主要功能是管理记账节点上的配置信息.

加入通道时需要需要试用CSCC的JoinChain接口, 同时它还提供两个接口.

### GetConfigBlock

客户端命令:

```shell
 peer channel fetch config mychannel.block 
```
每次有配置更新的时候, 都会更新记账节点本地维护的最新配置区块.
```go
case GetConfigBlock:
	// 检查是否满足/Channel/Application/Readers的策略
	err = e.policyChecker.CheckPolicy(string(args[1]), policies.ChannelApplicationReaders, sp)
	return getConfigBlock(args[1])
//代码在core/scc/cscc/configure.go
```
getConfigBlock会调用GetCurrConfigBlock从本地缓存获取, 其实也可以通过最新Block的Meta_LAST_CONFIG来获取配置区块.


### GetChannels

客户端命令:

```shell
peer channel list
```

获取加入的链列表

```go
    case GetChannels:
        // 检查是否满足Member的策略
        err = e.policyChecker.CheckPolicyNoChannel(mgmt.Members, sp)
       return getChannels()
```

