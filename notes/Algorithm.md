## 小记

在一个　epoch 中，　dataset　和　cache 都是确定的，　pow 其实就是利用　dataset　中的数据、区块　hash　以及 guess 的　nonce 值来计算　digest 和 result。

在计算 result 过程中，会利用　dataset　中的数据以及fnv算法来将　result 值变得不可预测和不可破解，你这样乖乖一步一步的改变 nonce 值来寻找满足要求的　result 值。

result = crypto.Keccak256(append(seed, digest...)

pow 大致过程可能就是这样吧～

不得不服以太坊真是个牛逼的项目～

具体算法细节～暂时不看了，太费时了～


## hashimotoFull

hashimotoFull函数的入参为：巨大的辅助数组 dataset、区块头 hash（不包括nonce）、nonce，该函数主要定义了查找表 lookup ，并调用 hashimoto 函数来进行工作量证明计算。

```
// hashimotoFull aggregates data from the full dataset (using the full in-memory
// dataset) in order to produce our final value for a particular header hash and
// nonce.
func hashimotoFull(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	lookup := func(index uint32) []uint32 { // 定义了一份 lookup 函数
		offset := index * hashWords // hashWords == 16
		return dataset[offset : offset+hashWords] // 返回一个 64 bytes 数据集
	}
	return hashimoto(hash, nonce, uint64(len(dataset))*4, lookup)
}
```

## hashimoto

hashimoto函数的返回值是不可预知的，即运行函数前是无法确定给定参数的输出值（result[]）,所以只能一个一个nonce去尝试，毕竟如果result[]生成过程存在被破译的途径，那么必然有方法可以更快地找到符合条件的数组，通过更快的挖掘出区块，在整个以太坊系统中逐渐占据主导。所以Ethash共识算法应用了非常复杂的一系列运算，包含了多次、多种不同的哈希函数运算。

__具体算法目前还没有完全理解～后续再分析～__


又回到这里，继续阅读吧～

hashimoto　中入参　hash、nonce、size　都很好确定，关键在 lookup, 就从 [lookup] 开搞吧～ 　

```
// hashimoto 聚合来自完整数据集的数据，以便为特定的 hash 和 nonce 生成最终值(pow值)
// 关于参数 [size](https://github.com/xianfeng92/go-ethereum/blob/45bd4feddeadfbde5d1e560797155aacb0abbadf/consensus/ethash/algorithm.go#L405)
// 其返回值为两个长度均为32的byte数组 - digest[]和result[]
func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
	// Calculate the number of theoretical rows (we use one buffer nonetheless)
	rows := uint32(size / mixBytes) // mixBytes 128

	// Combine header+nonce into a 64 byte seed
	seed := make([]byte, 40) // 定义一个长度为40的字节数组
	copy(seed, hash) // 将hash的前40个字节拷贝给seed
	binary.LittleEndian.PutUint64(seed[32:], nonce)

	seed = crypto.Keccak512(seed) // 此时 seed 为 [64]byte
	seedHead := binary.LittleEndian.Uint32(seed) // 将 seed 从 []byte 转换为 []uint32 类型

	// Start the mix with replicated seed
	mix := make([]uint32, mixBytes/4) // mix 为长度为 32 []uint32
	for i := 0; i < len(mix); i++ {
		mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
	}
	// Mix in random dataset nodes
	temp := make([]uint32, len(mix))
        // 循环的从 dataset 中取值混入到 mix 中
	for i := 0; i < loopAccesses; i++ { //  loopAccesses 64
		parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows
		for j := uint32(0); j < mixBytes/hashBytes; j++ { // hashBytes 64  mixBytes 128
			copy(temp[j*hashWords:], lookup(2*parent+j)) // hashWords 16
		}
		fnvHash(mix, temp)
	}
	// Compress mix
	for i := 0; i < len(mix); i += 4 {
		mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
	}
	mix = mix[:len(mix)/4]

	digest := make([]byte, common.HashLength)
	for i, val := range mix {
		binary.LittleEndian.PutUint32(digest[i*4:], val)
	}
	return digest, crypto.Keccak256(append(seed, digest...))
}

```

## hashimotoLight

hashimotoLight 中并不需要整个　dataset　数据集，只需要一个　cache 就可以了

重点关注这里的　lookup 函数，这里的　rawData　其实就是　dataset 中所使用到的数据集合，所以说在　hashimoto(hash, nonce, size, lookup)　就可以轻易验证　nonce　值是否满足　pow 的要求


```
// hashimotoLight aggregates data from the full dataset (using only a small
// in-memory cache) in order to produce our final value for a particular header
// hash and nonce.
func hashimotoLight(size uint64, cache []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	keccak512 := makeHasher(sha3.NewKeccak512())

	lookup := func(index uint32) []uint32 { // 自己去生成指定　index　的　data，其实也就是指定 index 的　dataset　中的数据集合
		rawData := generateDatasetItem(cache, index, keccak512)

		data := make([]uint32, len(rawData)/4)
		for i := 0; i < len(data); i++ {
			data[i] = binary.LittleEndian.Uint32(rawData[i*4:])
		}
		return data
	} 
	return hashimoto(hash, nonce, size, lookup)
}

```

## generateDatasetItem

```
// generateDatasetItem 组合来自256个伪随机选择的缓存节点的数据，以及用于计算单个数据集节点的哈希值。
func generateDatasetItem(cache []uint32, index uint32, keccak512 hasher) []byte {
	// Calculate the number of theoretical rows (we use one buffer nonetheless)
	rows := uint32(len(cache) / hashWords)

	// Initialize the mix
	mix := make([]byte, hashBytes)

	binary.LittleEndian.PutUint32(mix, cache[(index%rows)*hashWords]^index)
	for i := 1; i < hashWords; i++ {
		binary.LittleEndian.PutUint32(mix[i*4:], cache[(index%rows)*hashWords+uint32(i)])
	}
	keccak512(mix, mix)

	// Convert the mix to uint32s to avoid constant bit shifting
	intMix := make([]uint32, hashWords)
	for i := 0; i < len(intMix); i++ {
		intMix[i] = binary.LittleEndian.Uint32(mix[i*4:])
	}
	// fnv it with a lot of random cache nodes based on index
	for i := uint32(0); i < datasetParents; i++ {
		parent := fnv(index^i, intMix[i%16]) % rows
		fnvHash(intMix, cache[parent*hashWords:])
	}
	// Flatten the uint32 mix into a binary one and return
	for i, val := range intMix {
		binary.LittleEndian.PutUint32(mix[i*4:], val)
	}
	keccak512(mix, mix)
	return mix
}

```

## Lookup

lookup 主要是根据其参数　index 来从　dataset　中取数据，那么　dataset　的数据如何产生的呢？

来看看　[dataset](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/notes/Ethash.md)


```
	lookup := func(index uint32) []uint32 { // 定义了一个 lookup 函数
		offset := index * hashWords // hashWords == 16
		return dataset[offset : offset+hashWords] // 返回一个 64 bytes 数据集，直接从　dataset　切片获取
	}
```


## generateCache

```
// generateCache 为输入 seed 创建给定大小的验证缓存
// 缓存生成过程包括首先按顺序填充32 MB　memory，然后执行　then performing two passes of Sergio Demian Lerner's RandMemoHash
// algorithm from Strict Memory Hard Hashing Functions (2014) 输出是a一组524288 64-byte 值
// 此方法将结果以机器字节顺序放入dest
func generateCache(dest []uint32, epoch uint64, seed []byte) {
	// Print some debug logs to allow analysis on low end devices
	logger := log.New("epoch", epoch)

	start := time.Now()
	defer func() {
		elapsed := time.Since(start)

		logFn := logger.Debug
		if elapsed > 3*time.Second {
			logFn = logger.Info
		}
		logFn("Generated ethash verification cache", "elapsed", common.PrettyDuration(elapsed))
	}()
	// Convert our destination slice to a byte buffer
	header := *(*reflect.SliceHeader)(unsafe.Pointer(&dest))
	header.Len *= 4
	header.Cap *= 4
	cache := *(*[]byte)(unsafe.Pointer(&header))

	// Calculate the number of theoretical rows (we'll store in one buffer nonetheless)
	size := uint64(len(cache))
	rows := int(size) / hashBytes

	// Start a monitoring goroutine to report progress on low end devices
	var progress uint32

	done := make(chan struct{})
	defer close(done)

	go func() {
		for {
			select {
			case <-done:
				return
			case <-time.After(3 * time.Second):
				logger.Info("Generating ethash verification cache", "percentage", atomic.LoadUint32(&progress)*100/uint32(rows)/4, "elapsed", common.PrettyDuration(time.Since(start)))
			}
		}
	}()
	// Create a hasher to reuse between invocations
	keccak512 := makeHasher(sha3.NewKeccak512())

	// Sequentially produce the initial dataset
	keccak512(cache, seed)
	for offset := uint64(hashBytes); offset < size; offset += hashBytes {
		keccak512(cache[offset:], cache[offset-hashBytes:offset])
		atomic.AddUint32(&progress, 1)
	}
	// Use a low-round version of randmemohash
	temp := make([]byte, hashBytes)

	for i := 0; i < cacheRounds; i++ {
		for j := 0; j < rows; j++ {
			var (
				srcOff = ((j - 1 + rows) % rows) * hashBytes
				dstOff = j * hashBytes
				xorOff = (binary.LittleEndian.Uint32(cache[dstOff:]) % uint32(rows)) * hashBytes
			)
			bitutil.XORBytes(temp, cache[srcOff:srcOff+hashBytes], cache[xorOff:xorOff+hashBytes])
			keccak512(cache[dstOff:], temp)

			atomic.AddUint32(&progress, 1)
		}
	}
	// Swap the byte order on big endian systems and return
	if !isLittleEndian() {
		swap(cache)
	}
}
```



## generateDataset


重点关注　copy(dataset[index*hashBytes:], item) 每次使用新生成的 item　填充 64 个dataset中的元素

而　item := generateDatasetItem(cache, index, keccak512)，故当　cache　和　index 确定时，就可以从　dataset　中取出确定的元素

这里的　cache　为　generateDataset的入参

```
// generateDataset生成用于 mining 的整个ethash数据集。
// 此方法将结果以机器字节顺序放入dest。
func generateDataset(dest []uint32, epoch uint64, cache []uint32) {
	// Print some debug logs to allow analysis on low end devices
	logger := log.New("epoch", epoch)

	start := time.Now()
	defer func() {
		elapsed := time.Since(start)

		logFn := logger.Debug
		if elapsed > 3*time.Second {
			logFn = logger.Info
		}
		logFn("Generated ethash verification cache", "elapsed", common.PrettyDuration(elapsed))
	}()

	// 弄清楚是否需要为机器交换字节
	swapped := !isLittleEndian()

	// Convert our destination slice to a byte buffer
	// 将目标切片转换为字节缓冲区
	header := *(*reflect.SliceHeader)(unsafe.Pointer(&dest))
	header.Len *= 4
	header.Cap *= 4
	dataset := *(*[]byte)(unsafe.Pointer(&header))

	// Generate the dataset on many goroutines since it takes a while
	// 利用多个　goroutine　去生成数据集
	threads := runtime.NumCPU()
	size := uint64(len(dataset))

	var pend sync.WaitGroup
	pend.Add(threads)

	var progress uint32
	for i := 0; i < threads; i++ {
		go func(id int) {
			defer pend.Done()

			// Create a hasher to reuse between invocations
			keccak512 := makeHasher(sha3.NewKeccak512())

			// Calculate the data segment this thread should generate
			batch := uint32((size + hashBytes*uint64(threads) - 1) / (hashBytes * uint64(threads)))
			first := uint32(id) * batch
			limit := first + batch
			if limit > uint32(size/hashBytes) {
				limit = uint32(size / hashBytes)
			}
			// Calculate the dataset segment
			percent := uint32(size / hashBytes / 100)
			for index := first; index < limit; index++ {
				item := generateDatasetItem(cache, index, keccak512)
				if swapped {
					swap(item)
				}
				copy(dataset[index*hashBytes:], item) //每次使用新生成的 item　填充 64 个dataset中的元素　

				if status := atomic.AddUint32(&progress, 1); status%percent == 0 {
					logger.Info("Generating DAG in progress", "percentage", uint64(status*100)/(size/hashBytes), "elapsed", common.PrettyDuration(time.Since(start)))
				}
			}
		}(i)
	}
	// Wait for all the generators to finish and return
	pend.Wait()
}

```








