# 以太坊中 tx 的具体流程分析

-----------------------------------------------------------------------

## 发起 tx

###  MetaMask 中发起 tx

MetaMask中转账tx的相关操作界面是这样子的:


图中需要我们填写的字段有: Recipient Address、 Amount、 TRANSACTION DATA、GasLimit、Gas Price. 填写好相关字段后,点击 SUBMIT 以后就会创建一笔新的 tx, 并发送到以太坊的网络中.

先来看看源码中newTransaction：

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
其实我们在MetaMask 中输入的信息是用来初始化一个 tx, 其中 TRANSACTION DATA 会赋值给Payload, V、R、S是MetaMask对 tx 发起者的Address 所做的签名.使用signer.Sender(tx)可对txdata 的V、R 、S三个数进行解密得到这个交易的签名公钥，即发送方的 address 。发送方的地址在交易数据中是没有的，这主要是为了防止交易数据被篡改，任何交易数据的变化后通过signer.Sender方法都不能得到正确的地址。

以太坊中, Downloader 专门从网络获取 tx 以及 Block.  当一个节点获取到 tx 时, 会将其加入到 tx_pool. tx_pool 会验证这笔 tx, 通过验证并且nonce值满足要求即会将其加入到 pending 中.  矿工在挖矿前都会处理 tx_pool中处理pending状态的 tx.


## StateProcessor

