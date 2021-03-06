## interpreter

解释器 [interpreter](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/interpreter.go)用来执行(非预编译的)合约指令。

---------------------------------------------------

Interpreter结构体通过一个Config类型的成员变量，间接持有一个包括256个operation对象在内的数组JumpTable。

```
// Interpreter 的结构
type Interpreter struct {
	evm      *EVM
	cfg      Config // Interpreter 结构中持有一个 Config
	gasTable params.GasTable
	intPool  *intPool

	readOnly   bool   // Whether to throw on stateful modifications
	returnData []byte // Last CALL's return data for subsequent reuse
}

// Config 是 Interpreter 的配置选项
type Config struct {
	// Debug enabled debugging Interpreter options（启用调试解释器选项）
	Debug bool
	// Tracer is the op code logger
	Tracer Tracer
	// NoRecursion disabled Interpreter call, callcode,
	// delegate call and create.
	NoRecursion bool
	// Enable recording of SHA3/keccak preimages
	EnablePreimageRecording bool
	// JumpTable contains the EVM instruction table. This
	// may be left uninitialised and will be set to the default
	// table.
        // JumpTable 包含EVM指令表。这可能是未初始化的，并且将被设置为默认表。
	JumpTable [256]operation // 包括256个operation对象在内的数组JumpTable
}

```

__每个 operation 对象正对应一个已定义的虚拟机指令__ ，它所含有的四个函数变量 execute, gasCost, validateStack, memorySize 提供了这个虚拟机指令所代表的所有操作。每个指令长度1byte， __Contract对象的成员变量Code类型为 []byte，就是这些虚拟机指令的任意集合__ 。operation对象的函数操作，主要会用到Stack，Memory, IntPool 这几个自定义的数据结构。


operation在操作过程中，会需要几个数据结构： Stack，实现了标准容器 -栈的行为；Memory，一个字节数组，可表示线性排列的任意数据;还有一个intPool，提供对big.Int数据的存储和读取。

已定义的operation，种类很丰富，包括：

算术运算：ADD，MUL，SUB，DIV，SDIV，MOD，SMOD，EXP...；
逻辑运算：LT，GT，EQ，ISZERO，AND，XOR，OR，NOT...；
业务功能：SHA3，ADDRESS，BALANCE，ORIGIN，CALLER，GASPRICE，LOG1，LOG2...等等

LOGn指令操作，它用来创建n个Log对象，这里n最大是4。还记得Log在何时被用到么？每个交易(Transaction,tx)执行完成后，会创建一个Receipt对象用来记录这个交易的执行结果。Receipt携带一个Log数组，用来记录tx操作过程中的所有变动细节，而这些Log，正是通过合适的LOGn指令-即合约指令数组(Contract.Code)中的单个byte，在其对应的operation里被创建出来的。每个新创建的Log对象被缓存在StateDB中的相对应的stateObject里，待需要时从StateDB中读取。


### Run函数

```
// Run loops and evaluates the contract's code with the given input data and returns
// the return byte-slice and an error if one occurred.
//
// It's important to note that any errors returned by the interpreter should be
// considered a revert-and-consume-all-gas operation except for
// errExecutionReverted which means revert-and-keep-gas-left.
func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
	//  depth 深度不能超过 1024
	in.evm.depth++
	defer func() { in.evm.depth-- }()

	// 重置前一个调用的返回数据。保留旧的缓冲区并不重要，因为每次返回调用都会返回新的数据。
	in.returnData = nil

	// 如果没有代码，就不要为执行而烦恼。
	if len(contract.Code) == 0 {
		return nil, nil
	}

	var (
		op    OpCode        // 当前操作码
		mem   = NewMemory() // 绑定内存
		stack = newstack()  // 本地栈
		// 为了优化，我们使用 UIT64 作为程序计数器。
		// It's theoretically possible to go above 2^64. The YP defines the PC
		// to be uint256. Practically much less so feasible.
		pc   = uint64(0) // program counter
		cost uint64
		// tracer 使用的一些 copies
		pcCopy  uint64 // needed for the deferred Tracer
		gasCopy uint64 // for Tracer to log gas remaining before execution
		logged  bool   // deferred Tracer should ignore already logged steps
	)
	contract.Input = input // 获取 tx 的输入

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
	// The Interpreter main run loop (contextual). This loop runs until either an
	// explicit STOP, RETURN or SELFDESTRUCT is executed, an error occurred during
	// the execution of one of the operations or until the done flag is set by the
	// parent context.
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		if in.cfg.Debug {
			// Capture pre-execution values for tracing.
			logged, pcCopy, gasCopy = false, pc, contract.Gas
		}

		// Get the operation from the jump table and validate the stack to ensure there are
		op = contract.GetOp(pc) // GetOp returns the n'th element in the contract's byte array 获取合约字节码中第n个元素
		operation := in.cfg.JumpTable[op] // JumpTable 中查找 op 所对应的操作符
		if !operation.valid {
			return nil, fmt.Errorf("invalid opcode 0x%x", int(op)) // 无效操作符
		}
		if err := operation.validateStack(stack); err != nil { // 判断是否有足够的堆栈项可用于执行操作
			return nil, err
		}
		// If the operation is valid, enforce and write restrictions
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
		// 如果 gas 不足，会消耗掉 gas 并返回一个错误
		// cost is explicitly set so that the capture state defer method can get the proper cost
		cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize) // 计算 operation 所需消耗的 gas
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
		res, err := operation.execute(&pc, in.evm, contract, mem, stack) // 指令的执行

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
		case operation.halts: // 退出循环
			return res, nil
		case !operation.jumps: // 继续执行 合约的字节码
			pc++
		}
	}
	return nil, nil
}

```

每个 operation ：

gasCost = 内存扩展所需的 gas + LogGas + LogTopicGas + memorySizeGas


详细代码如下：

* gasCost

```
func makeGasLog(n uint64) gasFunc {
	return func(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
		requestedSize, overflow := bigUint64(stack.Back(1))
		if overflow {
			return 0, errGasUintOverflow
		}

		gas, err := memoryGasCost(mem, memorySize) // 计算内存扩展所需的 gas，它只针对被扩展的内存区域，而不是总内存
		if err != nil {
			return 0, err
		}

		if gas, overflow = math.SafeAdd(gas, params.LogGas); overflow { // LogGas == 375 Per LOG* operation.
			return 0, errGasUintOverflow
		}
		if gas, overflow = math.SafeAdd(gas, n*params.LogTopicGas); overflow {
			// LogTopicGas == 375   其中 n 取值 0，1,2,3,4 分别对应 LOG0 LOG1 LOG2 LOG3 LOG4
                       // Multiplied by the * of the LOG*, per LOG transaction. e.g. LOG0 incurs 0 * c_txLogTopicGas, LOG4 incurs 4 * c_txLogTopicGas.
			return 0, errGasUintOverflow
		}

		var memorySizeGas uint64
		if memorySizeGas, overflow = math.SafeMul(requestedSize, params.LogDataGas); overflow { // LogDataGas == 8  Per byte in a LOG* operation's data.
			return 0, errGasUintOverflow
		}
		if gas, overflow = math.SafeAdd(gas, memorySizeGas); overflow {
			return 0, errGasUintOverflow
		}
		return gas, nil
	}
}

```


------------------------------------------------------

## 补充

[operation的gas消耗](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/jump_table.go)。

