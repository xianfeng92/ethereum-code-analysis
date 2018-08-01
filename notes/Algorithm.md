
## hashimotoFull

hashimotoFull函数的入参为：巨大的辅助数组 dataset、区块头hash（不包括nonce）、nonce，该函数主要定义了查找表 lookup ，并调用 hashimoto 函数

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