* /core/[state_processor.go](https://github.com/xianfeng92/go-ethereum/blob/master/core/state_processor.go)

执行tx的入口函数是 StateProcessor 的Process()函数，其会遍历block的所有交易。实现代码如下：

```
func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
var (
receipts types.Receipts // 交易执行后的收据
usedGas  = new(uint64)
header   = block.Header()
allLogs  []*types.Log
gp       = new(GasPool).AddGas(block.GasLimit()) // GasPol 跟踪在Block 中执行 tx 期间可用的 gas， 即一个 Block 中最多可以消耗多少　gas
)
// Mutate the the block and state according to any hard-fork specs
if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
misc.ApplyDAOHardFork(statedb)
}
// 循环处理区块中的 tx
for i, tx := range block.Transactions() {
statedb.Prepare(tx.Hash(), block.Hash(), i)
// 交易的处理函数，会返回一个收据　receipt
receipt, _, err := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, usedGas, cfg) 
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


Receipts 是 tx 执行的一种凭证，其一般会记录 tx 执行前后的一些状态和 Log，其结构如下：
```
// Receipt 代表交易的结果
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

PostState 存储了创建该Receipt对象时，整个Block内所有“帐户”的状态，即当时所在Block里所有　stateObject　对象的 RLP Hash值；Receipt 中有一个Log类型的数组，其中每一个Log对象记录了Tx中一小步的操作。这里Bloom，被用以验证某个给定的Log是否处于Receipt已有的Log数组中。每一个tx的执行结果，__由一个Receipt对象来表示__；更具体一点，是由一组Log对象来记录。这个Log数组很重要，比如在不同Ethereum节点(Node)的相互同步过程中，待同步区块的Log数组有助于验证同步中收到的block是否正确和完整，所以会被单独同步(传输)。


## ApplyTransaction

ApplyTransaction()的具体实现，它的基本流程如下图：

![](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/images/ExecTx.png)

```
// ApplyTransaction 尝试将 tx 应用于给定的状态数据库,并将输入参数用于其环境
// 它返回 tx 的收据以及使用的gas。如果tx执行失败，则返回错误，表示块无效。

func ApplyTransaction(config *params.ChainConfig, bc *BlockChain, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, uint64, error) {
// 根据 ChainConfig 以及 header.Number 封装一个 msg
msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
if err != nil {
return nil, 0, err
}
// 在EVM环境中创建一个新的 context， author 为交易发起者或者合约调用者， msg 主要是封装 tx 相关信息
context := NewEVMContext(msg, header, bc, author)
// 创建一个包含 tx 和调用机制的所有相关信息的新环境。从这里可以出以太坊会为每个 tx 创建一个 vmenv 来执行 tx
vmenv := vm.NewEVM(context, statedb, config, cfg)
// 在当前状态上执行交易
_, gas, failed, err := ApplyMessage(vmenv, msg, gp) // 返回值为执行　tx　所消耗的gas
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

### 1 tx 封装成Message对象

```
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
此时 message 中已经包含一个 tx 所有的信息, 并且已经将交易发起者的　address　提取出来了.


### 2 evm 执行环境的创建

首先, 使用 msg,、header、区块链 bc以及挖矿address 来创建一个 context, 然后依据 context、状态数据库 statedb 、区块链的配置以及虚拟机解释器( Interpreter) 的配置来创建一个虚拟机执行环境(vmenv). 在该环境中, 完成 tx 的执行.

```
// Create a new context to be used in the EVM environment
context := NewEVMContext(msg, header, bc, author)
// Create a new environment which holds all relevant information
// about the transaction and calling mechanisms.
vmenv := vm.NewEVM(context, statedb, config, cfg)
```

### 3 tx的处理

有了vmenv,接下来就是 tx 的处理了.

涉及的函数:
```
// ApplyMessage 通过给定消息（Message）计算新的状态
_, gas, failed, err := ApplyMessage(vmenv, msg, gp)

func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, uint64, bool, error) {
return NewStateTransition(evm, msg, gp).TransitionDb()
}
```
 state_processor 负责调用 ApplyMessage 完成msg(tx)的执行.  __简单一点的解释, 所有的 tx 其实是告诉 state_processor(矿工) 来做一些事--根据 tx 的内容来改变相关账户的一些状态值. 这里,state_processor(矿工)的可信度是相当重要的, 一个好的区块链应该确保所有state_processor(矿工)对 tx 的处理结果的完全可信, 这才是区块链技术的精髓__.

在 tx 的处理中,会使用虚拟机环境vmenv, msg以及 gaspool的gp值生成一个StateTransition对象,然后调用其TransitionDb函数来完成 tx.

 StateTransition 对象的结构如下：
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


再来看看 TransitionDb 中的处理逻辑:

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
执行流程如下:

* 计算固定的gas消耗: IntrinsicGas
分为两个部分，每一个tx预设的消耗量，如果tx为合约创建,则固定gas 消耗为53000 gas,tx只是转账,则为21000; 针对tx.data.Payload的Gas消耗，Payload类型是[]byte，关于它的固有消耗依赖于[]byte中非0字节和0字节的长度。如果此时出现gas不足,则会退出 tx 的执行.

* 依据 msg.To() == nil 来判断 tx 为合约创建还是转账或合约调用

1 msg.To() == nil 为 true, 则为合约创建

2 msg.To() == nil 为 false, 则为合约调用


###  3.1合约创建

tx 如果为合约创建, 则调用 evm.Create 来创建一个合约对象.

```
ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
```

合约创建的入参 code：指令数组 即为智能合约编译后的字节码.

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
evm.StateDB.CreateAccount(contractAddr) // 在StateDB中生成一个合约账户
if evm.ChainConfig().IsEIP158(evm.BlockNumber) {
evm.StateDB.SetNonce(contractAddr, 1)
}
// 向合约中转入一定量(value)的以太币
evm.Transfer(evm.StateDB, caller.Address(), contractAddr, value) // 转入账户 contractAddr, 转出账户 caller.Address()

// 创建一个合约实例 contract
contract := NewContract(caller, AccountRef(contractAddr), value, gas)
// 将编译后的字节码 code 存储到 contract
contract.SetCallCode(&contractAddr, crypto.Keccak256Hash(code), code) // 初始化一个合约，并将编译后的字节码 code 设给　contract的Code 属性上

if evm.vmConfig.NoRecursion && evm.depth > 0 {
return nil, contractAddr, gas, nil
}

if evm.vmConfig.Debug && evm.depth == 0 {
evm.vmConfig.Tracer.CaptureStart(caller.Address(), contractAddr, true, code, gas, value)
}
start := time.Now()

// 调用合约, 其中 input 为 nil, 检查code中的 operation 的合法性以及存储 code 所需的 memory
ret, err = run(evm, contract, nil)

// 检查 code size 是否过大
maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) && len(ret) > params.MaxCodeSize

