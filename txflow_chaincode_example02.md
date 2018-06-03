## 交易流程 - chaincode example02

BYFN用到的用户链码是examples/chaincode/go/chaincode_example02/chaincode_example02.go

本次链码调用, 客户端使用的参数是["invoke","a","b","10"].会调用了下面的invoke(i是小写). 它同时支持delete query等.  参数代表的意思是 从a转10 到b.

```go

func (t *SimpleChaincode) invoke(stub shim.ChaincodeStubInterface, args []string) pb.Response {

    A := args[0]
    B := args[1]
	// 发送GET_STATE给背书节点 获取key a的值, 这里是100
	Avalbytes, err := stub.GetState(A)
	Aval, _ = strconv.Atoi(string(Avalbytes))
	// 发送GET_STATE给背书节点 获取key b的值, 这里是200
	Bvalbytes, err := stub.GetState(B)
	Bval, _ = strconv.Atoi(string(Bvalbytes))
    
	// 获取转账金额
	X, err = strconv.Atoi(args[2])

    // 运算后把最终结果用PUT_STATE送回到账本
	Aval = Aval - X
	Bval = Bval + X
	err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
	err = stub.PutState(B, []byte(strconv.Itoa(Bval)))

	return shim.Success(nil)
}
//代码在examples/chaincode/go/chaincode_example02/chaincode_example02.go
```

这里的逻辑是先获取值, 修改值, 把最终结果再塞回去. PUT_STATE在并不直接修改账本, 而是用于生成读写集. 读写集部分可参考实例化链码.



