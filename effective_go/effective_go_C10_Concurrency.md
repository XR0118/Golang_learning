# Concurrency

在 Go 语言中，共享变量都是通过 channel 来传递。
任意时间都只有一个 goroutine 能够访问该值。通过这样的设计避免了数据竞争(data races)。

## Goroutines

goroutine 的模型：一个与其他 goroutines 在同一个地址空间中同时执行的函数.
通过在函数或者 methods 前面加上 `go` 关键字来创建一个新的 goroutine。
当调用结束时，这个 goroutine 会自动退出。简单的例子:

```Go
go list.Sort()  // run list.Sort concurrently; don't wait for it.

// 在调用中使用匿名函数(function literals)
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // 注意最后的括号，函数定义完之后必须要调用
}
```

在 Go 中，function literals 是闭包，其实现保证了在 function literals 被调用且调用结束前，传入变量存活

## Channels

channels 由 make 关键字创建，最终的值是指向其底层数据结构的引用。
如果 make 时加上了可选的 int 参数，那么会给这个 channel 加上一个对应大小的 buffer, 默认为0(即同步 channel)。 channel 创建的例子:

```Go
ci := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
```

unbuffered channel 将通信(值的交换)与同步相结合(保证两个计算（goroutine）处于已知状态)。

channel使用的简单例子:

```Go
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```

receiver 始终阻塞，直到有数据要接收。
如果 channel 未缓冲，则 sender 将阻塞，直到接收方收到该值。
如果 channel 有缓冲区，sender 只会在向缓冲区拷贝值的时候阻塞；
如果缓冲区满了，则表示等待某个 receiver 取回值。

可以像信号量一样使用缓冲信道，例如限制吞吐量。
在下面的这个例子中，获取到的 request 被传到 handle 函数中进行性处理,处理时会传入一个信号量到 sem channel中，最后再从 sem channel 中接受一个信号量，表示处理完一次，可以传入下一个。这样，就通过 sem channel 的缓冲限制了 process 的调用数量。

```Go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // Wait for active queue to drain.
    process(r)  // May take a long time.
    <-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // Don't wait for handle to finish.
    }
}
```

一旦有 MaxOutstanding 个 handler 在执行 process 时，再尝试往该 channel 发送信号就会阻塞，直到该 channel 中的一个 process 调用完成，并且 channel 的一个信号被 receive

**阻塞情况总结：** 1) channel 接受时，直到有数据要接受之前一直阻塞；2) channel 要向外发送数据，但是 channel 中没有数据时会一直阻塞；3) channel 的缓冲区满了，再向该 channel 写数据时会阻塞

**注意：** 在 Go 的 for 循环中， 循环变量在每次迭代中会重复使用，因此 req 变量会在所有的 goroutine 中共享。错误例子如下:

```Go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}

// 解决方法1: 匿名函数中传入变量
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}

// 解决方法2: 循环中重新定义新的变量, 通过同名变量来隐藏外层的循环变量
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // Create new instance of req for the goroutine.
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

## channels of channels

channel 是一个 first-class 的值类型，和其他类型一样能够在内存中申请或者传递。
该属性的常见用途是**实现安全的并行解复用**。

完整的例子:

```Go
package main

import (
    "fmt"
)

type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int // channel inside the request object on which to receive the answer
}

func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}


func main() {
    clientRequests := make(chan *Request)
    go handle(clientRequests)
    request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
    // Send request
    clientRequests <- request
    // Wait for response.
    fmt.Printf("answer: %d\n", <-request.resultChan)
}
```

## Parallelization

利用 channel 和多核 CPU 进行并行计算的例子:

```Go
type Vector []float64

// Apply the operation to v[i], v[i+1] ... up to v[n-1].
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // signal that this piece is done
}

const numCPU = 4 // number of CPU cores

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // Buffering optional but sensible.
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // Drain the channel.
    for i := 0; i < numCPU; i++ {
        <-c    // 每次计算完成就会返回一个信号，直到返回4个则计算完成
    }
    // All done.
}
```

另外可以用 `runtime.NumCPU()` 获取硬件的 CPU 核数。
或者可以使用 `runtime.GOMAXPROCS` 来设置(或者读取)用户指定的该 Go 程序能够使用的 CPU 核数。 传入参数 0 可以查询可用的核心数量 `var numCPU = runtime.GOMAXPROCS(0)`

## A leaky buffer

例子:

```Go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // Grab a buffer if available; allocate if not.
        select {
        case b = <-freeList:
            // Got one; nothing more to do.
        default:
            // None free, so allocate a new one.
            b = new(Buffer)
        }
        load(b)              // Read next message from the net.
        serverChan <- b      // Send to server.
    }
}

func server() {
    for {
        b := <-serverChan    // Wait for work.
        process(b)
        // Reuse buffer if there's room.
        select {
        case freeList <- b:
            // Buffer on free list; nothing more to do.
        default:
            // Free list full, just carry on.
        }
    }
}
```
