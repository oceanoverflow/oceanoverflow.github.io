# Decoding Dubbo Protocol
  
  之前看 `shadowsocks` 源码的时候看过好多遍 `SOCKS5` 的 `RFC` ，看的一脸懵逼，不知道这些奇奇怪怪的网络协议究竟该如何理解，后来才发现网络协议也并
没有那么神秘，无非就是为了让两台计算机使用网络进行通信时能互相理解对方而设定的一系列规则而已。就好像一个中国人和一个法国人聊天，法国人讲法语，而中国人
用中文，他们自然很难理解对方究竟要表达什么意思，但是如果二者约定同时用两者都会的语言——英语进行交流，那么两者就可以正常交流下去了，因为两者协议了一套规
则，可以相互理解的规则（这里的规则就是英语）。

  一般来说要是想自己定义一套网络协议，首先需要做的事情就是定义这个协议中特定位置的字段分别代表什么含义，例如下图中的 `Dubbo` 协议，它就规定了一个报文
中各个字段代表的含义，如下面第一个字节和第二个字节合在一起就叫做一个 `Magic Number` ，以此类推。

![Dubbo](https://code.aliyun.com/middlewarerace2018/docs/raw/master/assets/dubbo-protocol.png)

  所以，解析网络协议的本质就是去解析一个字节数组，并将它包含的数据转换成特定的含义的过程，下面我们就以解析 `Dubbo` 协议为例，来看一下如何正确地读取字
节数组中的信息，在这之前，我们先了解一下golang标准库中的两个函数。

  第一个就是 `bytes.NewBuffer(buf []byte) *bytes.Buffer` ，因为我们处理协议本质上就是处理字节数组，使用该函数对字节数组进行封装而不是直接操作字
节数组的好处自然不用多说。当使用上述函数的时候，返回的 `Buffer` 会以 `buf` 这个字节数组作为它的初始内容，而且在这之后该 `Buffer` 就会全权掌控该字
节数组（也就是说之后我们不可以再碰 `buf`了）。

  第二个就是 `binary.Read(r io.Reader, order ByteOrder, data interface{}) error`，该函数可以非常方便的将结构化的二进制数据从 `r` 中以固
定的字节序（ `ByteOrder` ）读到 `data`相应的数据字段中，由于`bytes.Buffer` 实现了`io.Reader` 中的 `Read` 方法，所以它也能作为第一个参数传入
`binary.Read` 进行使用。

  很多协议的报文都是变长的，为了方便起见，一般都是先解析定长部分的数据，然后再解析其变长部分的数据。

## Dubbo 协议分析

  在正式写协议解析类的代码之前，最好先明确该协议中各个字段的含义，所谓磨刀不误砍柴工嘛。
  
![Dubbo](https://code.aliyun.com/middlewarerace2018/docs/raw/master/assets/dubbo-protocol.png)

* Magic 字段，共2字节，包含 `Magic High` 和 `Magic Low` 字段，用于特定标识该报文是否采用 `Dubbo` 协议而存在的，如果是则要求其值等于0xdabb。
* Req/Res，1比特，用于表示该报文是请求报文还是答复报文，当值为1时说明该报文为请求报文，0时为答复报文。
* 2way，1比特，当且仅当 `Req/Res` 为1时有效，也就是该报文为请求报文时，指明是否期望服务器返回数据，当值为1时则表示希望服务器返回数据。
* Event，1比特，标记该报文是否为事件消息，例如心跳消息，则设置为1。
* Serialization ID，5比特，用于表示变长数据部分序列化的方法，例如当使用 `fastjson` 时它的值为6。
* Status，1字节，用于表示请求的状态，其值可以是

    * 20 - OK
    * 30 - CLIENT_TIMEOUT
    * 31 - SERVER_TIMEOUT
    * 40 - BAD_REQUEST
    * 50 - BAD_RESPONSE
    * 60 - SERVICE_NOT_FOUND
    * 70 - SERVICE_ERROR
    * 80 - SERVER_ERROR
    * 90 - CLIENT_ERROR
    * 100 -SERVER_THREADPOOL_EXHAUSTED_ERROR
      
* Request ID，64比特即8字节，用于唯一标识该请求。
* Data Length，变长部分的数据长度，4字节长度。
* Variable Part 报文中数据的变长部分，其使用的序列化方法由上述的`serializationID` 字段决定。

## 解析过程

  由于从 `Magic Number` 到 `Data Length` 这个数据字段，其总长度都是固定的，所以我们可以定义一个 `dubboHeader` 结构体包含这些数据字段，其中各
个数据字段的类型由协议中各个字段的长度决定，例如 `MagicHigh` 的长度为一个字节，所以我们在结构体中使用一个 `byte` 来表示它就可以了，再比如
`RequestID` 的长度是8字节，我们可以用 `uint64` 来表示，依次类推。

  由于我们要解析的是一个字节数组，为了后面方便起见，我们使用 `bytes.NewBuffer` 来封装它，这样就可以得到一个 `io.Reader` ，然后再使用
`binary.Read` 就可以解码得到 `dubboHeader` 这个结构体了。

  当使用 `bytes.Buffer` 的时候需要注意一点，它里面有类似于指针概念，也就是当我们调用 `Read` 方法读取多少字节时，内部的指针 `off` 也会向前移动多
少字节的长度，所以当我们使用 `binary.Read(buf, binary.BigEndian, &dubboHeader)` ，它就会将内部的指针移动 `len(dubboHeader)` 个长度，这样我们就可以利用
`buf.Next` 方法获取剩下变长部分的数据啦。

  定长部分解决好之后，我们来看变长部分应该怎么解决，协议中 ` Serialization ID` 字段规定了变长部分数据的序列化方法，例如这里使用 `json` 格式进行序
列化，所以我们只要获取剩余部分的字节数组，然后调用`json.Unmarshal` 就可以获得剩余部分的数据内容了。

## 完整代码段

  自定义解析过程中可能出现的错误类型

```golang
var ErrInvalidMagicNumber = errors.New("Invalid Magic Number")

var ErrInvalidDataLength = errors.New("Invalid Data Length")

var ErrUnmarshallingJSON = errors.New("Encounter Error when Unmarshalling JSON")
```

  自定义变长部分数据对应的结构体，注意，结构体中的数据字段必须为大写，否则使用 `json.Unmarshal` 会出错。

```golang
type VariablePart struct {
    DubboVersion     string `json:"DubboVersion"`
    ServiceName      string `json:"ServiceName"`
    ServiceMethod    string `json:"ServiceMethod"`
    MethodName       string `json:"MethodName"`
    MethodParamTypes string `json:"MethodParamTypes"`
    MethodArguments  string `json:"MethodArguments"`
    Attachments      string `json:"Attachments"`
}
```

  按协议规则一步步解析字节数组并得到相应的信息。

```golang
func Parse(b []byte) (*VariablePart, error) {
    buf := bytes.NewBuffer(b)
 
    var dubboHeader struct {
        MagicHigh  byte
        MagicLow   byte
        Misc       byte
        Status     byte
        RequestID  uint64
        DataLength uint32
    }

    var err error
    err = binary.Read(buf, binary.BigEndian, &dubboHeader)
    if err != nil {
        return nil, err
    }
    if !(dubboHeader.MagicHigh == 0xda && dubboHeader.MagicLow == 0xbb) {
        return nil, ErrInvalidMagicNumber
    }

    partBytes := buf.Next(int(dubboHeader.DataLength))
    if len(partBytes) < int(dubbo.DataLength) {
        return nil, ErrInvalidDataLength
    }
    
    partBuffer := bytes.NewBuffer(partBytes)

    var variablePart VariablePart

    switch dubboHeader.Misc & 0x1F {
    case 1:
        //
    case 2:
        //
    default:
        err = json.Unmarshal(partBuffer, &variablePart)
        if err != nil {
            return nil, ErrUnmarshallingJSON
        }
    }
    return &variablePart, nil
}
```
