# Magnet Link
  
  相信大部分人都使用过磁力链接来下载过网上的文件吧，但是估计很少人会去关心那稀奇古怪的磁力链接究竟是什么鬼，今天就让我们来解析一下磁力链接，看一下它里
面隐藏的信息吧。

  其实类似下面这种看似复杂的链接地址也就是由几个部分组成，开头的 `magnet` 我们把它称为 `scheme` 或者说是协议名，就像一般的 `http` 地址一样，
`http` 也算是一种协议名，紧接着 `magnet:?` 后面就是几对键值对了，一般键值对之间以 `&` 符号划分，常见的键有：

```
magnet:?xt=urn:btih:0FF55426B9713084FF5229A16D21712B1E6C1625&dn=%e5%8d%97%e5%be%81%e5%8c%97%e6%88%98%20-%206415&tr=http%3a%2f%2ftracker.nexushd.org%2fannounce.php%3fpasskey%3d346ae8170f8427b078512150f3683285
```

* `xt` ：eXact Topic 的缩写，其实本质上就是一个文件的哈希值，用于唯一标示这个文件而存在的，这是整个磁力链接中不可或缺的一部分，少了它，基本上链接地址就无法解析了。
* `dn` ：Display Name 的缩写，这个是用于向用户展示的文件的名称，这一项可以没有。
* `tr` ： Tracker 的缩写，表示Tracker服务器的地址，取决于使用的技术，这一项也是选填的。
* `ws` ： Web Seed 的缩写，表示网络种子，选填。
* `urn` ：Uniform Resource Name 的缩写，也就是统一资源名称。
* `btih` ： BitTorrent Info Hash 的缩写，表示种子的散列函数。

知道了磁力链接地址中的关键信息，也就基本可以看懂链接地址中的各个部分分别代表什么含义了，但是由于其中一些信息还是经过特殊编码的，人类读起来还是比较费劲
的，所以我们还是实现代码来解析它吧。

  首先，来定义一个结构体来定义一下磁力链接中的组成部分，这里只选了几个重要的，`InfoHash` 代表的就是 `"urn:btih:"` 后面那一部分玩意儿，
`Trackers` 数组代表的就是 `Tracker` 服务器的地址，`DisplayName` 就对应磁力链接中的 `dn` 。

```golang
type Magnet struct {
    InfoHash Hash
    Trackers []string
    DisplayName string
}

const xtPrefix = "urn:btih:"

func (m Magnet) String() string {
    ret : = "magnet:?xt="
    ret += xtPrefix + hex.EncodeToString(m.InfoHash[:])
    if m.DisplayName != "" {
        ret += "&dn=" + url.QueryEscape(m.DisplayName)
    }
    for _, tr := range m.Trackers {
        ret += "&tr=" + url.QueryEscape(tr)
    }
    return ret
}
```

  解析磁力链接的过程和解析一般的 `url` 的过程非常类似，关键的部分是哈希值部分的解析，由于 `BTIH` 这个部分的哈希值可能用 `Hex` 编码也可能用
`Base32` 编码，所以我们要根据其哈希长度选择解码的方式，这里定义了一个函数变量 `var decode func(dst, src []byte) (int, error)` 用于接受我们
具体的解码函数。

```golang
func ParseMagnetURI(uri string) (m Magnet, err eror) {
    u, err := url.Parse(uri)
    if err != nil {
        err = fmt.Errorf("error parsing uri : %s", err) 
        return
    }
    if u.Scheme != "magnet" {
        err = fmt.Errorf("unexpected scheme: %q", u.Scheme)
        return
    }
    xt := u.Query().Get("xt")
    if !strings.HasPrefix(xtPrefix) {
        err = fmt.Errorf("bad xt parameter")
        return
    }
    
    infoHash := xt[len(xtPrefix):]
    
    var decode func(dst, src []byte) (int, error)
    switch len(infoHash) {
    case 40:
        decode = hex.Decode
    case 32:
        decode = base32.StdEncoding.Decode
    }

    if decode == nil {
        err = fmt.Errorf("unhandled xt parameter encoding: encoded lenght", len(infoHash))
        return
    }
    n, err := decode(m.InfoHash[:], []byte(infoHash))
    if err != nil {
        err = fmt.Errorf("error decoding xt: %s", err)
        return
    }
    if n != 20 {
        panic(n)
    }
    m.DisplayName = u.Query().Get("dn")
    m.Trackers = u.Query()["tr"]
    return
}
```
