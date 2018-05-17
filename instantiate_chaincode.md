## 链码实例化

### 实例化步骤

1. 调用LSCC的部署操作, 把链码计算哈希后生成的ChaincodeData存到链上
2. 从节点文件系统中读取链码源码, 生成docker镜像执行初始化操作

### 详细流程(客户端, Peer节点 和 链码容器)

1. 客户端发送SignedProposal给背书节点
   - ChanelID:mychannel 
   - CCID.Name:lscc 
   - Input Args: deploy + ChainID + CDS + policy + escc + vscc)
2. 背书节点调用ProcessProposal 开始处理这个请求
   1. 背书节点从Proposal, ChannelHeader, Signature,  重放和权限方面校验请求的有效性和合法性
   2. 背书节点创建交易模拟器和上下文, 然后调用simulateProposal->callChaincode, 调用相关链码执行模拟
   3. 背书节点根据传入的参数创建CCContext
   4. 背书节点根据channel name和传入的参数 调用chaincode , ExecuteChaincode->Execute
   5. Lunch启动chaincode, 因为当前我们调用的lscc, 已经启动, 所以Launch会直接返回
   6. 创建Chaincode Message 类型为TRANSACTION
   7. 调用ChainSupport.Execute()->sendExecuteMessage,  往handler.nextState 发送数据
   8. processStream收取message, 进行状态切换,因为是在ready下收到TRANSACTION, 无状态切换
   9. processStream通过与ChainCode已建立好的连接,把消息发出去ChatStream.Send.
3. 链码端 通过 chatWithPeer收到消息
   1. handleMessage 进行消息处理, 在ready收到TRANSACTION不会状态切换,但是会执行beforeTransaction
   2. beforeTransaction->handleTransaction会调用LSCC的**Invoke**
   3. 因为是deloy会调用executeDeploy,用GetState和账本交互,查询是否已经实例化过这个chaincode,  校验实例化策略, 生成ChaincodeData
   4. 调用createChaincode->putState->handlePutState. 以chainname为key, chaincodeData为value存到账本里.
   5. chaincode收到PUT_STATE调用enterBusyState触发RESPONSE 事件.发送RESPONSE给shim
   6. shim 发送message给responseChannel.
   7. 发送COMPLETE message给chaincode support, 切换到COMPLETE state.
4. 背书节点从callChaincode返回到 ProcessProposal
   1. 判断chainID是否是lscc, 并且是args 是depoy或upgrade, 这种情况下需要调用用户链码的Init
   2. 根据要deploy的chaincode创建NewCCContext 
   3. 调用Execute, 和最开始调用Execute类似. 这次执行的是 用户链码(mycc)的Init, 进行初始化
   4. 调用Launch->launchAndWaitForRegister->Start 去生成docker文件, deploy image. 等待链码端发送REGISTER
   5. 背书节点调用beforeRegisterEvent  并发送gREGISTER 给链码测. 在created收到REGISTER会进入established状态,调用enterEstablishedState->notifyDuringStartup 写readyNotify 通知已经启动
   6. Notify使launchAndWaitForRegister 退出, 返回至Lunch, 并发送ready给链码, 最后返回到Execute()
   7. 创建Chaincode Message 类型为INIT
   8. 调用ChainSupport.Execute()->sendExecuteMessage,  往handler.nextState 发送数据
   9. processStream收取message, 进行状态切换,因为是在ready下收到TRANSACTION, 无状态切换
   10. processStream通过与ChainCode已建立好的连接,把Init消息发出去ChatStream.Send().
   11. 之后会收到链码测的PUT_STATE, 执行enterBusyState调用模拟器txContext.txsimulator.SetState返回RESPONSE, 直到链码测回COMPLETE.
   12. 回到callChaincode, 通过 GetTxSimulationResults获取读写集
5. 背书节点从callChaincode返回到 ProcessProposal
   1. 调用endorseProposal() 对读写集进行背书
   2. 获取背书链码 escc
   3. 开始新一轮的callChaincode 
   4. ………………..省略流程,参考调用lscc, 之后会详解ESCC这个Chaincode
6. 回到 ProcessProposal 返回给客户端RESPONSE



步骤省略了如下步骤, 会放到之后的链码调用处解释:

客户端收到RESPONSE会发送SIngedTX  给排序节点, 排序节点排好序后 发回给commter节点, 进行VSCC校验,写入账本.


