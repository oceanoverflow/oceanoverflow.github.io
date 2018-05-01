# iobridge

  每个人小时候都或多或少经历过父母冷战吧，当他们冷战的时候，我们这些可怜的小屁孩就成了传话筒，一般事情就会发展成下面这种情况了：

  「老爸 -> 我 ：“叫你老妈安静一点”   
    然后
   我 -> 老妈 ：“老妈，老爸叫你安静一点”」
 
  「老妈 -> 我 ：“你老爸才需要安静呢”   
    然后 
   我 -> 老爸 ：“老爸，老妈说你才需要安静呢”」

  上述对话如此往复，诶，可怜天下小屁孩，要是冷战的话，老爸老妈必须通过一个中介人——我来进行语言上的交流。

  在现代网络编程中，也会出现与上述例子极其相似的情况，例如，现在有三台服务器，分别是A，B，C，其中A能与B直接通信，B能与C直接通信，但是A不能与C直接通
信，在这种情况下，A要想能与C交流，就必须要经过B，下面我们就来看如何实现上面这种情况吧。

  下面的 `iobridge` 函数就做了一件传话筒的工作，其原理非常简单，就是从我们的数据源 `src` 读取（调用 `Read` 方法）数据，然后将它写到我们的目的地
（调用 `Write` 方法）。也就是相当于我们上述过程中我从老妈那里获取消息，然后再将这个消息发给老爸的过程，注意到，这里函数中数据流的方法是单向的。

  因为在读数据的过程中经常需要开辟新内存，为了节约运行时的成本，我们通过之前写过的 `LeakyBuffer` 模型来获取内存。

```golang
func iobridge(src io.Reader, dst io.Writer, shutdown chan struct{}) {
    defer func() {
        shutdown <- struct{}
    }()
    
    for {
        buf := leakyBuffer.Get()
        n, err := src.Read(buf)
        if err != nil {
            if !(err == io.EOF || UseClosedConn(err)) {
                log.Print("error reading %s:", src, err)
            }
            break
        }
        _, err := dst.Write(buf[:n])
        if err != nil {
            log.Prinf("error writing %s: %s\n", dst, err)
            break
        }
        leakyBuffer.Put(buf)
    }
}

func UseClosedConn(err error) bool {
    operr, ok := err.(*net.OpErr)
    return ok && operr.Err.Error() == "use of closed network connection"
}
```

  因为我们的 `net.Conn` 同时定义了 `Read(b []byte) (n int, err error)` 方法和 `Write(b []byte) (n int, err error)` 方法，而这两个方法
分别由接口 `io.Reader` 和 `io.Writer` 定义，所以 `net.Conn` 既是 `io.Reader` 也是 `io.Writer` 。故上述方法中的第一个和第二个参数我们都可以
使用 `net.Conn` 做我们的参数。

```golang
// defined in package io
type Reader interface {
    Read(p []byte) (n int, err error)
}

// defined in package io
type Writer interface {
    Write(p []byte) (n int, err error)
}

// defined in package net
type Conn interface {
    Read(b []byte) (n int, err error)
    Write(b []byte) (n int, err error)
    ...
    ...
}
```

  这里的main函数应该运行在中间那台服务器也就是B上面，B作为客户端向远程服务器C发起TCP连接，并返回一个套接字conn1，同时B也作为服务器监听端口，服务来
自A的连接请求，`Accept` 成功返回另一个套接字conn2，那么如何使得来个无法直接接触的服务器A和C有来有往的进行交流对话呢，这里的技巧就是起两个
goroutine分别调用 `iobridge` 方法，使得两个 `net.Conn` conn1和conn2相互传递数据以互通有无，`shutdown` 这个通道这里算一个hack，用于阻塞
main routine 不让其马上返回。

```golang
func main() {
    conn1, err := net.Dial("tcp", "remotehost")
    if err != nil {
        log.Println("Error Dialing")
        return
    }

    l. err := net.Listen("tcp", ":2333")
    conn2, err := listener.Accept()
    if err != nil {
        log.Println("Error Accepting Connection")
        return
    }

    shutdown := make(chan struct{}, 2)

    go iobridge(conn1, conn2, shutdown)
    go iobridge(conn2, conn1, shutdown)

    <-shutdown
}
```

## 总结
  设有服务器A，B，C，除了A与C不能直接接触，其余两两都能沟通，要想A能与C进行沟通交流，解决方案就是A作为客户端调用 `net.Dial` 向B发起TCP连接，如果
连接成功，则A上会有一个 `net.Conn` sA，B上也有一个 `net.Conn` sB1，此时B也作为客户端向C发起TCP连接，B上又多了一个 `net.Conn` sB2，C上也有一
个`net.Conn` sC。注意这些 `net.Conn` 的本质都是一个套接字，可以直接当成文件描述符来进行操作。

  此时A和C要想互相交流，唯一的方法就是将sB1和sB2进行某种程度的联系，通过起两个 `iobridge` 函数，我们就将sB1读到的数据发送给sB2，也可sB2读到的数
据发送给sB1，注意，一旦sB1将读到的数据发送给sB2，sB2会马上把这个数据发送给sC，这样就达到了A与C相互交流的目的了。

  聪明的小伙伴一定注意到了，上面讲的就是代理（shadowsocks）的基本工作原理了，是不是很简单也很有趣呢。
  