// 如果 contract 创建成功,计算存储 code 所需消耗的 gas
// 如果存储 code时 gas 不足,则会出现 ErrCodeStoreOutOfGas
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
// 返回 字节码 code的大小, 合约地址 以及剩余的 gas
return ret, contractAddr, contract.Gas, err
}
```
* Create中首先会对检查一些条件: 如合约创建者是否有足够的以太币转入到合约地址中,虚拟机中的调用/创建是否超过堆栈的最大深度.

* 创建一个合约地址,并在 StateDB中创建一个合约账户,并向合约中转入一定数量的以太币

* contract 实例的创建以及将字节码 code 赋值给 contractAddr 

*  Run code, 检查 code 中的 operation是否合法,以及计算和分配code所需要的 memory

* check code 的size , 确保其值小于MaxCodeSize


###  3.2 合约调用

msg.To() == nil 为 false, 则为合约调用. Call方法的入参 input 即为调用合约的参数.

```
// Increment the nonce for the next transaction
st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
```

```
// 调用以给定的input作为参数执行与addr相关联的合约,它还处理所需的任何值传递，并采取必要的步骤来创建帐户
// 在执行错误或失败的值传递的情况下会revert
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
if evm.vmConfig.NoRecursion && evm.depth > 0 {
return nil, gas, nil
}

// 如果我们试图执行高于调用深度限制，则失败
if evm.depth > int(params.CallCreateDepth) {
return nil, gas, ErrDepth
}
// 检查账户是否有足够的余额
if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
return nil, gas, ErrInsufficientBalance
}

var (
to       = AccountRef(addr)
snapshot = evm.StateDB.Snapshot()
)
if !evm.StateDB.Exist(addr) {
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
evm.StateDB.CreateAccount(addr)
}
evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)

// 初始化一个新的合约并设置EVM要使用的code
// 该合约仅是此执行上下文的范围环境
contract := NewContract(caller, to, value, gas)
// 从 stateDB 中取出 addr 所对应的 codeHash 和 code,将其设置给 contract
contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

start := time.Now()

// Capture the tracer start/end events in debug mode
if evm.vmConfig.Debug && evm.depth == 0 {
evm.vmConfig.Tracer.CaptureStart(caller.Address(), addr, false, input, gas, value)

defer func() { // Lazy evaluation of the parameters
evm.vmConfig.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
}()
}
// 合约的调用, ret即为合约调用的返回值
ret, err = run(evm, contract, input)

// When an error was returned by the EVM or when setting the creation code
// above we revert to the snapshot and consume any gas remaining. Additionally
// when we're in homestead this also counts for code storage gas errors.
// 
if err != nil {
evm.StateDB.RevertToSnapshot(snapshot)
if err != errExecutionReverted {
contract.UseGas(contract.Gas)
}
}
return ret, contract.Gas, err
}
```

Call会把对应的Code读出来，依次解析，Code中会把所有的public签名的函数标志（4字节）push到栈里。然后依据 input 中需要调用函数的签名标志（前4字节）来匹配 Code， 匹配之后跳转到对应的 opcode 。


## Run

合约创建和调用都需要执行一个 Run　函数。合约创建时　input == nil,而合约调用时 input != nil。

```
// 检查和计算 contract code 中每个 operation 的合法性以及所需消耗的内存
// 除了 errExecutionReverted, 其余所有由 interpreter 返回的错误都会 revert-and-consume-all-gas
func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
if in.intPool == nil {
in.intPool = poolOfIntPools.get()
defer func() {
poolOfIntPools.put(in.intPool)
in.intPool = nil
}()
}

