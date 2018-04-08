## 查询链码

query  其实也是调用链码的Invoke,  但是因为只是查询账本状态, 不更改账本信息, 所以不需要把TX 发给Orderer. 但是依然会调用ESCC依然需要调用.

### 客户端命令:

```shell
 setGlobals 0 ## 设置peer.org1的环境变量
 peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```



### 流程

调用查询前, 脚本有实例化peer0.org2上的链码, 所以这个节点对应的容器会跑起来, 因为要调用链码的Init. 实例化完成后, 其它节点的链码容器并不会跑起来, 但是区块文件是会同步过的. 所以这次query会把peer0.org1的链码容器带起来.

- 客户端发送SignedProposal给背书节点
  - ChanelID:mychannel 
  - CCID.Name:mycc 
  - Input Args: query + 'a'
- 背书节点调用ProcessProposal 开始处理这个请求
  - 校验有效性, 合法性, channel是否存在,是否是重复的交易, 同时通过查询LSCC获取此链码是否已经实例化过等信息(ChaincodeData).
  - 创建交易模拟器和上下文, 调用相关链码执行模拟
  - 因为链码容器未启动所以会先创建容器docker image, 启动链码容器, 发送TRANSACTION给链码容器
  - 链码容器会发GET_STATE给节点, 节点返回REPSONSE, 完成后
  - 容器返回结果给背书节点, GetTxSimulationResults获取读集
  - 发送给ESCC进行背书, 成功后, 返回ProposalResponse 给客户端
- 客户端把结果打印到屏幕上, 不需要发送给Orderer



### log文件

[query peer0](logs/query_peer0.log)