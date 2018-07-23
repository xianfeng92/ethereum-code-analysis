##  StateDB

StateDB有一个trie.Trie类型成员trie，它又被称为storage trie，这个MPT结构中存储的都是stateObject对象，每个stateObject对象以其地址作为插入节点的Key。

StateDB结构定义如下：
```
// StateDBs within the ethereum protocol are used to store anything
// within the merkle trie. StateDBs take care of caching and storing
// nested states. It's the general query interface to retrieve:
// * Contracts
// * Accounts
// StateDB 使用the merkle trie 存储以太坊协议中使用的任何内容
// StateDB负责缓存和存储嵌套状态
type StateDB struct {
	db   Database
	trie Trie // 负责管理 stateObject 对象

	// This map holds 'live' objects, which will get modified while processing a state transition.
        // Address ===> stateObject
	stateObjects      map[common.Address]*stateObject
	stateObjectsDirty map[common.Address]struct{}

	// DB error.
	// State objects are used by the consensus core and VM which are
	// unable to deal with database-level errors. Any error that occurs
	// during a database read is memoized here and will eventually be returned
	// by StateDB.Commit.
	dbErr error

	// The refund counter, also used by state transitioning.
	refund uint64

	thash, bhash common.Hash
	txIndex      int
	logs         map[common.Hash][]*types.Log
	logSize      uint

	preimages map[common.Hash][]byte

	// Journal of state modifications. This is the backbone of
	// Snapshot and RevertToSnapshot.
	journal        *journal
	validRevisions []revision
	nextRevisionId int

	lock sync.Mutex
}
```

在 StateDB 的主要作用是管理 stateObject 对象， stateObject 与具体的以太坊账户 Address 关联。StaseObject 对象中主要存储着以太坊 Address 相关的余额（Account）、合约字节码（Code）。每当一个stateObject有改动，亦即“账户”信息（如：收到一笔转账，余额增加了）有变动时，这个stateObject对象会更新，并且这个stateObject会标为 dirty （即，dirtyCode 为 True），此时所有的数据改动还仅仅存储在map里。

当IntermediateRoot()调用时，所有标为dirty的stateObject才会被一起写入trie。

```
// IntermediateRoot 计算当前 stateTrie的 root hash
// 在事务之间调用以获取进入事务收据的根哈希。
func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash {
	s.Finalise(deleteEmptyObjects)
	return s.trie.Hash()
}
```

而整个trie中的内容只有在 CommitTo()调用时被一起提交到底层数据库。可见，这个map被用作本地的一级缓存，trie是二级缓存，底层数据库是第三级，各级数据结构的界限非常清晰，这样逐级缓存数据，每一级数据向上一级提交的时机也根据业务需求做了合理的选择。

具体关系图如下所示：


再来看一下stateObject对象：

```
// StaseObject 表示正在修改的以太坊帐户
// 使用模式如下：
// 首先，获取stateObject
// 可以通过stateObject访问和修改帐户值（Account）
// 最后，调用CommitTrie将修改后的 storage trie写入数据库中
type stateObject struct {
	address  common.Address // 账户地址
	addrHash common.Hash // 以太坊账户地址的hash值
	data     Account     // 账户相关数据
	db       *StateDB

	// DB error.
	// State objects are used by the consensus core and VM which are
	// unable to deal with database-level errors. Any error that occurs
	// during a database read is memoized here and will eventually be returned
	// by StateDB.Commit.
	dbErr error

	// 写缓存
	trie Trie // storage trie, which becomes non-nil on first access
	code Code // 合约字节码，在加载合约时被设置

	cachedStorage Storage // Storage entry cache to avoid duplicate reads
	dirtyStorage  Storage // 需要刷新到磁盘的存储条目

	// 缓存标志
	// 当stateObject对象被标记为suicided时，将在状态 update 阶段将该对象从Trie中删除。
	dirtyCode bool // 如果代码已更新，则为 True
	suicided  bool
	deleted   bool
}

```
每个stateObject对象管理着Ethereum世界里的一个“账户”。stateObject有一个成员变量data，类型是Accunt结构体，里面存有账户Ether余额，合约发起次数，最新发起合约指令集的哈希值，以及一个MPT结构的顶点哈希值。

Account 结构如下：
```
// Account is the Ethereum consensus representation of accounts.
// These objects are stored in the main account trie.
type Account struct {
	Nonce    uint64 // 合约调用次数
	Balance  *big.Int // 账户余额
	Root     common.Hash // merkle root of the storage trie
	CodeHash []byte
}
```

stateObject内部也有一个Trie类型的成员trie，被称为storage trie，它里面存放的是一种被称为State的数据。State跟每个账户相关，格式是[Hash, Hash]键值对。有意思的是，stateObject内部也有类似StateDB一样的二级数据缓存机制，用来缓存和更新这些State。


stateObject定义了一种类型名为storage的map结构，用来存放[]Hash,Hash]类型的数据对，也就是State数据。当SetState()调用发生时，storage内部State数据被更新，相应标示为"dirty"。之后，待有需要时(比如updateRoot()调用)，那些标为"dirty"的State数据被一起写入storage trie，而storage trie中的所有内容在CommitTo()调用时再一起提交到底层数据库。





















