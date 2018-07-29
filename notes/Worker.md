## Worker

* miner/[worker](https://github.com/xianfeng92/go-ethereum/blob/master/miner/worker.go)


worker 主要和Agent 互动,组装区块给Agent,并接收Agent挖出的Block.

```
// Copyright 2015 The go-ethereum Authors
// This file is part of the go-ethereum library.
//
// The go-ethereum library is free software: you can redistribute it and/or modify
// it under the terms of the GNU Lesser General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// The go-ethereum library is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
// GNU Lesser General Public License for more details.
//
// You should have received a copy of the GNU Lesser General Public License
// along with the go-ethereum library. If not, see <http://www.gnu.org/licenses/>.

package miner

import (
"bytes"
"fmt"
"math/big"
"sync"
"sync/atomic"
"time"

mapset "github.com/deckarep/golang-set"
"github.com/ethereum/go-ethereum/common"
"github.com/ethereum/go-ethereum/consensus"
"github.com/ethereum/go-ethereum/consensus/misc"
"github.com/ethereum/go-ethereum/core"
"github.com/ethereum/go-ethereum/core/state"
"github.com/ethereum/go-ethereum/core/types"
"github.com/ethereum/go-ethereum/core/vm"
"github.com/ethereum/go-ethereum/ethdb"
"github.com/ethereum/go-ethereum/event"
"github.com/ethereum/go-ethereum/log"
"github.com/ethereum/go-ethereum/params"
)

const (
resultQueueSize  = 10
miningLogAtDepth = 5

// txChanSize is the size of channel listening to NewTxsEvent.
// The number is referenced from the size of tx pool.
txChanSize = 4096
// chainHeadChanSize is the size of channel listening to ChainHeadEvent.
chainHeadChanSize = 10
// chainSideChanSize is the size of channel listening to ChainSideEvent.
chainSideChanSize = 10
)

// Agent can register themself with the worker
type Agent interface {
Work() chan<- *Work
SetReturnCh(chan<- *Result)
Stop()
Start()
GetHashRate() int64
}

// Work is the workers current environment and holds
// all of the current state information
// Work结构体主要用以携带数据，被视为 mining 一个区块时所需的数据环境。
type Work struct {
config *params.ChainConfig
signer types.Signer

state     *state.StateDB // apply state changes here
ancestors mapset.Set     // ancestor set (used for checking uncle parent validity)
family    mapset.Set     // family set (used for checking uncle invalidity)
uncles    mapset.Set     // uncle set
tcount    int            // tx count in cycle tx计数器
gasPool   *core.GasPool  // available gas used to pack transactions 打包区块可使用的gas

Block *types.Block // the new block 此处的 Block 为打包处理完的区块，等待 Seal

header   *types.Header
txs      []*types.Transaction // 记录当前已经处理的从 txpool 传来的 txs
receipts []*types.Receipt // 记录 tx 的执行后的 receipt

createdAt time.Time
}

// 返回一个经过授权确认的Block加上更新过的Work
type Result struct {
Work  *Work
Block *types.Block
}

// worker is the main object which takes care of applying messages to the new state
type worker struct {
config *params.ChainConfig // 链的配置
engine consensus.Engine // 共识算法接口

mu sync.Mutex

// update loop
mux          *event.TypeMux
txsCh        chan core.NewTxsEvent // 当 tx 进入transaction pool时，将发布 NewTxsEvent
txsSub       event.Subscription
chainHeadCh  chan core.ChainHeadEvent
chainHeadSub event.Subscription
chainSideCh  chan core.ChainSideEvent
chainSideSub event.Subscription // channal
wg           sync.WaitGroup

agents map[Agent]struct{} // 目前只有 CpuAgent
recv   chan *Result

eth     Backend
chain   *core.BlockChain
proc    core.Validator
chainDb ethdb.Database

coinbase common.Address // 接收挖矿奖励的地址
extra    []byte

currentMu sync.Mutex // 互斥锁
current   *Work // Work结构体主要用以携带数据，被视为 mining 一个区块时所需的数据环境

snapshotMu    sync.RWMutex
snapshotBlock *types.Block
snapshotState *state.StateDB

uncleMu        sync.Mutex
possibleUncles map[common.Hash]*types.Block // 存储可能的叔区块

unconfirmed *unconfirmedBlocks // set of locally mined blocks pending canonicalness confirmations

// atomic status counters
mining int32 // 标志是否在挖矿中
atWork int32 //
}

func newWorker(config *params.ChainConfig, engine consensus.Engine, coinbase common.Address, eth Backend, mux *event.TypeMux) *worker {
worker := &worker{
config:         config,
engine:         engine,
eth:            eth,
mux:            mux,
txsCh:          make(chan core.NewTxsEvent, txChanSize),// 使用make初始化Channel,并且可以设置容量
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
// Subscribe NewTxsEvent for tx pool 订阅 tx pool 的事件
worker.txsSub = eth.TxPool().SubscribeNewTxsEvent(worker.txsCh) // 当 eth.TxPool() 有新 txs 就会通知 txsCh
// Subscribe events for blockchain 订阅区块链的相关事件
worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
go worker.update()
// 单独开启一个线程，监听 NewTxsEvent ChainHeadEvent ChainSideEvent
// 监听中， 如果有新的区块产生，则会重新 commitNewWork ，然后 push 一个 work 给 Agent 进行 mining

go worker.wait() // 等待 Agent 的 mining 结果，如果成功挖出一个Block会将其写入数据库，并进行广播
worker.commitNewWork() // 组装一个 work， 并 push 给 Agent 进行 mining

return worker
}

func (self *worker) setEtherbase(addr common.Address) {
self.mu.Lock()
defer self.mu.Unlock()
self.coinbase = addr
}

func (self *worker) setExtra(extra []byte) {
self.mu.Lock()
defer self.mu.Unlock()
self.extra = extra
}

func (self *worker) pending() (*types.Block, *state.StateDB) {
if atomic.LoadInt32(&self.mining) == 0 {
// return a snapshot to avoid contention on currentMu mutex
self.snapshotMu.RLock()
defer self.snapshotMu.RUnlock()
return self.snapshotBlock, self.snapshotState.Copy()
}

self.currentMu.Lock()
defer self.currentMu.Unlock()
return self.current.Block, self.current.state.Copy()
}

func (self *worker) pendingBlock() *types.Block {
if atomic.LoadInt32(&self.mining) == 0 {
// return a snapshot to avoid contention on currentMu mutex
self.snapshotMu.RLock()
defer self.snapshotMu.RUnlock()
return self.snapshotBlock
}

self.currentMu.Lock()
defer self.currentMu.Unlock()
return self.current.Block
}

func (self *worker) start() {
self.mu.Lock()
defer self.mu.Unlock()

atomic.StoreInt32(&self.mining, 1)

// spin up agents
for agent := range self.agents { // 目前就一个 CpuAgent
agent.Start()
}
}

func (self *worker) stop() {
self.wg.Wait()

self.mu.Lock()
defer self.mu.Unlock()
if atomic.LoadInt32(&self.mining) == 1 {
for agent := range self.agents {
agent.Stop()
}
}
atomic.StoreInt32(&self.mining, 0)
atomic.StoreInt32(&self.atWork, 0)
}

func (self *worker) register(agent Agent) {
self.mu.Lock()
defer self.mu.Unlock()
self.agents[agent] = struct{}{}
agent.SetReturnCh(self.recv)
}

func (self *worker) unregister(agent Agent) {
self.mu.Lock()
defer self.mu.Unlock()
delete(self.agents, agent)
agent.Stop()
}

func (self *worker) update() {
defer self.txsSub.Unsubscribe()
defer self.chainHeadSub.Unsubscribe()
defer self.chainSideSub.Unsubscribe()

for {
// A real event arrived, process interesting content
select {
// Handle ChainHeadEvent 监听是否有区块挖出
case <-self.chainHeadCh:
// 如果有新区块挖出，则准备新的 work，并开始mining
self.commitNewWork()

// Handle ChainSideEvent 监听是否有叔区块
case ev := <-self.chainSideCh:
self.uncleMu.Lock()
self.possibleUncles[ev.Block.Hash()] = ev.Block
self.uncleMu.Unlock()

// Handle NewTxsEvent 监听新 tx
// txpool 每 add 一个 tx，就会通知 Send(NewTxsEvent)
case ev := <-self.txsCh:
// Apply transactions to the pending state if we're not mining.
// 如果不进行挖掘（mining），则将处理 transactions ，将其变为 pending 状态
// Note all transactions received may not be continuous with transactions
// already included in the current mining block. These transactions will
// be automatically eliminated.
if atomic.LoadInt32(&self.mining) == 0 {
// 判断是否在 mining 中。
// 组装一个 work
self.currentMu.Lock() // 互斥的操作
txs := make(map[common.Address]types.Transactions)
for _, tx := range ev.Txs {
acc, _ := types.Sender(self.current.signer, tx) // 从 tx 的the signature (V, R, S）提取处 sender的address
txs[acc] = append(txs[acc], tx)
// 将 tx 添加到 txs[acc] 列表中，即记录每个 address 的所有交易， nonce值一定要连续
}
txset := types.NewTransactionsByPriceAndNonce(self.current.signer, txs) // 取每个 address 的tx，并按照 price 排序
self.current.commitTransactions(self.mux, txset, self.chain, self.coinbase) // 执行 txset
self.updateSnapshot()
self.currentMu.Unlock()
} else {
// If we're mining, but nothing is being processed, wake on new transactions
if self.config.Clique != nil && self.config.Clique.Period == 0 {
self.commitNewWork()
}
}

// System stopped
case <-self.txsSub.Err():
return
case <-self.chainHeadSub.Err():
return
case <-self.chainSideSub.Err():
return
}
}
}
// worker.wait()会在一个channel处一直等待Agent完成挖掘发送回来的新Block和Work对象
// 这个Block会被写入数据库，加入本地的区块链试图成为最新的链头
func (self *worker) wait() {
for {
for result := range self.recv {
atomic.AddInt32(&self.atWork, -1)

if result == nil {
continue
}
block := result.Block
work := result.Work

// Update the block hash in all logs since it is now available and not when the
// receipt/log of individual transactions were created.
for _, r := range work.receipts {
for _, l := range r.Logs {
l.BlockHash = block.Hash()
}
}
for _, log := range work.state.Logs() {
log.BlockHash = block.Hash()
}
// 将块和所有关联的状态写入数据库
stat, err := self.chain.WriteBlockWithState(block, work.receipts, work.state)
if err != nil {
log.Error("Failed writing block to chain", "err", err)
continue
}
// Broadcast the block and announce chain insertion event
// 广播块和宣告链插入事件
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

// Insert the block into the set of pending ones to wait for confirmations
// 将块插入待处理的集合中等待确认
self.unconfirmed.Insert(block.NumberU64(), block.Hash())
}
}
}

// push sends a new work task to currently live miner agents.
// 推送一个新的 work 任务给当前的 miner agents
func (self *worker) push(work *Work) {
if atomic.LoadInt32(&self.mining) != 1 {
return
}
for agent := range self.agents {
atomic.AddInt32(&self.atWork, 1)
if ch := agent.Work(); ch != nil {
ch <- work
}
}
}

// makeCurrent creates a new environment for the current cycle.
func (self *worker) makeCurrent(parent *types.Block, header *types.Header) error {
state, err := self.chain.StateAt(parent.Root())
if err != nil {
return err
}
work := &Work{
config:    self.config,
signer:    types.NewEIP155Signer(self.config.ChainID),
state:     state,
ancestors: mapset.NewSet(),
family:    mapset.NewSet(),
uncles:    mapset.NewSet(),
header:    header,
createdAt: time.Now(),
}

// when 08 is processed ancestors contain 07 (quick block)
for _, ancestor := range self.chain.GetBlocksFromHash(parent.Hash(), 7) {
for _, uncle := range ancestor.Uncles() {
work.family.Add(uncle.Hash())
}
work.family.Add(ancestor.Hash())
work.ancestors.Add(ancestor.Hash())
}

// Keep track of transactions which return errors so they can be removed
work.tcount = 0
self.current = work
return nil
}

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

func (env *Work) commitTransactions(mux *event.TypeMux, txs *types.TransactionsByPriceAndNonce, bc *core.BlockChain, coinbase common.Address) {
if env.gasPool == nil {
env.gasPool = new(core.GasPool).AddGas(env.header.GasLimit)
}

var coalescedLogs []*types.Log

for {
// If we don't have enough gas for any further transactions then we're done
if env.gasPool.Gas() < params.TxGas {
log.Trace("Not enough gas for further transactions", "have", env.gasPool, "want", params.TxGas)
break
}
// Retrieve the next transaction and abort if all done
tx := txs.Peek()
if tx == nil {
break
}
// Error may be ignored here. The error has already been checked
// during transaction acceptance is the transaction pool.
//
// We use the eip155 signer regardless of the current hf.
from, _ := types.Sender(env.signer, tx)
// Check whether the tx is replay protected. If we're not in the EIP155 hf
// phase, start ignoring the sender until we do.
if tx.Protected() && !env.config.IsEIP155(env.header.Number) {
log.Trace("Ignoring reply protected transaction", "hash", tx.Hash(), "eip155", env.config.EIP155Block)
txs.Pop()
continue
}
// Start executing the transaction
env.state.Prepare(tx.Hash(), common.Hash{}, env.tcount)

err, logs := env.commitTransaction(tx, bc, coinbase, env.gasPool) //交易的执行
switch err {
case core.ErrGasLimitReached:
// Pop the current out-of-gas transaction without shifting in the next from the account
log.Trace("Gas limit exceeded for current block", "sender", from)
txs.Pop()

case core.ErrNonceTooLow:
// New head notification data race between the transaction pool and miner, shift
log.Trace("Skipping transaction with low nonce", "sender", from, "nonce", tx.Nonce())
txs.Shift()

case core.ErrNonceTooHigh:
// Reorg notification data race between the transaction pool and miner, skip account =
log.Trace("Skipping account with hight nonce", "sender", from, "nonce", tx.Nonce())
txs.Pop()

case nil:
// Everything ok, collect the logs and shift in the next transaction from the same account
coalescedLogs = append(coalescedLogs, logs...) // 将每个 tx 执行的 logs 放入 coalescedLogs
env.tcount++ // 记录处理的 tx 数量
txs.Shift()

default:
// Strange error, discard the transaction and get the next in line (note, the
// nonce-too-high clause will prevent us from executing in vain).
log.Debug("Transaction failed, account skipped", "hash", tx.Hash(), "err", err)
txs.Shift()
}
}

if len(coalescedLogs) > 0 || env.tcount > 0 {
// make a copy, the state caches the logs and these logs get "upgraded" from pending to mined
// logs by filling in the block hash when the block was mined by the local miner. This can
// cause a race condition if a log was "upgraded" before the PendingLogsEvent is processed.
cpy := make([]*types.Log, len(coalescedLogs))
for i, l := range coalescedLogs {
cpy[i] = new(types.Log)
*cpy[i] = *l // 复制 coalescedLogs
}
go func(logs []*types.Log, tcount int) {
if len(logs) > 0 {
mux.Post(core.PendingLogsEvent{Logs: logs})
}
if tcount > 0 {
mux.Post(core.PendingStateEvent{})
}
}(cpy, env.tcount)
}
}

func (env *Work) commitTransaction(tx *types.Transaction, bc *core.BlockChain, coinbase common.Address, gp *core.GasPool) (error, []*types.Log) {
snap := env.state.Snapshot()

receipt, _, err := core.ApplyTransaction(env.config, bc, &coinbase, gp, env.state, env.header, tx, &env.header.GasUsed, vm.Config{})
if err != nil {
env.state.RevertToSnapshot(snap)
return err, nil
}
// 将 tx 以及其执行完的 receipt， 存储到 work 中
env.txs = append(env.txs, tx)
env.receipts = append(env.receipts, receipt)

return nil, receipt.Logs
}
```

