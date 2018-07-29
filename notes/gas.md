## gas

[Gas](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/gas.go), 是Ethereum里对所有活动进行消耗资源计量的单位。这里的活动是泛化的概念，包括但不限于：转帐，合约的创建，合约指令的执行，执行中内存的扩展等等。所以Gas可以想象成现实中的汽油或者燃气。__Gas是Ethereum系统的血液。一切资源，活动，交互的开销，都以Gas为计量单元__。如果定义了一个GasPrice，那么所有的Gas消耗亦可等价于以太币Ether。

关于 gas 的计算:

```
// schema: [opcode, pop, push, gasUsed, halts, jumps, writes, valid, reverts, returns]

halts   bool // 是否会停止之后的执行
jumps   bool // pc计数器是否需要增长
writes  bool // 是否会修改state
valid   bool // 是否是合法的操作
reverts bool // 是否会revert state会隐式地停止之后的操作
returns bool // 是否会返回数据
// 没有填写的均为false
var opcodes = {
	    // arithmetic
        0x00: ["STOP", 0, 0, 0, halts: true, valid: true],    // 不对栈进行操作，停止执行代码
        0x01: ["ADD", 2, 1, 3, valid: true],    // 取出栈顶两个元素x,*y（x表示pop，*y表示返回栈顶元素指针，以下均同此）,y = x + *y 
        0x02: ["MUL", 2, 1, 5, valid: true],    // 取出栈顶两个元素x,y，push(x*y)
        0x03: ["SUB", 2, 1, 3, valid: true],    // 取出栈顶两个元素x,*y，y = x - *y
        0x04: ["DIV", 2, 1, 5, valid: true],    // 取出栈顶两个元素x，*y，y = x / *y
        0x05: ["SDIV", 2, 1, 5, valid: true],   // x, y;TODO
        0x06: ["MOD", 2, 1, 5, valid: true],    // x, y; push x % y
        0x07: ["SMOD", 2, 1, 5, valid: true],   // TODO
        0x08: ["ADDMOD", 3, 1, 8, valid: true], // x, y, z; push (x + y) % z
        0x09: ["MULMOD", 3, 1, 8, valid: true], // x, y, z; push (x * y) % z
        0x0a: ["EXP", 2, 1, gasExp, valid: true], // evm-tools中显示为10; x, y; push x ^ y
        0x0b: ["SIGNEXTEND", 2, 1, 5, valid: true], // x; TODO
    
        // boolean
        0x10: ["LT", 2, 1, 3, valid: true],  // x, *y; if x < *y; *y = 1; else *y = 0;
        0x11: ["GT", 2, 1, 3, valid: true],  // x, *y; if x > *y; *y = 1; else *y = 0;
        0x12: ["SLT", 2, 1, 3, valid: true], // x, *y;TODO
        0x13: ["SGT", 2, 1, 3, valid: true], // x, *y:TODO
        0x14: ["EQ", 2, 1, 3, valid: true],  // x, *y; if x == *y *y = 1; else *y = 0;
        0x15: ["ISZERO", 1, 1, 3, valid: true], // *x; if *x > 0 ; *x = 0; else *x = 1;
        0x16: ["AND", 2, 1, 3, valid: true], // x, y; push x and y
        0x17: ["OR", 2, 1, 3, valid: true], // x, *y; *y = x or *y
        0x18: ["XOR", 2, 1, 3, valid: true], // x, *y; *y = x xor *y
        0x19: ["NOT", 1, 1, 3, valid: true], // *x; *x = not *x
        0x1a: ["BYTE", 2, 1, 3, valid: true], // x, *y; TODO
        0x1b: ["SHL", 2, 1, 3], //
        0x1c: ["SHR", 2, 1, 3],
        0x1d: ["SAR", 2, 1, 3],
    
        // crypto
        0x20: ["SHA3", 2, 1, gasSha3, valid: true], // evm-tools中显示为30; x, y; data = memory(offset: x, size: y), push Hash(data)
        
        // contract context
        0x30: ["ADDRESS", 0, 1, 2, valid: true], // push contract.address
        0x31: ["BALANCE", 1, 1, 400, valid: true], // Homestead为20；EIP150/EIP158为400； *x; *x = balance(*x);
        0x32: ["ORIGIN", 0, 1, 2, valid: true], // push origin
        0x33: ["CALLER", 0, 1, 2, valid: true], // push caller
        0x34: ["CALLVALUE", 0, 1, 2, valid: true], // push contract.value
        0x35: ["CALLDATALOAD", 1, 1, 3, valid: true], // x; inputData[x:x+32],inputData需先转为byte数组
        0x36: ["CALLDATASIZE", 0, 1, 2, valid: true], // push input.length
        0x37: ["CALLDATACOPY", 3, 0, gasCallDataCopy, valid: true], // evm-tools中显示为3; x, y, z;memory[x:x+z] = inputData[y:y+z]
        0x38: ["CODESIZE", 0, 1, 2, valid: true], // push contract.code.length
        0x39: ["CODECOPY", 3, 0, gasCodeCopy, valid: true], // evm-tool中显示为3; x, y, z;memory[x: x+z]=code[y:y+z]
        0x3a: ["GASPRICE", 0, 1, 2, valid: true], // push gasPrice
        0x3b: ["EXTCODESIZE", 1, 1, 700, valid: true], // Homestead为20；EIP150/EIP158为700；*x; *x = codesize(*x)
        0x3c: ["EXTCODECOPY", 4, 0, gasExtCodeCopy, valid: true], // evm-tools中显示为20;x, y, z, g; memory[y:y+g] = code(x)[z:z+g]
        0x3d: ["RETURNDATASIZE", 0, 1, 2, valid: true], // push len(returnData)
        0x3f: ["RETURNDATACOPY", 3, 0, gasReturnDataCopy, valid: true],  // x, y, z; memeory[x:x+z] = returnData[y:y+z];
    
        // blockchain context
        0x40: ["BLOCKHASH", 1, 1, 20, valid: true], // x; push(hash(x));x必须在最近的257块之内
        0x41: ["COINBASE", 0, 1, 2, valid: true], // push coinbase
        0x42: ["TIMESTAMP", 0, 1, 2, valid: true], // push time
        0x43: ["NUMBER", 0, 1, 2, valid: true], // push blockNumber
        0x44: ["DIFFICULTY", 0, 1, 2, valid: true], // push difficulty
        0x45: ["GASLIMIT", 0, 1, 2, valid: true], // push gasLimit
      
        // storage and execution
        0x50: ["POP", 1, 0, 2, valid: true], // x;
        0x51: ["MLOAD", 1, 1, gasMLoad, valid: true], // evm-tools中显示为3; x; push memory[x:x+32]
        0x52: ["MSTORE", 2, 0, gasMStore, valid: true], // evm-tools中显示为3; x, y; memory[x:x+32] = y;
        0x53: ["MSTORE8", 2, 0, gasMStore8, valid: true], // evm-tools中显示为3; x, y; memory[x] = y;
        0x54: ["SLOAD", 1, 1, 200, valid: true], // Homestead为50；EIP150/EIP158为200； x; push( stateOf(hash(x)) );
        0x55: ["SSTORE", 2, 0, gasSStore, writes: true, valid: true], // evm-tools中显示为0; x, y; setState(hash(x), hash(y));
        0x56: ["JUMP", 1, 0, 8, jumps: true, valid: true], // pos; pc = pos;
        0x57: ["JUMPI", 2, 0, 10, valid: true], // pos, cond; if cond != 0 pc = pos; else pc++;
        0x58: ["PC", 0, 1, 2, valid: true], // push pc;
        0x59: ["MSIZE", 0, 1, 2, valid: true], // push memory.length
        0x5a: ["GAS", 0, 1, 2, valid: true], // push contract.gas
        0x5b: ["JUMPDEST", 0, 0, 1, valid: true], // do nothing
    
        // logging
        0xa0: ["LOG0", 2, 0, makeGasLog(0), writes: true, valid: true], // evm-tools中显示为375
        0xa1: ["LOG1", 3, 0, makeGasLog(1), writes: true, valid: true], // evm-tools中显示为750
        0xa2: ["LOG2", 4, 0, makeGasLog(2), writes: true, valid: true], // evm-tools中显示为1125
        0xa3: ["LOG3", 5, 0, makeGasLog(3), writes: true, valid: true], // evm-tools中显示为1500
        0xa4: ["LOG4", 6, 0, makeGasLog(4), writes: true, valid: true], // evm-tools中显示为1875
        // makeGasLog(size)
        // mStart, mSize; for i:=0; i<size; i++ { topics[i]=stack.pop } log.topics = topics; log.data = memory[mStart:mSize];
        
        // unofficial opcodes used for parsing
        0xb0: ["PUSH"],
        0xb1: ["DUP"],
        0xb2: ["SWAP"],
        
        // closures
        0xf0: ["CREATE", 3, 1, gasCreate32000, writes: true, valid: true, returns: true], // vlaue, offset, size; input = memory[offset:offset+size]; createContract
        0xf1: ["CALL", 7, 1, gasCall40, valid: true, returns: true], // _, addr, value, inOffset, inSize, retOffset, retSize; call;
        0xf2: ["CALLCODE", 7, 1, gasCallCode40, valid: true, returns: true], // _, addr, value, inOffset, inSize, retOffset, retSize; callCode;
        0xf3: ["RETURN", 2, 0, gasReturn0, halts: true, valid: true, returns: ], // offset, size; return memory[offset:offset+size]
        0xf4: ["DELEGATECALL", 6, 1, gasDelegateCall, valid: true, returns: true], // _, addr, inOffset, inSize, retOffset, retSize; delegateCall;
        
        0xfa: ["STATICCALL", 6, 1, gasStaticCall, valid: true, returns: true], // _, addr, inOffset, inSize, retOffset, retSize; staticCall;
        0xfd: ["REVERT", 2, 0, gasRevert ,valid: true, returns: true, reverts: true], // offset, size; return memory[offset:offset+size]
        0xff: ["SELFDESTRUCT", 1, 0, gasSuicide, halts: true, writes: true, valid: true], // x; addBalance(hash(x), contract.balance) 
    	
        // arbitrary length storage (proposal for metropolis hardfork)
        0xe1: ["SLOADBYTES", 3, 0, 50], // not use now
        0xe2: ["SSTOREBYTES", 3, 0, 0], // not use now
        0xe3: ["SSIZE", 1, 1, 50], // not use now
}

// i 代表是一个字节个数，如PUSH1代表压入1个单字节的数，PUSH2代表压入一个双字节的数，下面同此。
for i := 1; i <= 32; i++ {
    opcodes[0x60 + i - 1] = ["PUSH" + string(i), 0, 1, 3]; // push x;
}

for i := 1; i <= 16; i++ {
    opcodes[0x80 + i - 1] = ["DUP" + string(i), i, i+1, 3] // push stack[stack.len - i];
    opcodes[0x90 + i - 1] = ["SWP" + string(i), i+1, i+1, 3] // stack[stack.len - (i + 1)], stack[stack.len - 1] = stack[stack.len - 1], stack[stack.len - (i + 1)]
}
交易时，计算的gas，主要分为IntrinsicGas，执行evm的gas，最后会进行refund操作，所以交易费基本等于=(IntrinsicGas+evmGas-refundGas) * gasPrice:
IntrinsicGas:

func IntrinsicGas(data []byte, contractCreation, homestead bool) (uint64, error) {
	// Set the starting gas for the raw transaction
	var gas uint64
	// 判断是否为创建合约交易，所需的gas不一致
	if contractCreation && homestead {
		gas = params.TxGasContractCreation
	} else {
		gas = params.TxGas
	}
	// Bump the required gas by the amount of transactional data
	if len(data) > 0 {
		// Zero and non-zero bytes are priced differently
		var nz uint64
		// 计算非0的数据个数
		for _, byt := range data {
			if byt != 0 {
				nz++
			}
		}
		// Make sure we don't exceed uint64 for all data combinations
		if (math.MaxUint64-gas)/params.TxDataNonZeroGas < nz {
			return 0, vm.ErrOutOfGas
		}
		gas += nz * params.TxDataNonZeroGas

        // 计算0的数据个数
		z := uint64(len(data)) - nz
		if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
			return 0, vm.ErrOutOfGas
		}
		gas += z * params.TxDataZeroGas
	}
	// 总的来说 gas = 交易类型的gas + 0数据的gas×0数据的size + 非0数据的gas×非0数据的size
	return gas, nil
}
refundGas:

func (st *StateTransition) refundGas() {
	// refund的数量为使用gas的一半
	refund := st.gasUsed() / 2
	// 取refund和GetRefund()中较小的一个
	// GetRefund()在evm执行过程中，仅有两个操作可能会增加该部分的值：
	// 1. SstoreRefundGas： 15000，执行Sstore操作，删除一个地址
	// 2. SuicideRefundGas： 合约自毁24000
	if refund > st.state.GetRefund() {
		refund = st.state.GetRefund()
	}
	// 当前剩余gas加上refund的gas
	st.gas += refund

	// 计算需要退回的金额
	remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
	// 把这部分金额退还给发送交易的地址
	st.state.AddBalance(st.msg.From(), remaining)

	// 将剩下的gas加到全局的gas中，保证下一条交易使用的数据是对的。
	st.gp.AddGas(st.gas)
}

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

