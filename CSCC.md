## 配置系统链码(CSCC)

配置管理系统链码(Configuration SystemChaincode)主要功能是管理记账节点上的配置信息.

CSCC代码在:/core/scc/cscc中, Invoke函数支持以下功能:

### JoinChain

节点申请加入某个通道时被调用.

### GetConfigBlock

获取链的配置区块, 从本地缓存中获取(chains.list[channel_id]).

### GetChannels

获取节点已经加入的通道列表

