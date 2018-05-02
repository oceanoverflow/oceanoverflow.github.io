# TLS

  在之前的博客中涉及到的 `TCP` 网络编程都是非常不安全的，因为它将我们的数据直接以明文的形式在公网上面传输，如果有恶意第三方的存在，我们传输的数据就
可以被捕捉监听甚至修改，从而导致各种各样无法预测的结果，所以，考虑到安全性，我们还是要学一学如果用 `TLS` 进行网络编程，以提高我们的客户端和服务器之
前传输数据的安全性。

  `TLS` 即 `Transport Layer Security` ，就是为了保证传输层上的数据安全，服务器和客户端会在 `handshake` 阶段协商出只有两者知晓的一个对称密钥
，然后之后的通信就利用这个对称密钥将数据进行加密后进行传输，很幸运的是，在我们使用 `TLS` 进行网络编程的时候，并不用去关注这些繁杂的细节，因为golang
的标准库都已经帮我们封装好了（hooray！）。

## 准备工作

  使用 `TLS` 进行网络编程的时候，我们需要在服务器上放一个证书，证书的话可以使用 `openssl` 工具来生成，这里作为例子就先生成一个自签名的证书吧。

  要生成一个证书，第一步当然就是生成一个私钥了，这里我们指定使用 `RSA` 算法生成一个 `2048bit` 的私钥。

`$ openssl genrsa -out server.key 2048`
 
  计算私钥是比较耗时的一个过程，无论多好的电脑，都需要在毫秒级别才能生成一个RSA私钥，这里输出的 `server.key` 就是我们要的私钥，生成的私钥是经过
`pem` 格式编码的，也就是类似如下的格式：

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA1DYBcVnQFyFEW9AhHY+6+Shp7LIT5ZtjbFfu2ZQgg3aTsXVF..........
-----END RSA PRIVATE KEY-----
```

  当拥有了私钥之后，为了得到对应的证书，我们还需要有一个 `Certificate Signing Request` ，简称为 `CSR` ，也就是证书签名请求。

`$ openssl req -new -sha256 -key server.key -out server.csr`

  这个命令会要求你输入相关的信息，填入相关信息后就会得到一个 `server.csr` 文件，这个文件也是用`pem` 格式编码的，类似如下：

```
-----BEGIN CERTIFICATE REQUEST-----
MIIC0jCCAboCAQAwbjELMAkGA1UEBhMCQ04xETAPBgNVBAcMCEhhbmd6aG91MQww..........
-----END CERTIFICATE REQUEST-----
```

  最后一步就是利用上面步骤中得到的私钥和证书签名请求来最终获得一个自签名的证书。

`$ openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt`

  最终获得的证书 `server.crt` 也是 `pem` 格式编码的，类似如下：

```
-----BEGIN CERTIFICATE-----
MIIDWDCCAkACCQCFvy51fQRGMTANBgkqhkiG9w0BAQUFADBuMQswCQYDVQQGEwJD..........
-----END CERTIFICATE-----
```

  利用下面的命令我们可以查看我们刚刚生成的符合 `x509` 标准的证书的具体信息，里面包含了两个比较重要的信息就是公钥（public key）和签名
（signature）。

`$ openssl x509 -in server.crt -noout -text`

  上面这么多命令必须按顺序执行，也就是说第一步生成私钥，第二步生成证书签名请求，最后才能生成我们的证书。

## TLS server

  在 `TLS` 编程中，服务器端需要得到私钥和证书的信息。我们使用 `tls.LoadX509KeyPair` 来获得我们之前生成证书和私钥信息，然后在监听时作为
参数导入。

```golang
package main

import (
    "crypto/tls"
    "net"
    "log"
    "bufio"
    "fmt"
)

func main() {
    cert, err := tls.LoadX509KeyPair("server.crt", "server.key")
    if err != nil {
        log.Println(err)
        return
    }

    config := &tls.Config{Certificates: []tls.Certificate{cert}}
    listener, err := tls.Listen("tcp", ":443", config)
    if err != nil {
        log.Println(err)
        return
    }
    defer listener.Close()
    
    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Println(err)
            continue
        }
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()
    r := bufio.NewReader(conn)
    for {
        msg, err := r.ReadString('\n')
        if err != nil {
            log.Println(err)
            return
        }
        
        fmt.Println(msg)

        n, err := conn.Write([]byte("world\n"))
        if err != nil {
            log.Println(n, err)
            return
        }
    }
}
```

## TLS client

  与一般的TCP网络编程类似，作为客户端我们调用 `tls.Dial` 而非 `net.Dial` 来与服务器建立连接，但与 `net.Dial` 不同的是，我们还需要多传递一个
`tls.Config` 参数，来指定 `TLS` 连接时的各种参数。

```golang
package main

import (
    "fmt"
    "log"
    "crypto/tls"
)

func main() {
    conf := &tls.Config{
        InsecureSkipVerify: true,
    }

    conn, err := tls.Dial("tcp", "127.0.0.1:443", conf)
    if err != nil {
        log.Println(err)
        return
    }
    defer conn.Close()

    n, err := conn.Write([]byte("hello\n"))
    if err != nil {
        log.Println(err)
        return
    }

    buf := make([]byte, 100)
    n , err := conn.Read(buf)
    if err != nil {
        log.Println(n, err)
        return
    }

    fmt.Println(string(buf[:n]))
}
```

## 抓取tcp包进行验证

`$ tcpdump host 127.0.0.1 port 433`

  当我们这时候使用类似 `tcpdump` 这样的抓包工具进行网络抓包时，我们不再能看到明文了，看到的只能是一堆乱七八糟的东西，因为客户端和服务器已经将数据
经过加密处理了。
