# Event Timer

  还记得我们在 [EventProcessEngine](EventProcessEngine.md) 里讲过的 `Event Manager` 吗，我们这里要实现的 `Event Timer` 其实和那个是
有点耦合的，本篇博客与[EventProcessEngine](EventProcessEngine.md) 是姊妹篇，如果没有看过的小伙伴建议先看上一篇再来看这一篇。

  关于定时器，golang的标准库已经提供了一个免费的解决方案给我们了 。标准库里的 `Timer` 结构体里包含一个 `<- chan Time` 通道，在经历过一定的时间
后，该通道就会返回，这样就达到了倒计时的目的，当然还有一个 `Reset` 方法用于重置我们的定时器。

```golang
type Timer struct {
    C <- chan Time
}

func AfterFunc(d Duration, f func()) *Timer 

func NewTimer(d Duration) *Timer

func (t *Timer) Reset(d Duration) bool

func (t *Timer) Stop() bool
```

  那么为什么要实现一个自己的定时器呢，当然是觉得官方提供的不够用啦。

  先说一说我们的 `EventTimer` 会在哪里用到，例如在分布式系统中，主节点需要向从节点周期性发送心跳消息，以期望从节点可以判定主节点有没有出现宕机
之类的故障，所以每个从节点都需要设置一个心跳超时的定时器，如果在规定的时间内，从节点接收到了来自主节点的心跳消息，那么从节点将定时器重置，如果没有
收到，那么从节点会发起申请成为主节点的请求。

  下面例举几个在PBFT算法中定义的几个定时器超时的事件。

```golang
// viewChangeTimerEvent is sent when the view change timer expires
type viewChangeTimerEvent struct{}

// viewChangeResendTimerEvent is sent when the view change resend timer expires
type viewChangeResendTimerEvent struct{}

// batchTimerEvent is sent when the batch timer expires
type batchTimerEvent struct{}
```

  所以我们接下来需要实现的定时器需要有两个功能，第一点就是老功能，定时操作，第二点是增加的新功能，当定时器超时将我们预设好的事件发送给我们的事件管理
器（Event Manager）作进一步处理。

  首先，我们来定义一下我们自己实现的 `Timer` 需要实现的接口，作为一个定时器，第一要务肯定是定时了，所以我们需要它有倒计时的功能，当倒计时结束我们
发送事先定义好的事件给我们的事件管理器（Event Manager），第二点我们需要在不需要它的时候随时停止它。

```golang
type Event interface{}

type Timer interface {
    SoftReset(timeout time.Duration, event Event)
    Reset(timeout time.Duration, event Event)
    Stop()
    Halt()
}

type threaded struct {
    exit chan struct{}
}

type timerImpl struct {
    threaded 
    timerChan <- chan time.Time
    startChan chan *timerStart
    stopChan  chan struct{}
    manager   Manager
}

type timerStart struct {
    hard     bool
    event    Event
    duration time.Duration
}

func newTimer(manager Manager) Timer {
    et := &timerImpl{
        threaded:  threaded{make(chan struct{})},
        startChan: make(chan *timerStart),
        stopChan:  make(chan struct{}),
        manager: manager,
    }
    go et.loop()
    return et
}
```

  interface `Timer` 中的 `Reset` 和 `SoftReset` 方法的功能都是让定时器重新开始倒计时，倒计时结束向我们的事件管理器发送事件，然后由事件管理器
对我们的事件进行集中处理。它们之间的区别就是 `Reset` 会清除掉之前所有定时的事件，而 `SoftReset` 则不会，`Stop` 和 `Halt` 则非常简单，就是停止
定时器，但`Stop` 会清除掉所有定时的事件。

  `timerImpl`  实现了 `Timer` 接口，这里就可以清楚地看到 `Reset` 和 `SoftReset` 就是向 `startChan` 中发送我们的倒计时事件，也就是和我们
的定时器申明，在 `timeout` 这么长的时间后，发射一个 `event` 事件。

```golang
func (t *timerImpl) Reset(timeout time.Duration, event Event) {
    t.startChan <- &timerStart{
        duration: timeout,
        event: event,
        hard: true,
    }
}

func (t *timerImpl) SoftReset(timeout time.Duration, event Event) {
     t.startChan <- &timerStart{
         duration: timeout,
         event: event,
         hard: false,
     }
}

func (t *timerImpl) Stop() {
    et.stopChan <- struct{}
}

func (t *timerImpl) Halt() {
    select {
    case <-t.threaded.exit:
        logger.Warning("Attempted to halt a threaded object twice")
    default:
        close(t.threaded.exit)
    }
}
```

  接下来我们用工厂模式来新建我们的 `Timer`， 使用工厂模式有很多好处，我们会在另一篇博客中详细讲述它的用法和好处。

