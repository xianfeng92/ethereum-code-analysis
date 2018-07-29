## Miner

*  miner/[miner](https://github.com/xianfeng92/go-ethereum/tree/master/miner)

挖掘新区块的流程入口在Miner里,具体入口在Miner结构体的创建函数.在New()里，针对新对象miner的各个成员变量初始化完成后，会紧跟着创建worker对象，然后将Agent对象登记给worker，最后用一个单独线程去运行miner.Update()函数.这个update()会订阅(监听)几种事件，均跟Downloader相关。当收到Downloader的StartEvent时，意味者此时本节点正在从其他节点下载新区块，这时miner会立即停止进行中的挖掘工作，并继续监听；如果收到DoneEvent或FailEvent时，意味本节点的下载任务已结束-无论下载成功或失败-此时都可以开始挖掘新区块(self.Start(self.coinbase) )，并且此时会退出Downloader事件的监听。

```
// Copyright 2014 The go-ethereum Authors
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

// Package miner implements Ethereum block creation and mining.
package miner

import (
"fmt"
"sync/atomic"

"github.com/ethereum/go-ethereum/accounts"
"github.com/ethereum/go-ethereum/common"
"github.com/ethereum/go-ethereum/consensus"
"github.com/ethereum/go-ethereum/core"
"github.com/ethereum/go-ethereum/core/state"
"github.com/ethereum/go-ethereum/core/types"
"github.com/ethereum/go-ethereum/eth/downloader"
"github.com/ethereum/go-ethereum/ethdb"
"github.com/ethereum/go-ethereum/event"
"github.com/ethereum/go-ethereum/log"
"github.com/ethereum/go-ethereum/params"
)

// Backend wraps all methods required for mining.
type Backend interface {
AccountManager() *accounts.Manager
BlockChain() *core.BlockChain
TxPool() *core.TxPool
ChainDb() ethdb.Database
}

// Miner creates blocks and searches for proof-of-work values.
type Miner struct {
mux *event.TypeMux

worker *worker

coinbase common.Address
mining   int32 // 是否在 mining 中
eth      Backend
engine   consensus.Engine

canStart    int32 // 表示 能否进行mining
shouldStart int32 // should start indicates whether we should start after sync
}

func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
miner := &Miner{
eth:      eth,
mux:      mux,
engine:   engine,
// 在创建 worker 时， 会调用 worker.commitNewWork()，即构建一个 block header，获取当前的最新 block 作为其 parent
// 利用 header 和 parent 生成一个 work，在 work 中记录 从 tx_pool 中获取 pending 状态的 tx 以及对应的recepits
worker:   newWorker(config, engine, common.Address{}, eth, mux),
canStart: 1,// 可以开始 mining
}
miner.Register(NewCpuAgent(eth.BlockChain(), engine))
go miner.update() // 一个单独线程去运行miner.Update()函数， 当一个节点更新成最新状态时，就 start Mining

return miner
}

// update keeps track of the downloader events. Please be aware that this is a one shot type of update loop.
// It's entered once and as soon as `Done` or `Failed` has been broadcasted the events are unregistered and
// the loop is exited. This to prevent a major security vuln where external parties can DOS you with blocks
// and halt your mining operation for as long as the DOS continues.
// 这个update()会订阅(监听)Downloader相关事件。
func (self *Miner) update() {
events := self.mux.Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})
out:
for ev := range events.Chan() {
switch ev.Data.(type) {
case downloader.StartEvent:
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

func (self *Miner) Start(coinbase common.Address) {
atomic.StoreInt32(&self.shouldStart, 1)
self.SetEtherbase(coinbase)
if atomic.LoadInt32(&self.canStart) == 0 {
log.Info("Network syncing, will start miner afterwards")
return
}
atomic.StoreInt32(&self.mining, 1) // 表示在 mining 中

log.Info("Starting mining operation")
self.worker.start()
self.worker.commitNewWork()
}

func (self *Miner) Stop() {
self.worker.stop()
atomic.StoreInt32(&self.mining, 0)
atomic.StoreInt32(&self.shouldStart, 0)
}

func (self *Miner) Register(agent Agent) {
if self.Mining() {
agent.Start()
}
self.worker.register(agent)
}

func (self *Miner) Unregister(agent Agent) {
self.worker.unregister(agent)
}

func (self *Miner) Mining() bool {
return atomic.LoadInt32(&self.mining) > 0
}

func (self *Miner) HashRate() (tot int64) {
if pow, ok := self.engine.(consensus.PoW); ok {
tot += int64(pow.Hashrate())
}
// do we care this might race? is it worth we're rewriting some
// aspects of the worker/locking up agents so we can get an accurate
// hashrate?
for agent := range self.worker.agents {
if _, ok := agent.(*CpuAgent); !ok {
tot += agent.GetHashRate()
}
}
return
}

func (self *Miner) SetExtra(extra []byte) error {
if uint64(len(extra)) > params.MaximumExtraDataSize {
return fmt.Errorf("Extra exceeds max length. %d > %v", len(extra), params.MaximumExtraDataSize)
}
self.worker.setExtra(extra)
return nil
}

// Pending returns the currently pending block and associated state.
func (self *Miner) Pending() (*types.Block, *state.StateDB) {
return self.worker.pending()
}

// PendingBlock returns the currently pending block.
//
// Note, to access both the pending block and the pending state
// simultaneously, please use Pending(), as the pending state can
// change between multiple method calls
func (self *Miner) PendingBlock() *types.Block {
return self.worker.pendingBlock()
}

func (self *Miner) SetEtherbase(addr common.Address) {
self.coinbase = addr
self.worker.setEtherbase(addr)
}
```

