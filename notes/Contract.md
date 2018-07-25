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

其中， caller tx 的发起者，即转账转出方， self 为转入方地址，它们的类型都是用接口 ContractRef 来表示的。Code 为指令数组，其中每一个byte都对应于一个__预定义的虚拟机指令__。CodeHash 是 Code 的RLP哈希值。Input是数据数组，是指令所操作的数据集合。


