```golang
type TimerFactory interface {
    CreateTimer() Timer
}

type timerFactoryImpl struct {
    manager Manager
}

func NewTimerFactory(manager Manager) TimerFactory {
    return &timerFactoryImpl(manager)
}

func (etf *timerFactoryImpl) CreateTimer() Timer {
    return newTimer(etf.manager)
}
```

  最后我们来看一下定时器最核心的函数 `loop()` 的实现。

```golang
func (et *timerImpl) loop() {
	var eventDestChan chan<- Event
	var event Event

	for {
		// A little state machine, relying on the fact that nil channels will block on read/write indefinitely

		select {
    // case 1.
		case start := <-et.startChan:
			if et.timerChan != nil {
				if start.hard {
					logger.Debug("Resetting a running timer")
				} else {
					continue
				}
			}
			logger.Debug("Starting timer")
			et.timerChan = time.After(start.duration)
			if eventDestChan != nil {
				logger.Debug("Timer cleared pending event")
			}
			event = start.event
			eventDestChan = nil
    // case 2.
		case <-et.stopChan:
			if et.timerChan == nil && eventDestChan == nil {
				logger.Debug("Attempting to stop an unfired idle timer")
			}
			et.timerChan = nil
			logger.Debug("Stopping timer")
			if eventDestChan != nil {
				logger.Debug("Timer cleared pending event")
			}
			eventDestChan = nil
			event = nil
    // case 3.
		case <-et.timerChan:
			logger.Debug("Event timer fired")
			et.timerChan = nil
			eventDestChan = et.manager.Queue()
    // case 4.
		case eventDestChan <- event:
			logger.Debug("Timer event delivered")
			eventDestChan = nil
    // case 5.
		case <-et.exit:
			logger.Debug("Halting timer")
			return
		}
	}
}
```

  虽然又一点恐怖，还是一步一步来吧。

  首先，说一下大致的思路，`loop()` 方法在开头定义了 `eventDestChan` 通道变量和 `event` 变量分别用于记录事件发送的目的地和事件本身。`loop()` 
方法使用 `for select` 机制去轮询 `timerImpl` 中定义的几个通道。并且5个case条件中都是等待从通道中读取数据，2和5都是非正常流程，而1，3，4则是
正常的流程。我们先从非正常流程说起。

  当程序某处调用 `Stop()` 方法时，`stopChan` 里就会有数据 `struct{}` ，`case 2` 就会被执行，我们分别将`timerChan`，`eventDstChan`，
`event` 设置为nil就可以达到暂停的目的了，非常简单。

  当程序某处调用 `Halt()` 方法时，同样道理， `case 5` 会被执行，整个定时器将会被暂停，然后返回，这里也没有什么难度。

  接下来我们来捋一捋正常的流程也就是倒计时结束时发射事件的流程。一般来说正常的流程无非是程序某处调用了 `Reset()` 或者 `SoftReset()` 方法，那么
我们的 `startChan` 里面就会有一个 `timerStart` 结构体，此时 `case 1` 就会被执行，然后判断当前时候是否有正在倒计时的时间，如果有并且新事件为
hard，那么我们可以重置之前的倒计时，如果不是，则进入下一个for循环中。之后我们通过设置 `et.timerChan = time.After(start.duration)` 来真正
开始倒计时，并且设置倒计时结束后要发生的event，但此时 `eventDestChan` 仍应设置为nil，毕竟倒计时还没有结束呢，超时事件不能马上发射。

  5，4， 3， 2， 1，眼瞧着我们之前设置好的倒计时结束了，那么我们的 `case 3` 就会被紧接着执行了，倒计时结束，说明超时事件已经触发了，但是现在我们
超时事件究竟应该发送给谁还不知道，通过设置 `eventDestChan = et.manager.Queue()` 就将我们的事件目的地设置为我们的事件管理器，然后通过设置
`et.timerChan = nil` 就可以暂停我们的定时器了。

  执行完 `case 3`， 注意到我们现在 `event` 和 `eventDstChan` 都已经不为nil了，那么终于可以执行我们的 `case 4` 了，此处才真正地将我们的超时
事件发送给我们的事件处理的目的地，之后的流程就交给我们的事件管理器（Event Manager）来处理啦。





















