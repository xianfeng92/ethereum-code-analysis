# Trie树

字典树（Trie）可以保存一些字符串->值的对应关系。基本上，它跟 Java 的 HashMap 功能相同，都是 key-value 映射，只不过 Trie 的 key 只能是字符串。

Trie 的强大之处就在于它的时间复杂度。__它的插入和查询时间复杂度都为 O(k) ，其中 k 为 key 的长度，与 Trie 中保存了多少个元素无关__。sTrie 的缺点是空间消耗很高。典型应用是用于统计和排序大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。它的优点是：最大限度地减少无谓的字符串比较，查询效率比哈希表高。

 Trie的核心思想是空间换时间。__利用字符串的公共前缀来降低查询时间的开销以达到提高效率的目的__。

Trie树的基本性质可以归纳为： 
* 根节点不包含字符，除根节点以外每个节点只包含一个字符。
* 从根节点到某一个节点，路径上经过的字符连接起来，为该节点对应的字符串。
* 每个节点的所有子节点包含的字符串不相同。

基本思想:

* 插入过程

对于一个单词，从根开始，沿着单词的各个字母所对应的树中的节点分支向下走，直到单词遍历完，将最后的节点标记为红色，表示该单词已插入Trie树。

* 查询过程

从根开始按照单词的字母顺序向下遍历trie树，一旦发现某个节点标记不存在或者单词遍历完成而最后的节点未标记为红色，则表示该单词不存在，若最后的节点标记为红色，表示该单词存在。


## Trie树数据结构

利用串构建一个字典树，这个字典树保存了串的公共前缀信息，因此可以降低查询操作的复杂度。
下面以英文单词构建的字典树为例，这棵Trie树中每个结点包括26个孩子结点，因为总共有26个英文字母(假设单词都是小写字母组成)。

```
public class Trie_Tree{

/**
* 内部节点类
*
*/
private class Node{
private int dumpli_num;//该字串的反复数目，该属性统计反复次数的时候实用,取值为0、1、2、3、4....
private int prefix_num;//以该字串为前缀的字串数,包含该字串本身！
private Node childs[];//此处用数组实现，当然也能够map或list实现以节省空间
private boolean isLeaf;//是否为叶子节点(即,单词节点)
public Node(){
dumpli_num=0;
prefix_num=0;
isLeaf=false;
childs=new Node[26]; // 26个字母
}
}    

private Node root;//树根  
public Trie_Tree(){
// 初始化trie树
root=new Node();
}

/**
* 插入字串,用循环实现
* @param words
*/
public void insert(String words){
insert(this.root, words);
}

/**
* 插入字串，用循环实现
* @param root
* @param words
*/
private void insert(Node root,String words){
words=words.toLowerCase(); //转化为小写
char[] chrs=words.toCharArray(); //字符数组

for(int i=0,length=chrs.length; i<length; i++){
// 用相对于a字母的值作为下标索引，也隐式地记录了该字母的值
int index=chrs[i]-'a';
if(root.childs[index]!=null){
// 已经存在了，该子节点的prefix_num++
root.childs[index].prefix_num++;
}else{
// 假设不存在
root.childs[index]=new Node();
root.childs[index].prefix_num++;                
}    

// 假设到了字串结尾，则做标记
if(i==length-1){
root.childs[index].isLeaf=true;
root.childs[index].dumpli_num++;
}
// root指向子节点，继续处理
root=root.childs[index];
}
}

/**
* 遍历Trie树，查找全部的words以及出现次数
* @return HashMap<String, Integer> map
*/
public HashMap<String,Integer> getAllWords(){

 return preTraversal(this.root, "");

}

/**
* 前序遍历
* @param root    子树根节点
* @param prefixs 查询到该节点前所遍历过的前缀
* @return
*/
private  HashMap<String,Integer> preTraversal(Node root,String prefixs){
HashMap<String, Integer> map=new HashMap<String, Integer>();

if(root!=null){
if(root.isLeaf==true){
////当前即为一个单词
map.put(prefixs, root.dumpli_num);
}

for(int i=0,length=root.childs.length; i<length;i++){
if(root.childs[i]!=null){
char ch=(char)(i+'a');
// 递归调用前序遍历
String tempStr=prefixs+ch;
map.putAll(preTraversal(root.childs[i], tempStr));
   }
  }
 }
 
return map;
}
}
```

