## Ethash

### dataset

dataset 入参为　block，每30000个 block 为一个 epoch。

重点在　ethash.datasets.get　函数，来看看 ethash.datasets.get

获取到　current　以后会调用其　generate，　再来看看　generate


```
func (ethash *Ethash) dataset(block uint64) *dataset {
	epoch := block / epochLength // epochLength == 30000
	currentI, futureI := ethash.datasets.get(epoch)
	current := currentI.(*dataset)

	// Wait for generation finish.
	current.generate(ethash.config.DatasetDir, ethash.config.DatasetsOnDisk, ethash.config.PowMode == ModeTest)

	// If we need a new future dataset, now's a good time to regenerate it.
	if futureI != nil {
		future := futureI.(*dataset)
		go future.generate(ethash.config.DatasetDir, ethash.config.DatasetsOnDisk, ethash.config.PowMode == ModeTest)
	}

	return current
}
```

## ethash.datasets.get

```
// get 获取检索或创建给定 epoch 的 item。第一个返回值始终是非零。如果lru认为某个 item 在不久的将来会有用，则第二个返回值是非零的。
func (lru *lru) get(epoch uint64) (item, future interface{}) {
	lru.mu.Lock()
	defer lru.mu.Unlock()

	// Get or create the item for the requested epoch.
　　　　　　　　// 从缓存中查找 epoch 的值
	item, ok := lru.cache.Get(epoch)
	if !ok {
		if lru.future > 0 && lru.future == epoch {
			item = lru.futureItem
		} else {
			log.Trace("Requiring new ethash "+lru.what, "epoch", epoch)
			item = lru.new(epoch)
		}
		lru.cache.Add(epoch, item)
	}
	// Update the 'future item' if epoch is larger than previously seen.
	if epoch < maxEpoch-1 && lru.future < epoch+1 {
		log.Trace("Requiring new future ethash "+lru.what, "epoch", epoch+1)
		future = lru.new(epoch + 1)
		lru.future = epoch + 1
		lru.futureItem = future
	}
	return item, future
}
```


## generate

generate　主要调用　generateCache　来生成　cache，再来看看　[generateCache](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/notes/Algorithm.md)

不管　generateCache　使用多复杂多牛逼的算法（懒得去理解～费劲），我们其实只需要知道　catch 是和 epoch 对应的。一个特定的　epoch 对应一个特定的　catch。

基于这些其实很多东西都好理解了～

```
// generate 确保在使用之前生成 cache 内容
func (c *cache) generate(dir string, limit int, test bool) {
	c.once.Do(func() {
		size := cacheSize(c.epoch*epochLength + 1)
		seed := seedHash(c.epoch*epochLength + 1)
		if test {
			size = 1024
		}
		// 如果我们不在磁盘上存储任何内容，则使用　generateCache 生成并返回即可
		if dir == "" {
			c.cache = make([]uint32, size/4)
			generateCache(c.cache, c.epoch, seed)
			return
		}
		// Disk storage is needed, this will get fancy
		var endian string
		if !isLittleEndian() {
			endian = ".be"
		}
		path := filepath.Join(dir, fmt.Sprintf("cache-R%d-%x%s", algorithmRevision, seed[:8], endian))
		logger := log.New("epoch", c.epoch)

		// We're about to mmap the file, ensure that the mapping is cleaned up when the
		// cache becomes unused.
		runtime.SetFinalizer(c, (*cache).finalizer)

		// Try to load the file from disk and memory map it
		var err error
		c.dump, c.mmap, c.cache, err = memoryMap(path)
		if err == nil {
			logger.Debug("Loaded old ethash cache from disk")
			return
		}
		logger.Debug("Failed to load old ethash cache", "err", err)

		// No previous cache available, create a new cache file to fill
		c.dump, c.mmap, c.cache, err = memoryMapAndGenerate(path, size, func(buffer []uint32) { generateCache(buffer, c.epoch, seed) })
		if err != nil {
			logger.Error("Failed to generate mapped ethash cache", "err", err)

			c.cache = make([]uint32, size/4)
			generateCache(c.cache, c.epoch, seed)
		}
		// Iterate over all previous instances and delete old ones
		for ep := int(c.epoch) - limit; ep >= 0; ep-- {
			seed := seedHash(uint64(ep)*epochLength + 1)
			path := filepath.Join(dir, fmt.Sprintf("cache-R%d-%x%s", algorithmRevision, seed[:8], endian))
			os.Remove(path)
		}
	})
}

```

























