# Miner

Miner包主要实现了区块的创建以及区块的 mining,该包中主要由miner、agent以及worker组成。

![TX_consensus](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/images/minerStructure.png)

*  miner/[miner](https://github.com/xianfeng92/go-ethereum/tree/master/miner)

## New():Miner

挖掘新区块的流程入口在Miner里,具体入口在Miner结构体的创建函数.在New()里，针对新对象miner的各个成员变量初始化完成后，会紧跟着创建worker对象，然后将Agent对象登记给worker，最后用一个单独线程去运行miner.Update()函数.这个update()会订阅(监听)几种事件，均跟Downloader相关。

```
// Miner creates blocks and searches for proof-of-work values.
type Miner struct {
mux *event.TypeMux

worker *worker

coinbase common.Address
mining   int32 // 是否在 mining 中
eth      Backend
engine   consensus.Engine

canStart    int32 // 是否能进行mining
shouldStart int32 // 同步后是否应该mining
}

func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
miner := &Miner{
eth:      eth,
mux:      mux,
engine:   engine,
// 在创建 worker 时， 会调用 worker.commitNewWork()，即构建一个 block header，获取当前的最新 block 作为其 parent
// 利用 header 和 parent 生成一个 work，在 work 中记录 从 tx_pool 中获取 pending 状态的 tx 以及对应的recepits
worker: newWorker(config, engine, common.Address{}, eth, mux),
canStart: 1,// 可以开始 mining
}
miner.Register(NewCpuAgent(eth.BlockChain(), engine))
go miner.update() // 一个单独线程去运行miner.Update()函数， 当一个节点更新成最新状态时，就 start Mining

return miner
}

// 将Agent对象登记给worker
func (self *Miner) Register(agent Agent) {
	if self.Mining() {
		agent.Start()
	}
	self.worker.register(agent)
}

```

### NewWorker

在创建worker对象时，其对订阅ChainHeadEvent，ChainSideEvent，TxPreEvent 这三个事件，然后单独开启线程去执行 worker.update() 和 worker.wait()，最后调用 worker.commitNewWork()，即构建一个 block header，获取当前的最新 block 作为其 parent。利用 header 和 parent 生成一个 work，在 work 中记录 从 tx_pool 中获取 pending 状态的 tx 以及对应的recepits。

```
func newWorker(config *params.ChainConfig, engine consensus.Engine, coinbase common.Address, eth Backend, mux *event.TypeMux) *worker {
	worker := &worker{
		config:         config,
		engine:         engine,
		eth:            eth,
		mux:            mux,
		txCh:           make(chan core.TxPreEvent, txChanSize),
		chainHeadCh:    make(chan core.ChainHeadEvent, chainHeadChanSize),
		chainSideCh:    make(chan core.ChainSideEvent, chainSideChanSize),
		chainDb:        eth.ChainDb(),
		recv:           make(chan *Result, resultQueueSize),
		chain:          eth.BlockChain(),
		proc:           eth.BlockChain().Validator(),
		possibleUncles: make(map[common.Hash]*types.Block),
		coinbase:       coinbase,
		agents:         make(map[Agent]struct{}),
		unconfirmed:    newUnconfirmedBlocks(eth.BlockChain(), miningLogAtDepth),
	}
	// Subscribe TxPreEvent for tx pool
	worker.txSub = eth.TxPool().SubscribeTxPreEvent(worker.txCh)
	// Subscribe events for blockchain
	worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
	worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
	go worker.update()

	go worker.wait()
	worker.commitNewWork()

	return worker
}
```

![WORKER](https://github.com/xianfeng92/ethereum-code-analysis/blob/master/images/Worker.png)

#### worker.update()

worker.update()分别监听ChainHeadEvent，ChainSideEvent，TxPreEvent几个事件，每个事件会触发worker不同的反应。ChainHeadEvent是指区块链中已经加入了一个新的区块作为整个链的链头，这时worker的回应是立即开始准备挖掘下一个新区块(也是够忙的)；ChainSideEvent指区块链中加入了一个新区块作为当前链头的旁支，worker会把这个区块收纳进possibleUncles[]数组，作为下一个挖掘新区块可能的Uncle之一；TxPreEvent是TxPool对象发出的，指的是一个新的交易tx被加入了TxPool，这时如果worker没有处于挖掘中，那么就去执行这个tx，并把它收纳进Work.txs数组，为下次挖掘新区块备用。ChainHeadEvent并不一定是外部源发出。由于worker对象有个成员变量chain(eth.BlockChain)，所以当worker自己完成挖掘一个新区块，并把它写入数据库，加进区块链里成为新的链头时，worker自己也可以调用chain发出一个ChainHeadEvent，从而被worker.update()函数监听到，进入下一次区块挖掘。

```
func (self *worker) update() {
	defer self.txSub.Unsubscribe()
	defer self.chainHeadSub.Unsubscribe()
	defer self.chainSideSub.Unsubscribe()

	for {
		// A real event arrived, process interesting content
		select {
		// Handle ChainHeadEvent
		case <-self.chainHeadCh:
			self.commitNewWork()

		// Handle ChainSideEvent
		case ev := <-self.chainSideCh:
			self.uncleMu.Lock()
			self.possibleUncles[ev.Block.Hash()] = ev.Block
			self.uncleMu.Unlock()

		// Handle TxPreEvent
		case ev := <-self.txCh:
			// Apply transaction to the pending state if we're not mining
			if atomic.LoadInt32(&self.mining) == 0 {
				self.currentMu.Lock()
				acc, _ := types.Sender(self.current.signer, ev.Tx)
				txs := map[common.Address]types.Transactions{acc: {ev.Tx}}
				txset := types.NewTransactionsByPriceAndNonce(self.current.signer, txs)

				self.current.commitTransactions(self.mux, txset, self.chain, self.coinbase)
				self.currentMu.Unlock()
			} else {
				// If we're mining, but nothing is being processed, wake on new transactions
				if self.config.Clique != nil && self.config.Clique.Period == 0 {
					self.commitNewWork()
				}
			}

		// System stopped
		case <-self.txSub.Err():
			return
		case <-self.chainHeadSub.Err():
			return
		case <-self.chainSideSub.Err():
			return
		}
	}
}
```

#### worker.wait()

worker.wait()会在一个channel处一直等待Agent完成挖掘发送回来的新Block和Work对象。这个Block会被写入数据库，加入本地的区块链试图成为最新的链头。注意，此时区块中的所有交易，假设都已经被执行过了，所以这里的操作，不会再去执行这些交易对象。当这一切都完成，worker就会发送一条事件(NewMinedBlockEvent{})，等于通告天下：我挖出了一个新区块！这样监听到该事件的其他节点，就会根据自身的状况，来决定是否接受这个新区块成为全网中公认的区块链新的链头。

```
func (self *worker) wait() {
	for {
		mustCommitNewWork := true
		for result := range self.recv {
			atomic.AddInt32(&self.atWork, -1)

			if result == nil {
				continue
			}
			block := result.Block
			work := result.Work


			// 更新所有Log中的 block hash 值，
			for _, r := range work.receipts {
				for _, l := range r.Logs {
					l.BlockHash = block.Hash()
				}
			}
			for _, log := range work.state.Logs() {
				log.BlockHash = block.Hash()
			}
			// 将 block 添加到 chain 中
			stat, err := self.chain.WriteBlockWithState(block, work.receipts, work.state)
			if err != nil {
				log.Error("Failed writing block to chain", "err", err)
				continue
			}
			// check if canon block and write transactions
			if stat == core.CanonStatTy {
				// implicit by posting ChainHeadEvent
				mustCommitNewWork = false
			}
			// 广播 block 并宣布 chain insertion event
			self.mux.Post(core.NewMinedBlockEvent{Block: block})
			var (
				events []interface{}
				logs   = work.state.Logs()
			)
			events = append(events, core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
			if stat == core.CanonStatTy {
				events = append(events, core.ChainHeadEvent{Block: block})
			}
			self.chain.PostChainEvents(events, logs)

			// 将 block 插入到 unconfirmed， 等待进一步确认
			self.unconfirmed.Insert(block.NumberU64(), block.Hash())

			if mustCommitNewWork {
				self.commitNewWork()
			}
		}
	}
}
```

### worker.commitNewWork()

commitNewWork()会在worker内部多处被调用，注意它每次都是被直接调用，并没有以goroutine的方式启动。commitNewWork()内部使用sync.Mutex对全部操作做了隔离。

```
func (self *worker) commitNewWork() {
self.mu.Lock()
defer self.mu.Unlock()
self.uncleMu.Lock()
defer self.uncleMu.Unlock()
self.currentMu.Lock()
defer self.currentMu.Unlock()

tstart := time.Now()
parent := self.chain.CurrentBlock() // 获取当前区块链中的头区块

tstamp := tstart.Unix() // 计算时间戳
if parent.Time().Cmp(new(big.Int).SetInt64(tstamp)) >= 0 { // 确保时间戳必须大于parent
tstamp = parent.Time().Int64() + 1
}
// this will ensure we're not going off too far in the future
if now := time.Now().Unix(); tstamp > now+1 {
wait := time.Duration(tstamp-now) * time.Second
log.Info("Mining too far in the future", "wait", common.PrettyDuration(wait))
time.Sleep(wait)
}

num := parent.Number()
header := &types.Header{ // 封装区块头
ParentHash: parent.Hash(), // 父区块的hash
Number:     num.Add(num, common.Big1),// 区块的num
// CalcGasLimit computes the gas limit of the next block after parent
GasLimit:   core.CalcGasLimit(parent),
Extra:      self.extra,
Time:       big.NewInt(tstamp), // 区块的时间戳
}
// Only set the coinbase if we are mining (avoid spurious block rewards)
if atomic.LoadInt32(&self.mining) == 1 {
header.Coinbase = self.coinbase
}
if err := self.engine.Prepare(self.chain, header); err != nil {
log.Error("Failed to prepare header for mining", "err", err)
return
}
// If we are care about TheDAO hard-fork check whether to override the extra-data or not
if daoBlock := self.config.DAOForkBlock; daoBlock != nil {
// Check whether the block is among the fork extra-override range
limit := new(big.Int).Add(daoBlock, params.DAOForkExtraRange)
if header.Number.Cmp(daoBlock) >= 0 && header.Number.Cmp(limit) < 0 {
// Depending whether we support or oppose the fork, override differently
if self.config.DAOForkSupport {
header.Extra = common.CopyBytes(params.DAOForkBlockExtra)
} else if bytes.Equal(header.Extra, params.DAOForkBlockExtra) {
header.Extra = []byte{} // If miner opposes, don't let it use the reserved extra-data
}
}
}
// Could potentially happen if starting to mine in an odd state.
err := self.makeCurrent(parent, header) // 根据 parent 和 header 创建一个 work 对象
if err != nil {
log.Error("Failed to create mining context", "err", err)
return
}
// Create the current work task and check any fork transitions needed
work := self.current
if self.config.DAOForkSupport && self.config.DAOForkBlock != nil && self.config.DAOForkBlock.Cmp(header.Number) == 0 {
misc.ApplyDAOHardFork(work.state)
}
pending, err := self.eth.TxPool().Pending() // 取出 TxPool 中 pending 的 tx
if err != nil {
log.Error("Failed to fetch pending transactions", "err", err)
return
}
txs := types.NewTransactionsByPriceAndNonce(self.current.signer, pending) // 取出 tx_pool 中状态为 pending 的 tx
work.commitTransactions(self.mux, txs, self.chain, self.coinbase) // 处理 txs

// compute uncles for the new block.
var (
uncles    []*types.Header
badUncles []common.Hash
)
for hash, uncle := range self.possibleUncles {
if len(uncles) == 2 {
break
}
if err := self.commitUncle(work, uncle.Header()); err != nil {
log.Trace("Bad uncle found and will be removed", "hash", hash)
log.Trace(fmt.Sprint(uncle))

badUncles = append(badUncles, hash)
} else {
log.Debug("Committing new uncle to block", "hash", hash)
uncles = append(uncles, uncle.Header())
}
}
for _, hash := range badUncles {
delete(self.possibleUncles, hash)
}
// Create the new block to seal with the consensus engine
// 将 txs 组装区块
if work.Block, err = self.engine.Finalize(self.chain, header, work.state, work.txs, uncles, work.receipts); err != nil {
log.Error("Failed to finalize block for sealing", "err", err)
return
}
// We only care about logging if we're actually mining.
if atomic.LoadInt32(&self.mining) == 1 {
log.Info("Commit new mining work", "number", work.Block.Number(), "txs", work.tcount, "uncles", len(uncles), "elapsed", common.PrettyDuration(time.Since(tstart)))
self.unconfirmed.Shift(work.Block.NumberU64() - 1)
}
self.push(work) // 发布一个 work
self.updateSnapshot()
}

func (self *worker) commitUncle(work *Work, uncle *types.Header) error {
hash := uncle.Hash()
if work.uncles.Contains(hash) {
return fmt.Errorf("uncle not unique")
}
if !work.ancestors.Contains(uncle.ParentHash) {
return fmt.Errorf("uncle's parent unknown (%x)", uncle.ParentHash[0:4])
}
if work.family.Contains(hash) {
return fmt.Errorf("uncle already in family (%x)", hash)
}
work.uncles.Add(uncle.Hash())
return nil
}

func (self *worker) updateSnapshot() {
self.snapshotMu.Lock()
defer self.snapshotMu.Unlock()

self.snapshotBlock = types.NewBlock(
self.current.header,
self.current.txs,
nil,
self.current.receipts,
)
self.snapshotState = self.current.state.Copy()
}
```

这个函数的基本逻辑如下：

* 准备新区块的时间属性 Header.Time，一般均等于系统当前时间，不过要确保父区块的时间(parentBlock.Time())要早于新区块的时间，父区块来自当前区块链的链头了。

* 创建新区块的Header对象，其各属性中：Num可确定(父区块Num +1)；Time可确定；ParentHash可确定;其余诸如Difficulty，GasLimit等，均留待之后共识算法中确定。

* 调用Engine.Prepare()函数，完成Header对象的准备,即 header.Difficulty 属性赋值。

* 根据新区块的位置(Number)，查看它是否处于DAO硬分叉的影响范围内，如果是，则赋值予header.Extra。

* 根据已有的Header对象，创建一个新的Work对象，并用其更新worker.current成员变量。

* 如果配置信息中支持硬分叉，在Work对象的StateDB里应用硬分叉。

* 准备新区块的交易列表，来源是TxPool中那些最近加入的tx，并执行这些交易。

* 准备新区块的叔区块uncles[]，来源是worker.possibleUncles[]，而possibleUncles[]中的每个区块都从事件ChainSideEvent中搜集得到。注意叔区块最多有两个。

* 调用Engine.Finalize()函数，对新区块“定型”，填充上Header.Root, TxHash, ReceiptHash, UncleHash等几个属性。

* 如果上一个区块(即旧的链头区块)处于unconfirmedBlocks中，意味着它也是由本节点挖掘出来的，尝试去验证它已经被吸纳进主链中。

* 把创建的Work对象，通过channel发送给每一个登记过的Agent，进行后续的挖掘。

commitNewWork()完成了待挖掘区块的组装，block.Header 创建完毕，交易数组 txs，叔区块 Uncles[]都已取得，并且由于所有交易被执行完毕，相应的 Receipt[]也已获得。万事俱备，可以交给Agent进行‘挖掘’了。
 

### miner.update()

这个update()会订阅(监听)几种事件，均跟Downloader相关。当收到Downloader的StartEvent时，意味者此时本节点正在从其他节点下载新区块，这时miner会立即停止进行中的挖掘工作，并继续监听；如果收到DoneEvent或FailEvent时，意味本节点的下载任务已结束-无论下载成功或失败-此时都可以开始挖掘新区块，并且此时会退出Downloader事件的监听。

从miner.Update()的逻辑可以看出，对于任何一个Ethereum网络中的节点来说，挖掘一个新区块和从其他节点下载、同步一个新区块，根本是相互冲突的。这样的规定，保证了在某个节点上，一个新区块只可能有一种来源，这可以大大降低可能出现的区块冲突，并避免全网中计算资源的浪费。

```
// 这个update()会订阅(监听)Downloader相关事件。
func (self *Miner) update() {
events := self.mux.Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})
out:
for ev := range events.Chan() {
switch ev.Data.(type) {
case downloader.StartEvent: // StartEvent事件
atomic.StoreInt32(&self.canStart, 0)// self.canStart 设为 0， 表示现在不可以 mining
if self.Mining() {
self.Stop()
atomic.StoreInt32(&self.shouldStart, 1)
log.Info("Mining aborted due to sync")
}
case downloader.DoneEvent, downloader.FailedEvent:
shouldStart := atomic.LoadInt32(&self.shouldStart) == 1

atomic.StoreInt32(&self.canStart, 1) // self.canStart 设为 1， 表示现在可以 mining
atomic.StoreInt32(&self.shouldStart, 0)
if shouldStart {
self.Start(self.coinbase) // 开始 mining
}
// unsubscribe. we're only interested in this event once
events.Unsubscribe()
// stop immediately and ignore all further pending events
break out
}
}
}
```

#### miner.start()

miner.start 主要调用 worker.start() 以及 worker.commitNewWork()。

```
func (self *Miner) Start(coinbase common.Address) {
	atomic.StoreInt32(&self.shouldStart, 1)
	self.SetEtherbase(coinbase)

	if atomic.LoadInt32(&self.canStart) == 0 {
		log.Info("Network syncing, will start miner afterwards")
		return
	}
	atomic.StoreInt32(&self.mining, 1)

	log.Info("Starting mining operation")
	self.worker.start()
	self.worker.commitNewWork()
}
```

##### worker.start()

worker.start 中继续调用agent.Start。

```
func (self *worker) start() {
	self.mu.Lock()
	defer self.mu.Unlock()

	atomic.StoreInt32(&self.mining, 1)

	// spin up agents
	for agent := range self.agents {
		agent.Start()
	}
}
```

######　agent.Start()

agent.Start中开启一个线程执行CpuAgent.update，其会调用共识算法引擎来 Seal block，worker.wait()会在一个channel处一直等待Agent完成挖掘发送回来的新Block和Work对象。这个Block会被写入数据库，加入本地的区块链试图成为最新的链头。

```
func (self *CpuAgent) Start() {
	if !atomic.CompareAndSwapInt32(&self.isMining, 0, 1) {
		return // agent already started
	}
	go self.update()
}

func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) {
	if result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
		log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
		self.returnCh <- &Result{work, result}
	} else {
		if err != nil {
			log.Warn("Block sealing failed", "err", err)
		}
		self.returnCh <- nil
	}
}
```









