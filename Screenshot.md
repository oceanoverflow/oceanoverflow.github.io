# 使用Golang获取手机屏幕截图

  在今年年前的时候，像冲顶大会，芝士超人这样的答题赚钱类应用异常火爆，当然了，聪明的程序猿们也不是吃素的，为了提高答对题目的概率，就写了一些程序（脚本
）来自动化答题过程，整个作弊的思路并不难，关键点就在于获取手机的当前截屏，然后通过文字识别来获取题目的信息，最后再调用浏览器搜索答案，在浏览器中出现最
多的那个基本就可以锁定是答案了（当然还有其他的作弊方法，这里不一一列举），这种作弊方法在初期真是屡试不爽，所以今天就让我们来看一下整个作弊流程的关键环
节——自动获取手机截屏的代码实现吧。

  因为手机分为 `Android` 阵营和 `iOS` 阵营，而且两者实现截屏的方法也稍有不同，所以我们先定义一个 `Screenshot` interface对两种平台进行抽象，之
后会分别定义 `iOS` 和 `Android` 这两个结构体来实现这个接口（有没有慢慢开始体会到 `interface` 的好处呢），下面的代码中我们也利用了工厂模式来返回 
`Screenshot` 实例。

```golang
type Screenshot interface {
    GetImage() (image.Image. error)
}

func NewScreenshot(cfg *Config) Screenshot {
    if cfg.Device == "iOS" {
        return NewIOS(cfg)
    }
    return NewAndroid(cfg)
}

type Config {
    Device     string
    WdaAddress string
    .....
}
```

## Android 实现
 
  我们先定义个 `Android` 结构体，这个结构体实现了 `GetImage` 方法，所以可以在上面的工厂方法中作为 `Screenshot` 接口返回。而在真正运行我们的代码
之前，我们需要确保安卓机已经正确地连接至电脑，并且要确保电脑上已经安装过 `adb` 这个命令行工具。这里实现截屏主要靠调用 `adb` 的 `screencap` 和
`pull` 这两个子命令，`screencap` 对当前手机进行截屏并保存到手机本地，`pull` 命令将手机上的截图传到电脑本地（然后就可以接着一通骚操作了），我们通
过 `exec.Command` 接口对 `adb` 进行调用，整体思路比较清晰，难点应该就在我们不熟悉 `adb` 命令的使用。

```golang
type Android struct {}

func NewAndroid(cfg *Config) *Android {
    return new(Android)
}

func (android *Android) GetImage() (img image.Image, err error) {
    err := exec.Command("adb", "shell", "screencap", "-p". "/sdcard/screenshot.png").Run()
    if err != nil {
        return
    }
    dstImagePath := "/Desktop" + "origin.png"
    err = exec.Command("adb", "pull", "/sdcard/screenshot.png", dstImgPath).Run()
    if err != nil {
        return
    }
    f, err := os.Open(dstImagePath)
    if err != nil {
        return nil, err
    }
    defer f.Close()
    return png.Decode(f)
}
```

## iOS 实现

  依葫芦画瓢，同样定义一个 `IOS` 结构体，来实现 `GetImage` 方法，在具体截屏的实现中，由于 `iOS` 系统的封闭性，所以自然没有像 `Android` 的
`adb` 那样的命令行调试工具，不过也不是说啥事也做不了，在 `iOS` 中，我们可以向特定接口地址 `wdaAddress` 发送 `http` 的`GET` 请求，然后通过解析
返回的 `json` 数据，这个 `json` 数据中就编码了我们的图片信息，然后再利用 `base64` 进行解码获得一个字节数组，再对该字节数组进行 `png` 解码，最终
就可以获得我们想要的 `image` 了。

```golang
type IOS struct {
    wdaAddress string
}

type screenshotRes {
    Value     string `json:"value"`
    SessionID string `json:"sessionId"`
    Status    int    `json:"status"`
}

func NewIOS(cfg *Config) *IOS {
    ios := new(IOS)
    ios.wdaAddress = cfg.WdaAddress
    if ios.wdaAddress == "" {
        panic("Please specify the wda address")
    }
    return ios
}

func (ios *IOS) GetImage() (img image.Image, err error) {
    body, err := util.HTTPGet(ios.wdaAddress)
    if err != nil {
         return
    }
    res := new(screenshotRes)
    err = json.Unmarshal(body, res)
    if err != nil {
        return
    }
    pngValue, err := base64.stdEncoding.DecodeString(res.Value)
    if err != nil {
        return
    }
    src, err := png.Decode(bytes.NewReader(pngValue))
    if err != nil {
        return
    }
    img = src
    return
}
```

## 总结
  
  在本篇博客中，我们看到了工厂模式在具体程序设计中的应用，灵活使用工厂模式可以提供我们工程代码的水平，也见识到了如何利用 `adb` 工具对 `Android` 机
器进行调试，当然了 `adb` 的功能不仅仅局限于上述两点，它还可以干非常多的事情，有兴趣的小伙伴可以自己研究一下，最后还有就是因为 `iOS` 系统本身的封闭性
，我们只能通过调用web接口来获取其截图信息。

  最后，作为程序员圈子里的一员，在日常生活中还是要多思考，啥东西要是可以用代码实现自动化，就马上动手去实现它吧。
