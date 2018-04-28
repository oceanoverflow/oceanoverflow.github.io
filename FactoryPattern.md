# 工厂模式
  工厂模式是设计模式里面的一种，具体点属于创建模式（creational pattern），它提供了一种创建对象的最佳方式。与传统的new来进行初始化对象不同，
工厂模式在初始化对象时不会向客户端暴露创建的逻辑，也就减少了出错时到处修改的麻烦，工厂方法虽然稍为增加了程序猿的工作量，但会给系统带来更大的可
拓展性和尽量少的修改。

  下面我们来看一下如何使用工厂模式，工厂模式有几个要点，掌握住了几个要点，就可以轻松写代码了。

* Creator: 申明我们的工厂方法，会返回抽象的Product
* Product: 申明我们的工厂方法会返回的东西
* Concrete Creator: 具体的工厂，返回一个特定的Product或者说是Concrete Product
* Concrete Product: 具体的Product的实现

我们以实现一个制造电脑组件的工厂方法作为例子来深入了解一下工厂模式的使用吧，我们的工厂会生产鼠标，键盘，显示器这三种组件，我们把工厂定义为接口
`Factory` ，而生产的产品定义为 `Product` 。`Factory`  通过 `Create()` 来生产 `Product` ，这就是工厂方法的精髓所在。

```golang
type Factory interface {
    Create(spec Spec) Component
}

type Component interface {
    Make(io.Writer) error
}

type Spec struct {
    MadeIn string
}
```

  下面我们都实现了三种产品，分别是 `Mouse` ，`Keyboard` 和 `Screen` 。注意到他们都实现了 `Component` 接口，说明它们都是电脑的组件。
  
```golang
type Mouse struct {
    MadeIn    string
    Bluetooth bool
}

func (m *Mouse) Make(w io.Writer) error {
    _, err := fmt.Fprintf(w, "The mouse is made in %s, and bluetooth enabled is %v\n", m.MadeIn, m.Bluetooth)
    return err
}
```

```golang
type Keyboard struct {
    MadeIn    string
    Bluetooth bool
}

func (k *Keyboard) Make(w io.Writer) error {
    _, err := fmt.Fprintf(w, "The Keyboard is made in %s, and bluetooth enabled is %v\n", k.MadeIn, k.Bluetooth)
    return err
}
```

```golang
type Screen struct {
    MadeIn     string
    Resolution uint64
}

func (s *Screen) Make(w io.Writer) error {
    _, err := fmt.Fprintf(w, "The Screen is made in %s, And the resolution is %d\n", s.MadeIn, s.Resolution)
    return err
}
```

  然后我们定义具体的工厂，来生产具体的电脑组件，`MouseFactory` ，`KeyboardFactory` 和 `ScreenFactory` 都实现了 `Factory` 接口，并且返回
具体的 `Component` 例如 `Mouse` ，`Keyboard` 和 `Screen` 。也就是说让子类决定实例化哪一个工厂类，工厂模式使得其创建过程延迟到子类，当然这里
说子类并不是特别恰当，毕竟golang里没有类的概念。

```golang
type MouseFactory struct{}

func (mf *MouseFactory) Create(spec Spec) Component {
    return &Mouse{
        MadeIn:    "China",
        Bluetooth: true,
    }
}
```

```golang
type KeyboardFactory struct{}

func (kf *KeyboardFactory) Create(spec Spec) Component {
    return &Keyboard{
        MadeIn:    "China",
        Bluetooth: false,
    }
}
```

```golang
type ScreenFactory struct{}

func (sf *ScreenFactory) Create(spec Spec) Component {
    return &Screen{
        MadeIn: "China",
        Resolution: 4000000,
    }
}
```

  `Computer` 这个结构体包含了不同的工厂，这就可以让我们自定义`Computer` 所包含的内容。

```golang
type Computer struct {
    ComponentFactories []Factory
}


func (c *Computer) Make(w io.Writer) error {
    spec := Spec{
        MadeIn: "China",
    }

    if _, err := fmt.Fprintln(w, "Computer Manufacture Begin"); err != nil {
        return err
    }

    for _, factory := range c.ComponentFactories {
        component := factory.Create(spec)
        if err := component.Make(w); err != nil {
            return err
        }
    }

    _, err := fmt.Fprintln("Computer Manufacture Done")
    return err
}
```

  注意到通过工厂模式来产生实例时非常简单，因为我们之前已经把代码封装好了，这样就减少了出现错误或者业务需求改变时修改代码的机会了。

```golang
c := &Computer{
    ComponentFactories: []Factory{
        &MouseFactory{},
        &ScreenFactory{},
        &KeyboardFactory{},
    },
}

c.Make(os.Stdout)
```

  工厂方法是实现代码去耦合的非常精妙的一种方法，希望读者可以仔细品味并加以掌握。如果下次你的项目中需要在很多地方初始化相同的对象，你可以抛弃传统的
new方法而转向使用工厂模式来新建对象。
