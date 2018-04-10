## 交易流程 - 提交节点

Commiter节点通过orderer的Deliver接口获取区块后会:

- 调用交易验证器(txValidator)的Validate对Block进行校验

  - Block数据结构
  - 签名和Creator是否匹配
  - TxID是否是由Nonce + Creator生成
  - chian是否存在
  - TxID是否有冲突
  - 调用VSCC来验证, 背书信息是否满足此chian的背书策略, 背书策略通过LSCC获取ChaincodeData 得到.
  - 合法的交易对应的TRANSACTIONS_FILTER设为VALID, 非法的根据错误类型设置不同的值

-  调用TxMgr的ValidateAndPrepare对block做MVCC的验证, 返回有效的UpdateBatch

  - 通过Validator的validataTx对交易中的读集进行校验
  - 流程是通过GetState获取Key在当前DB的版本信息, 与读集中的版本进行对比
  - 上步TRANSACTIONS_FILTER设为VALID的RwSet, 如果这里读集校验失败, 对应的TRANSACTIONS_FILTER设为非法
  - 校验合法的txRXSet会append到UpdateBatch返回

- 调用fsBlockStore的AddBlock 添加区块到账本

  - 是否超过区块大小
  - 追加区块大小和区块内容
  - 保存检查点信息
  - 建立区块索引
  - 更新检查点和区块链信息

- 调用txMgr的Commit 把有效的写集apply到state DB里

  - 调用VersionDb的ApplyUpdates, 把之前保存的UpdateBatch逐个写到state DB中

-  调用HistoryDB的Commit, 写History DB

  ​