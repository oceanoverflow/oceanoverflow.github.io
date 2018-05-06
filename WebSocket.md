# WebSocket

  如果服务器会定期地更新消息，但是具体的时间间隔并不确定，客户端为了能够及时拿取最新的消息，就得不断的发送 `HTTP` 请求来轮询服务器，也就是说为了能够
及时获取最新资讯，由于 `HTTP` 本身的特点，服务请求的发起方只能是客户端，服务器本身并没有消息推送的功能，这样就导致了资源浪费的问题，很多客户端发送的
请求都是无效的，当客户端和服务器一对一交流时倒还好，如果很多客户端向服务器以这样的形式咨询，服务器将会承受非常大的压力了。

  所以，为了避免资源浪费，我们来引入 `WebSocket` 解决上述问题，`WebSocket` 可以说是 `HTTP` 协议的一个升级，它同样基于 `TCP` 之上，但与传输层的
`socket` 不同，它的 `socket` 是应用层的概念。要想使用 `WebSocket` ，客户端和服务器必须同时支持才可以使用。

  `WebSocket` 中比较重要的一个阶段就是握手阶段了，如下图所示，客户端向服务器发送一个协议升级的 `HTTP` 请求，并且附带上一些其他的元信息用于握手。

![WebSocket](http://www.cuelogic.com/blog/wp-content/uploads/2013/11/websocketlifecycle_30624_l.png)

```
Request headers from client

GET /mychat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

  服务器收到该协议升级的 `HTTP` 请求后，会返回一个 `HTTP` 答复，向客户端确认同意建立 `WebSocket` 连接，从此之后，它们之间就不再用 `HTTP` 的形式
交流了，而是直接改用 `WebSocket` 格式的报文进行交流，服务器也具备了相应推送消息的能力了。

```
Response headers from server

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

  下面我们来看一下使用 `"github.com/gorilla/websocket"` 时各个步骤对应的代码吧。

  客户端代码的风格和使用 `TCP` 进行编程非常类似，第一步，客户端向服务器发送协议升级请求，对应
`websocket.DefaultDialer.Dial(u.string(), nil)` 。

```golang
var addr = flag.String("addr", "localhost:8080", "http service addr")
u := url.URL{Scheme: "ws", Host: *addr, Path: "/echo"}
log.Printf("connecting to %s", u.String())

c, _, err := websocket.DefaultDialer.Dial(u.string(), nil)
if err != nil {
    log.Fatal("dial:", err)
} 
defer c.Close()
```

  在服务端处理 `WebSocket` 请求与处理 `HTTP` 请求类似，我们同样需要定义一个 `handler` ，但与普通处理 `HTTP` 请求不同的是，此 `handler` 会对
协议进行升级，也就是 `upgrader.Upgrade(w, r, nil)` ，这段代码相当于服务端向客户端发送一个协议升级的确认消息。在这之后，服务器就可以一直读取客户
端发来的请求和向客户端返回答复了（代码的逻辑在 `for` 循环之中），也就是说服务器只起了一个 `handler` 来处理这段 `session` 。

```golang
func echo(w http.ResponseWriter, r *http.Request) {
    c, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Print("upgrade:", err)
        return
    }
    defer c.Close()
    for {
        mt, message, err := c.ReadMessage()
        if err != nil {
            log.Println("read:", err)
            break
        }
        log.Printf("recv: %s", message)
        err = c.WriteMessage(mt, message)
        if err != nil {
            log.Println("write:", err)
            break
        }
    }
}

func main() {
    http.HandleFunc("echo", echo)
    log.Fatal(http.ListenAndServe(*addr, nil))
}
```

  客户端在握手之后就可以读取和发送数据，在这里，由于在 `WebSocket` 中服务器可以主动向客户端推送数据，所以我们起一个 `goroutine` 专门用于接受服务
器主动推送的数据（ `c.ReadMessage()` ）。

```golang
done := make(chan struct{})

go func() {
    defer close(done)
    for {
        _, message, err := c.ReadMessage()
        if err != nil {
            log.Println("read:", err)
            return
        }
        log.Printf("recv:", message)
    }
}
```

  下面是客户端发送数据的逻辑，这里我们每隔一秒钟向服务器发送当前的时间，`c.WriteMessage(websocket.TextMessage, []byte(t.string()))` ，这里
我们注意到，`WebSocket` 中的消息是有类型的，这个函数的第一个参数就是指明该消息的类型，这里我们使用 `websocket.TextMessage` 。

  那么如何让这段 `session` 结束呢，结束会话一般来说都是客户端主动发起的，客户端主要发送一个结束对话的消息类型就可以了，
`c.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))` ，这里注意该结束对话的消
息类型为 `websocket.CloseMessage` 。
 
```golang
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

interrupt := make(chan os.Signal, 1)
signal.Notify(interrupt, os.Interrupt)

for {
    select {
    case <-done:
        return
    case t := <-ticker.C:
        err := c.WriteMessage(websocket.TextMessage, []byte(t.String()))
        if err != nil {
            log.Println("write:", err)
            return
        }
    case <-interrupt:
        log.Println("interrupt")
        
        err := c.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
        if err != nil {
            log.Println("write close:", err)
            return
        }
        select {
        case <-done:
        case <-time.After(time.Second)
        }
        return
    }
}
```
