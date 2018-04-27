# Leaky Buffer

  在科学上网服务中，客户端不断地向shadowsocks服务器发送数据，服务器从网络中读取经过加密的数据然后进行解密操作，为了避免重复的分配内存和释放内存操作，
我们引入 `LeakyBuffer` 这种编程模型，`LeakyBuffer` 通常也被叫做 `LeakyBucket` ，或者说是漏桶模型。`LeakyBuffer` 维护了固定个数个字节数组
（free list），如果我们的程序需要使用内存， 那么它会从 `LeakyBuffer` 维护的 `freelist` 去拿，如果 `freelist` 中没有了，则再重新make一个，
当程序使用完了这段内存，不像通常那么被golang的runtime进行垃圾回收操作，我们把它重新放回 `LeakyBuffer` 维护的 `freelist` 中。

  下面我们来看一下 `Leaky Buffer` 的实现吧。

```golang
const leakyBufferSize = 4096
const maxNum = 2048

type LeakyBuffer struct {
    bufferSize  int
    freeList chan []byte
}
```

  `LeakyBuffer` 的实现非常简单，它主要维护了两个变量，一就是每个 `buffer`  的大小，而就是一个 `freelist` ，真正用于维护我们的内存。

  在上面的代码中我们定义每个 `buffer` 的大小是4096，这里的单位是字节，`freelist` 的长度是2048，所以我们这个 `LeakyBuffer` 总共可以使用的内存
大小就是 4096 * 2048 B 也就是 8MB ，这对于一般的网络应用已经足够了。

```golang
var leakyBuf = NewLeakyBuffer(maxNum, leakyBufferSize)

func NewLeakyBuffer(n, bufSize int) *LeakyBuffer {
    return &LeakyBuffer{
        bufferSize: bufSize,
        freeList: make(chan []byte, n)
    }
}
```

  `NewLeakyBuffer` 返回一个 `LeakyBuffer` 指针，它维护的 `freelist`  就是一个容量为n的通道。

  下面我们来看看如何使用 `LeakyBuffer` ，`LeakyBuffer` 只定义了两种操作，存和取。

## 取操作
  `Get` 操作从 `LeakyBuffer` 所维护的 `freelist` 的通道中取出一个字节数组，如果 `freelist` 为空，则说明没有多余的空闲字节数组，则自己make一个字节数组然后返回。

```golang
func (lb *LeakyBuffer) Get() (b []byte) {
    select {
    case b := <- lb.freeList:
    default:
        b := make([]byte, lb.bufferSize)
    }
    return
}
```


## 存操作
  当程序使用完了这段内存空间后，我们将它重新放回 `LeakyBuffer` 中进行回收而不是用 `golang runtime` 进行垃圾回收，注意一种特殊情况，就是当
`freelist` 已经满了，这时我们会执行default case， 也就是什么事情也不干，然后这个无家可归的字节数组就会自动被系统进行垃圾回收，这就是
`LeakyBuffer` 这个名字中 Leaky 的由来，也就是有可能会泄漏出去。

```golang
func (lb *LeakyBuffer) Put(b []byte) {
     if lb.bufferSize != len(b) {
         panic("invalid buffer size that's put into leaky buffer")
     }
     select {
     case lb.freeList <- b:
     default:
     }
     return
}
```

  上述就是 `LeakyBuffer` 的实现了，如果下次你遇到需要重复分配很多相同大小的内存空间时，不妨可以使用一下这个编程模型。

## Golang 标准库的实现
  在 golang 标准库中也实现了类似于 `LeakyBuffer` 的接口，`sync.Pool` ，有兴趣的读者不妨阅读一下它的源码，这里就不再重复介绍了。
