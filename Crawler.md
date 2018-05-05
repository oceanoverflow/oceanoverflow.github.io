# Crawler
  
  爬虫算是练习并发编程的一个非常好的例子了，因为它既不枯燥，难度也不大，而且能在短时间内产出一个有趣的结果。爬虫一般通过某种搜索算法（深度优先搜索或者
广度优先搜索）在互联网这个海量数据库中去寻找我们感兴趣的东西，如果使用得当的话，爬虫可以极大地提高我们的工作效率。

  那么接下去我们从一个单线程的爬虫程序出发，然后慢慢将它改造成一个并发的爬虫程序。在这里，我们并不关心究竟如何去匹配我们感兴趣的元素(例如使用CSS进行元
素匹配)，而是给出一个爬虫的基本思路。

  首先，爬虫中非常重要的一个步骤就是匹配当前搜索的网页中的元素，并将符合我们要求的东西下载或保存起来，并且还会记录当前网页中的超链接地址，以便于我们继
续进行深度优先搜索或者广度优先搜索。一般来说，我们感兴趣的东西都是符合特定规则的，而具体的规则我们可以在单个网页上进行正则匹配来获取，但是，由于不同网
页的匹配规则并不相同，所以我们定义一个interface来抽象该行为。在这之后无论我们想要爬豆瓣还是新浪微博亦或是其他自己感兴趣的网站，只要重新定义一个结构体
实现该方法就可以。

  `Fetcher` 这个 `interface` 定义了如下行为，从一个特定的url去取我们感兴趣的内容以及这个网页上的所有的url链接。

```golang
type Fetcher interface {
    Fetch(url string) (body string, urls []string, err error)
}
```

  例如我们要爬豆瓣网上的东西，我们只要定义好一个结构体，并且实现上述`Fetch` 方法就可以了。所以无论以后你决定改爬别的网站了，再定义如下的结构体并实现
相应的方法就可以了，这就是为什么上面我们定义了一个 `Fetcher` interface 来抽象这个行为的原因了。

```golang
type Result struct {
    body string
    urls []string
}

type doubanFetcher struct {
    map[string]*Result
    sync.RWMutex
}

func (f *doubanFetcher) Fetch(url string) (body string, urls []string, err error) {
     // 这里写具体的匹配逻辑
}
```

  爬虫还有一个重点就是搜索的策略，应该是广度优先还是深度优先，但是所有的链接形成为拓扑图并不一定是一个有向无环图，所以我们的爬虫在爬取的过程中可能会遇
到相同的链接，但是注意这个逻辑同样应在在 `Fetch` 函数中实现。

## 单线程版本实现

  下面我们来看一下单线程深度优先的爬虫应该怎么写。

```golang
func Crawl(url string, depth int, fetcher Fetcher) {
    if depth <= 0 {
        return
    }

    body, urls, err := fetcher.Fetch(url)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("Founf: %s %q\n", url, body)
    for _, u := range urls {
        Crawl(u, depth-1, fetcher)
    }
    return
}
```

  由于是深度优先搜索，所以我们用递归的策略来进行搜索，递归最重要的一点就是递归条件的出口条件，如果递归函数没有出口，那么就是死循环了。这里我们在函数的
参数中引入一个深度的变量来控制是否应该继续进行递归，如果当前深度小于等于0，则退出。然后用 `fether`  来 `Fetch` 对应的 `url`上的内容，由于使用了接
口，所以让我们这个函数的实现变得非常的灵活，而不是局限于哪个具体的实现，这就是接口的好处。然后根据当前网页中的链接进行深度优先搜索，
`Crawl(u, depth-1, fetcher)` ，每搜索一层，深度减一，直到条件不再满足就可以退出了。

  不过上述的爬虫还是单线程的，单线程的效率肯定不高，因为程序大部分时间都是在等待 `I/O`，`CPU` 大部分时间都是空闲的，所以，为了提高 `CPU`的占用率，
我们应该考虑使用并发。

## 并发版本实现

  在 `golang` 里面写并发程序是相对比较简单的，但是也没有那么简单，下面的实现就是比较tricky的。这里为什么叫它并发而不是多线程呢，是因为`golang` 中
的 `go` 机制并不是以线程作为基本单位的。

```golang
func Crawl(url string, depth int, fetcher Fetcher, ret chan string) {
    defer close(ret)
    if depth <= 0 {
        return
    }

    body, urls, err := fetcher.Fetch(url) 
    if err != nil {
        ret <- err.Error()
        return
    }
     
    ret <- fmt.Sprintf("Found: %s %q", url, body)

    result := make([]chan string, len(urls))
    for i, u := range urls {
        result := make(chan string)
        go Crawl(u, depth-1, fetcher, result)
    }

    for i := range result {
        for s := range result[i] {
            ret <- s
        }
    }
    
    return
}
```

  并发版本和单线程版本的基本框架是一样的，有下面几个区别。

  首先我们定义个一个 `chan string` 也就是一个通道，一般来说，`go` 和`chan` 都是手拉手用在一起的。这个通道用于接受我们爬虫的结果。
  
  这个通道在整个函数返回时会关闭，也就是函数最初用的 `defer close(ret)` 。

  当 `fetch.Fetch(url)` 方法返回时，也就是这个网页中的内容已经检索完毕了，我们将结果放到刚才的通道中等待其他地方的程序读取该通道中的内容。

  下面就是函数的精髓了，由于上面通过 `fetch.Fetch(url)` 获得了一个`urls` 数组，因为要进行深度优先搜索，我们为这个 `urls` 中的 `url` 也准备一个
`chan string` 用于存储它们抓取的内容，然后调用 `go Crawl(u, depth-1, fetcher, result)` 来并发爬取。

  也就是说下面这段函数很快就返回了，但是它们并不一定都已经拿到相应的内容了。

```golang
for i, u := range urls {
    result := make(chan string)
    go Crawl(u, depth-1, fetcher, result)
}
```

  所以，我们要等待它们拿回内容，下面代码的意思就是子任务完成后，将它拿到的内容传给上一层的通道中。这个就好比在一个公司中，`CEO` 将任务分别分配给下面
各个部门的 `Manager` ，然后各个部门的 `Manager` 又细分任务后将任务分给部门下面的各个 `Worker` ，`Worker` 完成自己的任务后，将任务结果传给负责
自己的 `Manager` ，然后由 `Manager` 再汇报给 `CEO` 。`CEO` 是总任务的分配者同时也是所有结果的收集者。

```golang
for i := range result {
    for s := range result[i] {
        ret <- s
    }
}
```

## 总结
  
  并发程序能充分利用现在 `CPU` 的所有核心，它利用单线程程序运行中部分代码段会因为等待 `I/O`（例如发送http请求和等待http答复）而占用 `CPU` 时间的
特点，同时启动多个 `goroutine` 来最大程度上占用 `CPU` 时间，从而提高程序的运行效率。
