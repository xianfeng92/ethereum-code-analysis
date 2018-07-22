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

valueNode []byte   // 充当MPT的叶子节点，其实是字节数组[]byte的一个别名，不带子节点
// 所携带数据部分的RLP哈希值，长度32byte，数据的RLP编码值（key ）作为valueNode（value）的匹配项存储在数据库里
// 当我们从MTP树中找到 valueNode时，根据其RLP编码值可到数据库中找到真实的 valueNode
)
```

* fullNode 最多可以携带17个子节点，数组中前16位分别对应16进制(hex)下的0-9a-f（参考 indices )这样对于每个子节点，根据其key值16进制形式下的第一位的值，就可挂载到Children数组的某个位置

* fullNode和shortNode的成员变量 nodeFlag 存储的是hashNode,即 fullNode或者shortNode对象的RLP哈希值.一旦fullNode或shortNode的成员变量(包括子结构)发生任何变化，它们的hashNode就一定需要更新. 这里的 所有的hashNode组合一起其实也就是一棵 Merker Tree. 

* shortNode 其实是 PatriciaTrie 的体现,即如果存在一个父节点只有一个子节点，那么这个父节点将与其子节点合并。这样可以缩短Trie中不必要的深度，大大加快搜索节点速度。

*  valueNode为整个MPT树的叶子节点, 所携带数据部分的RLP哈希值，长度32byte,当我们从MTP树中找到 valueNode时，根据该RLP编码值可到数据库中找到真实的valueNode

任何一个[k,v]类型数据被插入一个MPT时，会以k字符串为路径沿着root向下延伸,在此次插入结束时首先成为一个valueNode，k会以自顶点root起到到该节点止的key path形式存在有了这个key path以后，可以很快的在MTP树中查找节点

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
return true, &shortNode{n.Key, nn, t.newFlag()}, nil //更新插入的node值
}
// Otherwise branch out at the index where they differ.
branch := &fullNode{flags: t.newFlag()}
var err error
_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
if err != nil {
return false, nil, err
}
_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
if err != nil {
return false, nil, err
}
// Replace this shortNode with the branch if it occurs at index 0.
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
// We've hit a part of the trie that isn't loaded yet. Load
// the node and insert into it. This leaves all child nodes on
// the path to the value in the trie.
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













































