# Protocol Buffer

  Protobuf是一种用于信息交换的数据格式，你们可能会问我可以用JSON，再不济XML也可以用啊，为什么要接触这个完全没有用过的玩意儿呢？答案只有两个字：高效。
使用XML不仅会占用更多的存储空间，而且会降低整个系统的性能。

  搞过分布式系统的朋友一定知道，消息在分布式系统中起着至关重要的作用，例如两个节点之间相互交换自己的状态信息，一个网络中节点之间的交流就需要负载在消息
之上，与人类一样，只有节点之间有一套共用的语言，分布式系统才能正常工作下去。而且protobuf可以让我们非常方便的定义消息的类型，它还提供了供我们序列化和
反序列化的API，下面我们就来看看该如何使用它。

## 前期准备
  首先需要安装protobuf，使用macOS的用户可以考虑使用brew进行安装，安装完了之后还需要安装Go语言的protobuf插件—proto-gen-go，这个插件会被安装在
`$GOPATH/bin`目录下。

```
$ brew install protobuf
$ go get -u github.com/golang/protobuf/protoc-gen-go
```

## 定义我们自己的消息格式

  使用protobuf，我们只需要定义一个以 `.proto` 结尾的文件，这里我们以PBFT算法中的几个消息类型为例子进行讲解，首先注意到”=1”， “=2”这样的标记，
这些标记是为了二进制编码用的。其次，如果一个消息中有多个类似的成员，也就是一个数组，我们可以使用 `repeated` 这个关键字来表示。

```protobuf
syntax = "proto3";

import "google/protobuf/timestamp.proto";

package pbft;

message request {
    google.protobuf.Timestamp timestamp = 1;
    bytes payload = 2;
    uint64 replica_id = 3;
    bytes signature = 4;
}

message request_batch {
    repeated request batch = 1;
}

message pre_prepare {
    uint64 view = 1;
    uint64 sequence_number = 2;
    string batch_digest = 3;
    request_batch request_batch = 4;
    uint64 replica_id = 5;
}

message prepare {
    uint64 view = 1;
    uint64 sequence_number = 2;
    string batch_digest = 3;
    uint64 replica_id = 4;
}

message commit {
    uint64 view = 1;
    uint64 sequence_number = 2;
    string batch_digest = 3;
    uint64 replica_id = 4;
}
```

## 使用protoc编译器编译我们的消息
```
protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/messages.proto
```
  在命令行中运行上述命令，`-I` 表示你的源文件所在的目录，`--go_out`  表示你指定的生成目标文件（messages.pb.go）的目录，一般来说与源文件所在的目录
相同，最后就是我们需要编译的源文件的路径。

## 使用protobuf提供的API对消息进行写操作
  
  我们可以使用 `proto.Marshal()` 对数据进行序列化。序列化之后得到的是一个字节数组（ []byte ）。

```golang
prep := &pb.Prepare{
    view: 1,
    SequenceNumber: 2,
    BatchDigest: "aaaaa",
    ReplicaId: 11,    
}

out, err := proto.Marshal(prep)

if err != nil {
    fmt.Printf("Error marshalling prepare: %s", err) 
    return 
}

if err := ioutil.WriteFile(fname, out, 0644); err != nil {
    fmt.Printf("Error writing file: %s", err) 
    return    
}
```

## 使用protobuf提供的API对消息进行读操作
  
  对编码过的数据调用 `proto.Unmarshal` 就可以进行反序列化操作得到原来的信息。

```golang
in, err := ioutil.ReadFile(fname)

if err != nil {
    fmt.Printf("Error reading file: %s", err) 
    return    
}

prep := &pb.Prepare{}

if err := proto.Unmarshal(in, prep); err != nil {
    fmt.Printf("Error unmarshalling prepare: %s", err)
    return
}
```

  当然了，上面两个例子是将序列化的数据存储到文件中，同样的，我们可以将序列化后的数据直接在网上传播，因为序列化后的数据已经是 `wire format` 了。
