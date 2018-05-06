# PubSub
  消息的发布和订阅这种设计模式在大型开源项目中还是会经常被使用到的，它的要点是一堆消息订阅者（ `Subscriber` ）会订阅某个自己感兴趣的 `topic` ，一
旦发布者（ `Publisher` ）向这个 `topic` 发布消息，消息的订阅者就能收到该消息。

  这种设计模式的灵感其实就来自于现实生活之中，我们（ `Subscriber` ）向我们感兴趣的杂志社订阅想要阅读的月刊，那么我们订阅的消息（算一个
`subscription` ）就会被杂志社（ `Publisher` ）记录下来，一旦杂志可以上市发行了，杂志社就会主动向我们推送（ `Publish` ）杂志消息。

![PubSub](https://docs.oracle.com/cd/E19575-01/819-3669/images/jms-publishSubscribe.gif)

  下面我们来实现一个简单的发布者订阅者模型吧。

  由于这个模型的核心就在于很多 `subscriber` 会向某一个 `topic` 订阅消息，所以需要一个结构体来记录特定的 `topic` 对应的所有 `subscriber` ，这
里 `subscriber` 用一个集合（ `Set` ）来表示。

```golang
type PubSub struct {
    sync.RWMutex
    subscriptions map[string]*Set
}

type Set struct {
    items map[interface{}]struct{}
    lock *sync.RWMutex
}

func NewPubSub() *PubSub {
    return &PubSub{
        subscriptions: make(map[string]*Set),
    }
}
```

  `Subscription` 应该具有监听行为，通过 `Listen` 来获取 `Publisher` 发布的消息。而结构体 `subscription` 则相当于一个订阅的注册信息。

```golang
type Subscription interface {
    Listen() (interface{}, error)
}

type subscription struct {
    topic string
    ttl   time.Duration
    c     chan interface{}    
}

type (s *subscription) Listen() (interface{}, error) {
    select {
    case <-time.After(s.ttl):
        return nil, errors.New("timed out")
    case iterm := <-s.c:
        return item, nil
    }
}
```

  其实订阅和取消订阅的本质就是将自己从订阅名单（ `subscriptions map[string]*Set` ）里面进行增加或删除的操作，因为我们向杂志社订阅了某本杂志，那
么关于我们的订阅信息一定会在杂志出版商那里存在。

```golang
func (ps *PubSub) Subscribe(topic string, ttl time.Duration) Subscription {
    sub := &subscription{
        topic: topic.
        ttl:   ttl,
        c:     make(chan interface{}, subscriptionBuffSize),
    }
    ps.Lock()
    s, exists := ps.subscriptions[topic]
    if !exists {
        s = NewSet()
        ps.subscriptions[topic] = s
    }
    ps.Unlock()

    s.Add(sub)
    time.AfterFunc(ttl, func() {
        ps.unSubscribe(sub)
    })
    return
}

func (ps *PubSub) unSubscribe(sub *subscription) {
    ps.Lock()
    defer ps.Unlock()
    ps.subscriptions[sub.topic].Remove(sub)
    if ps.subscriptions[sub.topic].Size() != 0 {
        return
    }
    delete(ps.subscriptions, sub.topic)
}
```

  最后我们订阅的杂志终于要发布了，杂志的出版社会根据自己手中的订阅名单，逐一向订阅该 `topic` 的订阅者发送杂志，当然只有订阅了的人才能收到。但是由于
订阅者可能出去旅游了，也就是说他最近并没有调用 `Listen` 将收到的订阅消息拿出，而我们接受订阅消息的邮箱大小是有上限的（ `c: make(chan interface
{}, subscriptionBuffSize)` ，我们设置了 `subscriptionBuffSize` 来表示一个通道中最多可以存放的消息数量），如果该邮箱内已有的消息超出这个限制，
则选择不将消息放入邮箱中，否则则将消息放入这个邮箱中。

```golang
func (ps *PubSub) Publish(topic string, item interface{}) error {
    ps.RLock()
    defer ps.RUnlock()
    s, subscribed := ps.subscriptions[topic]
    if !subscribed {
        return errors.New("no subscribers")
    }
    for _, sub := range s.ToArray() {
        c := sub.(*subscription).c
        if len(c) == subscriptionBuffSize {
            continue
        }
        c <- item
    }
    return nil
}
```
