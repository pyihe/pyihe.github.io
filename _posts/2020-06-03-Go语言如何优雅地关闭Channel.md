---
layout: post
title: '[译]Go语言如何优雅地关闭channel'
date: 2020-06-03
author: pyihe
tags: [Golang, 译文]
---

## 前言

这是一篇译文，网上有很多关于这篇文章的翻译，但这并不影响自己想自己翻译这篇文章的的愿望，毕竟自己看重的是自己实践翻译这样一件事情，并且从中得到收获。

## 原文介绍

原文来自[Go101](https://github.com/go101/go101)，该项目被托管在Github上。

原文链接为[How to Gracefully Close Channels](https://go101.org/article/channel-closing.html)，如果可以，阅读原文是最好的选择。

## 译文内容

几天前我写了一篇解释[Go channel规则](https://go101.org/article/channel.html)的文章，在[reddit](https://www.reddit.com/r/golang/comments/5k489v/the_full_list_of_channel_rules_in_golang/)和[HN](https://news.ycombinator.com/item?id=13252416)上这篇文章获得了许多赞同，但是仍然存在很多对Go channel设计细节的批判声。

我收集了一些关于Go channel的以下几点设计以及规则的批判：  
1. 在没有设定状态标志的情况下，没有简便的方法去判断一个channel是否已经关闭 
2. 调用者不知道channel是否已经关闭的情况下去执行关闭操作是很危险的，因为关闭已关闭的channel会造成panic
3. 调用者不知道channel是否已经关闭的情况下向channel发送数据是很危险的，因为向已关闭的channel发送数据会造成panic

这些批判听起来很合理（事实上并不）。是的，确实没有判断一个channel是否已经被关闭了的内置函数。

如果你可以确定没有数据以后也不会有数据发送至一个channel的话，确实有一个简单的方法去检查channel是否是关闭的。这个方法在[上一篇文章](https://go101.org/article/channel-use-cases.html#check-closed-status)。这里，为了更好的统一，该方法再次被列在了下面的例子中。
```go
package main

import "fmt"

type T int

func IsClosed(ch <-chan T) bool {
    select {
    case <-ch:
        return true
    default:
    }

    return false
}

func main() {
    c := make(chan T)
    fmt.Println(IsClosed(c)) // false
    close(c)
    fmt.Println(IsClosed(c)) // true 
}
```
正如上面提到的，这并不是判断一个channel是否关闭的通用方法。

实际上，即使有一个简单的内置函数`closed`检查channel是否已经被关闭，但是它的作用是非常局限的，就像用于获取当前channel缓冲中的数据个数的内置函数`len`一样。原因是在一次内置函数调用刚返回后被校验的channel的状态有可能改变，所以返回值已经不能反应被校验channel的最新状态。虽然当`closed(ch)`返回`true`时停止向`ch`发送数据没问题，但是如果`closed(ch)`返回`false`，则关闭channel或者继续发送数据到channel是不安全的。

### channel关闭准则

使用Go channel的一个基本准则是***不要从receiver侧关闭channel，如果channel有多个并发的senders时也不要关闭***。换句话说，我们应该只在只有一个sender时，在sender侧关闭channel。

（下面，我们将上面的准则称之为***channel关闭准则***）

当然，这不是一个关闭channel的通用准则。通用准则是***不要关闭（或者发送数据到）已经关闭的channel***。如果我们能保证不再有协程关闭或者发送数据到一个没关闭和非空的channel，此时协程便可安全的关闭channel。然而，靠channel的一个receiver或者某一个sender达到这样的保障是需要很大的努力的，并且通常会使代码变得复杂。正相反，坚持上面提到的***channel关闭准则***更容易。

### 粗暴地关闭channel

如果你无论如何要从receiver侧或者多个sender中的一个关闭channel，你可以使用[恢复机制](https://go101.org/article/control-flows-more.html#panic-recover)阻止可能的panic让你的程序崩溃。这里有一个例子（假设channel元素类型是T）。
```go
func SafeClose(ch chan T) (justClosed bool) {
    defer func() {
        if recover() != nil {
        //在defer函数调用中，返回值可以被改变
        justClosed = false
        }
    }()

    //这里假设ch != nil
    close(ch)   // 如果ch已经关闭了会造成panic
    return true // 等价于给justClosed赋值true，然后返回
}
```
这个方法明显违背了***channel关闭准则***

同样的想法可以用在向一个有可能已关闭的channel发送数据的情形。
```go
func SafeSend(ch chan T, value T) (closed bool) {
    defer func() {
        if recover() != nil {
            closed = true
        }
    }()

    ch <- value  // ch已关闭的话会造成panic
    return false // <=> 等价于给closed赋值false，然后返回
}
```
粗暴的解决方法不仅仅违背了***channel关闭准则***，并且程序有可能出现数据竞争。

### 礼貌地关闭channel

许多人更喜欢用`sync.Once`关闭channel：
```go
type MyChannel struct {
    C    chan T
    once sync.Once
}

func NewMyChannel() *MyChannel {
    return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
    mc.once.Do(func() {
        close(mc.C)
    })
}
```
当然，我们也可以用`sync.Mutex`来避免多次关闭channel:
```go
type MyChannel struct {
    C      chan T
    closed bool
    mutex  sync.Mutex
}

func NewMyChannel() *MyChannel {
    return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    if !mc.closed {
        close(mc.C)
    	mc.closed = true
    }
}

func (mc *MyChannel) IsClosed() bool {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    return mc.closed
}
```
这些方法可能很礼貌，但是它们可能无法避免数据竞争。目前，当channel关闭操作和发送操作同时发生时Go不保证没有数据竞争发生。如果同一个channel的`SafeClose`函数和发送操作同时发生，有可能发生数据竞争（虽然这种数据竞争通常不会带来多大伤害）。

### 优雅地关闭channel

上面的`SafeSend`函数的一个缺点是，它的调用不能像在`select`语句块`case`关键字中的发送操作那样使用。`SafeSend`和`SafeClose`函数的另一个缺点就是很多人，包括我，可能会觉得上面使用`panic`/`recover`和`sync`包的方法并不优雅。接下来，将会介绍一些针对所有场景，纯净的，不使用`panic`/`recover`和`sync`包的channel使用方法。

（在接下来的例子中，通过使用`sync.WaitGroup`让例子变得完整，在实际练习中使用它并不总是必要的。）

**1. M个receiver，一个sender，sender通过关闭数据channel来终止数据传输**

这是最简单的情形，当sender不再想发送数据时让它关闭channel即可。
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)
    
    // ...
    const Max = 100000
    const NumReceivers = 100

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int)

    // the sender
    go func() {
    	for {
            if value := rand.Intn(Max); value == 0 {
            //唯一的sender可以在任何时候安全的关闭channel
            close(dataCh)
                return
            } else {
                dataCh <- value
            }
        }
    }()

    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func() {
            defer wgReceivers.Done()
    
            //接收数据直到数据channel关闭
            //并且dataChan的缓冲队列变空
            for value := range dataCh {
                log.Println(value)
            }
        }()
    }

    wgReceivers.Wait()
}
```
**2. 一个receiver，N个sender，receiver通过关闭额外的信号channel来告诉sender停止发送数据**

这是一个比之前一个稍微复杂一点的情形，我们不能让receiver关闭数据channel来停止数据传输，因为这样做会违背***channel关闭准则***。但是我们可以让receiver关闭一个额外的信号channel来通知所有的sender停止发送数据。
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const Max = 100000
    const NumSenders = 1000

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(1)

    // ...
    dataCh := make(chan int)
    stopCh := make(chan struct{})
    //stop channel是一个额外的信号channel
    //它的发送者是receiver channel
    //它的接收者是dataChan的sender

    // senders
    for i := 0; i < NumSenders; i++ {
        go func() {
            for {
                //接收操作尝试尽可能早的退出协程
                //具体到这个例子，这并不是必要的
                select {
                case <- stopCh:
                    return
                default:
                }

                //即使stopChan被关闭了，
                //如果向dataChan的send操作没阻塞
                //几次循环后第二个select语句块中的第一个case可能仍然不会被执行
                //但是在这个例子中是可以接受的，所以上面的第一个select语句可以被忽略
                select {
                case <- stopCh:
                    return
                case dataCh <- rand.Intn(Max):
                }
            }
        }()
    }

    // the receiver
    go func() {
    	defer wgReceivers.Done()
    
    	for value := range dataCh {
            if value == Max-1 {
                //dataChan的receiver同样是stopChan的sender
                //这里关闭stopChan的操作是安全的
                close(stopCh)
                return
            }

            log.Println(value)
        }
    }()

    // ...
    wgReceivers.Wait()
}
```
正如注释中提到的，信号channel的发送者是数据channel的receiver。信号channel是靠它唯一的sender来关闭的，这坚持了***channel关闭准则***。

在这个例子中，`dataChan`一直没有被关闭。没错，channel不是必须被关闭，如果一个channel不再有协程引用了，不管是否被关闭，最终都会被垃圾回收器回收。所以这里关闭channel的优雅之处就是不关闭channel。

**3. M个receiver，N个sender，它们中的任何一个通过通知中间人来关闭额外的信号channel以停止数据传输**

这是最复杂的情形，我们不能让任何一个receiver或者sender关闭数据channel，也不能让任何一个receiver通过关闭信号channel来通知所有sender和所有receiver退出数据传输，任何一种做法都违背了***channel关闭准则***。然而，我们可以引进一个可以关闭信号channel的中间者角色，接下来的例子里的技巧是如何利用尝试发送操作的机制来通知中间者关闭信号channel。
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
    "strconv"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const Max = 100000
    const NumReceivers = 10
    const NumSenders = 1000

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int)
    stopCh := make(chan struct{})
        //stopChan是一个信号channel
        //它的sender是下面出现的中间者协程
        //它的receiver是dataChan所有的sender和receiver
    toStop := make(chan string, 1)
        //toStop channel用来通知中间者协程关闭信号channel(stopCh)
        //它的sender是dataCh中的任何一个sender和receiver
        //它的receiver是下面出现的中间者协程
        //它必须是带缓冲的channel

    var stoppedBy string

    // 中间者协程
    go func() {
        stoppedBy = <-toStop
        close(stopCh)
    }()

    // senders
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(Max)
                if value == 0 {
                    //这里，尝试发送操作是为了通知中间人关闭信号channel
                    select {
                    case toStop <- "sender#" + id:
                    default:
                    }
                    return
                }

                //这里的尝试接收操作是为了尽可能早的退出sender协程。
                //尝试接收和尝试发送操作的select语句块是被Go编译器特别优化了的，
                //所以它们是很高效的
                select {
                case <- stopCh:
                    return
                default:
                }

				//即使stopCh被关闭了，
				//在dataCh的发送端没有阻塞的情况下，
				//这里select中的第一个case经过几轮循环可能仍然不会被执行（理论上是永远）。
				//如果不能接受这样，那么上面的尝试接收操作就是必须的
                select {
                case <- stopCh:
                    return
                case dataCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }

    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func(id string) {
            defer wgReceivers.Done()

            for {
				//跟sender一样，这里的尝试接收操作也是为了尽可能早的退出接收协程
                select {
                    case <- stopCh:
                        return
                    default:
                }
                
                //即使stopCh被关闭了，
                //在dataCh的接收端没有阻塞的情况下，
                //这里select中的第一个case经过几轮循环可能仍然不会被执行（理论上是永远）。
                //如果无法接受这样，那么上面的尝试接收操作就是必须的
                select {
                    case <- stopCh:
                        return
                    case value := <-dataCh:
                        if value == Max-1 {
                        //这里同样的技巧被用来通知中间者关闭信号channel
                            select {
                                case toStop <- "receiver#" + id:
                                default:
                            }
                            return
                        }

                        log.Println(value)
                }
            }
        }(strconv.Itoa(i))
    }

    // ...
    wgReceivers.Wait()
    log.Println("stopped by", stoppedBy)
}
```
在这个例子中，仍然坚持了***channel关闭准则***。

请注意，`toStop` channel的缓冲大小是1，这是为了如果在中间者协程准备好从`toStop`接收通知之前第一个通知就已经发送过来时避免错过通知。

我们也可以设置`toStop`的容量大小为sender和receiver的数量之和，这样我们就不需要带有尝试发送机制的`select`语句块的来通知中间人了。
```go
...
toStop := make(chan string, NumReceivers + NumSenders)
...
value := rand.Intn(Max)
if value == 0 {
    toStop <- "sender#" + id
    return
    }
