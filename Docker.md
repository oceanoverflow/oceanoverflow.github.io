# Docker & Golang

  还记得 `java` 当初响遍大江南北的口号吗，`Write one，run anywhere` ，到了互联网时代，`Docker` 这家公司打出了更为牛逼的口号，`Build，Ship，
and Run Any App，Anywhere` ，它们两者有类似的地方，都是为了追求各种平台的兼容性和通用性。`java` 的跨平台是因为它在不同平台下有不同的 `jvm` 实现
，不过它仅仅局限于 `java` 这个生态系统里面，相比之下，`Docker` 的野心就更大了，它并没有给我们把编程语言给限定死了，它提供了一个与操作系统隔离的环境
，可以让我们写任何应用，而且可以在几乎所有平台下跑。

  具体 `Docker` 的细节这里就不具体探讨了，网上也有一堆文章，今天我们主要看一下如何使用它，也就是如何将我们的应用打包成一个 `Docker` 镜像。

  下面我们要实现的是一个用 `golang` 写的 `http` 服务器，但是这个服务器并不是跑在裸机上，而是跑在 `Docker` 容器中的。顺便一提，`Docker` 也是用
`golang` 写的哦，是不是感觉很厉害呢。

## 书写自己的应用逻辑

下面定义了我们要实现的 `http` 服务器的前端，无奈直男不会写 `CSS` ，就用 `HTML` 写了一下最基本的提交 `FORM` 表单的功能。

```html
<html>
    <head>
        <title></title>
    </head>
    <body>
        <form action="/hash" method="post">
            interface: <input type="text" name="interface">
            method: <input type="text" name="method">
            parameterTypesString: <input type="text" name="parameterTypesString">
            parameter: <input type="text" name="parameter">
            <input type="submit" value="hash">
        </form>
    </body>
</html>
```

  下面使用了golang中的标准库 `net/http` 和 `html/template` 起一个服务器，我们自己定义的 `hashHandler` 会根据请求的类型分别进行处理，如果是
`GET` 请求则展示我们的前端界面，如果是 `POST` 请求则处理我们浏览器提交的表单并将表单输出到命令行中。到这里，一个非常简单的 `http` 服务器就写好了。

```golang
package main

import (
    "fmt"
    "html/template"
    "log"
    "net/http"
    "strings"
)

func hashHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Println("method: ", r.Method)
    if r.Method == "GET" {
        t, _ := template.ParseFiles("hash.tpl")
        t.Execute(w, nil)
    } else {
        r.ParseForm()
        for k, v := range r.Form {
            fmt.Printf("key: %s, value: %s\n", k, v)
        }
    }
}

func main() {
    http.HandleFunc("/hash", hashHandler)
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

## Dockerfile的撰写

  到现在，就没我们 `golang` 什么事情了，有请我们今天的主角 `Docker` 登场了。为了能让我们的 `http` 服务器在容器中运行，我们需要写一个
`Dockerfile` ，`Dockerfile` 的作用就是让 `Docker` 知道该如何构建镜像，当然平时大部分时候我们都可以直接用别人打包好的镜像，但是自己会写
`Dockerfile` 也是一个非常重要的能力哦。

  `FROM` 指令指定了我们的基础镜像是什么，也就是说我们构建的镜像是基于哪个镜像的基础之上的，这里 `golang:alpine` 作为我们的基础镜像。而`COPY` 指
令则将我们的文件从操作系统中拷贝到容器环境中。`WORKDIR` 命令则制定了在 `Docker` 内部的工作目录，类似于 `cd` 命令，`RUN` 命令在容器中执行指令，这
里我们把go源文件编译成二进制。`CMD` 指令指定了启动该容器时内部应该执行的命令。当然 `Dockerfile` 的命令还有很多，具体还是参考官方的文档吧。

```dockerfile
FROM golang:alpine
COPY . /home
WORKDIR /home
RUN go build -o server ./server.go
CMD /home/server
```

  写完了 `Dockerfile` ，就可以构建镜像了，不过构建镜像的事情我可干不来，还是交给 `Docker` 去做吧。下面的命令就帮助我们构建一个名为
`oceanoverflow/server` 的 `golang` 服务器镜像。构建镜像的过程可能会比较花时间，需要耐心等待。

`$ docker build -t oceanoverflow/server .`

## 运行Docker容器

  构建完了我们使用 `docker image ls` 就可以看到我们新构建的镜像啦，最后一步就是起容器服务了。

  `-p` 命令这里将容器内的端口8080映射到操作系统的端口8080上去，如果不指定8080，那我们在自己的操作系统上就无法获取到它提供的服务，所以 `-p` 还是非
常重要的。

`$ docker run -p 8080:8080 --rm -it go-server`

  如果一切无误，到浏览器中输入 `localhost:8080/hash` 就可以看到我们跑在 `Docker` 里面的 `http` 服务器已经成功地跑起来啦。

## 总结
  
  让自己的应用跑在 `Docker` 容器中非常简单，第一步就是构建自己的应用程序的源码，第二步正确编写适应自己业务场景的 `Dockerfile` 然后利用`Docker` 
自带的功能打包成一个镜像，最后一步就是运行容器了。就这么三步走，可谓是 `Docker`在手，天下我有了。

