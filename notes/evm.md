## EVM

*  core/vm/[evm.go](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/evm.go)

-----------------------------------------------------
## 合约预编译

如果待执行的Contract对象恰好属于一组 __预编译的合约集合__-此时以指令地址CodeAddr为匹配项-那么它可以直接运行；没有经过预编译的Contract，才会由Interpreter解释执行。这里的"预编译"，可理解为不需要编译(解释)指令(Code)。预编译的合约，其逻辑全部固定且已知，所以执行中不再需要Code，仅需Input即可。
```
// run runs the given contract and takes care of running precompiles with a fallback to the byte code interpreter.
func run(evm *EVM, contract *Contract, input []byte) ([]byte, error) {
	if contract.CodeAddr != nil {
		precompiles := PrecompiledContractsHomestead
		if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
			precompiles = PrecompiledContractsByzantium
		}
		if p := precompiles[*contract.CodeAddr]; p != nil {
			return RunPrecompiledContract(p, input, contract)
		}
	}
	return evm.interpreter.Run(contract, input)
}

// PrecompiledContractsHomestead contains the default set of pre-compiled Ethereum
// contracts used in the Frontier and Homestead releases.
var PrecompiledContractsHomestead = map[common.Address]PrecompiledContract{ // 默认的预编译合约
	common.BytesToAddress([]byte{1}): &ecrecover{},
	common.BytesToAddress([]byte{2}): &sha256hash{},
	common.BytesToAddress([]byte{3}): &ripemd160hash{},
	common.BytesToAddress([]byte{4}): &dataCopy{},
}

// RunPrecompiledContract runs and evaluates the output of a precompiled contract.
func RunPrecompiledContract(p PrecompiledContract, input []byte, contract *Contract) (ret []byte, err error) {
	gas := p.RequiredGas(input) // 计算预编译合约所需 gas
	if contract.UseGas(gas) {
		return p.Run(input)
	}
	return nil, ErrOutOfGas
}

// Precompiled contract gas prices 具体预编译合约的 gas

EcrecoverGas            uint64 = 3000   // Elliptic curve sender recovery gas price
Sha256BaseGas           uint64 = 60     // Base price for a SHA256 operation
Sha256PerWordGas        uint64 = 12     // Per-word price for a SHA256 operation
Ripemd160BaseGas        uint64 = 600    // Base price for a RIPEMD160 operation

```

当调用的是预编译合约 sha256hash， 其 run 方法如下：

```
func (c *sha256hash) Run(input []byte) ([]byte, error) {
	h := sha256.Sum256(input)
	return h[:], nil
}
```

----------------------------------------------------

### Create

#### 合约创建 （当 tx 的 to == nil）：

