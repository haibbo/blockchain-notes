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

- [Peer 节点加入通道](Join_channel.md)
    - [配置管理系统链码](CSCC.md)

- [更新通道锚节点信息](updateAnchorPeers.md)

- [安装chaincode](install_chaincode.md)

- [实例化chaincode](instantiate_chaincode.md)

- [chaincode 查询](query_chaincode.md)

- chaincode 调用

    ​





