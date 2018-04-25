# 实现一个有限状态机

  在实际工程代码当中，常常会用到有限状态机(Finite-state Machine. FSM)，例如在p2p网络中一个节点可能拥有多种状态，有限状态机可以用于表示有限个状态
及所有可能的状态之间转移的情况。利用有限状态机就可以轻松地解决节点在不同状态之间转移的情况。这种编程模型可以避免丑陋的switch语句。

  以门的开启和关闭举例，一扇门拥有两种状态（state），open和closed，我们将门关上，那它的状态就由open转移到closed，这里我们将open称为初始状态，
将closed称为目标状态，分别用src和dst表示。而导致状态转移的动作我们称它为事件（event），状态的转移都是由特定事件触发的。
 
```golang
type EventDesc struct {
    Name string 
    Src []string 
    Dst string  
}

type Events []EventDesc
```

  这里Name表示触发状态转移的事件名称，Src是状态转移过程中的初始状态，注意到它是一个数组，因为可以有多个不同的初始状态转移到同一个目标状态，而Dst是
状态转移过程中的目标状态。上面的结构体就代表了有限状态机中触发状态转移的事件，可以把它理解成一个导火索，有了它，有限状态机才在不同的状态之间跳跃。

  下面我们再来定义有限状态机的结构体，一个有限状态机需要哪些信息呢，首先，肯定需要一个值来标定它现在处于什么状态，也就是有限个状态中它处于哪个阶段，
还需要一个东西来表示各个状态之间如何转移，如在事件E发生的情况下状态A转移到状态B。还需要一个回调方法集合（callbacks），它可以用于在状态转移发生时需
要分别调用什么方法。

```golang
type eKey struct {
    event string
    src string
}

type cKey struct {
    target string
    callbackType int
}

type FSM struct {
    current string
    transitions map[eKey]string
    callbacks map[cKey]string
    transition func()
}

```

  还是拿开门关门举例子，当门刚开始的状态是open，那么current=open，我们用手关门，就会查看transitions这个map中open和关门这个动作所对应的目标状态
是什么，发现是closed，然后在状态转移的过程中还会调用注册过的callback方法。以上就是基本的实现一个FSM的思路。

  下面我们来讨论一下回调方法（callback）的种类。
  
```golang
const (
    callbackNone int = iota
    callbackBeforeEvent
    callbackLeaveState
    callbackEnterState
    callbackAfterState
)
```

  触发一次状态的导火索是一个特定的事件Event，我们记为E，因为E的发生，状态A变成状态B，所以我们一共可以定义四种回调方法，分别是在事件发生之前和之后，
也就是callbackBeforeEvent，callbackAfterEvent，还有就是离开初始状态和进入目标状态的回调函数，callbackLeaveState和callbackEnterState。

  而一个callback其实就是一个回调函数 `type Callback func(*Event)` 。

```golang
func NewFSM(initial string, events []EventDesc, callbacks map[string]Callback) *FSM {
	var f FSM
	f.current = initial
	f.transitions = make(map[eKey]string)
	f.callbacks = make(map[cKey]Callback)

	// Build transition map and store sets of all events and states.
	allEvents := make(map[string]bool)
	allStates := make(map[string]bool)
	for _, e := range events {
		for _, src := range e.Src {
			f.transitions[eKey{e.Name, src}] = e.Dst
			allStates[src] = true
			allStates[e.Dst] = true
		}
		allEvents[e.Name] = true
	}

	// Map all callbacks to events/states.
	for name, c := range callbacks {
		var target string
		var callbackType int

		switch {
		case strings.HasPrefix(name, "before_"):
			target = strings.TrimPrefix(name, "before_")
			if target == "event" {
				target = ""
				callbackType = callbackBeforeEvent
			} else if _, ok := allEvents[target]; ok {
				callbackType = callbackBeforeEvent
			}
		case strings.HasPrefix(name, "leave_"):
			target = strings.TrimPrefix(name, "leave_")
			if target == "state" {
				target = ""
				callbackType = callbackLeaveState
			} else if _, ok := allStates[target]; ok {
				callbackType = callbackLeaveState
			}
		case strings.HasPrefix(name, "enter_"):
			target = strings.TrimPrefix(name, "enter_")
			if target == "state" {
				target = ""
				callbackType = callbackEnterState
			} else if _, ok := allStates[target]; ok {
				callbackType = callbackEnterState
			}
		case strings.HasPrefix(name, "after_"):
			target = strings.TrimPrefix(name, "after_")
			if target == "event" {
				target = ""
				callbackType = callbackAfterEvent
			} else if _, ok := allEvents[target]; ok {
				callbackType = callbackAfterEvent
			}
		default:
			target = name
			if _, ok := allStates[target]; ok {
				callbackType = callbackEnterState
			} else if _, ok := allEvents[target]; ok {
				callbackType = callbackAfterEvent
			}
		}

		if callbackType != callbackNone {
			f.callbacks[cKey{target, callbackType}] = c
		}
	}

	return &f
}
```

