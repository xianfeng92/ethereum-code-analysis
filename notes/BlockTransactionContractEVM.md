
## SHA3

SHA-3在2015年8月由美国标准技术协会(NIST)正式发布，作为Secure Hash Algorithm家族的最新一代标准，它相比于SHA-2和SHA-1，采用了完全不同的设计思路，性能也比较好。需要注意的是，SHA-2目前并没有出现被成功攻克的案例，SHA-3也没有要立即取代SHA-2的趋势，NIST只是考虑到SHA-1有过被攻克的案例，未雨绸缪的征选了采用全新结构和思路的SHA-3来作为一种最新的SHA方案。


## RLP

RLP(Recursive Length Prefix),递归长度前缀编码，__它是以太坊序列化所采用的编码方式。RLP主要用于以太坊中数据的网络传输和持久化存储__。它可以将一个任意嵌套的字节数组([]byte)，编码成一个“展平”无嵌套的[]byte。RLP是可逆的，它提供了互逆的编码、解码方法。

Ethereum 中具体使用的哈希算法，就是__对某个类型对象的RLP编码值做了SHA3哈希运算，可称为RLP Hash__。 Ethereum 在底层存储中特意选择了专门存储和读取 [k, v] 键值对的第三方数据库，[k, v] 中的 v 就是某个结构体对象的RLP编码值([]byte)，k大多数情况就是v的RLP编码后的SHA-3哈希值。

## Hash and Address

```
const (
	HashLength    = 32
	AddressLength = 20
)

// Hash represents the 32 byte Keccak256 hash of arbitrary data.
type Hash [HashLength]byte

// Address represents the 20 byte address of an Ethereum account.
type Address [AddressLength]byte

```
在Ethereum 代码里，所有用到的哈希值，都使用该Hash类型，长度为32bytes，即256 bits；Ethereum 中所有跟帐号(Account)相关的信息，比如交易转帐的转出帐号(地址)和转入帐号(地址)，都会用该Address类型表示，长度20bytes, 即160bits。


## Block

区块(Block)是Ethereum的核心结构体之一。Block中带有一个Header(指针), Header结构体带有Block的所有属性信息。下面为Header的结构：

* Core/types/block.go:

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

Transaction，是Ethereum里__标示一次交易的结构体__，它的成员变量包括转帐金额，转入方地址等等信息。

* Core/types/transaction.go

```
type Transaction struct {
	data txdata
	// caches 避免多次调用导致性能损失
	hash atomic.Value
	size atomic.Value
	from atomic.Value
}

```

Transaction中主要数据结构是txdata，如下：

```
type txdata struct {
	AccountNonce uint64          `json:"nonce"    gencodec:"required"`
	Price        *big.Int        `json:"gasPrice" gencodec:"required"`
	GasLimit     uint64          `json:"gas"      gencodec:"required"`
	Recipient    *common.Address `json:"to"       rlp:"nil"` // nil means contract creation
	Amount       *big.Int        `json:"value"    gencodec:"required"`
	Payload      []byte          `json:"input"    gencodec:"required"`

	// Signature values
	V *big.Int `json:"v" gencodec:"required"`
	R *big.Int `json:"r" gencodec:"required"`
	S *big.Int `json:"s" gencodec:"required"`

	// This is only used when marshaling to JSON.
	Hash *common.Hash `json:"hash" rlp:"-"`
}

```

为了防止交易重播，以太坊要求每笔交易必须有一个nonce值。每一个账户从同一个节点发起交易时，这个nonce值从0开始计数，发送一笔nonce对应加1。当前面的nonce处理完成之后才会处理后面的nonce。注意这里的前提条件是相同的地址在相同的节点发送交易。每个tx都声明了自己的(Gas)Price 和 GasLimit。Price指的是单位Gas消耗所折抵的Ether多少，它的高低意味着执行这个tx有多么昂贵。GasLimit 是该tx执行过程中所允许消耗资源的总上限，通过这个值，我们可以防止某个tx执行中出现恶意占用资源的问题，这也是Ethereum中有关安全保护的策略之一。拥有独立的Price和GasLimit, 也意味着每个tx之间都是相互独立的。Recipient 为转入方地址，为空是表明是在创建合约。Amount 转账金额；Payload是重要的数据成员，它既可以__作为所创建合约的指令数组__，其中每一个byte作为一个单独的虚拟机指令；也可以作为__数据数组__，由合约指令进行操作。合约由以太坊虚拟机(Ethereum Virtual Machine, EVM)创建并执行; R、S、V 为交易发起者的签名，代表交易发起者的身份。

