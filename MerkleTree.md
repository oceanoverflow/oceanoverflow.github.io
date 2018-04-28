# Merkle Tree
  `Merkle Tree` 在各个领域都有着广泛的应用，就拿我们熟悉的 `bittorrent` 来举例，在我们使用 `bittorrent` 下载的时候，客户端会向许多并不信任的
节点去下载目标文件的某个片段，利用 `Merkle proofs` 我们就可以验证我们下载的部分片段是否属于整个文件的一部分。`Merkle proofs` 可以用于验证某个
数据子集是否从属于更大的集合。

  那么下面我们来看一下如何实现一个 `Merkle Tree` 吧。与传统的二叉树类似，每个 `Merkle Tree` 中由许多节点组成，每个节点都指向它的左孩子节点，右
孩子节点和父节点，同时，还有一个flag指明它是否为叶子结点等其他元信息。`Content` 是每个节点所存储的内容，只要实现了该接口 `Content` 的结构体都能
被节点所存储。

## 数据结构

```golang
type Node struct {
    Parent *Node
    Left   *Node
    Right  *Node
    leaf   bool
    dup    bool
    Hash   []byte
    C      Content
}

type Content interface {
    CalculateHash() []byte
    Equals(other Content) bool
}

type MerkleTree struct {
    Root       *Node
    merkleRoot []byte
    Leafs      []*Node
}
```

  在 `Merkle Tree` 当中，有一个非常重要的特性就是获取节点的哈希值，我们来定义一个helper function 用于计算节点的哈希值。计算哈希分为两种情况，
第一种就是如果当前被计算的节点为叶子结点，那么我们调用interface `Content` 中的 `CalculateHash` 方法来计算所存储内容的哈希值，如果不是叶子结点，
则说明是中间层的节点，那么就计算左子节点和右子节点的哈希值的拼接值的哈希值，即 `H(P) = H( H(P.L) + H(P.R) )`。

```golang
func (n *Node) calculateNodeHash() []byte {
    if n.leaf {
        return n.C.CalculateHash()
    }
    h := sha256.New()
    h.Write(append(n.Left.Hash, n.Right.Hash...))
    return h.Sum(nil)
}
```

## 新建一棵树

```golang
func NewTree(cs []Content) (*MerkleTree, error) {
    root, leafs, err := buildWithContent(cs)
    if err != nil {
        return nil, err
    }
    mk := &MerkleTree{
        Root: root,
        merkleRoot: root.Hash,
        Leafs: leafs
    } 
    return mk, nil
}
```


```golang
func buildWithContent(cs []Content) (*Node, []*Node, error) {
    if len(cs) == 0 {
        return nil, nil, errors.New("Error: cannot construct tree with no content")
    }    
    var leafs []*Node
    for _, c := range cs {
        leafs = append(leafs, &Node{
            Hash: c.CalculateHash(),
            C: c,
            leaf: true,
        })
    }
    if len(cs) % 2 == 1 {
        duplicate := &Node{
            Hash: leafs[len(leafs)-1].Hash,
            C: leafs[len(leafs)-1].C,
            leaf: true,
            dup: true,
        }
        leafs = append(leafs, duplicate)
    }
    root := buildIntermediate(leafs)
    return root, leafs, nil
}
```

  `buildIntermediate` 函数利用递归的方法建立树，递归在树的建立中是一种非常常用的套路，只需要特别关注一下它的退出条件就可以了，这里的退出条件就是
最后仅剩两个节点了，说明已经可以返回根节点，那么函数执行到这里就可以返回了。

```golang
func buildIntermediate(nl []*Node) *Node {
    var nodes []*Node
    for i := 0; i < len(nl); i += 2 {
        h := sha256.New()
        var l, r int = i, i+1
        if r == len(nl) {
            r = len(nl)-1
        }
        h.Write(nl[l].Hash, nl[r].Hash...)
        n := &Node{
            Left:  nl[l],
            Right: nl[r],
            Hash:  h.Sum(nil),
        }
        nodes = append(nodes, n)
        nl[l].Parent = n
        nl[r].Parent = n
        if len(nl) == 2 {
            return n
        }
    }
    return buildIntermediate(nodes)
}
```

  只有当叶子结点的个数为 `2^n` 的时候，才能保证每一层的中节点都能两两配对，但一般来说叶子结点的个数不一定恰好为 `2^n` ，在这种情况下，就可以将每一
层的最右边的结点进行自我复制，自己与自己配对然后再计算哈希值，上面的算法就采用了这样的解决方法。

```
     ┌───┴──┐       ┌────┴───┐         ┌─────┴─────┐          ┌─────┴─────┐
  ┌──┴──┐   │    ┌──┴──┐     │      ┌──┴──┐     ┌──┴──┐    ┌──┴──┐     ┌──┴──┐
┌─┴─┐ ┌─┴─┐ │  ┌─┴─┐ ┌─┴─┐ ┌─┴─┐  ┌─┴─┐ ┌─┴─┐ ┌─┴─┐   │  ┌─┴─┐ ┌─┴─┐ ┌─┴─┐ ┌─┴─┐ 
   (5-leaf)         (6-leaf)             (7-leaf)                (8-leaf)
```

## 验证内容是否存在在 Merkle Tree 中

  那么，我们又如何验证某个部分的内容是否真正存在于我们的 `MerkleTree` 中呢。具体的思路非常简单，就是遍历该树的叶子结点，看有没有与我们要验证的内容
相一致的，如果没有则直接返回false，如果有则开始真正的验证阶段，验证的基本思路也不难，就是一个从叶子节点往根节点走并且验证这条路上的哈希值是否都正确的
过程，先获取当前遍历的节点的父节点，然后判断父节点的左子节点和右子节点的哈希值是否相匹配，如果不匹配，则直接返回false，如果匹配，则继续验证知道根节点
为止。

```golang
func (m *MerkleTree) VerifyContent(expectedMerkleRoot []byte, content Content) bool {
    for _, l := range m.Leafs {
        if l.C.Content.Equals(content) {
            currentParent := l.Parent
            for currentParent != nil {
                h := sha256.New()
               h.Write(append(currentParent.Left.calculateNodeHash(), currentParent.Right.calculateNodeHash()...))  
                if bytes.Compare(h.Sum(nil), currentParent.Hash) != 0 {
                    return false
                } 
                currentParent = currentParent.Parent
            }
            return true
        }
    }
    return false
}
```



