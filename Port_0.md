# port 0, WTF?
  
  我们都知道 `TCP` 或者 `UDP` 的端口号可以从0一直到65535，而0到1023端口则被称为熟知端口，是系统保留使用的端口号，0号端口在 `TCP/IP` 中也属于保留
端口号，但它并不可以在一般的 `TCP` 或者 `UDP` 通信中使用。

  但是，如果0号端口不能被使用的话，那它又有什么用呢，简单来讲，就是如果程序监听了0号端口，那么它就会触发操作系统自动去寻找在0到65535这个范围内可以使
用的端口号（即未被其他程序占用），这样就相当于动态分配一个端口号了，和 `DHCP` 协议有着异曲同工之妙，不过 `DHCP` 是用于动态分配IP地址，而0号端口则用
于动态分配端口号，而且两者的工作原理也不同。

  所以，如果想要得到一个动态的端口号，我们只要让程序监听0号端口就可以了，即 `net.Listen("tcp", "127.0.0.1:0")` ，下面给出一个获取动态端口号的示
例代码。

```golang
package main

import (
    "net"
    "strconv"
)

func GetFreePort() (port int, err error) {
    listener, err := net.Listen("tcp", "127.0.0.1:0")
    if err != nil {
        return 0, err
    }
    defer listener.Close()
     
    addr := listener.Addr().String()
    _, portString, err := net.SplitHostPort(addr)
    if err != nil {
        return 0, err
    }
    return strconv.Atoi(portString)
}

func main() {
    port, err := GetFreePort()
    if err != nil {
        fmt.Println(err)
        return
    }

    fmt.Println(port)
}
```