添加tx的方法：

```
func newTransaction(nonce uint64, to *common.Address, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
	if len(data) > 0 {
		data = common.CopyBytes(data)
	}
	d := txdata{
		AccountNonce: nonce,
		Recipient:    to,
		Payload:      data,
		Amount:       new(big.Int),
		GasLimit:     gasLimit,
		Price:        new(big.Int),
		V:            new(big.Int),
		R:            new(big.Int),
		S:            new(big.Int),
	}
	if amount != nil {
		d.Amount.Set(amount)
	}
	if gasPrice != nil {
		d.Price.Set(gasPrice)
	}

	return &Transaction{data: d}
}

```

## Transaction Execute




### StateProcessor

执行tx的入口函数是StateProcessor的Process()函数，其实现代码如下：

* /core/state_processor.go

```
func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
	var (
		receipts types.Receipts // 交易执行后的收据
		usedGas  = new(uint64)
		header   = block.Header()
		allLogs  []*types.Log
		gp       = new(GasPool).AddGas(block.GasLimit()) // GasPol 跟踪在Block 中执行 tx 期间可用的 gas， 即一个 Block 中最多可以消耗多少 gas
	)
	// Mutate the the block and state according to any hard-fork specs
	if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
		misc.ApplyDAOHardFork(statedb)
	}
        // 循环处理区块中的每笔交易
	for i, tx := range block.Transactions() {
		statedb.Prepare(tx.Hash(), block.Hash(), i)
		receipt, _, err := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, usedGas, cfg) // 交易的处理函数
		if err != nil {
			return nil, nil, 0, err
		}
		receipts = append(receipts, receipt)
		allLogs = append(allLogs, receipt.Logs...)
	}
	// Finalize the block, applying any consensus engine specific extras (e.g. block rewards)
	p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), receipts)

	return receipts, allLogs, *usedGas, nil
}
```

先来看看 Receipts 的结构：

```
// Receipt represents the results of a transaction.
type Receipt struct {
	// Consensus fields
	PostState         []byte `json:"root"`
	Status            uint64 `json:"status"`
	CumulativeGasUsed uint64 `json:"cumulativeGasUsed" gencodec:"required"`
	Bloom             Bloom  `json:"logsBloom"         gencodec:"required"`
	Logs              []*Log `json:"logs"              gencodec:"required"`

	// Implementation fields (don't reorder!)
	TxHash          common.Hash    `json:"transactionHash" gencodec:"required"`
	ContractAddress common.Address `json:"contractAddress"`
	GasUsed         uint64         `json:"gasUsed" gencodec:"required"`
}
```

PostState 存储了创建该Receipt对象时，整个Block内所有“帐户”的状态，即当时所在Block里所有stateObject对象的 RLP Hash值；Receipt 中有一个Log类型的数组，其中每一个Log对象记录了Tx中一小步的操作。这里Bloom，被用以验证某个给定的Log是否处于Receipt已有的Log数组中。

具体Log结构如下：

```
// Log represents a contract log event. These events are generated by the LOG opcode and
// stored/indexed by the node.
type Log struct {
	// Consensus fields:
	// address of the contract that generated the event
	Address common.Address `json:"address" gencodec:"required"`
	// list of topics provided by the contract.
	Topics []common.Hash `json:"topics" gencodec:"required"`
	// supplied by the contract, usually ABI-encoded
	Data []byte `json:"data" gencodec:"required"`

	// Derived fields. These fields are filled in by the node
	// but not secured by consensus.
	// block in which the transaction was included
	BlockNumber uint64 `json:"blockNumber"`
	// hash of the transaction
	TxHash common.Hash `json:"transactionHash" gencodec:"required"`
	// index of the transaction in the block
	TxIndex uint `json:"transactionIndex" gencodec:"required"`
	// hash of the block in which the transaction was included
	BlockHash common.Hash `json:"blockHash"`
	// index of the log in the receipt
	Index uint `json:"logIndex" gencodec:"required"`

	// The Removed field is true if this log was reverted due to a chain reorganisation.
	// You must pay attention to this field if you receive logs through a filter query.
	Removed bool `json:"removed"`
}
```

