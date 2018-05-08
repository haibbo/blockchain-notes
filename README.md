# Hyperledger fabric 学习笔记

## 简介
本系列文章, 是我学习fabric过程中的笔记. 

## 从一个例子开始

基于hyperledger官方提供的fabric-samples里的BYFN（Build Your First Net-work）介绍超级账本的构建过程，首先是利用提供的脚本快速地构建网络，然后是分解构建过程。分析每个步骤背后的原理, 部分会分析代码.

下面是Hyperleder提供的详细文档

[构建你的第一个fabric网络](http://hyperledger-fabric.readthedocs.io/en/latest/build_network.html)

### 下载BYFN代码
BYFN是包含在fabric-samples的first-network目录下的，先通过git下载源代码：
```shell
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples/first-network
```
后面的操作默认都在此路径下进行。

## 启动BYFN 网络

```shell
./byfn.sh -m up # 启动网络
```

上面这条命令就可以启动了这个fabric的网络, 并执行了一些操作, 下面是脚本背后具体的操作.
- 生成[证书信息(MSP)](generate_certs.md)
- 生成 channel artifacts 
  - [创世区块](genesis_block.md)
  - [channel的配置交易](config_tx.md)
  - [锚节点更新](anchor_tx.md)
- [启动docker](docker_start.md)
    - [Orderer启动之参数传递](orderer_start_paramaters.md)
- [创建channel](create_channel.md)
- [Peer 节点加入通道](join_channel.md)
    - [配置管理系统链码](CSCC.md)
- [更新通道锚节点信息](update_anchor_peers.md)
- [安装chaincode](install_chaincode.md)
- [实例化chaincode](instantiate_chaincode.md)
  - [实例化链码 - 客户端](instantiate_application.md)
  - [实例化链码 - 生命周期系统链码(LSCC)](instantiate_cc_lscc.md)
  - [系统链码与用户链码](instantiate_what_is_chaincode.md)
  - [实例化链码 - 链码状态机](instantiate_cc_sm.md)
  - [实例化链码 - 交易模拟器与读写集](txsim_and_rwset.md)
- [chaincode 查询](query_chaincode.md)
- [交易流程(chaincode 调用)](invoke_chaincode.md)
  - [交易流程 - 客户端](txflow_application.md)
  - [交易流程 - 背书节点](txflow_endorser.md)
  - [交易流程 - 背书节点与链码容器](txflow_endorser_cc_container.md)
  - [交易流程 - 用户链码example02](txflow_chaincode_example02.md)
  - [交易流程 - 排序节点](txflow_orderer.md)
  - [交易流程 - 提交节点](txflow_committer_peer.md)
    - [数据存储](txflow_datastore.md)
    - [4种DB](txflow_four_DBs.md)
    - [区块链](txflow_blockchain.md)
    - [添加区块](txflow_addblock.md)


### 共识算法
- [POW 工作量证明](consensus/POW.md)
- [Raft 一致性算法](consensus/Raft.md)
- [PBFT 实用拜占庭容错算法](consensus/PBFT.md)

### Gossip 协议在Fabric中的实现
- [消息类型](gossip/gossip_message_type.md)
- [发现和成员管理](gossip/gossip_discovery.md)
- [主节点选举](gossip/gossip_election.md)
- [状态同步](gossip/gossip_state.md)
- [消息广播和分区](gossip/gossip_emmiter_batch.md)

