## evm

### call

* core/vm/[evm.go](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/evm.go)

```
// 以给定的输入作为参数执行与 addr 相关联的合约
// 它还处理所需的任何值传递，并采取必要的步骤来创建帐户，在执行错误或值传递失败的情况下 revert 状态
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
	if evm.vmConfig.NoRecursion && evm.depth > 0 { // evm.vmConfig.NoRecursion 禁用 Interpreter call, callcode, delegate call and create
		return nil, gas, nil
	}

	// 如果我们试图执行高于堆栈调用深度的限制，则失败
	if evm.depth > int(params.CallCreateDepth) { //  CallCreateDepth == 2014,调用/创建堆栈的最大深度。
		return nil, gas, ErrDepth
	}
	// 检查交易发起者账户余额
	if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, gas, ErrInsufficientBalance
	}

	var (
		to       = AccountRef(addr) // 交易接收方地址
		snapshot = evm.StateDB.Snapshot() // 对 EVM 状态数据库做一个快照
	)
	if !evm.StateDB.Exist(addr) { // 交易接收方地址不在 evm 数据库中
		precompiles := PrecompiledContractsHomestead
		if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
			precompiles = PrecompiledContractsByzantium
		}
		if precompiles[addr] == nil && evm.ChainConfig().IsEIP158(evm.BlockNumber) && value.Sign() == 0 {
			// Calling a non existing account, don't do antything, but ping the tracer
			if evm.vmConfig.Debug && evm.depth == 0 {
				evm.vmConfig.Tracer.CaptureStart(caller.Address(), addr, false, input, gas, value)
				evm.vmConfig.Tracer.CaptureEnd(ret, 0, 0, nil)
			}
			return nil, gas, nil
		}
		evm.StateDB.CreateAccount(addr) // 在 evm 状态数据库中创建一个 addr 账户
	}
	evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value) // 转账

	// 初始化一个新的 contract 并设置 EVM 要使用的 code
	// 该 contract 仅作用于此执行 context 的范围环境
	contract := NewContract(caller, to, value, gas)
	contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

	start := time.Now()

	// Capture the tracer start/end events in debug mode
	if evm.vmConfig.Debug && evm.depth == 0 {
		evm.vmConfig.Tracer.CaptureStart(caller.Address(), addr, false, input, gas, value)

		defer func() { // Lazy evaluation of the parameters
			evm.vmConfig.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
		}()
	}
	ret, err = run(evm, contract, input)

	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != errExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}
	return ret, contract.Gas, err
}
```


CanTransfer, Transfer，在Context初始化时从外部传入，目前使用的均是一个本地实现：

* core/[evm.go](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/evm.go)

```
// NewEVMContext 在EVM中创建一个新的 context 
func NewEVMContext(msg Message, header *types.Header, chain ChainContext, author *common.Address) vm.Context {
	// If we don't have an explicit author (i.e. not mining), extract from the header
	var beneficiary common.Address
	if author == nil {
		beneficiary, _ = chain.Engine().Author(header) // Ignore error, we're past header validation
	} else {
		beneficiary = *author
	}
	return vm.Context{
		CanTransfer: CanTransfer, // 传入函数
		Transfer:    Transfer,
		GetHash:     GetHashFn(header, chain),
		Origin:      msg.From(),
		Coinbase:    beneficiary,
		BlockNumber: new(big.Int).Set(header.Number),
		Time:        new(big.Int).Set(header.Time),
		Difficulty:  new(big.Int).Set(header.Difficulty),
		GasLimit:    header.GasLimit,
		GasPrice:    new(big.Int).Set(msg.GasPrice()),
	}
}


// CanTransfer 检查地址帐户是否有足够的资金"转移"
func CanTransfer(db vm.StateDB, addr common.Address, amount *big.Int) bool {
	return db.GetBalance(addr).Cmp(amount) >= 0
}

// Transfer 从 sender 余额中减去 amount， 向 recipient 余额中添加 amount
func Transfer(db vm.StateDB, sender, recipient common.Address, amount *big.Int) {
	db.SubBalance(sender, amount)
	db.AddBalance(recipient, amount)
}

```