```
// Create creates a new contract using code as deployment code.
// parameter caller:tx 发起者  code：指令数组  value：转账金额
// return contractAddr:合约地址 leftOverGas:剩余gas
func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {

	// Depth check execution. Fail if we're trying to execute above the limit.
	if evm.depth > int(params.CallCreateDepth) { // CallCreateDepth ==  1024  调用/创建堆栈的最大深度
		return nil, common.Address{}, gas, ErrDepth
	}
	if !evm.CanTransfer(evm.StateDB, caller.Address(), value) { // 余额是否充足
		return nil, common.Address{}, gas, ErrInsufficientBalance
	}
	// Ensure there's no existing contract already at the designated address
	nonce := evm.StateDB.GetNonce(caller.Address())
	evm.StateDB.SetNonce(caller.Address(), nonce+1)

	contractAddr = crypto.CreateAddress(caller.Address(), nonce) // 生成合约地址
	contractHash := evm.StateDB.GetCodeHash(contractAddr)
	if evm.StateDB.GetNonce(contractAddr) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) {
		return nil, common.Address{}, 0, ErrContractAddressCollision
	}
	// Create a new account on the state
	snapshot := evm.StateDB.Snapshot()
	evm.StateDB.CreateAccount(contractAddr) // 生成一个合约账户
	if evm.ChainConfig().IsEIP158(evm.BlockNumber) {
		evm.StateDB.SetNonce(contractAddr, 1)
	}
	evm.Transfer(evm.StateDB, caller.Address(), contractAddr, value) // 转入账户 contractAddr, 转出账户 caller.Address()

	// initialise a new contract and set the code that is to be used by the
	// EVM. The contract is a scoped environment for this execution context
	// only.
	contract := NewContract(caller, AccountRef(contractAddr), value, gas)
	contract.SetCallCode(&contractAddr, crypto.Keccak256Hash(code), code) // 初始化一个合约，并设置其 code（指令数组）

	if evm.vmConfig.NoRecursion && evm.depth > 0 {
		return nil, contractAddr, gas, nil
	}

	if evm.vmConfig.Debug && evm.depth == 0 {
		evm.vmConfig.Tracer.CaptureStart(caller.Address(), contractAddr, true, code, gas, value)
	}
	start := time.Now()

	ret, err = run(evm, contract, nil) // 执行 tx 中的合约调用

	// check whether the max code size has been exceeded
	maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) && len(ret) > params.MaxCodeSize
	// if the contract creation ran successfully and no errors were returned
	// calculate the gas required to store the code. If the code could not
	// be stored due to not enough gas set an error and let it be handled
	// by the error checking condition below.
	if err == nil && !maxCodeSizeExceeded {
		createDataGas := uint64(len(ret)) * params.CreateDataGas
		if contract.UseGas(createDataGas) {
			evm.StateDB.SetCode(contractAddr, ret)
		} else {
			err = ErrCodeStoreOutOfGas
		}
	}

	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if maxCodeSizeExceeded || (err != nil && (evm.ChainConfig().IsHomestead(evm.BlockNumber) || err != ErrCodeStoreOutOfGas)) {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != errExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}
	// Assign err if contract code size exceeds the max while the err is still empty.
	if maxCodeSizeExceeded && err == nil {
		err = errMaxCodeSizeExceeded
	}
	if evm.vmConfig.Debug && evm.depth == 0 {
		evm.vmConfig.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
	}
	return ret, contractAddr, contract.Gas, err
}
```

-----------------------------------------------------------------------------------

### Call

#### 合约调用（当 tx 的 to != nil）：

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
	contract := NewContract(caller, to, value, gas) //创建一个Contract对象，并初始化其成员变量caller, to, value 和 gas
	contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr)) // 赋值Contract对象的Code, CodeHash, CodeAddr成员变量。其中 evm.StateDB.GetCode(addr) 取出合约地址中的 指令数组（code）  

	start := time.Now()

	// Capture the tracer start/end events in debug mode
	if evm.vmConfig.Debug && evm.depth == 0 {
		evm.vmConfig.Tracer.CaptureStart(caller.Address(), addr, false, input, gas, value)

		defer func() { // Lazy evaluation of the parameters
			evm.vmConfig.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
		}()
	}
	ret, err = run(evm, contract, input) // 调用run()函数执行该合约的指令，input为调用合约的参数

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

----------------------------------------------------

## 小结

evm.go中的Call()和Create()函数：

* Call:

> ret, err = run(evm, contract, input)

* Create:

> ret, err = run(evm, contract, nil)


Call() 有一个入参input类型为[]byte，而Create()有一个入参code类型同样为[]byte，没有入参input. 这两个[]byte都是Transaction对象tx的成员变量Payload！调用EVM.Create()或Call()的入口在StateTransition.TransitionDb()中， 当tx.Recipent为空时，tx.data.Payload 被当作所创建Contract的Code(创建合约)；当tx.Recipient 不为空时，tx.data.Payload 被当作Contract的Input（合约调用）。

---------------------------------------------------------------------------------------------------------------

## 补充

关于[run函数](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/notes/interpreter.md)










