## 共识算法 - 工作量证明

工作量证明是比特币使用的一种共识算法, 皆在不可信的分布式网络中, 使各节点数据一致.

它是一种基于彩票中奖的算法, 记账权的获得是有一定概率的, 就像买彩票. 它是一种先记账再共识的算法, 缺点就是区块确认时间过长. 

PoW(Proof of Work)算法思路: 

- 限制一段时间内整个网络中出现提案的个数(增加提案成本)
- 放宽对最终一致性确认的需求,约定好大家都确认并沿着已知最长的链进行拓宽.

也就是说,如果比特币系统在某一个时刻同时出现了两个都合法的区块,那么两个都承认.于是,区块链上会出现两个合法的分支(术语叫 " 分叉 ").此时矿工可以选择任何一个分支继续,在某个分支的长度超过了另一个分支时(应该是超过两个区块?),短的那个分支马上作废.

这种分叉,在严格的商业网络中, 是不允许的, 它可以允许一段时间内数据是未更新的, 但不能是分叉的.

### 挖矿 - 提案的成本

挖矿就是同通过Hash运算找到一个特定的组合使它的Hash小于某一个特定值:

```
SHA-256(SHA-256 (Block Header)) < Target
```

Target 是一个前面有多个0 hash字符串,  具体是几个0, 由难度系数Bits决定, 所有节点的Bits都是相同的, 你无法更改自己的Block Header去修改它, 它每2016个区块会调整一次, 调整是基于这2016个区块花费的时间决定,由算法决定, 所以所有节点上的BIts都是一样的. 

区块头里能随意修改的只有Nonce, Timestamp需要在特定范围内修改, Merkle Root 和你打包的交易有关.

例如下面这个区块有18个零:

[0000000000000000001c5874d874f31ef8d6d7b8e0aa8586773467fb57136733](https://blockchain.info/block/0000000000000000001c5874d874f31ef8d6d7b8e0aa8586773467fb57136733)

一个 SHA-256 算法算出来的哈希值有 2\*\*256 种可能性,而前面有 18 个零意味着前面有 72 个 bits 是零.于是,满足条件的哈希值是有 2\*\*(256 - 72) 种可能性,概率是 1/ (2\*\*72) .Nonce只有32位, 所以大概率情况下, 遍历完nonce也没有算出达到要求的Hash值. 这时就需要从待记账区获取交易, 哪怕只更新一笔交易. 或是更新Timestamp.

设置这个难度系数就是为了让全网产生的区域名平均在 10 分钟一块.为了保证每 10 分钟产生一个区块,当算力不足的时候,难度下降,当算力充足的时候,难度提高.

大概10分钟才会产生一个合法的区块, 产生后丢到网络中去, 收到区块的节点很容易就能交易这个区块的合法性, 如果合法, 它会把这个区块帮忙丢出去, 然后基于这个区块的Hash值, 开始新区块的计算.

### 最终一致性

10分钟才产生一个合法的区块, 多个节点同时产生合法区块的概率较小, 即使同时出现了两个合法的区块. 也可以通过让矿工选择, 选择某一个区块的算力最多, 逐渐这个分支就会胜出, 越胜出, 选择它的算力就会越多, 当无人问津区块短的那个分支后, 就达到了一致性.

但是如果你超过51%的算力, 你就能取消自己的花费. 这就是51%的算力攻击, 能有这么多的算力很难. 如果有, 从利益的角度讲, 是否有必要会去黑这个网络呢?

假设作弊者的计算资源(算力)只占整个系统的 30%,那么连续两次获得记账权的概率是 9%,看起来作弊的可能性还是挺高的,如果是连续 6 次获得记账权呢？概率直降到万分之七.

在比特币中,这个 6 也就是 6 次确认,表示连续 6 个块过去了,交易被篡改的可能性将越来越小,最后变得几乎不可能被篡改.这也是区块链不可被篡改说法的由来.

试想,如果任何作弊者花了大量的成本获取了系统 30% 的计算资源(算力),最后只有万分之七的概率获得篡改的可能性,比起作弊,还不如诚实记账的收益高.
