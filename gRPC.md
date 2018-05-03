# gRPC
  
  首先，在了解 `gRPC` 之前，要搞清楚什么是 `RPC` ，`RPC` ， `Remote Procedure Call` ，即远程过程调度，它采用 `C/S`模型，本质就是客户端通过网
络向远程计算机程序也就是服务端发送请求服务，远程服务器执行后向客户端返回答复，`RPC` 是分布式系统中非常重要的一个组成部分。

  `RPC` 和本地方法调用最大的区别就在于程序是否在一个地址空间内执行，那么我们为什么要用到 `RPC` 呢，例如我们要写一个分布式爬虫的框架，因为自己的笔记
本电脑处理能力有限，为了提高爬虫效率，我们租借了几台 `8C16G` 的高性能服务器帮我们完成任务，这样就可以由客户端发送爬虫指令，具体的粗活累活可以由服务器
代替完成，如下面 `client` 向服务器发送一个 `crawl request` ，里面包含了我们要爬取的网站的具体信息，在服务器完成任务之后，向 `client` 返回一个
`crawl response`，也就是爬虫的结果。

![grpc](https://camo.githubusercontent.com/e4deb67aa41f71d46774f192b05b75be5c3da112/68747470733a2f2f7261772e6769746875622e636f6d2f6875696368656e2f7a6572672f6d61737465722f646f632f7a6572672e706e67)

  关于 `RPC` ，不同的语言有不同的解决方案，例如在golang的标准库中就提供了 `net/rpc` 这个解决方案，虽然这个方法效率很高，但是它的缺点也是比较致命
的，就是它仅能在golang程序中使用，而我们今天的主角，`gRPC` 就是一个跨语言平台的通用高性能 `RPC` 库，听着是不是很酷炫，下面就让我们来仔细学习一下吧
。

## 利用gRPC定义我们的服务

  在使用 `gRPC` 之前我们需要安装它，在命令行中输入以下命令。

`$ go get -u google.golang.org/grpc`

  `gRPC` 在定义服务时采用了 `protobuf` 的语法格式，我们需要定义 `gRPC` 中请求消息和答复消息的数据格式，这个我们应该已经非常熟悉了，就是定义两个
`message` （详情可以参考 `protobuf` 的官方文档），`CrawlRequest` 和 `CrawlResponse` ，除此之外我们还要定义一个 `service` ，说白了也就是我
们要定义的 `RPC` 方法。也就是说我们向服务器发送一个 `CrawlRequest` 消息，服务器处理完成后会向我们返回一个 `CrawlResponse` 。

```protobuf
service Crawl {
    rpc Crawl (CrawlRequest) returns (CrawlResponse) {}
}

message CrawlRequest {
    string url = 1;
    int64 timeout = 2;
}

message CrawlResponse {
    string content = 1;
}
```

  使用 `gRPC` 的好处就在于它可以根据我们定义的服务自动生成相应的服务端和客户端的代码，也就是说我们只要关心具体调用服务的逻辑，剩下具体的实现由框架帮
我们干就可以了。

  由于是 `protobuf` 文件，我们使用 `protoc` 来编译它就可以了。

`$ protoc --go_out=plugins=grpc:. *.proto`

## gRPC 原理

  `gRPC` 相当于给我们一个框架，具体的代码就在我们的 `crawl.proto` 这个文件经过 `protoc` 编译后的得到的 `crawl.pb.go` 这个文件中，下面的代码
都是由 `protoc` 经过编译后自动生成的，编译后的文件都具有类似的格式，基本看一个例子就可以一通百通了。

  编译后得到的代码在客户端部分定义了一个大写的 `CrawlClient` 这个interface，而小写的 `crawlClient` 结构体则具体实现了它，注意到一个
`crawlClient` 中包含了一个`grpc.ClientConn` 结构体，与 `net.Conn` 类似，它是真正发送网络请求的结构体，`grpc.Invoke` 则才是具体发送网络请求
的行为。

```golang
type CrawlClient interface {
    Crawl(ctx context.Context, in *CrawlRequest, opts ...grpc.CallOption) (*CrawlResponse, error)
}

type crawlClient struct {
    cc *grpc.ClientConn
}

type NewCrawlClient(cc *grpc.ClientConn) CrawlClient {
    return &crawlClient{cc}
}

func (c *crawlClient) Crawl(ctx context.Context, 
                            in *CrawlRequest, 
                            opts ...grpc.CallOption) (*CrawlResponse, error) {
    out := new(CrawlResponse)
    err := grpc.Invoke(ctx, "/protos.Crawl/Crawl", in, out, c.cc, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}
```

  与 `CrawClient` 不同的是，并没有结构体直接实现 `CrawlServer` 这个接口，毕竟 `gRPC` 仅仅是一个框架，具体的逻辑代码还是需要我们自己去填充的。
`RegisterCrawlServer` 方法则向 `gRPC` 注册该服务，注册的过程也需要我们自己在代码中调用。

```golang
type CrawlServer interface {
    Crawl(context.Context, *CrawlRequest) (*CrawlResponse, error)
}

func RegisterCrawlServer(s *grpc.Server, srv CrawlServer) {
    s.RegisterService(&_Crawl_serviceDesc, srv)
}

func _Crawl_Crawl_Handler(srv interface{}, 
                          ctx context.Context, 
                          dec func(interface{}) error, 
                          interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
    in := new(CrawlRequest)
    if err := dec(in); err != nil {
        return nil, err
    }
    if interceptor == nil {
        return srv.(CrawlServer).Crawl(ctx, in)
    }
    info := &grpc.UnaryServerInfo{
        Server:     srv,
        FullMethod: "/protos.Crawl/Crawl",
    }
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {
        return srv.(CrawlServer).Crawl(ctx, req.(*CrawlRequest))
    }
    return interceptor(ctx, in, info, handler)
}

var _Crawl_serviceDesc = grpc.ServiceDesc{
    ServiceName: "protos.Crawl",
    HandlerType: (*CrawlServer)(nil),
    Methods: []grpc.MethodDesc{
        {
            MethodName: "Crawl",
            Handler:    _Crawl_Crawl_Handler,
        },
    },
    Streams:  []grpc.StreamDesc{},
    Metadata: "crawl.proto",
}
```

## Server
  
  下面需要我们自己实现具体的业务逻辑（这里就是爬虫的业务逻辑），毕竟 `gRPC` 可不能猜到我们究竟要干什么事情，我们先定义一个包含 `http.Client` 的结
构体 `crawlServer` ，当然了，`crawlServer` 究竟要包含什么要看具体的业务场景了，这里因为是爬虫所以要包含 `http.Client` 用于发送http请求。

所以我们在 `Crawl` 方法中实现具体的爬虫逻辑。

```golang
type crawlServer struct{
    client *http.Client
}

func (s *crawlServer) Crawl(ctx context.Context, in *pb.CrawlRequest) (*pb.CrawlResponse, error) {
    return s.internalCrawl(in)
}

func (s *crawlServer) internalCrawl(in *pb.CrawlRequest) (*pb.CrawlResponse, error) {
    // .... 具体的爬虫逻辑
}
```

  与普通的 `tcp` 网络编程类似的，我们需要监听特定的端口，当这个端口监听到来自客户端的连接请求时，`gRPC` 会接受该连接请求，并且新起一个 `goroutine`
去处理该请求，然后读取该请求的消息类型，`gRPC` 内部会去检查这个消息的类型，最后决定调用哪个 handler（也就是之前Register过的）去处理该请求，具体处
理后向客户端返回答复。

```golang
listener, err := net.Listen("tcp", *address)
if err != nil {
    log.Fatal(err)
}
s:= grpc.NewServer()
pb.RegisterCrawlServer(s, &crawlServer{})
s.Serve(listener)
```

## Client

  在客户端这里，我们需要一个类似 `net.Conn` 这样的结构体用于发送和接受消息，在 `gRPC` 中，我们可以通过类似 `net.Dial` 的方法 `grpc.Dial` 来返
回一个 `grpc.ClientConn` 结构体，然后通过该结构体新建一个客户端 `pb.NewCrawlClient` ，最后用这个客户端发送我们的请求就可以了，在很多地方，客户
端也被叫做 `stub`， 因为它仅仅有方法调用的接口，而真正实现业务逻辑的方法还是在服务端上面。

```golang
conn, err := grpc.Dial(*address, grpc.WithInsecure())
if err != nil {
    log.Fatal(err)
}
client := pb.NewCrawlClient(conn)

request := pb.CrawlRequest{
    Url:     *url,
    Timeout: 10000,
}
response, err := client.Crawl(context.Background(), &request)
if err != nil {
    log.Fatal(err)
}
```

## 总结
  
  在使用 `gRPC` 时，我们需要明确自己的责任与分工，什么事情应该我们自己干，而什么事情又是框架帮我们干好而我们不需要重复去做的。其实说到底，我们使用
`RPC` 的目的就是让任务在远程执行后返回处理结果，具体的任务该怎么执行 `gRPC` 肯定是不知道的，所以相关的业务逻辑应该在服务端进行定义，毕竟活是服务端干
的（如上面结构体 `crawlServer` 实现的 `Crawl` 方法），我们只需要定义相关的 `Request` 和 `Response` 消息，再定义一个 `service` 服务，用
`protoc` 编译生成相应的代码，最后再定义服务端上面的处理逻辑就可以了。

