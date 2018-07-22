Ethereum 使用的Merkle-PatriciaTrie(MPT)结构，源自于Trie结构，又分别继承了PatriciaTrie和MerkleTree的优点.

下面是Merkle Patricia Trie的数据结构:
```
// Trie is a Merkle Patricia Trie.
// The zero value is an empty trie with no database.
// Use New to create a trie that sits on top of a database.
//
// Trie is not safe for concurrent use.
type Trie struct {
db           *Database
root         node // 整个MPT的根节点， 有四种类型的Node 分别为 fullNode， shortNode， hashNode， valueNode
originalRoot common.Hash // originalRoot 的作用是在创建Trie对象时承接入参 hashNode

// Cache generation values.
// cachegen increases by one with each commit operation.
// cacheGen是cache次数的计数器，每次Trie的变动提交后，cacheGen自增1
// new nodes are tagged with the current generation and unloaded
// when their generation is older than than cachegen-cachelimit.
cachegen, cachelimit uint16
}
```

root为整个MPT树的根节点, 在MPT中一共有四种节点分别为fullNode、shortNode、hashNode和valueNode:
```
type (
fullNode struct {
Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
flags    nodeFlag
}
// 最多可以携带17个子节点，数组中前16位分别对应16进制(hex)下的0-9a-f（参考 indices ）
// 这样对于每个子节点，根据其key值16进制形式下的第一位的值，就可挂载到Children数组的某个位置

shortNode struct {
Key   []byte
Val   node
flags nodeFlag
} // shortNode 的 key 对应的子节点为 Val， 即，shortNode 和其子节点 Node合并到了一起，体现了PatriciaTrie的特点

hashNode  []byte // hashNode是fullNode或者shortNode对象的RLP哈希值
// 在MPT中，hashNode几乎不会单独存在(有时遍历遇到一个hashNode往往因为原本的node被折叠了)，
// 而是以nodeFlag结构体的成员(nodeFlag.hash)的形式，被fullNode和shortNode间接持有
// 一旦fullNode或shortNode的成员变量(包括子结构)发生任何变化，它们的hashNode就一定需要更新

valueNode []byte // 充当MPT的叶子节点，其实是字节数组[]byte的一个别名，不带子节点
// 所携带数据部分的RLP哈希值，长度32byte，数据的RLP编码值（key ）作为valueNode（value）的匹配项存储在数据库里
// 当我们从MTP树中找到 valueNode时，根据其RLP编码值可到数据库中找到真实的 valueNode
)
```

* fullNode 最多可以携带17个子节点，数组中前16位分别对应16进制(hex)下的0-9a-f（参考 indices )这样对于每个子节点，根据其key值16进制形式下的第一位的值，就可挂载到Children数组的某个位置

* fullNode和shortNode的成员变量 nodeFlag 存储的是hashNode,即 fullNode或者shortNode对象的RLP哈希值.一旦fullNode或shortNode的成员变量(包括子结构)发生任何变化，它们的hashNode就一定需要更新. 这里的 所有的hashNode组合一起其实也就是一棵 Merker Tree. 

* shortNode 其实是 PatriciaTrie 的体现,即如果存在一个父节点只有一个子节点，那么这个父节点将与其子节点合并。这样可以缩短Trie中不必要的深度，大大加快搜索节点速度。

*  valueNode为整个MPT树的叶子节点, 所携带数据部分的RLP哈希值，长度32byte,当我们从MTP树中找到 valueNode时，根据该RLP编码值可到数据库中找到真实的valueNode


任何一个[k,v]类型数据被插入一个MPT时，会以k字符串为路径沿着root向下延伸,在此次插入结束时首先成为一个valueNode，k会以自顶点root起到到该节点止的key path形式存在有了这个key path以后，可以很快的在MTP树中查找节点


下面代码为MPT树中插入一个节点:

