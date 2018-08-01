
## consensus

共识算法族对外暴露的是Engine接口，其有两种实现体，分别是基于运算能力的Ethash算法和基于“同行”认证的的Clique算法。在Engine接口的声明函数中，VerifyHeader()，VerifyHeaders()，VerifyUncles()用来验证区块相应数据成员是否合理合规，可否放入区块；Prepare()函数往往在Header创建时调用，用来对Header.Difficulty等属性赋值；__Finalize()函数在区块的数据成员都已具备时被调用，比如叔区块(uncles)已经具备，全部交易Transactions已经执行完毕，全部收据(Receipt[])也已收集完毕，此时Finalize()会最终生成Root，TxHash，UncleHash，ReceiptHash等成员__。

而Seal()和VerifySeal()是Engine接口所有函数中最重要的。Seal()函数可对一个调用过Finalize()的区块进行授权或封印，并将封印过程产生的一些值赋予区块中剩余尚未赋值的成员(Header.Nonce, Header.MixDigest)。__Seal()成功时返回的区块全部成员齐整，可视为一个正常区块，可被广播到整个网络中，也可以被插入区块链等__。所以，对于挖掘一个新区块来说，所有相关代码里Engine.Seal()是其中最重要，也是最复杂的一步。VerifySeal()函数基于跟Seal()完全一样的算法原理，通过验证区块的某些属性(Header.Nonce，Header.MixDigest等)是否正确，来确定该区块是否已经经过Seal操作。

```
// 共识算法接口
type Engine interface {

	// 获取区块创建者的地址
	Author(header *types.Header) (common.Address, error)


        // 验证 header 是否符合共识引擎的规则。可以在这里或通过 VerifySeal 来验证 Seal 是否成功 
	VerifyHeader(chain ChainReader, header *types.Header, seal bool) error


	VerifyHeaders(chain ChainReader, headers []*types.Header, seals []bool) (chan<- struct{}, <-chan error)


	VerifyUncles(chain ChainReader, block *types.Block) error

	// 验证 Seal 是否成功
	VerifySeal(chain ChainReader, header *types.Header) error


        // 依据特定的共识引擎的规则对区块头的相关属性赋值， 如 Header.Difficulty
	Prepare(chain ChainReader, header *types.Header) error

        // 打包一个区块
	Finalize(chain ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction,
		uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error)

        // 对打包好的区块做pow证明，即寻找一个合适的 nonce 值
	Seal(chain ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error)

	// 难度调整算法，它返回一个新块应该具有的难度。
	CalcDifficulty(chain ChainReader, time uint64, parent *types.Header) *big.Int

	// APIs returns the RPC APIs this consensus engine provides.
	APIs(chain ChainReader) []rpc.API
}

```

------------------------------


