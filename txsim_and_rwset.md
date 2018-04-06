## 交易模拟和读写集

 读写集官方资料: [link](https://hyperledger-fabric.readthedocs.io/en/release-1.0/readwrite.html)

实例化过程中会PutState ChaincodeData和执行用户链码中的Init. 这些对账本的操作都会保存在交易模拟器中, 背书节点会根据它们生成读写集(RwSet). 在模拟执行时账本数据不会更新, 在Commit阶段才会写入账本.

### 写集（WriteSet）

- 写集是模拟执行过程中交易写的键值, 无版本信息, 只有最新值
- 世界状态里面的都是三元组：（k, ver, val）
- 提交节点会遍历写集中的每个键，更新世界状态里对应的键值和版本号。

### 读集（Read Set）

- 读集是模拟执行过程中交易读取的已提交键值, 包含版本信息, 

- 版本是一个二元组Heigh(blockNumber, txNumber), 其中blockNumber是区块号, txNumber是区块内的交易编号.

- 提交节点根据读写集中的读集来验证交易.

  > 因为写集中只有最新值, 但是数据操作大多时候都是基于某一个值,进行增减, 而这里只有最终结果, 加入版本, 给最终结果一个起点.
  >
  > 举例子, a有100元, 消费20元, 实现上应该是先获取a的金额, 这里是100, 然后减去20, 最终结果就是80. 有可能同时别人也转给a了100元, 这两个交易, 如果同时需要提交到账本中, 必须有一个是无效的, 通过Read版本就可以做到. 如果交易排序后, 别人先转了100给a优先, 那a会先变成200, 并且更新版本. 当执行到80的写集时, 发现对应的读版本不匹配, a的值已经发生变化, 此写集就设置为无效.
  >


### 交易模拟器

链码在对账本数据的操作，都会被记录到了模拟器中. 交易模拟器的接口如下。

```go
// TxSimulator simulates a transaction on a consistent snapshot of the 'as recent state as possible'
// Set* methods are for supporting KV-based data model. ExecuteUpdate method is for supporting a rich datamodel and query support
type TxSimulator interface {
	QueryExecutor
	// 用于对指定的namespace进行key和value的设定。namespace对应Chaincode的
	SetState(namespace string, key string, value []byte) error
	// 删除指定namespace和key
	DeleteState(namespace string, key string) error
	 // 批量设置多个key的值
	SetStateMultipleKeys(namespace string, kvs map[string][]byte) error
	// ExecuteUpdate for supporting rich data model (see comments on QueryExecutor above)
	ExecuteUpdate(query string) error
	// 封装了事务模拟的结果，包含丰富的详细信息：    
    // - 交易的提交状态的更新将引起状态的更新；    
    // - 在提交交易时，对交易的执行环境进行有效性验证。
    // 对不同的账本以不同的形式展示上面提到的两点，实现对不同的数据模型的支持以及更好地展现相关信息
	GetTxSimulationResults() ([]byte, error)
}
//代码在core/ledger/ledger_interface.go
```

#### 逻辑示意图:

![mage-20180406223021](/var/folders/n6/4628kw492bg6xyg7b098h9380000gn/T/abnerworks.Typora/image-201804062230213.png)

- 每次链码调用的时候生成一个交易模拟器TxSimulator


- 所有的数据读取操作会通过queryHelper查询到真实的状态数据，并记录到rwset-Builder里，


- 数据的写入和删除操作只记录到rwsetBuilder里，并不会直接提交到状态数据库中。



### 数据分发

基于Golang 的上下文, 根据交易ID找到交易模拟器.



![mage-20180406223004](/var/folders/n6/4628kw492bg6xyg7b098h9380000gn/T/abnerworks.Typora/image-201804062230046.png)