```
func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
if len(key) == 0 {
if v, ok := n.(valueNode); ok {// 判断节点上插入的node值和已存在的node值是否相同
return !bytes.Equal(v, value.(valueNode)), value, nil // 此时插入node值和已存在的node值相同
}
return true, value, nil
}
switch n := n.(type) {
case *shortNode:
matchlen := prefixLen(key, n.Key) // 找到 Key 和 n.Key的相同前缀的长度，即要插入的key已经存在于MTP树
// If the whole key matches, keep this short node as is
// and only update the value.
if matchlen == len(n.Key) { // 此时从n节点开始递归插入，此时n.Val应该是一个valueNode，此时 key[matchlen:] 为零
dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
if !dirty || err != nil {
return false, n, err
}
return true, &shortNode{n.Key, nn, t.newFlag()}, nil // 更新插入的node值
}
// Otherwise branch out at the index where they differ.
branch := &fullNode{flags: t.newFlag()}
var err error
// 在branch 的 n.Key[matchlen] 处挂载 n 的前缀第matchle n+1位的前缀 n.Key[matchlen]
_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
if err != nil {
return false, nil, err
}
// 在branch的 key[matchlen] 处挂载要插入的node的第 matchlen+1 位前缀 key[matchlen]
_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
if err != nil {
return false, nil, err
}
// 如果该索引出现在索引0，则将该shortNode替换为branch
if matchlen == 0 {
return true, branch, nil
}
// Otherwise, replace it with a short node leading up to the branch.
return true, &shortNode{key[:matchlen], branch, t.newFlag()}, nil

case *fullNode:
dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
if !dirty || err != nil {
return false, n, err
}
n = n.copy()
n.flags = t.newFlag()
n.Children[key[0]] = nn
return true, n, nil

case nil:
return true, &shortNode{key, value, t.newFlag()}, nil

case hashNode:
//在MPT的查找，插入，删除中，如果遍历过程中遇到一个hashNode，
//首先需要从数据库里以这个哈希值为k，读取出相匹配的v，然后再将v解码恢复成fullNode或shortNode。
//在代码中这个过程叫resolve。
rn, err := t.resolveHash(n, prefix)
if err != nil {
return false, nil, err
}
dirty, nn, err := t.insert(rn, prefix, key, value)
if !dirty || err != nil {
return false, rn, err
}
return true, nn, nil

default:
panic(fmt.Sprintf("%T: invalid node: %v", n, n))
}
}
```
随着其他节点的不断插入和删除，根据MPT结构的要求，原有节点可能会变化成其他node实现类型，同时MPT中也会不断裂变或者合并出新的(父)节点。

* 假设一个shortNode S已经有一个子节点A，现在要新插入一个子节点B，那么会有两种可能，要么新节点B沿着A的路径继续向下，这样S的子节点会被更新；要么S的Key分裂成两段，前一段分配给S作为新的Key，同时裂变出一个新的fullNode作为S的子节点，以同时容纳B，以及需要更新的A。

* 如果一个fullNode原本只有两个子节点，现在要删除其中一个子节点，那么这个fullNode就会退化为shortNode，同时保留的子节点如果是shortNode，还可以跟它再合并。

* 如果一个shortNode的子节点是叶子节点同时又被删除了，那么这个shortNode就会退化成一个valueNode，成为一个叶子节点。


hashNode 主要存储在nodeFlag中:

```
// 有关节点的缓存相关元数据
type nodeFlag struct {
hash  hashNode // 缓存的 node 的hash值（可能是null）
gen   uint16   // 缓存生成计数器
dirty bool     // node 是否有需要写入数据库的变更
}
```
创建新nodeFlag对象的函数叫newFlags():

```
// newFlag 返回新创建节点的缓存flag值
func (t *Trie) newFlag() nodeFlag {
return nodeFlag{dirty: true, gen: t.cachegen}
}
```
在nodeFlag初始化过程中，__bool成员dirty置为true，表明了所代表的父节点有变更需要提交，同时hashNode成员hash，直接设空__。


基于效率和数据安全考虑，trie.Trie仅提供整个MPT结构的折叠(collapse)操作Hash()函数，它默认从顶点root开始遍历。

```
// Hash 函数返回trie的根hash值
// 它不写入数据库，即使没有trie，也可以使用它
func (t *Trie) Hash() common.Hash {
hash, cached, _ := t.hashRoot(nil, nil)
t.root = cached
return common.BytesToHash(hash.(hashNode))
}
```