## 二叉树的遍历

```
public class Tree<AnyType extends Comparable<? super AnyType>>
{
private static class BinaryNode<AnyType>
{
BinaryNode(AnyType theElement)
{
this(theElement, null, null);
}

BinaryNode(AnyType theElement, BinaryNode<AnyType> lt, BinaryNode<AnyType> rt)
{
element = theElement;
left = lt;
right = rt;
}

AnyType element;
BinaryNode<AnyType> left;
BinaryNode<AnyType> right;
}

private BinaryNode<AnyType> root;

public void insert(AnyType x)
{
root = insert(x, root);
}

public boolean isEmpty()
{
return root == null;
}

private BinaryNode<AnyType> insert(AnyType x, BinaryNode<AnyType> t)
{
if(t == null)
{
return new BinaryNode<>(x, null, null);
}

int compareResult = x.compareTo(t.element);

if(compareResult < 0)
{
t.left = insert(x, t.left);
}
else if(compareResult > 0)
{
t.right = insert(x, t.right);
}
else
{
;
}

return t;
}

/**
* 前序遍历
* 递归
*/
public void preOrder(BinaryNode<AnyType> Node)
{
if (Node != null)
{
System.out.print(Node.element + " ");
preOrder(Node.left);
preOrder(Node.right);
}
}

/**
* 中序遍历
* 递归
*/
public void midOrder(BinaryNode<AnyType> Node)
{
if (Node != null)
{
midOrder(Node.left);
System.out.print(Node.element + " ");
midOrder(Node.right);
}
}

/**
* 后序遍历
* 递归
*/
public void posOrder(BinaryNode<AnyType> Node)
{
if (Node != null)
{
posOrder(Node.left);
posOrder(Node.right);
System.out.print(Node.element + " ");
}
}

/*
* 层序遍历
* 递归
*/
public void levelOrder(BinaryNode<AnyType> Node) {
if (Node == null) {
return;
}

int depth = depth(Node);

for (int i = 1; i <= depth; i++) {
levelOrder(Node, i);
}
}

private void levelOrder(BinaryNode<AnyType> Node, int level) {
if (Node == null || level < 1) {
return;
}

if (level == 1) {
System.out.print(Node.element + "  ");
return;
}

// 左子树
levelOrder(Node.left, level - 1);

// 右子树
levelOrder(Node.right, level - 1);
}

public int depth(BinaryNode<AnyType> Node) {
if (Node == null) {
return 0;
}

int l = depth(Node.left);
int r = depth(Node.right);
if (l > r) {
return l + 1;
} else {
return r + 1;
}
}

/**
* 前序遍历
* 非递归
*/
public void preOrder1(BinaryNode<AnyType> Node)
{
Stack<BinaryNode> stack = new Stack<>();
while(Node != null || !stack.empty())
{
while(Node != null)
{
System.out.print(Node.element + "   ");
stack.push(Node);
Node = Node.left;
}
if(!stack.empty())
{
Node = stack.pop();
Node = Node.right;
}
}
}

/**
* 中序遍历
* 非递归
*/
public void midOrder1(BinaryNode<AnyType> Node)
{
Stack<BinaryNode> stack = new Stack<>();
while(Node != null || !stack.empty())
{
while (Node != null)
{
stack.push(Node);
Node = Node.left;
}
if(!stack.empty())
{
Node = stack.pop();
System.out.print(Node.element + "   ");
Node = Node.right;
}
}
}

/**
* 后序遍历
* 非递归
*/
public void posOrder1(BinaryNode<AnyType> Node)
{
Stack<BinaryNode> stack1 = new Stack<>();
Stack<Integer> stack2 = new Stack<>();
int i = 1;
while(Node != null || !stack1.empty())
{
while (Node != null)
{
stack1.push(Node);
stack2.push(0);
Node = Node.left;
}

while(!stack1.empty() && stack2.peek() == i)
{
stack2.pop();
System.out.print(stack1.pop().element + "   ");
}

if(!stack1.empty())
{
stack2.pop();
stack2.push(1);
Node = stack1.peek();
Node = Node.right;
}
}
}

/*
* 层序遍历
* 非递归
*/
public void levelOrder1(BinaryNode<AnyType> Node) {
if (Node == null) {
return;
}

BinaryNode<AnyType> binaryNode;
Queue<BinaryNode> queue = new LinkedList<>();
queue.add(Node);

while (queue.size() != 0) {
binaryNode = queue.poll();

System.out.print(binaryNode.element + "  ");

if (binaryNode.left != null) {
queue.offer(binaryNode.left);
}
if (binaryNode.right != null) {
queue.offer(binaryNode.right);
}
}
}

public static void main( String[] args )
{
int[] input = {4, 2, 6, 1, 3, 5, 7, 8, 10};
Tree<Integer> tree = new Tree<>();
for(int i = 0; i < input.length; i++)
{
tree.insert(input[i]);
}
System.out.print("递归前序遍历 ：");
tree.preOrder(tree.root);
System.out.print("\n非递归前序遍历：");
tree.preOrder1(tree.root);
System.out.print("\n递归中序遍历 ：");
tree.midOrder(tree.root);
System.out.print("\n非递归中序遍历 ：");
tree.midOrder1(tree.root);
System.out.print("\n递归后序遍历 ：");
tree.posOrder(tree.root);
System.out.print("\n非递归后序遍历 ：");
tree.posOrder1(tree.root);
System.out.print("\n递归层序遍历：");
tree.levelOrder(tree.root);
System.out.print("\n非递归层序遍历 ：");
tree.levelOrder1(tree.root);
}
}public class Tree<AnyType extends Comparable<? super AnyType>>
{
private static class BinaryNode<AnyType>
{
BinaryNode(AnyType theElement)
{
this(theElement, null, null);
}

BinaryNode(AnyType theElement, BinaryNode<AnyType> lt, BinaryNode<AnyType> rt)
{
element = theElement;
left = lt;
right = rt;
}

AnyType element;
BinaryNode<AnyType> left;
BinaryNode<AnyType> right;
}

private BinaryNode<AnyType> root;

public void insert(AnyType x)
{
root = insert(x, root);
}

public boolean isEmpty()
{
return root == null;
}

private BinaryNode<AnyType> insert(AnyType x, BinaryNode<AnyType> t)
{
if(t == null)
{
return new BinaryNode<>(x, null, null);
}

int compareResult = x.compareTo(t.element);

if(compareResult < 0)
{
t.left = insert(x, t.left);
}
else if(compareResult > 0)
{
t.right = insert(x, t.right);
}
else
{
;
}

return t;
}

/**
* 前序遍历
* 递归
*/
public void preOrder(BinaryNode<AnyType> Node)
{
if (Node != null)
{
System.out.print(Node.element + " ");
preOrder(Node.left);
preOrder(Node.right);
}
}

/**
* 中序遍历
* 递归
*/
public void midOrder(BinaryNode<AnyType> Node)
{
if (Node != null)
{
midOrder(Node.left);
System.out.print(Node.element + " ");
midOrder(Node.right);
}
}

/**
* 后序遍历
* 递归
*/
public void posOrder(BinaryNode<AnyType> Node)
{
if (Node != null)
{
posOrder(Node.left);
posOrder(Node.right);
System.out.print(Node.element + " ");
}
}

/*
* 层序遍历
* 递归
*/
public void levelOrder(BinaryNode<AnyType> Node) {
if (Node == null) {
return;
}

int depth = depth(Node);

for (int i = 1; i <= depth; i++) {
levelOrder(Node, i);
}
}

private void levelOrder(BinaryNode<AnyType> Node, int level) {
if (Node == null || level < 1) {
return;
}

if (level == 1) {
System.out.print(Node.element + "  ");
return;
}

// 左子树
levelOrder(Node.left, level - 1);

// 右子树
levelOrder(Node.right, level - 1);
}

public int depth(BinaryNode<AnyType> Node) {
if (Node == null) {
return 0;
}

int l = depth(Node.left);
int r = depth(Node.right);
if (l > r) {
return l + 1;
} else {
return r + 1;
}
}

/**
* 前序遍历
* 非递归
*/
public void preOrder1(BinaryNode<AnyType> Node)
{
Stack<BinaryNode> stack = new Stack<>();
while(Node != null || !stack.empty())
{
while(Node != null)
{
System.out.print(Node.element + "   ");
stack.push(Node);
Node = Node.left;
}
if(!stack.empty())
{
Node = stack.pop();
Node = Node.right;
}
}
}

/**
* 中序遍历
* 非递归
*/
public void midOrder1(BinaryNode<AnyType> Node)
{
Stack<BinaryNode> stack = new Stack<>();
while(Node != null || !stack.empty())
{
while (Node != null)
{
stack.push(Node);
Node = Node.left;
}
if(!stack.empty())
{
Node = stack.pop();
System.out.print(Node.element + "   ");
Node = Node.right;
}
}
}

/**
* 后序遍历
* 非递归
*/
public void posOrder1(BinaryNode<AnyType> Node)
{
Stack<BinaryNode> stack1 = new Stack<>();
Stack<Integer> stack2 = new Stack<>();
int i = 1;
while(Node != null || !stack1.empty())
{
while (Node != null)
{
stack1.push(Node);
stack2.push(0);
Node = Node.left;
}

while(!stack1.empty() && stack2.peek() == i)
{
stack2.pop();
System.out.print(stack1.pop().element + "   ");
}

if(!stack1.empty())
{
stack2.pop();
stack2.push(1);
Node = stack1.peek();
Node = Node.right;
}
}
}

/*
* 层序遍历
* 非递归
*/
public void levelOrder1(BinaryNode<AnyType> Node) {
if (Node == null) {
return;
}

BinaryNode<AnyType> binaryNode;
Queue<BinaryNode> queue = new LinkedList<>();
queue.add(Node);

while (queue.size() != 0) {
binaryNode = queue.poll();

System.out.print(binaryNode.element + "  ");

if (binaryNode.left != null) {
queue.offer(binaryNode.left);
}
if (binaryNode.right != null) {
queue.offer(binaryNode.right);
}
}
}

public static void main( String[] args )
{
int[] input = {4, 2, 6, 1, 3, 5, 7, 8, 10};
Tree<Integer> tree = new Tree<>();
for(int i = 0; i < input.length; i++)
{
tree.insert(input[i]);
}
System.out.print("递归前序遍历 ：");
tree.preOrder(tree.root);
System.out.print("\n非递归前序遍历：");
tree.preOrder1(tree.root);
System.out.print("\n递归中序遍历 ：");
tree.midOrder(tree.root);
System.out.print("\n非递归中序遍历 ：");
tree.midOrder1(tree.root);
System.out.print("\n递归后序遍历 ：");
tree.posOrder(tree.root);
System.out.print("\n非递归后序遍历 ：");
tree.posOrder1(tree.root);
System.out.print("\n递归层序遍历：");
tree.levelOrder(tree.root);
System.out.print("\n非递归层序遍历 ：");
tree.levelOrder1(tree.root);
}
}
```













