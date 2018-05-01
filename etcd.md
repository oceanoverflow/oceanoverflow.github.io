# 使用 etcd 来提供服务注册与发现功能
 
  `etcd` 是一个用于存储分布式系统中重要数据结构的存储引擎，而其本身就是一个分布式的键值数据库，例如在微服务架构中的服务注册和发现机制，就可以利用
`etcd` 作为我们的注册中心进行服务，用于存储服务注册信息。

  但是，什么又是服务注册和发现呢，其实这和现在盛行的微服务架构有着密切的联系，在以前那种非微服务时代，应用运行在物理机上面，服务实例对应的网络地址也是
相对固定不变的，而到了微服务时代，很多服务实例的网络地址则是动态分配的，而且服务实例也经常面临着失败或者更新的情况，所以这些服务实例对应的网络地址也经
常会变化，这对应用的开发者和使用者都是非常不利的，所以引入服务注册和发现这种机制，来适应微服务的这种问题。

![alt text](https://code.aliyun.com/middlewarerace2018/docs/raw/master/assets/dubbo-architecture.png)

  我们以 `Dubbo` 中的服务注册与发现为例，深入理解一下到底什么是服务注册与发现，这里有几个角色需要注意一下，`Registry` 用于服务信息的登记和提供，
`Provider` 是服务的提供商，它会将自己提供的服务信息注册到`Registry` 上，而 `Consumer` 则是相应服务的消费者，它会从 `Registry` 那里获取服务的
信息（一般来说就是接口的地址），再调用 `Provider` 上提供的真实服务，`Monitor` 这里仅仅作为记录一些调用次数之类的元信息。注意到这里 `Registry` 
通知 `Consumer` 服务信息是一个异步的过程，而 `Consumer` 调用 `Provider` 的服务则是同步的。

  `etcd` 底层使用 `raft` 协议来维持集群中服务器间数据的一致性，这在之后的博客中会详细探讨。而 `etcd` 也提供了非常简单的客户端接口供我们使用，今天
我们的目标就是写一个 `etcd` 的客户端供我们进行服务注册和发现等相关方面的使用。

  `etcd` 无论作为什么用途，其本质都是一个kv存储引擎，而我们要实现的客户端必须提供以下接口的服务。例如向 `etcd` 写入键值对，获取相应键对应的值，还有
获取相应键最新值的机制（通过watch机制）等等。

```golang
type Store interface {
    Put(key string, value []byte, options *WriteOptions) error
    Get(key string) (*KVPair, error)
    Delete(key string) error
    Exists(key string) (bool, error)
    Watch(key string, stopCh <-chan struct{}) (<-chan *KVPair, error)
    WatchTree(directory string, stopCh <-chan struct{}) (<-chan []*KVPair, error)
    NewLock(key string, options *LockOptions) (Locker, error)
    List(directory string) ([]*KVPair, error)
    DeleteTree(directory string) error
    AtomicPut(key string, value []byte, previous *KVPair, options *WriteOptions) (bool, *KVPair, error)
    AtomicDelete(key string, previous *KVPair) (bool, error)
    Close() 
}

type KVPair struct {
    Key       string
    Value     []byte
    LastIndex uint64
}

type WriteOptions struct {
    IsDir bool
    TTL   time.Duration
}

type LockOptions struct {
    Value     []byte
    TTL       time.Duration
    RenewLock chan struct{}
}
```

  `etcd` 官方已经给我们提供了一个client接口了（ github.com/coreos/etcd/client），我们只要在文件头导入它就行了。

`import etcd "github.com/coreos/etcd/client`

  我们对该服务进行一层封装以便实现我们的 `Store` 接口。

```golang
type Etcd struct {
    client etcd.KeysAPI
}

func New(addrs []string, options *Config) (Store, error) {
    s := &Etcd{}

    cfg := &etcd.Config{
        Endpoints:               entries,
        Transport:               etcd.DefaultTransport,
        HeaderTimeoutPerRequest: 3 * time.Second,
    }

    // Set options
    if options != nil {
        if options.TLS != nil {
            setTLS(cfg, options.TLS, addrs)
        }
        if options.ConnectionTimeout != 0 {
            setTimeout(cfg, options.ConnectionTimeout)
        }
        if options.Username != "" {
            setCredentials(cfg, options.Username, options.Password)
        }
    }
    
    c, err := etcd.New(*cfg)
    if err != nil {
        log.Fatal(err)
    }

    s.client = etcd.NewKeysAPI(c)

    // Periodic Cluster Sync
    go func() {
        for {
            if err := c.AutoSync(context.Background(), periodicSync); err != nil {
                return
            }
        }
    }()

    return s, nil
}
```

  当然，客户端与 `etcd` 服务器的通信一般都不是明文传输的，所以还需要定义用于传输加密相关的结构体。一般在通信过程中会使用TLS加密，所以安全性问题我们
就可以不用担心了。

```golang
type Config struct {
    ClientTLS         *ClientTLSConfig
    TLS               *tls.Config
    ConnectionTimeout time.Duration
    Bucket            string
    PersistConnection bool
    UserName          string
    Password          string
}

type ClientTLSConfig struct {
    CertFile   string
    KeyFile    string
    CACertFile string
}
```

  下面我们来重点看一下三个接口的实现，分别是 `Put` ，`Get` 和 `Watch` ，分别应对现实生活中，`Provider` 向 `etcd` 写入可以提供的服务信息，
`Consumer` 在 `etcd` 中读取服务信息，还有就是在 `etcd` 上注册的服务信息改变时，`Consumer` 可以及时获得改变后的值。

  `Get` 操作获取当前 `etcd` 已经注册的服务信息，也就是服务发现的过程。

```golang
func (s *Etcd) Get(key string) (pair *KVPair, err error) {
    getOpts := &etcd.GetOptions{
        Quorum: true,
    }

    result, err := s.client.Get(context.Background(), s.normalize(key), getOpts)

    if err != nil {
        return nil, err
    }

    pair = &KVPair{
        Key:       key,
        Value:     []byte(result.Node.Value)
        LastIndex: result.Node.ModifieIndex,
    }

    return pair, nil
}
```

  `Put` 操作可以向 `etcd` 注册相应的服务信息，简单的说就是服务注册。

```golang
func (s *Etcd) Put(key string, value []byte, opts *WriteOptions) error {
    setOpts := &etcd.SetOptions{}

    if opts != nil {
        setOpts.Dir = opts.IsDir
        setOpts.TTL = opts.TTL
    }

    _, err := s.client.Set(context.Background(), s.normalize(key), string(value), setOpts)
    return err
}
```

  当我们关心的某一项键值（对应服务）发生变化时候，我们需要及时获取最新的值以免我们的服务出现过期不可用的情况，例如我们我们提供服务的服务端的一个实例
挂掉了，某些编排系统如 `k8s` 会自动重启这些实例，但是重启后的实例的服务地址可能会出现改变，我们使用这些服务的客户端肯定需要知道更新过后的值，通过
`watch` 机制就可以及时获取更新后的键值对。

```golang
func (s *Etcd) Watch(key string, stopCh <-chan struct{}) (<-chan *KVPair, error) {
    opts := &etcd.WatcherOptions{Recursive: false}
    watcher := s.client.Watcher(s.normalize(key), opts)
    watchCh := make(chan *KVPair)

    go func() {
        defer close(watchCh)

        pair, err := s.Get(key)
        if err != nil {
            return
        }

        watchCh <- pair
        for {
            select {
            case <-stopCh:
                return
            default:
            }
           
            result, err := watcher.Next(context.Background())
            if err != nil {
                return
            }

            watchCh <- KVPair{
                Key:       key,
                Value:     []byte(result.Node.Value),
                LastIndex: result.Node.ModifiedIndex,
            }
        }
    }()

    retrun watchCh, nil
}
```

### 后记

  主要是笔者最近要参加阿里的第四届中间件大赛，所以对服务注册和发现做了一些调研工作。在像阿里这种大公司就会提供各种各样的服务端口，而且由于分布式的特性
，服务间通讯就变得尤为重要，通常服务提供者暴露服务，服务消费者调用服务，而它们之间的中介服务通常就由`etcd`，`consul` 或者 `zookeeper` 这样的高性
能键值数据库来提供。
