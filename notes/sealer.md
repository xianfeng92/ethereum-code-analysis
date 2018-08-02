## Sealer

* consensus/etash/[sealer](https://github.com/xianfeng92/go-ethereum/blob/master/consensus/ethash/sealer.go)

sealer 有两个函数组成，Seal 和 mine。 

## Seal

Seal()函数实现中，会以多线程(goroutine)的方式并行调用mine()函数，线程个数等于Ethash.threads；如果Ethash.threads被设为0，则Ethash选择以本地CPU中的总核数作为开启线程的个数。

```
// Seal implements consensus.Engine, attempting to find a nonce that satisfies
// the block's difficulty requirements.
// Seal 实现了 consensus.Engine 接口， 用于寻找一个满足工作量证明的 nonce 值
func (ethash *Ethash) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
	// If we're running a fake PoW, simply return a 0 nonce immediately
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		header := block.Header()
		header.Nonce, header.MixDigest = types.BlockNonce{}, common.Hash{}
		return block.WithSeal(header), nil
	}
	// If we're running a shared PoW, delegate sealing to it
	if ethash.shared != nil {
		return ethash.shared.Seal(chain, block, stop)
	}
	// Create a runner and the multiple search threads it directs
	abort := make(chan struct{})
	found := make(chan *types.Block)

	ethash.lock.Lock()
	threads := ethash.threads // 挖矿时启用的线程数
	if ethash.rand == nil { // 随机源
		seed, err := crand.Int(crand.Reader, big.NewInt(math.MaxInt64)) // seed 的产生,其取值范围为[0,big.NewInt(math.MaxInt64))
		if err != nil {
			ethash.lock.Unlock()
			return nil, err
		}
		// rand.NewSource(seed.Int64()) 通过 seed 产生一个新的随机源
		// rand.New() 使用随机源中的随机数产生一个新的随机函数
		ethash.rand = rand.New(rand.NewSource(seed.Int64())) // 到这里 ethash.rand 可以说足够的随机了
	}
	ethash.lock.Unlock()
	if threads == 0 { // threads 为0是，系统会自动设置其为本地计算机的 CPU 数
		threads = runtime.NumCPU()
	}
	if threads < 0 {
		threads = 0 // Allows disabling local mining without extra logic around local/remote
	}
	var pend sync.WaitGroup
	for i := 0; i < threads; i++ {
		pend.Add(1)
		go func(id int, nonce uint64) {
			defer pend.Done()
			ethash.mine(block, id, nonce, abort, found) // 每个 CPU 开启 mine, 如果有一个成功 found a block, 所有其他的CPU就会停止 mine

		}(i, uint64(ethash.rand.Int63()))
	}
	// Wait until sealing is terminated or a nonce is found
	var result *types.Block
	select {
	case <-stop:
		// Outside abort, stop all miner threads
		close(abort)
	case result = <-found: // 这里的 result 即为 Seal Block
		// One of the threads found a block, abort all others
		close(abort)
	case <-ethash.update:
		// Thread count was changed on user request, restart
		close(abort)
		pend.Wait()
		return ethash.Seal(chain, block, stop)
	}
	// Wait for all miners to terminate and return the block
	// 等待所有矿工终止并返回该区块
	pend.Wait()
	return result, nil
}

```

## mine

入参@id是线程编号，用来发送log告知上层；函数内部首先定义一组局部变量，包括之后调用hashimotoFull()时传入的hash、nonce、巨大的辅助数组dataset，以及结果比较的target；然后是一个无限循环，每次调用 hashimotoFull()进行一系列复杂运算，一旦它的返回值符合条件，就复制Header对象(深度拷贝)，并赋值Nonce、MixDigest属性，返回经过授权的区块。注意到在每次循环运算时，nonce还会自增+1，使得每次循环中的计算都各不相同。

```
// mine is the actual proof-of-work miner that searches for a nonce starting from
// seed that results in correct final block difficulty.
func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
	// Extract some data from the header
	var (
		header  = block.Header() // 区块头
		hash    = header.HashNoNonce().Bytes() // 区块头的hash（不包括 nonce）
		target  = new(big.Int).Div(maxUint256, header.Difficulty) // 需要计算寻找的目标值：target = maxUint256 /  header.Difficulty
		number  = header.Number.Uint64() // 区块号
		dataset = ethash.dataset(number) // 获取或生产指定区块的挖矿数据集（dataset）
	)
	// Start generating random nonces until we abort or find a good one
	var (
		attempts = int64(0) // 记录一共进行多少次尝试
		nonce    = seed
	)
	logger := log.New("miner", id)
	logger.Trace("Started ethash search for new nonces", "seed", seed)
search:
	for { // 开始寻找 nonce 值
		select {
		case <-abort:
			// Mining terminated, update stats and abort
			logger.Trace("Ethash nonce search aborted", "attempts", nonce-seed)
			ethash.hashrate.Mark(attempts)
			break search

		default:
			// We don't have to update hash rate on every nonce, so update after after 2^X nonces
			attempts++
			if (attempts % (1 << 15)) == 0 {
				ethash.hashrate.Mark(attempts)
				attempts = 0
			}
			// Compute the PoW value of this nonce
			// 计算此 nonce 的PoW值
			// 入参为 挖矿数据集 dataset、区块头 hash、nonce值
			digest, result := hashimotoFull(dataset.dataset, hash, nonce)
			if new(big.Int).SetBytes(result).Cmp(target) <= 0 { // 当 result 小于等于  target 时，即挖矿成功
				// Correct nonce found, create a new header with it
				header = types.CopyHeader(header)
				header.Nonce = types.EncodeNonce(nonce)
				header.MixDigest = common.BytesToHash(digest) // 将 digest 转换成 hash值存储在 header 中，待以后Ethash.VerifySeal()可以加以验证

				// Seal and return a block (if still needed)
				select {
				case found <- block.WithSeal(header):
					logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
				case <-abort:
					logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
				}
				break search
			}
			nonce++ // 
		}
	}
	// Datasets are unmapped in a finalizer. Ensure that the dataset stays live
	// during sealing so it's not unmapped while being read.
	runtime.KeepAlive(dataset)
}
```

关于[hashimotoFull]()




































