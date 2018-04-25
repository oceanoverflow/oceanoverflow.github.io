  在对消息的处理过程中，我们常常会遇到不同来源的消息，如在Golang中这些消息来自于不同的通道，但是这些消息在本质上都是相同的，所以，如何把这些消息
进行聚合后再处理，就是我们这篇文章需要探讨的内容。下面我们引入Fanin这种编程模型。

  假设Master节点需要处理3个Slave节点的消息，这3个Slave节点的消息我们使用Golang中的通道进行模拟。
  
```golang
type Message interface{}

func Fanin(cs ...<-chan *Message) <-chan *Message {
    var wg sync.WaitGroup
    
    out := make(chan *Message)
    
    send := func(c <-chan *Message) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }
    
    wg.Add(len(cs))
    
    for c := range cs {
        go send(c)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
} 
```

  在函数中我们新建一个用于消息输出的通道out，并且遍历所有cs中的输入通道c，将中的消息取出后再重新放入out当中，这样就达到了将多个通道合并为一个通道的
目的。

  但是上述方法也有其弊端，就是需要提前知晓所有的输入通道，但是在很多情况下，输入通道可能是动态增加的，那这种情况下我们应该怎么办呢？
  
```golang
type Message interface{}

type MessageFan struct {
    ins []<-chan *Message
    out <-chan *Message
    lock sync.Mutex
}

func NewMessageFan() *MessageFan {
    return &MessageFan{
        ins: []<-chan *Message,
        out: make(<-chan *Message),
    }
}

func (fan *MessageFan) AddFaninChannel(channel <-chan *Message) {
    fan.lock.Lock()
    defer fan.lock.Unlock()
    
    for _, c := range fan.in {
        if c == channel {
            fmt.Println("Received duplicate connection")
            return
        }
    }
    
    fan.ins = append(fan.ins, channel)
    
    go func() {
        for msg := range channel {
            out <- msg
        }
        
        fan.lock.Lock()
        defer fan.lock.Unlock()
        
        for i, c := range fan.ins {
            if c == channel {
                fan.ins = append(fan.ins[:i], fan.ins[i+1:]...)
            }
        }
    }()
}

func (fan *MessageFan) GetOutChannel() {
    return fan.out
}
```
  这里我们定义了一个`MessageFan`结构体，里面包含了一个输入通道的数组和一个输出通道，在实例化`MessageFan`的时候，注意，我们仅仅对`out`进行了`make`
操作，因为`out`通道是确定的，只有一个，而我们的输入通道却是`runtime`时决定的，所以无法使用`make`操作。
  当我们调用`AddFaninChannel`方法的时候，我们先遍历`ins`这个通道数组，看有没有相同的实现已经加入过的通道，有则返回，没有则将它加入`ins`，同样的，
在`goroutine`中轮询这个`channel`，将它的消息塞入`out`中，注意，在一般情况下该for循环是阻塞的，如果for循环返回，则说明`channel`已经被关闭了，即
在代码的某处执行了`close(channel)`这个操作，所以说明它已经没有作用了，我们就可以把它在我们的ins数组中删除。    
    
# 总结
  
  上述两种方法都实现了Fanin这种编程模型，当你遇到那种需要对不同来源但是类型却相同的消息进行集中处理时，就可以考虑使用这种编程模型啦。
    
    
    
    
    
