## 实例化链码

### 客户端

```shell
peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR ('Org1MSP.member','Org2MSP.member')"
```

- 在指定通道对安装过的链码进行实例化, 链码在实例化之前是和通道无关的, 实例化的时候才绑定通道.
- 需提供初始化参数, 节点上会创建容器, 并调用链码的Init方法进行初始化操作
- 实例化链码需要同时和Orderer和Peer节点打交道
- 执行实例化的用户,必须有在此通道上的Write权限, 默认是Admin
- 实例化可以指定背书策略,  这个例子的策略是, 组织1或组织2的任何一个member
- 链码实例化后进入ready状态.



### 链码实例化过程



![](_images/instantiate_cc_flow.png)

### Peer节点

客户端发过来的SignedProposal, 由Peer节点的系统链码LSCC的Invoke处理, 发现是DEPLOY,会调用executeDeploy.

用户链码相关的代码都在core/chaincode路径下. 其中core/chaincode/shim包中的代码主要是供链码容器侧调用使用，其他代码主要是Peer侧使用. 

系统链码并不运行在Docker容器内, 而是运行在Peer主进程内. 其他方面,系统链码和用户链码基本是一致的:

- 需要shim层, 和Peer沟通
- 操作账本需要利用shim.Chain-codeSubInterface提供的API
- 链码状态机和Peer侧状态机

```go
func (lscc *LifeCycleSysCC) executeDeploy(stub shim.ChaincodeStubInterface, chainname string, depSpec []byte, policy []byte, escc []byte, vscc []byte) (*ccprovider.ChaincodeData, error) {
	// 获取Signed Proposal中的CDS
	cds, err := utils.GetChaincodeDeploymentSpec(depSpec)
    
	// 检查名字和版本, 和Install时类似.
	lscc.isValidChaincodeName(cds.ChaincodeSpec.ChaincodeId.Name)
	err = lscc.isValidChaincodeVersion(cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version)
    // 做账号访问控制, 当前的实现是空
	err = lscc.acl(stub, chainname, cds)

    // 测试是否这个chaincode已经在LSCC中存在(实例化过)
	_, err = lscc.getCCInstance(stub, cds.ChaincodeSpec.ChaincodeId.Name)

	// 从文件系统中读出chaincode, 返回CDSPackage
	ccpack, err := ccprovider.GetChaincodeFromFS(cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version)
	
	// 得到ChianCodaData
	cd := ccpack.GetChaincodeData()

	// 把ESCC VSCC名字赋值, 还有背书策略
	cd.Escc = string(escc)
	cd.Vscc = string(vscc)
	cd.Policy = policy

	// 获取实例化策略并用它evalutate 
	cd.InstantiationPolicy, err = lscc.getInstantiationPolicy(chainname, ccpack)
	err = lscc.checkInstantiationPolicy(stub, chainname, cd.InstantiationPolicy)
    // 基于chaindata进行实例化
	err = lscc.createChaincode(stub, cd)
	return cd, err
}
//代码在core/scc/lscc/lscc.go
```

在core/scc/lscc/lscc.go中的函数调用顺序:

1. createChaincode
2. putChaincodeData(判断传入的是否是系统链码)
3. stub.PutState 最终会调用到core/chaincode/shim/handler.go中的 handlePutState

PutState是账本提供一个API, 用于添加或更新一对Key-value, 这对键值会被放到读写集中, 等待Committer验证后写到账本中.

实例化链码, 要往账本里写入什么呢?

```go
func (lscc *LifeCycleSysCC) putChaincodeData(stub shim.ChaincodeStubInterface, cd *ccprovider.ChaincodeData) error {
	.......
	cdbytes, err := proto.Marshal(cd)
	err = stub.PutState(cd.Name, cdbytes)

	return err
}
//代码在core/scc/lscc/lscc.go 
```

ChaincodeData的定义如下:

```go
type ChaincodeData struct {    
    // 链码名称   
    Name string    
    // 链码版本    
    Version string    
    // 链码的ESCC，默认是内置的escc
     Escc string    
    // 链码的VSCC，默认是内置的vscc    
    Vscc string    
    // 链码的背书策略    
    Policy []byte    
    // 链码源代码哈希和元数据哈希组成的
    CDSData    Data []byte    
    // 链码编号，预留字段    
    Id []byte    
    // 链码实例化策略    
    InstantiationPolicy
}
type CDSData struct {    
    // 链码源代码哈希=hash(chaincode)    
    CodeHash []byte    
    // 链码元数据哈希=hash(chaincodeName+chaincodeVersion)    
    MetaDataHash []byte}
}
```

以链码名为Key, ChaincodeData为value, 更新账本. 

