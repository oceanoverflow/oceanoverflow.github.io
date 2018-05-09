# CircuitBreaker

  我们都知道，如果家中因为启动过多大功率电器而导致电流过大，断路开关就会断开以保护这些电路和电器。在分布式系统中，集群中的某些服务器出现异常还是一件非常
正常的事情，如果客户端向这些异常的服务器进行 `RPC` 请求的话，这些服务器会因为异常而无法向客户端返回远程方法调用的结果，因为一般来说 `RPC` 是阻塞的，客
户端会像执行本地方法一样调用 `RPC` ，也就是说客户端会阻塞在这里无法执行 直到该方法本地超时，但由于为了得到正确的结果，客户端可能会反复多次调用该 `RPC`
，由于在这段时间内远程服务器都可能无法恢复正常，所以就会导致客户端一直调用该方法，但却无法得到结果，这样就会导致资源的浪费。

  为了避免上述情况的发生，我们可以在程序设计中借鉴断路开关的模式，也就是允许被保护的函数在规定上限内执行，如果一旦出现错误的次数超过上限，则下次函数再次
调用时则直接不让其执行（也就是断路开关直接断开），从而避免系统资源的浪费。但是上述方法存在一个问题，就是一旦总的错误次数超过上限，是不是就说明即使现在远
程服务器恢复正常，客户端也无法调用了呢。

  解决上述问题也很简单，通过模仿网络中很常见的 `Sliding Window` ，也就是移动窗口，允许在一定时间窗口内出现特定上限个错误，如果在该时间窗口内出现错误
的次数超过上限，则不再允许该函数执行，直到下一个时间窗口的到来才允许它重新执行。

  下面来设计一个滑动窗口版本的断路开关，我们通过判断一个时间窗口 `window` 内出现的错误次数 `failures` 是否超过 `failureThreshold` 来决定断路开关
是否应该断开（也就是不让函数继续执行）。

```golang
type CircuitBreaker struct {
    lastFailureTime  time.Time
    failures         uint64
    failureThreshold uint64
    window           time.Duration
}

func NewCircuitBreaker(failureThreshold uint64, window time.Duration) {
    return &CircuitBreaker{
        failureThreshold: failureThreshold,
        window:           window,
    }
}
```

  由于是滑动窗口，我们需要观察在现在的窗口出现错误的次数是否超过我们规定的次数，如果超过，则表示该函数还没有 `ready` ，也就是还不能执行，如果该窗口已经
过去，那么我们重置（ `reset` ）断路开关中记录的出错次数，以让函数可以重新执行。

```golang
func (cb *CircuitBreaker) ready() bool {
    if time.Since(cb.lastFailureTIme) > cb.window {
        cb.reset()
        return
    }
    failures := atomic.LoadUint64(&cb.failures)
    return failures < cb.failureThreshold
}

func (cb *CircuitBreaker) success() {
    cb.reset()
}

func (cb *CircuitBreaker) fail() {
    atomic.AddUint64(&cb.failures, 1)
    cb.lastFailureTime = time.Now()
}

func (cb *CircuitBreaker) reset() {
    atomic.StoreUint64(&cb.failures, 0)
    cb.lastFailureTime = time.Now()
}
```

  也就是说，在 `CircuitBreaker` 保护下，客户端准备调用函数时，会预先检察其是否具备资格，也就是说过去一个时间窗口没有出现超过上限个错误，如果不具备，
就报 `ErrBreakerOpen` 错误，也就是对应下图中的  `circuit open` ，因为断路开关断开了，自然没有办法继续执行。

  如果具备，则可以让受保护的函数继续执行，但是钥匙它在规定的时间内还没有返回结果，说明出现超时情况，则报 `ErrBreakerTimeout` 错误，并增加错误次数直
到其达到上限为止。

![Circuit Breaker](https://martinfowler.com/bliki/images/circuitBreaker/sketch.png)

```golang
var (
    ErrBreakerOpen = errors.New("breaker open")
    ErrBreakerTimeout = errors.New("breaker time out")
)


func (cb *CircuitBreaker) Call(fn func() error, d time.Duration) error {
    var err error
    if !cb.ready() {
        return 
    }

    if d == 0 {
        err = fn()
    } else {
        c := make(chan error, 1)
        go func() {
            c <- fn()
            close(c)
        }()

        t := time.NewTicker(d)
        select {
        case e := <-c:
            err = e
        case <-t.C:
            err = ErrBreakerTimeout
        }
        t.Stop()
    }

    if err == nil {
        cb.success()
    } else {
        cb.fail()
    } 
    return err
}
```