...
if value == Max-1 {
    toStop <- "receiver#" + id
    return
}
...
```

**4. 不一样的"M个receiver，一个sender"情形：由第三方协程发起关闭channel的请求**

有时候需要第三方发出关闭信息，对于这样的需求，我们可以利用额外的信号channel去通知sender关闭数据channel。例如：
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)
    
    // ...
    const Max = 100000
    const NumReceivers = 100
    const NumThirdParties = 15
    
    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)
    
    // ...
    dataCh := make(chan int)
    closing := make(chan struct{}) // 信号channel
    closed := make(chan struct{})
    
	//stop函数可以被安全的调用多次
    stop := func() {
        select {
        case closing<-struct{}{}:
            <-closed
        case <-closed:
        }
    }
	
	//第三方协程
    for i := 0; i < NumThirdParties; i++ {
        go func() {
            r := 1 + rand.Intn(3)
            time.Sleep(time.Duration(r) * time.Second)
            stop()
        }()
    }
    
    // the sender
    go func() {
        defer func() {
            close(closed)
            close(dataCh)
        }()
    
        for {
            select{
            case <-closing: return
            default:
            }
    
            select{
            case <-closing: return
            case dataCh <- rand.Intn(Max):
            }
        }
    }()
    
    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func() {
            defer wgReceivers.Done()
    
            for value := range dataCh {
                log.Println(value)
            }
        }()
    }
    
    wgReceivers.Wait()
}
```
`stop`函数中应用的想法是学习自Roger Peppe的一条评论[评论](https://groups.google.com/forum/#!msg/golang-nuts/lEKehHH7kZY/SRmCtXDZAAAJ)

**5. 不一样的"N个sender"情形：为了告诉receiver数据传输已经结束，必须关闭数据channel**

在之前提到的N个sender的情形中，为了坚持***channel关闭准则***，我们避免了关闭数据channel。然而，有时候为了让receiver知道数据传输已经完成，最终需要关闭数据channel。对于这种情形，我们可以通过使用一个中间channel将N个sender的情形转换成一个sender的情形，这个中间channel只有一个sender，所以我们可以通过关闭它来替代关闭原始的数据channel。
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
    "strconv"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)
    
    // ...
    const Max = 1000000
    const NumReceivers = 10
    const NumSenders = 1000
    const NumThirdParties = 15
    
    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)
    
    // ...
    dataCh := make(chan int)     // 永远不会关闭
    middleCh := make(chan int)   // 将会被关闭
    closing := make(chan string) // 信号channel
    closed := make(chan struct{})
    
    var stoppedBy string
    
	//stop函数可以被安全的调用多次
    stop := func(by string) {
        select {
        case closing <- by:
            <-closed
        case <-closed:
        }
    }
    
	//中间channel的协程
    go func() {
        exit := func(v int, needSend bool) {
            close(closed)
            if needSend {
                dataCh <- v
            }
            close(dataCh)
        }
    
        for {
            select {
            case stoppedBy = <-closing:
                exit(0, false)
                return
            case v := <- middleCh:
                select {
                case stoppedBy = <-closing:
                    exit(v, true)
                    return
                case dataCh <- v:
                }
            }
        }
    }()
    
	//一些第三方协程
    for i := 0; i < NumThirdParties; i++ {
        go func(id string) {
            r := 1 + rand.Intn(3)
            time.Sleep(time.Duration(r) * time.Second)
            stop("3rd-party#" + id)
        }(strconv.Itoa(i))
    }
    
    // senders
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(Max)
                if value == 0 {
                    stop("sender#" + id)
                    return
                }
    
                select {
                case <- closed:
                    return
                default:
                }
    
                select {
                case <- closed:
                    return
                case middleCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }
    
    // receivers
    for range [NumReceivers]struct{}{} {
        go func() {
            defer wgReceivers.Done()
    
            for value := range dataCh {
                log.Println(value)
            }
        }()
    }
    
    // ...
    wgReceivers.Wait()
    log.Println("stopped by", stoppedBy)
}
```

#### 其他情形？

应该还有更多的变化情形，但在上面出现的情形是最平常、最基础的情形。通过聪明地使用channel（和其他并发编程技术），我相信每个变化的情形都能找到一个坚持***channel关闭准则***的解决方案。


### 疑问

没有情形会强迫你违背***channel关闭准则***，如果你遇到了这样的情形，请重新思考你的设计并重构你的代码。


用Go语言编程就像是在进行艺术创作。