每一个tx的执行结果，__由一个Receipt对象来表示__；更具体一点，是由一组Log对象来记录。这个Log数组很重要，比如在不同Ethereum节点(Node)的相互同步过程中，待同步区块的Log数组有助于验证同步中收到的block是否正确和完整，所以会被单独同步(传输)。


### ApplyTransaction

```
// ApplyTransaction attempts to apply a transaction to the given state database
// and uses the input parameters for its environment. It returns the receipt
// for the transaction, gas used and an error if the transaction failed,indicating the block was invalid.

func ApplyTransaction(config *params.ChainConfig, bc *BlockChain, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, uint64, error) {
        // 根据 ChainConfig 以及 header.Number 封装一个 msg
	msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
	if err != nil {
		return nil, 0, err
	}
        // 在EVM环境中创建一个新的context， author为交易发起者或者合约调用者， msg主要是封装 tx 相关信息
	context := NewEVMContext(msg, header, bc, author)
        // 创建一个包含tx和调用机制的所有相关信息的新环境， 从这里可以出以太坊会为每tx创建一个 vmenv，来执行 tx
	vmenv := vm.NewEVM(context, statedb, config, cfg)
        // 在当前状态上执行交易
	_, gas, failed, err := ApplyMessage(vmenv, msg, gp) // 返回值 gas 为执行tx所消耗的gas
	if err != nil {
		return nil, 0, err
	}
	// Update the state with pending changes
	var root []byte
	if config.IsByzantium(header.Number) {
		statedb.Finalise(true)
	} else {
		root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
	}
	*usedGas += gas

	// 为 tx 创建一个 receipt， 主要包括 TxHash，GasUsed，failed
	receipt := types.NewReceipt(root, failed, *usedGas)
	receipt.TxHash = tx.Hash()
	receipt.GasUsed = gas
	// 如果 tx 创建了一个合约，则将合约地址存储在收据中。
	if msg.To() == nil {
		receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
	}
	// 设置 receipt 日志并创建 Bloom Filter
	receipt.Logs = statedb.GetLogs(tx.Hash())
	receipt.Bloom = types.CreateBloom(types.Receipts{receipt})

	return receipt, gas, err
}

```

看看 AsMessage:

```
// AsMessage 将 tx 封装成一个core.Message.
// AsMessage 需要一个签名去提取 the sender.
func (tx *Transaction) AsMessage(s Signer) (Message, error) {
	msg := Message{
		nonce:      tx.data.AccountNonce,
		gasLimit:   tx.data.GasLimit,
		gasPrice:   new(big.Int).Set(tx.data.Price),
		to:         tx.data.Recipient,
		amount:     tx.data.Amount,
		data:       tx.data.Payload,
		checkNonce: true,
	}

	var err error
	msg.from, err = Sender(s, tx)
	return msg, err
}
```

AsMessage 主要封装 tx 的相关信息，如 nonce，gasLimit，gasPrice，交易接受者 to，交易金额 amount 以及 Payload 数据。

ApplyTransaction 的核心是 ApplyMessage 的执行，即对msg的处理。下面看看ApplyMessage：

```
// ApplyMessage 通过给定消息（Message）计算新的状态
func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, uint64, bool, error) {
	return NewStateTransition(evm, msg, gp).TransitionDb()
}
```
ApplyMessage 中创建一个新的StateTransition，然后传入 TransitionDb 中进行tx的执行。


先看看StateTransition：

```
/*
The State Transitioning Model（状态转换模型）

状态转换是将 tx 应用到当前世界状态时所做的更改，状态转换模型执行所有必要的工作，以生成有效的新状态根。

1) Nonce handling
2) Pre pay gas
3) Create a new state object if the recipient is \0*32
4) Value transfer
== If contract creation ==
  4a) Attempt to run transaction data
  4b) If valid, use result as code for the new state object
== end ==
5) Run Script section
6) Derive new state root
*/
type StateTransition struct {
	gp         *GasPool
	msg        Message
	gas        uint64
	gasPrice   *big.Int
	initialGas uint64
	value      *big.Int
	data       []byte
	state      vm.StateDB
	evm        *vm.EVM
}

```

