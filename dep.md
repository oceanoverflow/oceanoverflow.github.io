# dep

  在写大型工程项目的时候，难免会依赖第三方的库（自己重复造轮子肯定不现实），如果依赖的库数量非常多时，自己去手动管理肯定是不现实的，所以我们需要一个依
赖管理工具帮助我们管理依赖的库。就像 `node` 有 `npm` ，`java` 有 `maven` 。而 `golang` 之前一直没有一个官方推荐的依赖管理工具，所以第三方的依赖
管理工具 `glide` ，`govendor` 之类的就应运而生，但由于不是官方出品的，自己用起来心里总有些毛毛的，后来，`golang` 官方终于推出了 `dep` 这个依赖管
理工具，终于给我们这些强迫症患者安了一个心。下面我们就以一个demo为例子，看看`dep` 如何使用吧。

![godep](https://github.com/golang/dep/raw/master/docs/assets/DigbyShadows.png)

## 安装

  由于 `dep` 是一个命令行工具，安装起来也非常的方便，下面给出两种安装方式。

#### 二进制安装

  这种安装方法通用型比较强，下面的命令会根据你的操作系统直接安装相应的二进制。

`$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh`

#### macOS 安装

  但是 `macOS` 用户一般还是比较习惯于使用`brew` 的，使用 `brew` 也是`no-brainer` 。

```
$ brew install dep
$ brew upgrade dep
```

## vendor 机制
  要了解 `dep` 存在的必要性就不得不提 `vendor` 机制了，我们都知道如果我们在 `go` 文件中导入一个非标准库，在编译时 `go` 会在 `$GOPATH` 下面去寻
找它，但是这就导致一个问题，如果两个项目A和B同时用到了这个库，但是A和B对这个库的版本需求是不同的，这样就很难办了，毕竟在 `$GOPATH` 下面只能存在一个
版本的库，所以为了解决这种情况，在`Go1.5`的时候，就出现 `vendor` 文件夹这个机制，也就是说，`go` 在编译项目的时候，会去这个项目目录下面寻找是否存在
`vendor` 这个文件夹，并且 `vendor` 下面是否存在我们需要的库，如果有则使用它，如果没有则去 `$GOPATH` 下去寻找。

  例如项目C依赖 `github.com/aaa/bbb` 这个库，那么 `go` 在编译时回去查看当前项目目录下有没有 `vendor/github.com/aaa/bbb` 这个文件夹目录，如
果有则使用这个版本，如果没有再去 `$GOPATH` 下面去找。而 `dep` 就是为了帮我们管理 `vendor` 这个目录下面的依赖而存在的。

## 新建项目
  例如我们要写一个与 `WebSocket` 相关的项目，无奈 `golang` 的标准库里实现的 `WebSocket` 我们并不是非常满意，所以我们去 `github` 上面搜了一下
，最后发现 `girilla` 家的 `websocket` 库还是比较符合我们项目的需求的，所以决定使用它。

  第一步就是新建一个 `dep` 项目，使用 `dep init` 命令，会帮我们生成两个文件和一个空的 `vendor` 文件夹。

```
$ dep init
$ ls
Gopkg.toml Gopkg.lock vendor/
```

  然后我们写了一个 `main.go` 文件用于存放我们项目的所有逻辑（bad practice），这个文件除了标准库之外当然还导入了我们需要的 `websocket` 依赖。

```golang
import (
    "flag"
    "html/template"
    "io/ioutil"
    "log"
    "net/http"
    "os"
    "strconv"
    "time"
    
    "github.com/gorilla/websocket"
)
```

  编辑完之后，在命令行中输入 `dep ensure` ，`dep` 会扫描我们当前项目中所有的依赖，并找出我们依赖的三方库，然后更新相应的文件（Gopkg.toml 和
Gopkg.lock），并更新我们的 `vendor` 文件夹，如下所示。

```
.
├── Gopkg.lock
├── Gopkg.toml
├── main.go
└── vendor
    └── github.com
        └── gorilla
            └── websocket
                ├── AUTHORS
                ├── LICENSE
                ├── README.md
                ├── client.go
                ├── client_clone.go
                ├── client_clone_legacy.go
                ├── compression.go
                ├── conn.go
                ├── conn_read.go
                ├── conn_read_legacy.go
                ├── doc.go
                ├── json.go
                ├── mask.go
                ├── mask_safe.go
                ├── prepared.go
                ├── server.go
                └── util.go
```

## 总结

  反正以后用 `dep` 作为go项目的依赖管理工具准没错的，如果你用 `glide` 或者 `govendor` 之类的还是趁早投靠 `dep` 吧。
