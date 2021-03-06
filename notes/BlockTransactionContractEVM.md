
## SHA3

SHA-3在2015年8月由美国标准技术协会(NIST)正式发布，作为Secure Hash Algorithm家族的最新一代标准，它相比于SHA-2和SHA-1，采用了完全不同的设计思路，性能也比较好。需要注意的是，SHA-2目前并没有出现被成功攻克的案例，SHA-3也没有要立即取代SHA-2的趋势，NIST只是考虑到SHA-1有过被攻克的案例，未雨绸缪的征选了采用全新结构和思路的SHA-3来作为一种最新的SHA方案。


## RLP

RLP(Recursive Length Prefix),递归长度前缀编码，__它是以太坊序列化所采用的编码方式。RLP主要用于以太坊中数据的网络传输和持久化存储__。它可以将一个任意嵌套的字节数组([]byte)，编码成一个“展平”无嵌套的[]byte。RLP是可逆的，它提供了互逆的编码、解码方法。

Ethereum 中具体使用的哈希算法，就是__对某个类型对象的RLP编码值做了SHA3哈希运算，可称为 RLP Hash__。 Ethereum 在底层存储中特意选择了专门存储和读取 [k, v] 键值对的第三方数据库，[k, v] 中的 v 就是某个结构体对象的RLP编码值([]byte)，k大多数情况就是v的RLP编码后的SHA-3哈希值。

## Hash and Address

```
const (
	HashLength    = 32
	AddressLength = 20
)

// Hash represents the 32 byte Keccak256 hash of arbitrary data.
type Hash [HashLength]byte

// Address represents the 20 byte address of an Ethereum account.
type Address [AddressLength]byte

```
在Ethereum 代码里，所有用到的哈希值，都使用该Hash类型，长度为32bytes，即256 bits；Ethereum 中所有跟帐号(Account)相关的信息，比如交易转帐的转出帐号(地址)和转入帐号(地址)，都会用该Address类型表示，长度20bytes, 即160bits。

--------------------------------------------------------------------


