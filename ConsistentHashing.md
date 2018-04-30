# Consistent Hashing
  
  比如说我们现在有3台服务器用于存储kv值，分别是 `{s1, s2, s3}` ，而我们用的哈希策略非常简单，例如 `h(x) = ( x^2 + 3 * x + 2 ) % 3` ，也就是
说当k=1时，对应的v值会被存放在s1上，但如果业务量增加，我们又需要增加一个服务器来均衡负载，此时哈希策略就变成 `h(x) = ( x^2 + 3 * x + 2 ) % 4`
，那按照这种策略，当k=1的时候，应该是把对应的v值存放在s3上面，然而s3上面并没有存放，为了能够获取到k=1对应的v值，我们就需要将原来三台服务器上的数据
做一个 `remapping` ，也就是说基本上所有的数据都要进行迁移，在小数据量的情况下这倒没什么问题，但是一旦数据量达到T级别，这种开销将是无法忍受的，所以，
为了当增删节点时尽量少的数据迁移，我们可以利用一致性哈希这种非常神奇的算法。

  所谓一致性哈希，就是当相应的哈希表调整大小的时候（对应到现实生活中就是服务器增删节点），只有 `K/n` 个数据需要迁移，这里的 `K` 是指键的数量，而 `n`
则是指所有的卡槽（对应服务器的数量），如果不用一致性哈希而是用传统的哈希表，当哈希表调整的时候，基本所有的键都要进行 `remapping`。

  一致性哈希有几个要点，第一个就是它所有的地址空间会形成一个环，例如它的地址空间是从 `[0, 2^32-1)` ，它的头部和尾部会进行拼接而形成一个环，还有就
是虚节点的概念，例如下图中我们有`{s1, s2, s3, s4}`这四个节点，但每个节点都会有自己的拷贝（称为虚节点），然后这些复制过的节点群会根据特定的哈希算法
分布在这个环上面，一般来说都是均匀分布的，而我们要存储的键值对，同样根据与节点相同的哈希算法，算出一个值后，顺时针（或者逆时针）去寻找与自己最近的虚节
点的位置，找到最近的虚节点后，就将自己存放在对应的节点上，这就是实现一致性哈希的基本思路，下面就让我们来看一下一致性哈希算法的实现吧。

![alt text](https://dajh2p2mfq4ra.cloudfront.net/assets/site-images/system_design/consistent_hashing_virtual.jpg)

## 一致性哈希的实现

  上面我们知道了一致性哈希的核心是一个环，环的地址空间假设是 `[0, 2^32-1)` ，我们可以用一个map记录虚节点对应的环上的位置，位置我们用一个uint32表
示就可以了，因为是遍历map的时候是随机的，所以我们还需要一个数组来记录环上面虚节点的顺序。

```golang
type uints []uint32

func (x uints) Len() int { return len(x) }
func (x uints) Less(i, j int) bool { return x[i] < x[j] }
func (x uints) Swap(i, j int) { x[i], x[j] = x[j], x[i] }

type Consistent struct {
    circle map[uint32]string
    sortedList uints
    virtualNode int
    sync.Mutex
}

func New() *Consistent {
    return &Consistent{
        circle: make(map[uint32]string),
        sortedList: make(uints, 0)
        virtualNode: 20,
    }
}
```

  虚节点相当于实节点的影分身，例如s2，s2复制为s2-1，s2-2，s2-3。这些原来的节点和复制出来的节点我们可以把它们形象地称为哨兵。

```golang
func (c *Consistent) KeyNum(key string, num int) string {
    return key + ":" + fmt.Sprintf("%d", num)
}
```

  要想把虚节点对应到一致性哈希的圆环上，我们还需要一个哈希函数用于把我们的哨兵和我们的键值对映射到环上面去。

```golang
func (c *Consistent) hashKey(key string uint32) {
    if len(key) < 64 {
        scratch := make([]byte, 64)
        copy(scratch[:], key)
        return crc32.ChecksumIEEE(scratch[:len(key)])
    }
    return crc32.ChecksumIEEE([]byte(key))
}
```

  之前说了，我们加入的键值对经过哈希后会在环上顺时针寻找离自己最近的虚节点的位置，因为我们之前已经用 `sortedList` 记录了节点在环上的相对顺序了，
所以就可以利用二分法去查找对应虚节点的位置啦。

```golang
func (c *Consistent) search(key uint32) int {
    i := sort.Search(len(c.sortedList), func(x int) bool {
        return c.sortedList[x] > key
    })
    if i >= len(c.sortedList) {
        i = 0
    } 
    return i
}
```

  当然了，我们一致性哈希就牛逼的地方就在于当动态增删节点的时候，需要`remapping` 的键值对的数量比较少，动态增删节点无非也就是将该节点影分身后再经过
哈希操作对应到环上或者进行相应的删除操作，当然还需要对我们的 `sortedList` 重新进行排序来对应增删的虚节点。

```golang
func (c *Consistent) Add(key string) {
    c.Lock()
    defer c.Unlock()

    for n := 0; n < virtualNode; n++ {
        c.circle[c.hashKey(c.KeyNum(key, n))] = key
    }

    c.updateList()
}

func (c *Consistent) Remove(key string) {
    c.Lock()
    defer c.Unlock()
 
    for n := 0; n < c.virtualNode; n++ {
        delete(c.circle, c.hashKey(c.KeyNum(key, n)))
    }

    c.updateList()
}

func (c *Consistent) updateList() {
    list := c.sortedList[:0]
    for k, _ := range c.circle {
        list = append(list, k)
    } 
    sort.Sort(list)
    c.sortedList = list
}
```

  最后就是当我们新增一个键值对时，我们可以查找对应的节点。

```golang
func (c *Consistent) Get(name string) (string, error) {
    c.Lock()
    defer c.Unlock()

    if len(c.circle) == 0 {
        return 0, errors.New("empty consistent circle")
    }
    key := c.hashKey(name)
    i := c.search(key)
    return c.circle[c.sortedList[i]], nil
}
```

