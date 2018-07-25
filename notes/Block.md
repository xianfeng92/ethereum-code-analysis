## Block

区块(Block)是Ethereum的核心结构体之一。Block中带有一个Header(指针), Header结构体带有Block的所有属性信息。下面为Header的结构：

* Core/types/[block.go](https://github.com/xianfeng92/go-ethereum/blob/master/core/types/block.go):

```
// Header represents a block header in the Ethereum blockchain.
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
	Number      *big.Int       `json:"number"           gencodec:"required"`
	GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
	GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
	Time        *big.Int       `json:"timestamp"        gencodec:"required"`
	Extra       []byte         `json:"extraData"        gencodec:"required"`
	MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
}
```
其中 ParentHash 是当前区块的父区块的Hash值， 我们可以将该Hash值和其他字符串一起组成key，然后在KV数据库中查询相应的value才能解析到；UncleHash 为当前区块的叔区块的hash值；Root、TxHash、ReceiptHash分别为三个MTP树（状态树，交易数和收据树）的root hash；Bloom类型是一个Ethereum内部实现的一个256bit长Bloom Filter，它可用来快速验证一个新收到的对象是否处于一个已知的大量对象集合之中；Number表示该区块在整个区块链(BlockChain)中所处的位置，每一个区块相对于它的父区块，其Number值是+1。

下面是Body的结构：

```
// Body is a simple (mutable, non-safe) data container for storing and moving
// a block's data contents (transactions and uncles) together.
type Body struct {
	Transactions []*Transaction
	       []*Header
}

```

Body主要是存储 Transaction 和 Uncle。


完整的Block结构如下：

```
// Block represents an entire block in the Ethereum blockchain.
type Block struct {
	header       *Header
	uncles       []*Header
	transactions Transactions

	// caches
	hash atomic.Value
	size atomic.Value

	// Td is used by package core to store the total difficulty
	// of the chain up to and including the block.
	td *big.Int

	// These fields are used by package eth to track
	// inter-peer block relay.
	ReceivedAt   time.Time
	ReceivedFrom interface{}
}
```