* etash/[consensus](https://github.com/xianfeng92/go-ethereum/blob/master/consensus/ethash/consensus.go)

## VerifyHeader

VerifyHeader(chain consensus.ChainReader, header *types.Header, seal bool) 先会对 Header 的完整性进行检查，然后调用 ethash.verifyHeader(chain, header, parent, false, seal) 对其相关的数据做验证

```
// 对区块头的验证
func (ethash *Ethash) VerifyHeader(chain consensus.ChainReader, header *types.Header, seal bool) error {
	// If we're running a full engine faking, accept any input as valid
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// Short circuit if the header is known, or it's parent not
	number := header.Number.Uint64()
	if chain.GetHeader(header.Hash(), number) != nil { // 检查该 Header 是否已经存储在本地数据库中
		return nil
	}
	parent := chain.GetHeader(header.ParentHash, number-1)
	if parent == nil { // 确保该 header 存在父区块
		return consensus.ErrUnknownAncestor
	}
	// 通过完整性检查，进行适当的验证
	return ethash.verifyHeader(chain, header, parent, false, seal)
}
```

在 verifyHeader(chain consensus.ChainReader, header, parent *types.Header, uncle bool, seal bool) 中会对 header 的 Extra、Difficulty、Time、gaslimint 属性做相关检查，确保其符合要求

```
func (ethash *Ethash) verifyHeader(chain consensus.ChainReader, header, parent *types.Header, uncle bool, seal bool) error {
	// Ensure that the header's extra-data section is of a reasonable size
	if uint64(len(header.Extra)) > params.MaximumExtraDataSize {
		return fmt.Errorf("extra-data too long: %d > %d", len(header.Extra), params.MaximumExtraDataSize)
	}
	// Verify the header's timestamp
	if uncle {
		if header.Time.Cmp(math.MaxBig256) > 0 {
			return errLargeBlockTime
		}
	} else {
		if header.Time.Cmp(big.NewInt(time.Now().Add(allowedFutureBlockTime).Unix())) > 0 {
			return consensus.ErrFutureBlock
		}
	}
	if header.Time.Cmp(parent.Time) <= 0 {
		return errZeroBlockTime
	}
	// Verify the block's difficulty based in it's timestamp and parent's difficulty
	expected := ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)

	if expected.Cmp(header.Difficulty) != 0 {
		return fmt.Errorf("invalid difficulty: have %v, want %v", header.Difficulty, expected)
	}
	// Verify that the gas limit is <= 2^63-1
	cap := uint64(0x7fffffffffffffff)
	if header.GasLimit > cap {
		return fmt.Errorf("invalid gasLimit: have %v, max %v", header.GasLimit, cap)
	}
	// Verify that the gasUsed is <= gasLimit
	if header.GasUsed > header.GasLimit {
		return fmt.Errorf("invalid gasUsed: have %d, gasLimit %d", header.GasUsed, header.GasLimit)
	}

	// Verify that the gas limit remains within allowed bounds
	diff := int64(parent.GasLimit) - int64(header.GasLimit)
	if diff < 0 {
		diff *= -1
	}
	limit := parent.GasLimit / params.GasLimitBoundDivisor

	if uint64(diff) >= limit || header.GasLimit < params.MinGasLimit {
		return fmt.Errorf("invalid gas limit: have %d, want %d += %d", header.GasLimit, parent.GasLimit, limit)
	}
	// Verify that the block number is parent's +1
	if diff := new(big.Int).Sub(header.Number, parent.Number); diff.Cmp(big.NewInt(1)) != 0 {
		return consensus.ErrInvalidNumber
	}
	// Verify the engine specific seal securing the block
	if seal {
		if err := ethash.VerifySeal(chain, header); err != nil {
			return err
		}
	}
	// If all checks passed, validate any special fields for hard forks
	if err := misc.VerifyDAOHeaderExtraData(chain.Config(), header); err != nil {
		return err
	}
	if err := misc.VerifyForkHashes(chain.Config(), header, uncle); err != nil {
		return err
	}
	return nil
}

```

##

Prepare 函数主要为 header 的 Difficulty 熟悉赋值，为后面的区块的Seal做准备。

```
func (ethash *Ethash) Prepare(chain consensus.ChainReader, header *types.Header) error {
	parent := chain.GetHeader(header.ParentHash, header.Number.Uint64()-1)
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	header.Difficulty = ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)
	return nil
}

```


## CalcDifficulty

依据当前所要创建区块的　header　的时间戳和其父区块计算当前所要创建区块的 difficulty 值。

```
func (ethash *Ethash) CalcDifficulty(chain consensus.ChainReader, time uint64, parent *types.Header) *big.Int {
	return CalcDifficulty(chain.Config(), time, parent)
}
```

依据　header 所在区块的高度不同，使用不同算法来计算　header　的　difficulty　值。

```
func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
	next := new(big.Int).Add(parent.Number, big1) // 计算　header 所在的区块高度
	switch {
	case config.IsByzantium(next):
		return calcDifficultyByzantium(time, parent)
	case config.IsHomestead(next):
		return calcDifficultyHomestead(time, parent)
	default:
		return calcDifficultyFrontier(time, parent)
	}
}
```

## VerifySeal

VerifySeal　检查 Seal 出来的　header　是否满足　pow 的要求，具体是通过重新计算　digest　和　pow 值来验证　header

为什么可以重新计算？

```
// 检查给定的块是否满足PoW难度要求
func (ethash *Ethash) VerifySeal(chain consensus.ChainReader, header *types.Header) error {
	// If we're running a fake PoW, accept any seal as valid
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		time.Sleep(ethash.fakeDelay)
		if ethash.fakeFail == header.Number.Uint64() {
			return errInvalidPoW
		}
		return nil
	}
	// If we're running a shared PoW, delegate verification to it
	if ethash.shared != nil {
		return ethash.shared.VerifySeal(chain, header)
	}
	// Ensure that we have a valid difficulty for the block
	if header.Difficulty.Sign() <= 0 {
		return errInvalidDifficulty
	}
	// Recompute the digest and PoW value and verify against the header
	number := header.Number.Uint64()

	cache := ethash.cache(number)
	size := datasetSize(number)
	if ethash.config.PowMode == ModeTest {
		size = 32 * 1024
	}
	// check Nonce 值是否满足 pow 的要求
	digest, result := hashimotoLight(size, cache.cache, header.HashNoNonce().Bytes(), header.Nonce.Uint64())
	// Caches are unmapped in a finalizer. Ensure that the cache stays live
	// until after the call to hashimotoLight so it's not unmapped while being used.
	runtime.KeepAlive(cache)

	if !bytes.Equal(header.MixDigest[:], digest) {
		return errInvalidMixDigest
	}
	target := new(big.Int).Div(maxUint256, header.Difficulty)
	if new(big.Int).SetBytes(result).Cmp(target) > 0 {
		return errInvalidPoW
	}
	return nil
}

```









































