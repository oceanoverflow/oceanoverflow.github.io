# 使用 etcd 进行负载均衡
  
  当服务提供者（Service Provider）将服务注册到 `etcd` 之后，为了更加理想地分配任务·，我们就要考虑负载均衡问题了，负载均衡的核心就是把让任务合理地
分摊到多个执行单元上执行，以期获得更高的吞吐量或者说是数据处理能力，通俗点讲就是大家都有肉吃，没有人会撑死也没人会饿死，大家都得到适合自己的那一份。

  下图中，`Service Consumer` 即服务的消费者，它通过向 `Provider` 源源不断地发送服务请求，负载均衡问题要解决的问题就是如何将请求合理地分配给不同
的 `Service Provider` 处理，例如，我们下图中有三台服务提供者，假如服务提供者的服务都是比较消耗时间的（例如计算哈希值），为了增加整个系统的吞吐量，
我们要考虑合适的任务分配策略，这就是负载均衡的核心所在，要是只将请求发送给其中一台 `Service Provider`，那么其它两台都会空闲着不执行任务，这样的话
整个系统的吞吐量肯定就上不来了，毕竟没有充分的利用所有的资源，所以，我们要把任务平均地分配给每个执行单元去处理。

![loadbalancing](https://code.aliyun.com/middlewarerace2018/docs/raw/master/assets/system-architecture.png)

  下面我们来实现一个负载均衡的例子，实现的思路非常简单，定义一个 `LoadBalancingService` 结构体，其中的 `etcdClient` 用于与 `etcd` 进行沟通交
流以获取相应的注册信息，`nodes`，`nodeMap`，`nodeCount` 用于记录已经注册在 `etcd` 的服务提供者的信息。

```golang
type LoadBalancingService struct {
    etcdClient client.Client
    connected  bool
    nodes      []string
    nodeMap    map[string]int
    nodeCount  int
    sync.RWMutex
}
```

```golang
func (s *LoadBalancingService) watch(watcher client.Watcher) {
    for {
        resp, err := watcher.Next(context.Background())
        if err == nil {
            if resp.Action == "set" {
                n := resp.Node.Value
                s.addNode(n)
            } else if resp.Action == "delete" {
                n := resp.PrevNode.Value
                s.removeNode(n)
            }
        }
    }
}
```

  上面的核心就是 `Watcher` 机制，在新事件到来之前，`Watcher` 的`Next` 方法会一直阻塞，直到事件返回为止，一般来说都是在 `for` 循环中调用 `Next`
以期达到轮询（polling）的效果。

```golang
type Watcher interface {
    Next(context.Context) (*Response, error)
}
```

  `Connect` 函数就是用于客户端与 `etcd` 连接并根据 `serviceName` 读取当前已经注册过的服务，并通过 `watch` 机制不断地更新当前可用的服务。

```golang
func  (s *LoadBalancingService) Connect(serviceName string, endPoints []string) error {
    if s.connected {
        log.Println("Can't connect twice")
        return errors.New("Already connected")
    }

    s.nodeMap = make(map[string]int)
    s.nodeCount = 0

    cfg := client.Config{
        Endpoints:   endpoints,
        Transport:   client.DefaultTransport,
        HeaderTimeoutPerRequest: time.Second,
    }

    var err error
    s.etcdClient, err = client.New(cfg)
    if err != nil {
        return err
    }
    kapi := client.NewKeysAPI(s.etcdClient)

    resp, err := kapi.Get(context.Background(), serviceName, nil)
    if err != nil {
        return err
    } else {
        if resp.Node.Dir {
            for _, peer := resp.Node.Nodes {
                n := peer.Value
                s.addNode(n)
            }
        }
    }

    watcher := kapi.Watcher(serviceName, &client.WatcherOptions{Recursive: true})
    go s.watch(watcher)
    service.connected = true
    return nil    
}
```

  `GetNode` 采用了随机算法获取当前节点群中的随机节点进行处理我们的任务，这里是负载均衡算法的核心，说白了中随机挑选一个进行发送服务请求让服务提供者进
行处理。

```golang
func (s *LoadBalancingService) GetNode() (string, error) {
    if !s.connected {
        return "", errors.New("Must call connect first")
    }

    s.RLock()
    defer s.RUnlock()

    if s.nodeCount == 0 {
        return "", ErrEmptyService 
    }

    return s.nodes[rand.Intn(s.nodeCount)], nil
}
```

  `addNode` 和 `removeNode` 分别对应远程服务提供者数量增加或者减少时的情况，并更新 `LoadBalancingService` 结构体中相应的数据结构。

```golang
func (s *LoadBalancingService) addNode(key string) {
    s.Lock()
    defer s.Unlock()

    if _, ok := s.nodeMap[key]; ok {
        if s.nodeCount >= len(s.nodes) {
            s.nodes = append(s.nodes, key)
        } else {
            s.nodes[s.nodeCount] = key
        }
        s.nodeMap[key] = s.nodeCount
        s.nodeCount++
    }
}

func (s *LoadBalancingService) removeNode(key string) {
    s.Lock()
    defer s.Unlock()

    if index, ok := s.nodeMap[key]; ok {
        for i := index; i < len(s.nodes)-1; i++ {
            s.nodes[i] = s.nodes[i+1]
        }
        s.nodeCount--
        delete(s.nodeMap, key)
    }
}
```

## 负载均衡算法的改良
  
  在上面的例子中，我们采用的是随机算法，这种是基于每台处理任务的 `Service Provider` 性能差不多的情况下作出的抉择，但是现实生活中情况并不总是如此，
每台服务器的性能，网络带宽都存在着或多或少的差异，所以，我们需要根据实际情况，对负载均衡的分配方法作相应的改进。

  假设我们的服务提供者的平均综合性能比值为1:2:3，那么每台服务提供者任务分配的比例也应该是1:2:3，所谓能者多劳就是这个意思了。在其他情况下我们也可以根
据流量比例分配任务，反正还是要具体问题具体分析。