// Increment the call depth which is restricted to 1024
in.evm.depth++
defer func() { in.evm.depth-- }()

// Reset the previous call's return data. It's unimportant to preserve the old buffer
// as every returning call will return new data anyway.
in.returnData = nil

// Don't bother with the execution if there's no code.
if len(contract.Code) == 0 {
return nil, nil
}

var (
op    OpCode        // current opcode
mem   = NewMemory() // bound memory
stack = newstack()  // local stack
// For optimisation reason we're using uint64 as the program counter.
// It's theoretically possible to go above 2^64. The YP defines the PC
// to be uint256. Practically much less so feasible.
pc   = uint64(0) // program counter
cost uint64
// copies used by tracer
pcCopy  uint64 // needed for the deferred Tracer
gasCopy uint64 // for Tracer to log gas remaining before execution
logged  bool   // deferred Tracer should ignore already logged steps
)
contract.Input = input

// Reclaim the stack as an int pool when the execution stops
defer func() { in.intPool.put(stack.data...) }()

if in.cfg.Debug {
defer func() {
if err != nil {
if !logged {
in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
} else {
in.cfg.Tracer.CaptureFault(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
}
}
}()
}

// Interpreter 的主循环,该循环遇到 STOP、RETURN、SELFDESTRUCT 或者发送错误时才会退出
for atomic.LoadInt32(&in.evm.abort) == 0 {
if in.cfg.Debug {
// Capture pre-execution values for tracing.
logged, pcCopy, gasCopy = false, pc, contract.Gas
}

// Get the operation from the jump table and validate the stack to ensure there are
// enough stack items available to perform the operation.
op = contract.GetOp(pc)
operation := in.cfg.JumpTable[op]
// code 中操作符是否合法
if !operation.valid {
return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
}
// 验证operation的堆栈（大小）
if err := operation.validateStack(stack); err != nil {
return nil, err
}
// 如果 operation 是有效的，对其进行写限制
if err := in.enforceRestrictions(op, operation, stack); err != nil {
return nil, err
}

var memorySize uint64
// calculate the new memory size and expand the memory to fit
// the operation
if operation.memorySize != nil {
memSize, overflow := bigUint64(operation.memorySize(stack))
if overflow {
return nil, errGasUintOverflow
}
// memory is expanded in words of 32 bytes. Gas
// is also calculated in words.
if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
return nil, errGasUintOverflow
}
}
// consume the gas and return an error if not enough gas is available.
// cost is explicitly set so that the capture state defer method can get the proper cost
cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
if err != nil || !contract.UseGas(cost) {
return nil, ErrOutOfGas
}
if memorySize > 0 {
mem.Resize(memorySize)
}

if in.cfg.Debug {
in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
logged = true
}

// execute the operation
res, err := operation.execute(&pc, in.evm, contract, mem, stack)
// verifyPool is a build flag. Pool verification makes sure the integrity
// of the integer pool by comparing values to a default value.
if verifyPool {
verifyIntegerPool(in.intPool)
}
// if the operation clears the return data (e.g. it has returning data)
// set the last return to the result of the operation.
if operation.returns {
in.returnData = res
}

switch {
case err != nil:
return nil, err
case operation.reverts:
return res, errExecutionReverted
case operation.halts:
return res, nil
case !operation.jumps:
pc++
}
}
return nil, nil
}
```

对于run函数，其正常执行的情况下,只有遇到STOP、RETURN、SELFDESTRUCT这些operation时才会退出主循环（这里走的是　operation.halts　这个 case），所以正常执行情况下返回的是　res, nil。



##  Gas消耗

在执行 tx 时，会用 IntrinsicGas 函数计算出固定的 gas 消耗， IntrinsicGas源码如下：

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

系统会根据 tx 使用的 gas, 返还一些 gas 给交易创建者。refundGas 函数源码如下：
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

