再来看一下hashRoot函数:
```
func (t *Trie) hashRoot(db *Database, onleaf LeafCallback) (node, node, error) {
if t.root == nil { // trie的root为null
return hashNode(emptyRoot.Bytes()), nil, nil
}
h := newHasher(t.cachegen, t.cachelimit, onleaf) // 返回一个Hasher对象
defer returnHasherToPool(h)
return h.hash(t.root, db, true)
}
```

hashRoot()函数内部调用Hasher结构体进行折叠操作：
```
// hash collapses a node down into a hash node, also returning a copy of the
// original node initialized with the computed hash to replace the original one.
// hash 将一个 node 折叠成一个 hash node，同时返回一个用计算哈希初始化的原始节点的副本来替换原来的节点。
func (h *hasher) hash(n node, db *Database, force bool) (node, node, error) {
// If we're not storing the node, just hashing, use available cached data
if hash, dirty := n.cache(); hash != nil {
if db == nil {
return hash, n, nil
}
if n.canUnload(h.cachegen, h.cachelimit) {
// Unload the node from cache. All of its subnodes will have a lower or equal
// cache generation number.
cacheUnloadCounter.Inc(1)
return hash, hash, nil
}
if !dirty {
return hash, n, nil
}
}
// Trie not processed yet or needs storage, walk the children
collapsed, cached, err := h.hashChildren(n, db)
if err != nil {
return hashNode{}, n, err
}
hashed, err := h.store(collapsed, db, force)
if err != nil {
return hashNode{}, n, err
}
// Cache the hash of the node for later reuse and remove
// the dirty flag in commit mode. It's fine to assign these values directly
// without copying the node first because hashChildren copies it.
cachedHash, _ := hashed.(hashNode)
switch cn := cached.(type) {
case *shortNode:
cn.flags.hash = cachedHash
if db != nil {
cn.flags.dirty = false
}
case *fullNode:
cn.flags.hash = cachedHash
if db != nil {
cn.flags.dirty = false
}
}
return hashed, cached, nil
}
```

```
// hashChildren replaces the children of a node with their hashes if the encoded
// size of the child is larger than a hash, returning the collapsed node as well
// as a replacement for the original node with the child hashes cached in.
func (h *hasher) hashChildren(original node, db *Database) (node, node, error) {
var err error

switch n := original.(type) {
case *shortNode:
// Hash the short node's child, caching the newly hashed subtree
collapsed, cached := n.copy(), n.copy()
collapsed.Key = hexToCompact(n.Key)
cached.Key = common.CopyBytes(n.Key)

if _, ok := n.Val.(valueNode); !ok {
collapsed.Val, cached.Val, err = h.hash(n.Val, db, false)
if err != nil {
return original, original, err
}
}
return collapsed, cached, nil

case *fullNode:
// Hash the full node's children, caching the newly hashed subtrees
collapsed, cached := n.copy(), n.copy()

for i := 0; i < 16; i++ {
if n.Children[i] != nil {
collapsed.Children[i], cached.Children[i], err = h.hash(n.Children[i], db, false)
if err != nil {
return original, original, err
}
}
}
cached.Children[16] = n.Children[16]
return collapsed, cached, nil

default:
// Value and hash nodes don't have children so they're left as were
return n, original, nil
}
}
```
折叠node的入口是hasher.hash()，__在执行中, hash() 和 hashChiildren() 相互调用以遍历整个MPT结构，store()对节点作RLP哈希计算__。折叠node的基本逻辑是：如果node没有子节点，那么直接返回；如果这个node带有子节点，那么首先将子节点折叠成hashNode。当这个node的子节点全都变成哈希值hashNode之后，再对这个node作RLP+哈希计算，得到它的哈希值，亦即hashNode。
注意到hash()和hashChildren()返回两个node类型对象，第一个@hash是入参n经过折叠的hashNode哈希值，第二个@cached是没有经过折叠的n，并且n的hashNode还被赋值了。


## trie 的遍历和折叠原理






































