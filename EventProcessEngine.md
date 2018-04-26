# 实现一个事件处理引擎

  在许多情况下，尤其是在搞一致性算法的时候，如何对事件（Event）集中进行处理就变成了一件很重要的事情，下面，我们来介绍如何写一个简单的事件处理引擎，
并且向上层提供一个简单可用（无脑）的API供其他程序员使用。

  首先我们需要明白我们要对各种各样的事件进行处理，例如，在PBFT中，我们的消息类型有`Preprepare`，`Prepare`，`Commit`，`ViewChange`等等，具体的
消息类型的定义我们可以使用protobuf来，而无论是什么事件，我们在Golang中我们都可以把它们抽象成一个 `interface{}` 。

```golang
type Event interface{}
```

  这里我们定义的`Event`是我们以后围绕着处理的中心，我们继续看。定义完了`Event`，我们还要定义一个`Event`的接受者（Receiver），也就是事物处理的核心。
  
```golang
type Receiver interface {
    ProcessEvent(e Event) Event
}
```
  
  在这里我们注意到这是一个`interface`，只要你定义的对象实现了`ProcessEvent`这个方法，那么它就是一个`Receiver`，当然，你自己定义的`ProcessEvent`
里面才是真正的业务逻辑，这里仅仅提供一个框架而已。

  定义完了事件的接受者，`Receiver`，我们来看下更高层次的管理者`Manager`，如果把`Manager`想象成公司里的领导，那么他负责的事情就是调度，并且指派任务
给专门的个体也就是我们的`Receiver`去处理。`Receiver`是具体干活的，而`Manager`则是分配任务的。


```golang
type Manager interface {
    Inject(Event)
    Queue() <-chan Event
    SetReceiver(Receiver)
    Start()
    Halt()
} 

type managerImpl {
    threaded 
    receiver Receiver
    events chan Event
}

type threaded struct {
    exit chan struct{}
}
```

  一个Manager的职能具体有两大点，一就是管理事件的队列（events这个通道），二就是取出队列中的事件，指派接受者（SetReceiver(Receiver)）进行处理
（Start）。
```golang
func (em *managerImpl) Queue() {
    return em.events
}

func (em *managerImpl) SetReceiver(receiver Receiver) {
    em.receiver = receiver
}

func NewManagerImpl() Manager {
    return &managerImpl{
        events: make(chan Event),
        threaded: threaded{make(chan struct{})}
    }
}
```

```golang
func (em *managerImpl) Start() {
    go em.eventLoop()
}
```
  `Manager`的启动就是就是开了一个goroutine对消息进行循环处理，下面来看一下`eventLoop`的实现。

```golang
func (em *managerImpl) eventLoop() {
    for {
        select {
        case e := <-em.events:
            em.Inject(e)
        case <-em.exit:
            logger.Debug("event told to exit")
            return
        }
    }
}
```

  `eventLoop`里面是咋一看是一个死循环，但它还是有出口的，它通过`for select`机制一直去轮询`em.events`和`em.exit`这两个通道，如果`em.exit`这
个通道有值，则说明该事件处理循环要退出了，这就是程序的一个出口，但如果`em.events`有消息返回，我们则将它插入到我们`manager`的事件队列里等待被处理，
也就是调用`em.Inject(e)`。

```golang
func (em *managerImpl) Inject(event Event) {
    if em.Receiver != nil {
        SendEvent(em.Receiver, event)
    }
}
```

  `Inject`行为的主体是`manager`，`manager`把任务（event）分配给我们的receiver。
  
```golang
func SendEvent(receiver Receiver, event Event) {
    next := Event
    for {
        next  := receiver.ProcessEvent(next)
        if next == nil {
            break
        }
    }
}
```
  `SendEvent`函数才是真正干活的主力，它主要就是调用`Receiver`实现定义好的`ProcessEvent`方法，因为处理一个事件后可能会返回另一件事件，所以循环
处理它直到为空时候返回。看到这里，你可能会不理解处理意见事件会什么会返回另一件事件呢，其实现实中处理完A事件返回B事件的例子有很多，例如在PBFT中，
处理完`Preprepare`事件，我们赶忙就要开始处理`Prepare`事件了。这种例子还有很多，以后有机会我们再详细介绍。

  上面就是我们整个事件处理的引擎了，不难，但是设计的非常精妙，下面我们来看如何使用它。
```golang
type aEvent Event
type bEvent Event
type cEvent Event

type Instance struct{
    manager Manager
}

func (i *Instance) ProcessEvent(e Event) Event {
    switch et := e.(type) {
    case aEvent:
        fmt.Println("Received A Event, return B Event")
        return bEvent
    case bEvent:
        fmt.Println("Received B Event, return C Event")
        return cEvent
    case cEvent:
        fmt.Println("Received C Event, return nil")
        return nil
    default:
        fmt.Println("Received undefined message type")
    }
}

func main() {
    var i Instance
    i.manager = NewManagerImpl()
    i.manager.SetReceiver(i)
    i.manager.Start()

    i.manager.Queue() <- aEvent
}
```

  这里我们定义了三种事件类型，并且定义了一个包含manager的结构体，这个结构体实现了`ProcessEvent`方法，也就是说它实现了`Receiver`这个接口，
`ProcessEvent`方法里通过判断接收到的event的类型，来进行相应的处理，注意到处理完`aEvent`会返回`bEvent`，这就说我们说的处理完一件事件再返回
一件事件的例子了。

  在`main`函数中，我们新起一个`Instance`实例，并初始化里面的`manager`，把`manager`的`receiver`设置为自己，因为是`Instance`实现了
`ProcessEvent`方法，所以说谁实现了`ProcessEvent`方法，`SetReceiver`就设置为谁，最后再用`Start`起一个goroutine不断地处理接收进行的事件。

  在我们向manager管理的事件队列中插入一个`aEvent`事件后，`eventLoop`就会开始处理事件，命令行中应该会依次输出：
```
Received A Event, return B Event
Received B Event, return C Event
Received C Event, return nil
```

  这个事件处理引擎的实现非常精妙，还是希望读者可以反复阅读几次。
