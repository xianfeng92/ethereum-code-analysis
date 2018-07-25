## gas

[Gas](https://github.com/xianfeng92/go-ethereum/blob/master/core/vm/gas.go), 是Ethereum里对所有活动进行消耗资源计量的单位。这里的活动是泛化的概念，包括但不限于：转帐，合约的创建，合约指令的执行，执行中内存的扩展等等。所以Gas可以想象成现实中的汽油或者燃气。__Gas是Ethereum系统的血液。一切资源，活动，交互的开销，都以Gas为计量单元__。如果定义了一个GasPrice，那么所有的Gas消耗亦可等价于以太币Ether。

关于 gas 的源码在具体场景中分析，这里就不单独分析了。
