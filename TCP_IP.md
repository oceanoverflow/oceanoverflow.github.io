# TCP/IP
  网络是分布式系统之间通信的媒介，掌握网络编程自然变成了程序猿们居家旅行必备技能之一了，与直接使用应用层的 `http` 协议不同，我们一般使用传输层的协议
例如 `tcp` 进行编程，掌握好了 `tcp` 编程，我们甚至能写一套类似`http` 这样的应用层协议，是不是很兴奋呢，那今天就让我们从最简单的例子出发，掌握 
`tcp/ip` 编程的精髓吧。

  一般来说大部分网络通信模型是 `c/s` 模型，也有部分是 `p2p` 模型，今天我们先了解一下 `c/s` 模型的编程使用吧。

  客户端要想与服务器通信，前提必须是服务器已经启动并且监听了某个端口了。如果服务器没有监听而客户端直接连接的话，就会报类似下面这样的错误：

  `dial tcp 127.0.0.1:8080: connect: connection refused`
 
  也就是说，服务器端要调用 `net.Listen` 接口对特定端口进行监听，监听会返回一个 `Listener` ，服务器一旦监听到来自客户端的请求，就会调用 `Accept`
来接受该请求，`Accept` 会返回一个 `net.Conn` ，以后服务器就可以使用 `net.Conn` 与客户端进行通信。

```golang
type Listener interface {
    Accept() (Conn, error)
    Close() error
    Addr() Addr
}

type Conn interface {
    Read(b []byte) (n int, err error)
    Write(b []byte) (n int, err error)
    Close() error
    LocalAddr() Addr
    RemoteAddr() Addr
    SetDeadline(t time.Time) error
    SetReadDeadline(t time.Time) error
    SetWriteDeadline(t time.Time) error
}
```

## server.go

  一般来说，服务器需要处理来自多个客户端的请求，所以我们使用goroutine来处理不同客户端的请求，这样就达到一个并发的目的。

  服务器处理客户端请求的主要的流程如下：
  `Listen => Accept => Read & Write => Close` 。

```golang
package main 

import (
    "log"
    "net"
    "fmt"
    "bufio"
    "strings"
)

func main() {
    fmt.Println("Launching server...")

    l, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatal(err)
    }
    defer l.Close()

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Fatal(err)
        }
        go handleConn(conn)
    }
}

func handleConn(c net.Conn) {
    for {
        m, _ := bufio.NewReader(c).ReadString('\n')
        fmt.Print("Message Received:", string(m))
        newMsg := strings.ToUpper(m)
        c.Write([]byte(newMsg + "\n"))
    } 
}
```

## client.go

  一般来说连接都是由客户端主动发起的，客户端调用 `net.Dial` 向服务器发起连接行为，在正常情况下，则也返回一个 `net.Conn` ，注意到客户端和服务器都
是通过 `net.Conn` 进行通信的，linux基础好的同学，就知道它的本质就是一个套接字（socket），而套接字的本质则是一个文件，所以我们可以对待正常文件一样
去使用它，例如使用 `fmt.Fprintf` 来对套接字写入数据。

  主要的流程如下： 
  `Dial => Read & Write => Close` 。

```golang
package main

import (
    "os"
    "log"
    "net"
    "fmt"
    "bufio"
)

func main() {
    c, e := net.Dial("tcp", "localhost:8080")
    if e != nil {
        log.Fatal(e)
    }
    reader := bufio.NewReader(os.Stdin)
    for {
        fmt.Print("Text to send: ")
        text, err := reader.ReadString('\n')
        if err != nil {
            log.Fatal(err)
        }
        fmt.Fprintf(c, text + "\n")
        m, e := bufio.NewReader(c).ReadString('\n')
        if e != nil {
            log.Fatal(e)
        }
        fmt.Print("Message from server: " + m)
    }
}
```

## 运行
  先对两个源文件进行编译。
```
$ go build *.go
```

  在一个命令行中输入：

```
$ ./server
```

  在另一个命令行中输入：

```
$ ./client
```
