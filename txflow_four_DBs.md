## 交易流程 - 四个DB

### StateDB

状态数据记录的是交易执行的结果, 它代表了对应Chain上所有键值的最新值, 也成为世界状态. 根据文件在系统的位置, 多个channel应该共享状态数据, 但是Key是compositeKey(ChainID+ 0x00 +Key). 所以数据是可以隔离开的.

常用的数据结构:

```go
type CompositeKey struct {
	Namespace string // 命名空间
	Key       string  // Key
}

type VersionedValue struct {
	Value   []byte
	Version *version.Height  // 区块高度
}

type VersionedKV struct {
	CompositeKey
	VersionedValue
}
//代码在core/ledger/kvledger/txmgmt/statedb/statedb.go

// 某个区块的某个TX
type Height struct {
	BlockNum uint64         // 第几个Block 
	TxNum    uint64			// 第几个交易
}
//代码在core/ledger/kvledger/txmgmt/version/version.go
```

##### Dump State DB

```shell
peer0.production git:release ❯❯❯ level-dump .//ledgersData/stateLeveldb
................
{ key: 'mychannel\u0000mycc\u0000a', value: '\u0001\u0004\u000090' }
{ key: 'mychannel\u0000mycc\u0000b', value: '\u0001\u0004\u0000210' }

```



### Block Index DB

查找一个区块就是先确定是哪个链，然后根据文件编号找到对应的文件，再根据文件的偏移量和占用的字节数确定区块的内容.

文件位置指针如下:

```go
type locPointer struct {
	offset      int		// 文件内的偏移量
	bytesLength int     // 区块占用的字节长度
}

// fileLocPointer
type fileLocPointer struct {
	fileSuffixNum int // 所在文件的编号
	locPointer
}
```

索引数据库有六种索引, 可以定位区块, 交易和交易验证码:

```go
func (index *blockIndex) indexBlock(blockIdxInfo *blockIdxInfo) error {

	flp := blockIdxInfo.flp
	batch := leveldbhelper.NewUpdateBatch()
	flpBytes, err := flp.marshal()

    //Index1 key:"h + blockNum" 根据哈希值获取区块（RetrieveBlockByHash）
	batch.Put(constructBlockHashKey(blockIdxInfo.blockHash), flpBytes)
	//Index2 key:"n + blockNum" 根据区块编号获取区块（RetrieveBlockByNumber）
	batch.Put(constructBlockNumKey(blockIdxInfo.blockNum), flpBytes)
	//Index3 key:"t + txID" 根据交易编号获取交易（RetrieveTxByID）
	for _, txoffset := range txOffsets {
		txFlp := newFileLocationPointer(flp.fileSuffixNum, flp.offset, txoffset.loc)
		txFlpBytes, marshalErr := txFlp.marshal()
		batch.Put(constructTxIDKey(txoffset.txID), txFlpBytes)
	}
	//Index4 - key:"a + blockNum + txNum"
    // 根据区块编号和交易编号获取交易（RetrieveTxByBlockNumTranNum）
	for txIterator, txoffset := range txOffsets {
		txFlp := newFileLocationPointer(flp.fileSuffixNum, flp.offset, txoffset.loc)
		txFlpBytes, marshalErr := txFlp.marshal()
		batch.Put(constructBlockNumTranNumKey(blockIdxInfo.blockNum, uint64(txIterator)), txFlpBytes)
	}
	// Index5 - key:"t + txID" 根据交易编号获取区块（RetrieveBlockByTxID）
	for _, txoffset := range txOffsets {
		batch.Put(constructBlockTxIDKey(txoffset.txID), flpBytes)
	}
	// Index6 - key:"v + txID"  根据交易编号获取交易验证码（RetrieveTxValidationCodeByTxID）
	for idx, txoffset := range txOffsets {
		batch.Put(constructTxValidationCodeIDKey(txoffset.txID), []byte{byte(txsfltr.Flag(idx))})
	}
    // 索引检查点
    batch.Put(indexCheckpointKey, encodeBlockNum(blockIdxInfo.blockNum))
}
```

#####  Dump

```shell
/c/f/f/peer0.production git:release ❯❯❯ level-dump .//ledgersData/chains/index
................
{ key: 'mychannel\u0000indexCheckpointKey', value: '\u0004' }
{ key: 'mychannel\u0000t07179411973a0741bc8c2e9d712e00dcf667e4b5e669c8344a31a785e5759e27',
  value: '\u0000��\u0002�\u0016' }
{ key: 'mychannel\u0000tb657116f94bff5b0303af22698d72b445fdc81e49f89552588d89aa5a272027a',
  value: '\u0000��\u0002�\u001a' }
{ key: 'mychannel\u0000v07179411973a0741bc8c2e9d712e00dcf667e4b5e669c8344a31a785e5759e27',
  value: '\u0000' }
{ key: 'mychannel\u0000vb657116f94bff5b0303af22698d72b445fdc81e49f89552588d89aa5a272027a',
  value: '\u0000' }

```



### History Index DB

历史数据记录了每个状态数据的历史消息, 每个历史消息的key以4个元素组成

- namespace：不同的chaincodeID
- writeKey：要写入数据的键
- blockNo：要写入数据所在的区块编号
- tranNo：要写入数据所在区块内的交易序号，从0开始

历史信息存储的是固定的空字节数组.

实现代码:

```go
func (historyDB *historyDB) Commit(block *common.Block) error {
	.............
	// 把每个合法交易的写集写到History数据库中
	for _, envBytes := range block.Data.Data {

		for _, nsRWSet := range txRWSet.NsRwSets {
			ns := nsRWSet.NameSpace
			for _, kvWrite := range nsRWSet.KvRwSet.Writes {
				writeKey := kvWrite.Key
				// 历史数据的composite key 是 ns~key~blockNo~tranNo
				compositeHistoryKey := historydb.ConstructCompositeHistoryKey(ns, writeKey, blockNo, tranNo)
				// 不需要value, 写空值进去 
				dbBatch.Put(compositeHistoryKey, emptyValue)
			}
		}
	}

}
//代码在core/ledger/kvledger/history/historydb/historyleveldb/historyleveldb.go
func ConstructCompositeHistoryKey(ns string, key string, blocknum uint64, trannum uint64) []byte {

	var compositeKey []byte
	compositeKey = append(compositeKey, []byte(ns)...)
	compositeKey = append(compositeKey, compositeKeySep...)
	compositeKey = append(compositeKey, []byte(key)...)
	compositeKey = append(compositeKey, compositeKeySep...)
	compositeKey = append(compositeKey, util.EncodeOrderPreservingVarUint64(blocknum)...)
	compositeKey = append(compositeKey, util.EncodeOrderPreservingVarUint64(trannum)...)

	return compositeKey
}
//代码在 core/ledger/kvledger/history/historydb/histmgr_helper.go
```

##### Dump

```shell
/c/f/f/peer0.production git:release ❯❯❯ level-dump .//ledgersData/historyLeveldb/
..........................
{ key: 'mychannel\u0000mycc\u0000a\u0000\u0001\u0003\u0000',
  value: '' }
{ key: 'mychannel\u0000mycc\u0000a\u0000\u0001\u0004\u0000',
  value: '' }
{ key: 'mychannel\u0000mycc\u0000b\u0000\u0001\u0003\u0000',
  value: '' }
{ key: 'mychannel\u0000mycc\u0000b\u0000\u0001\u0004\u0000',
  value: '' }
```

### IdStore DB

记录有那些账本.

```shell
peer0.production git:release ❯❯❯ level-dump .//ledgersData/ledgerProvider
{ key: 'lmychannel', value: '' }
```

