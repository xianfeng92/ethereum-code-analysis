## Agent

* miner/[agent](https://github.com/xianfeng92/go-ethereum/blob/master/miner/agent.go)

CpuAgent中与mine相关的函数，主要是update()和mine().

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
"sync"

"sync/atomic"

"github.com/ethereum/go-ethereum/consensus"
"github.com/ethereum/go-ethereum/log"
)

type CpuAgent struct {
mu sync.Mutex

workCh        chan *Work
stop          chan struct{}
quitCurrentOp chan struct{}
returnCh      chan<- *Result

chain  consensus.ChainReader
engine consensus.Engine

isMining int32 // isMining indicates whether the agent is currently mining
}

func NewCpuAgent(chain consensus.ChainReader, engine consensus.Engine) *CpuAgent {
miner := &CpuAgent{
chain:  chain,
engine: engine,
stop:   make(chan struct{}, 1),
workCh: make(chan *Work, 1),
}
return miner
}

func (self *CpuAgent) Work() chan<- *Work            { return self.workCh }
func (self *CpuAgent) SetReturnCh(ch chan<- *Result) { self.returnCh = ch }

func (self *CpuAgent) Stop() {
if !atomic.CompareAndSwapInt32(&self.isMining, 1, 0) {
return // agent already stopped
}
self.stop <- struct{}{}
done:
// Empty work channel
for {
select {
case <-self.workCh:
default:
break done
}
}
}

func (self *CpuAgent) Start() {
if !atomic.CompareAndSwapInt32(&self.isMining, 0, 1) {
return // agent already started
}
go self.update()
}

//update函数会一直监听Work对象，如果收到由worker.commitNewWork()结束后发出Work对象
//就启动mine()函数；如果收到停止(mine)的消息，就退出一切相关操作。
func (self *CpuAgent) update() {
out:
for {
select {
case work := <-self.workCh: // // 监听 Work ，是否有组装好的区块
self.mu.Lock()
if self.quitCurrentOp != nil {
close(self.quitCurrentOp)
}
self.quitCurrentOp = make(chan struct{})
go self.mine(work, self.quitCurrentOp) // 开启一个新线程利用共识算法对区块 Seal
self.mu.Unlock()
case <-self.stop:
self.mu.Lock()
if self.quitCurrentOp != nil {
close(self.quitCurrentOp)
self.quitCurrentOp = nil
}
self.mu.Unlock()
break out
}
}
}

// CpuAgent.mine()会直接调用Engine.Seal()函数，利用Engine实现体的共识算法对传入的Block进行最终的授权
// 如果成功，就将Block同Work一起通过channel发还给worker，那边worker.wait()会接收并处理
func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) { // 找nonce值，做 pow ，尝试封装一个 block
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

func (self *CpuAgent) GetHashRate() int64 {
if pow, ok := self.engine.(consensus.PoW); ok {
return int64(pow.Hashrate())
}
return 0
}
```

