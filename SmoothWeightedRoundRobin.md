# Smooth Weighted Round Robin

  我们在之前的文章中就讲过负载均衡，但是那篇文章中的例子比较简单，仅仅是利用随机算法来进行负载均衡，并没有考虑服务器的实际情况，在很多情况下，服务器的性
能可能存在显著差异，这时候如果采用随机算法，那么性能较差的服务器会承受过多的压力，而性能较好的服务器则大部分时间处于空闲状态，系统整体的吞吐量就无法上来。

  例如服务器的性能比例如下：`{a:3, b: 2, c:1}` ，一般来说可以使用加权的轮询算法（ `Weighted Round Robin` ）进行负载均衡，但是普通的加权轮询会得
出类似这样的序列， `a a a b b c` ，这样的序列有一个问题，就是它并不够平滑，我们更希望得到像这样的序列 `a b a c b a` 。也就是说一般的轮询调度在调度
的过程中并没有考虑瞬时的情况，所以会导致对某一台服务器的瞬时访问。

  下面我们就来介绍一下平滑版本的加权轮询算法，来解决上述瞬时访问可能出现的问题。我们对每个节点赋予两个权重， `Weight` ，这是一个较为固定的值，就是前面
给出的 `{a:3, b: 2, c:1}` ，还需要一个 `CurrentWeight` 来记录当前的权重，初始时均为0，这个权重是在每次选择节点时动态变化的。

  例如现在有三台服务器，其权重比值为 `{a:3, b: 2, c:1}` ，刚开始时它们的 `CurrentWeight` 均为0，按照下面的算法，每一回合中各个节点的
`CurrentWeight` 都会加上它相应的的 `Weight` ，`total` 变量则用于记录每一个节点增加的 `Weight` 的总和，并记录当前权重最高的那一个节点，也就是我们的 `best` ，在一轮计算之后，再让 `best` 节点的 `CurrentWeight` 减去 `total` 值。

  也就是说开始时每台服务器的 `CurrentWeight` 为 `{a:0, b: 0, c:0}` ，在选择节点的过程中，每个节点均会加上自己对应的 `Weight` ，这样就变成
`{a:3, b: 2, c:1}` ， 此时 `best=a` ， 最后 a 对应的 `CurrentWeight` 要减去 `total` ，也就是 `{a:-3, b: 2, c:1}`  。

  然后 `{a:-3, b: 2, c:1}` 作为下一轮挑选节点时的初始状态，以此类推，最后可以得到下面这样的序列  `a b a c b a` ，这样就不会导致对服务器 `a` 集中
访问的情况，也就是说访问序列更加平滑了。

  整个周期计算的流程如下所示。

```
a   b   c
0   0   0

3   2   1   Next() : a
-3  2   1

0   4   2   Next() : b
0  -2   2

3   0   3   Next() : a
-3  0   3

0   2   4   Next() : c
0   2   -2 

3   4   -1  Next() : b
3  -2   -1

6   0   0   Next() : a
0   0   0

```

## 普通版本平滑轮询调度算法

```golang
type Weighted struct {
    Server string
    Weight int
    CurrentWeight int
}
```

  普通版本的算法，每次选择结点的时候，我们为每个候选结点的 `CurrentWeight` 增加 `Weight` ，选取 `CurrentWeight` 最大的节点作为我们最后返回的结果
，然后将该节点减去所有节点增加的 `Weight` 之和。 

```golang
func Next(servers []*Weighted) (best *Weight) {
    total := 0 
    for i := 0; i < len(servers); i++ {
        w := servers[i]
        
        if w == nil {
            continue
        }

        w.CurrentWeight += w.Weight
        total += w.CurrentWeight 
        if best == nil || w.CurrentWeight > best.CurrentWeight {
            best = w
        }
    }
    if best == nil {
        return nil
    }
    best.CurrentWeight -= total
    return best
} 
```

## 考虑节点失效情况时的算法

  上面那种算法不能很好的应对节点失效的情况，为了解决节点失效时的这种突发情况，我们多引入一个权重也就是 `EffectiveWeight` ，该权重初始时与 `Weight`
的值相同，例如上述的例子中都是  `{a:3, b: 2, c:1}` ，但是 `EffectiveWeight` 会在节点失败的时候清0，当节点恢复正常时再慢慢递增直到达到原本的
`Weight` ，这样就能很好地考虑了节点失效再重启时的真实情况。因为引入了 `EffectiveWeight` ， 我们在相应计算的时候就不直接采用 `Weight` 而是直接使用
`EffectiveWeight` 来代替计算。 

```golang
type Weighted struct {
    Server string
    Weight int
    CurrentWeight int
    EffectiveWeight int
}
```

```golang
func (w *Weighted) fail() {
    w.EffetiveWeight -= w.Weight
    if w.EffectiveWeight < 0 {
        w.EffectiveWeight = 0
    }
}
```

```golang
func Next(servers []*Weighted) (best *Weight) {
    total := 0
    for i := 0; i < len(servers); i++ {
        w := servers[i]

        if w == nil {
            continue
        }

        w.CurrentWeight += w.EffectiveWeight
        total += w.EffectiveWeight
        if w.EffectiveWeight < w.Weight {
            w.EffectiveWeight++
        }
        if best == nil || w.CurrentWeight > best.CurrentWeight {
            best = w
        }
    }
    if best == nil {
        return nil
    }
    best.CurrentWeight -= total
    return best
}
```
