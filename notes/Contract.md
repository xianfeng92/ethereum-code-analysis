## Contract

合约(Contract)是EVM用来执行(虚拟机)指令的结构体。先来看下Contract的定义：

* core/vm/[contract.go](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/contract.go)

```
// AccountRef implements ContractRef.
//
// Account references are used during EVM initialisation and
// it's primary use is to fetch addresses. Removing this object
// proves difficult because of the cached jump destinations which
// are fetched from the parent contract (i.e. the caller), which
// is a ContractRef.
type AccountRef common.Address

// Address casts AccountRef to a Address
func (ar AccountRef) Address() common.Address { return (common.Address)(ar) }

// Contract represents an ethereum contract in the state database. It contains
// the the contract code, calling arguments. Contract implements ContractRef
type Contract struct {
	// CallerAddress is the result of the caller which initialised this
	// contract. However when the "call method" is delegated this value
	// needs to be initialised to that of the caller's caller.
	CallerAddress common.Address
	caller        ContractRef
	self          ContractRef

	jumpdests destinations // result of JUMPDEST analysis.

	Code     []byte
	CodeHash common.Hash
	CodeAddr *common.Address
	Input    []byte

	Gas   uint64
	value *big.Int

	Args []byte

	DelegateCall bool
}
```

其中， caller tx 的发起者，即转账转出方， self 为转入方地址，它们的类型都是用接口 ContractRef 来表示的。CodeHash 是 Code 的RLP哈希值。Input是数据数组，是指令所操作的数据集合。Code 为指令数组，其中每一个byte都对应于一个__预定义的虚拟机指令__，[StateDB](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/notes/ethDB.md) 提供方法SetCode()，可以将指令数组Code存储在某个stateObject对象中;方法GetCode()，可以从某个stateObject对象中读取已有的指令数组Code。stateObject 是Ethereum里用来管理一个账户所有信息修改的结构体，它以一个Address类型变量为唯一标示符。StateDB 在内部用一个巨大的map结构来管理这些stateObject对象。所有账户信息-包括Ether余额，指令数组Code, 该账户发起合约次数nonce等-它们发生的所有变化，会首先缓存到StateDB里的某个stateObject里，然后在合适的时候，被StateDB一起提交到底层数据库。注意，一个Contract所对应的stateObject的地址，是Contract的self地址，也就是转帐的转入方地址。


EVM 目前有五个函数可以创建并执行Contract，按照作用和调用方式，可以分成两类:

* Create(), Call(): 二者均在StateProcessor的ApplyTransaction()被调用以执行单个交易，并且都有调用转帐函数完成转帐。

* CallCode(), DelegateCall(), StaticCall()：三者由于分别对应于不同的虚拟机指令(1 byte)操作，不会用以执行单个交易，也都不能处理转帐。


关于[Create和Call](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/notes/evm.md)