TransitionDb函数：

```
 // TransitionDb 通过 message 的执行来改变账户状态
// 返回 usedGas 以及 是否成功执行 tx
func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
	if err = st.preCheck(); err != nil {
		return
	}
	msg := st.msg // 获取 tx 相关信息
	sender := st.from() // err checked in preCheck

	homestead := st.evm.ChainConfig().IsHomestead(st.evm.BlockNumber)
	contractCreation := msg.To() == nil // 判断 tx 是否为合约创建

	gas, err := IntrinsicGas(st.data, contractCreation, homestead)
	if err != nil {
		return nil, 0, false, err
	}
	if err = st.useGas(gas); err != nil {
		return nil, 0, false, err
	}

	var (
		evm = st.evm
		// vm errors do not effect consensus and are therefor
		// not assigned to err, except for insufficient balance
		// error.
		vmerr error
	)
	if contractCreation { // 合约创建
		ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
	} else { // 转账或者合约调用
		// nonce值增1，待下一个 tx 使用
		st.state.SetNonce(sender.Address(), st.state.GetNonce(sender.Address())+1)
		ret, st.gas, vmerr = evm.Call(sender, st.to().Address(), st.data, st.gas, st.value)
	}
	if vmerr != nil { // 出现 EVM 错误的情况
		log.Debug("VM returned with error", "err", vmerr)
		// The only possible consensus-error would be if there wasn't
		// sufficient balance to make the transfer happen. The first
		// balance transfer may never fail.
		if vmerr == vm.ErrInsufficientBalance {
			return nil, 0, false, vmerr
		}
	}
	st.refundGas()
        // 奖励所属区块的创建者 ETH = st.gasUsed() * st.gasPrice
	st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))

	return ret, st.gasUsed(), vmerr != nil, err
}

```

先看看 IntrinsicGas 函数：

```
// IntrinsicGas 计算 message 中携带的 data （tx.data.Payload） 所需付的 gas
func IntrinsicGas(data []byte, contractCreation, homestead bool) (uint64, error) {
	// Set the starting gas for the raw transaction
	var gas uint64
	if contractCreation && homestead {
		gas = params.TxGasContractCreation // 53000  Per transaction that creates a contract. NOTE: Not payable on data of calls between transactions.
	} else {
		gas = params.TxGas // 21000  Per transaction not creating a contract. NOTE: Not payable on data of calls between transactions.
	}
	// 根据 transactional data 的大小来计算所需消耗的gas
	if len(data) > 0 {
		// Zero and non-zero bytes are priced differently
		var nz uint64
		for _, byt := range data { // 统计 data 中非零字节的数量
			if byt != 0 {
				nz++
			}
		}
		// Make sure we don't exceed uint64 for all data combinations
		if (math.MaxUint64-gas)/params.TxDataNonZeroGas < nz {
			return 0, vm.ErrOutOfGas
		}
		gas += nz * params.TxDataNonZeroGas // gas += gas + nz * 68

		z := uint64(len(data)) - nz
		if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
			return 0, vm.ErrOutOfGas
		}
		gas += z * params.TxDataZeroGas // gas = gas + z * 4
	}
	return gas, nil
}

```


再看看 refundGas 函数：

```
func (st *StateTransition) refundGas() {
	// gas的退回
	refund := st.gasUsed() / 2
	if refund > st.state.GetRefund() {
		refund = st.state.GetRefund()
	}
	st.gas += refund

	// tx 发起者
	sender := st.from()

	remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice) // 计算剩余 gas 可兑换的 ETH = st.gas * st.gasPrice
	st.state.AddBalance(sender.Address(), remaining)

        // 将 remaining gas 返回到 block gas pool, 以便执行下一个 tx 时使用
	st.gp.AddGas(st.gas)
}


 // GetRefund 返回 refund counter 的当前值
func (self *StateDB) GetRefund() uint64 {
	return self.refund
}

```

