1. 在new一个FSM的时候，我们先定义它的初始状态current，还有状态机中所有可能的状态迁移情况以及它的回调函数。
2. 这里我们建立所有可能的状态转移的情况。
3. 注册回调函数，也就是在事件发生前后，和改变状态的时候的回调函数。

  在初始化完一个FSM之后，也就是说规定了FSM该拥有的所有状态以及状态转移时候应该调用的函数之后，我们来看看如何让这个状态机动起来，我们之前已经知道，
触发状态机变化的就是外部的特定的事件，所以我们需要灵活地去处理这些事件，来使得我们的状态机可以动起来。
  
  在上面我们定义了EventDesc这个结构体，注意到那个结构体是多对一的，也就是多个初始状态对应一个目标状态，我们现在定义一个真正的一对一的事件，这里的
一对一指的是一个初始状态对应一个目标状态。

```golang
type Event struct {
    FSM *FSM
    Event string
    Src string
    Dst string
    Err error
    Args []interface{}
    canceled bool 
    async bool
}

```

  所以我们如何调用FSM并且进而使它的状态进行转化并且调用相应预先注册好的回调函数呢，我们通过向FSM实例传送一个event事件，FSM会先判断当前是否正在转移之
中，也就是 f.transition 是否为nil，若不为nil则说明状态机正在状态转移之中，否则则说明不是，继而综合FSM当前的状态和此次遇到的事件决定目标状态，然后按
顺序调用回调函数beforeEventCallbacks => leaveStateCallbacks => enterStateCallbacks => afterEventCallbacks。这样一次状态转移就算完成了。

```golang
func (f *FSM) Event(event string, args ...interface{}) error {
	if f.transition != nil {
		return &InTransitionError{event}
	}

	dst, ok := f.transitions[eKey{event, f.current}]
	if !ok {
		for ekey := range f.transitions {
			if ekey.event == event {
				return &InvalidEventError{event, f.current}
			}
		}
		return &UnknownEventError{event}
	}

	e := &Event{f, event, f.current, dst, nil, args, false, false}

	err := f.beforeEventCallbacks(e)
	if err != nil {
		return err
	}

	if f.current == dst {
		f.afterEventCallbacks(e)
		return &NoTransitionError{e.Err}
	}

	f.transition = func() {
		f.current = dst
		f.enterStateCallbacks(e)
		f.afterEventCallbacks(e)
	}

	err = f.leaveStateCallbacks(e)
	if err != nil {
		return err
	}

	err = f.Transition()
	if err != nil {
		return &InternalError{}
	}

	return e.Err
}

func (f *FSM) Transition() error {
	  if f.transition == nil {
		  return &NotInTransitionError{}
	  }
	  f.transition()
	  f.transition = nil
	  return nil
}
```

## 有限状态机的简单实践

```golang
package main

import (
    "fmt"
    "github.com/looplab/fsm"
)

func main() {
    fsm := fsm.NewFSM(
        "closed",
        fsm.Events{
            {Name: "open", Src: []string{"closed"}, Dst: "open"},
            {Name: "close", Src: []string{"open"}, Dst: "closed"},
        },
        fsm.Callbacks{},
    )

    fmt.Println(fsm.Current())

    err := fsm.Event("open")
    if err != nil {
        fmt.Println(err)
    }

    fmt.Println(fsm.Current())

    err = fsm.Event("close")
    if err != nil {
        fmt.Println(err)
    }

    fmt.Println(fsm.Current())
}

```

  上述例子就定义了一个简单的开门和关门的状态机，这里我们门这个状态机的初始状态是closed，它可以利用open这个事件将状态从closed状态转移到open状态，
也可以利用close这个事件将状态从open状态转移到closed状态。注意这里没有定义回调函数（fsm.Callbacks{}为空），所以这两件事件触发的时候什么事情也不
会发生。

  而调用事件就是通过fsm.Event("open”)或fsm.Event("close”)来完成的。
  
  上述就是实现一个简单的有限状态机的代码啦，具体的实现可以参考[Finite State Machine](http://github.com/looplab/fsm)。
  
  简单总结一下，实现有限状态机需要下面几件事情。
1. 有限状态机的初始状态
2. 有限状态机各个状态之间迁移的各种情况，也就是状态拓扑图
3. 状态迁移时的回调方法

