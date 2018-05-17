## 实例化链码 - 系统链码与用户链码

链码( chaincode )分为系统链码和用户链码.

### 系统链码

系统链码负责Fabric节点的处理逻辑,包括系统配置, 背书校验 , 运行在节点进程中, 在Peer节点启动时自动完成注册和部署. 代码在code/scc/*下面.

- 生命周期管理系统链码（LSCC) 
- 配置管理系统链码（CSCC）   
- 查询管理系统链码（QSCC）     
- 交易背书系统链码（ESCC）
- 交易验证背书链码（VSCC）

之前介绍过CSCC. 实例化链码会用到LSCC, ESCC和VSCC有默认的实现，也可以根据功能需求实现新的ESCC和VSCC. 

### 用户链码

链码是独立可运行的应用程序, 运行在基于Docker的安全容器中, 在启动的时候和背书节点建立gRPC连接，在运行过程中通过接口和背书节点通信. 

它提供了基于分布式账本的状态处理逻辑. 它的生命周期被生命周期管理系统链码(LSCC)进程管理.

它的代码,由开发人员完成后, install 到 peer 节点, 然后再实例化绑定通道.

用户链码与 Peer 之间建立的是 gRPC 连接，系统链码建立的是 Golang 的通道连接

### 共同点

- 链码相关的代码在core/chaincode, core/chaincode/shim由链码测调用. 其他的在Peer端调用.

- 必须实现的接口

  ```go
  type Chaincode interface {
      // 初始化工作，一般情况下仅调用一次
  	Init(stub ChaincodeStubInterface) pb.Response
      // 用于在 proposal 请求中查询或更新账本数据, 状态知道交易被commit后才会更新到账本里面
  	Invoke(stub ChaincodeStubInterface) pb.Response
  }
  // 代码在core/chaincode/shim/interfaces.go
  ```

  > Invoke实现了查询接口, 不需要专门的Query接口

- 链码通过shim.ChaincodeStubInterface 实现与账本的交互, 链码是不允许直接操作账本的.

  ```go
  // ChaincodeStubInterface is used by deployable chaincode apps to access and
  // modify their ledgers
  type ChaincodeStubInterface interface {
  	// 返回调用函数名称和参数的列表, 第一个元素是函数名称, 类型是字节数组
  	GetArgs() [][]byte

  	// 返回调用函数名称和参数的列表, 第一个元素是函数名称, 类型是字符串
  	GetStringArgs() []string

  	// 分别返回调用的函数名称和参数, 类型是字符串
  	GetFunctionAndParameters() (string, []string)

  	// 返回调用函数名称和参数拼接在一个的字符数组. 有什么用呢?
  	GetArgsSlice() ([]byte, error)

  	// 获取交易号, 每次调用这个值都是唯一的
  	GetTxID() string

      // 根据指定条件查询状态数据库里存储的值
  	InvokeChaincode(chaincodeName string, args [][]byte, channel string) pb.Response

  	// 根据指定键查询状态数据库里存储的值
  	GetState(key string) ([]byte, error)

      // 向状态数据库写入键值对( 写入到读写集中, 记帐时才真正写入)
  	PutState(key string, value []byte) error

  	// 删除状态数据库中对应的值
  	DelState(key string) error

      // 查询状态数据库里键在[startKey, endKey)之间的值
  	GetStateByRange(startKey, endKey string) (StateQueryIteratorInterface, error)

  	// 部分组合键查询
  	GetStateByPartialCompositeKey(objectType string, keys []string) (StateQueryIteratorInterface, error)

  	// 构造组合键
  	CreateCompositeKey(objectType string, attributes []string) (string, error)

  	// 分割组合键
  	SplitCompositeKey(compositeKey string) (string, []string, error)

  	// 根据指定条件查询状态数据库里存储的值
  	GetQueryResult(query string) (StateQueryIteratorInterface, error)

  	// 查询一个键的历史数据
  	GetHistoryForKey(key string) (HistoryQueryIteratorInterface, error)

  	// 获取提交交易的身份信息 ( 包括 MSP 和证书信息 )
  	GetCreator() ([]byte, error)

  	// 获取一些私密信息, 这部分信息不会写入账本数据
  	GetTransient() (map[string][]byte, error)

  	// 获取交易的绑定信息, 包含随机数和提交交易的身份信息
  	GetBinding() ([]byte, error)

  	// 获取签名的 Proposal, 签名者是和提交交易的身份是一样的
  	GetSignedProposal() (*pb.SignedProposal, error)

  	// 获取提交时的时间戳, 基于UTC
  	GetTxTimestamp() (*timestamp.Timestamp, error)

  	// 设置事件的名称和内容
  	SetEvent(name string, payload []byte) error
  }
  // 代码在core/chaincode/shim/interfaces.go
  ```

  ​

