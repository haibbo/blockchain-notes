## 查询链码

query  也是调用链码的 Invoke,  因为只是查询账本状态, 不更改账本信息, 所以不需要把TX 发给 Orderer. 但是依然会调用 ESCC ( 有必要吗? ).

### 客户端命令:

```shell
 setGlobals 0 ## 设置peer.org1的环境变量
 peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

### 流程

本次查询请求是发给peer0.org1节点的. 之前 byfn 脚本是在peer0.org2上执行的链码实例化, 因为实例化要调用链码的Init, 所以 peer0.org2 上的容器容器已经跑了起来了 .  实例化完成后, 其它节点的链码容器并不会跑起来, 但是包含 ChaincodeData 的区块文件会在节点间同步. 这次对 peer0.org1的 query 会把它的链码容器带起来.

- 客户端发送SignedProposal给背书节点
  - ChanelID:mychannel 
  - CCID.Name:mycc 
  - Input Args: query + 'a'
- 背书节点调用 ProcessProposal 开始处理这个请求
  - 校验有效性, 合法性, channel是否存在,是否是重复的交易, 同时通过查询LSCC获取此链码是否已经实例化过等信息(ChaincodeData).
  - 创建交易模拟器和上下文, 调用相关链码执行模拟
  - 因为链码容器未启动所以会先创建容器docker image, 启动链码容器, 发送TRANSACTION给链码容器
  - 链码容器会发GET_STATE给节点, 节点返回REPSONSE, 完成后
  - 容器返回结果给背书节点, GetTxSimulationResults获取读集
  - 发送给ESCC进行背书, 成功后, 返回ProposalResponse 给客户端
- 客户端把结果打印到屏幕上, 不需要发送给Orderer



### log文件

[query peer0](logs/query_peer0.log)