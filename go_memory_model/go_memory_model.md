# Go memory model

## Advice

程序中再修改由多个 goroutine 同时访问的数据时，必须要**序列化这类访问**！

## Happens Before

在单个的 goroutine 中，所有的程序运行顺序都是按照程序的顺序来执行的。
只有当重新排序不会改变 goroutine 中由语言规范决定的行为时，编译器和处理器才可以重新排序在单个goroutine中执行的读写操作。

为了指定读取和写入的要求，我们定义 *happens before*，Go 程序中执行内存操作的部分顺序。

在单个的 goroutine 中，*happens-before order* 就是程序上变现出的顺序。

一次对变量 v 的读取动作 r **允许**观测到 一次对 v 的写入动作 w 的结果，需要满足下面两个条件：

1. r 不能在 w 之前发生

2. 在 w 动作后，在 r 动作前，不能有其他的对 v 的写入操作 w‘

而 r **保证**能够观察到 w 操作的条件是：

1. w 在 r 之前发生

2. 在操作 w 和 r 之间，不能发生任何对共享变量 v 的写操作

后者的要求比前者更加严格，在 w 和 r 之间不能有任何的并发写操作。

在单个的 goroutine 中，因为没有并发，因此两者时等价的。但是当多个 goroutinue 访问同一个共享变量 v 时，就必须用 *synchronization event* 来建立 *happens-before 条件* 来确保读操作能够读取期望的写操作结果。

对变量的初始化 0 值操作也是一种写操作。

读或者写操作的对象如果大于一个机器字就会表现为**未指定顺序的多机器字操作**

## Synchronization

### Initialization

程序的初始化会启动一个单个 goroutine, 然后这个 goroutine 可能会创建其他的 goroutine 来同时运行。

**重要：** 如果一个包 p 中引入了另一个包 q, q 的 `init` 函数会在 p 的所有操作之前进行。

主程序的入口函数 `main.main` 会在所有的 `init` 函数执行完毕后开始运行。

### Goroutine creation

go 关键字声明开始一个新的 goroutinue 是发生在这个 goroutine 执行之前。例如：

```Go
var a string

func f() {
    print(a)
}

func hello() {
    a = "hello, world"
    go f()
}
```

调用 hello 函数会在未来的某个时刻打印对应的输出(可能是在 hello 函数返回之后)

### Goroutine destruction

goroutine 的退出发生的时间都是随机的。例如：

```Go
var a string

func hello() {
    go func() { a = "hello" }()
    print(a)
}
```

这里，a 变量赋值并不会在任何的同步事件之后，因此不能保证其他的 goroutine 能够观测到该变量的变化。

如果一定需要这个变化被其他的 goroutine 观测到，那么必须使用同步机制例如 lock 或者 channel来建立相对的顺序。

### Channel communication

channel 间的通信是 goroutine 之间保持同步的最主要方法。
通常在不同的 goroutine 之间，每次一个 goroutine 向一个 channel 发送信息就会对应另外一个 goroutine 从这个 channel 中拿信息。

**向 channel 发送信息总是要在与其对应的 channel 接受信息 complete 之前**。

例如下面这个程序：

```Go
var c = make(chan int, 10)
var a string

func f() {
    a = "hello, world"
    c <- 0 // close(c)
}

func main() {
    go f()
    <-c
    print(a)
}
```

上面这个程序能够保证打印出 "hello, world"。 对 a 变量的写入操作在向 c 发送消息之前，因此确定会发生在对应的接受 c 消息完成之前。

**channel 的关闭会发生在 channel 接收之前，在关闭后会从该 channel 收到 0值， 因为 channel 已经关闭了**。

根据上面这个规则，将上面例子程序中的 `c <- 0`  替换成 `close(c)` 能够保证程序的行为相同。

**一个无缓存的 channel 的接收会发生在那个 channel 的发送 complete 之前**。

例子程序如下：

```Go
var c = make(chan int)
var a string

func f() {
    a = "hello, world"
    <-c
}

func main() {
    go f()
    c <- 0
    print(a)
}
```

**简单的总结**：发的时候没有人收，那么发的就阻塞；收的时候没有人发，那么发的就阻塞

上面的程序也能够保证打印出 "hello world"。对变量 a 的写入发生在 c channel 信息的接受之前，接收 c channel 的信息又发生在 c channel 的 complete 之前，因此能打印出消息。

如果上面的 channel 换成一个有缓存的 channel(例如 `c = make(chan int， 1)`)那么就不一定能打印出信息，因为多个 goroutine 之间的执行顺序不确定。

**一个 channel 的 第 k 个大小为 C 的接收信号会在第 k + C 个发送信号完成之前**。

这条规则是针对前面的有缓冲 channel 规则的概括。
这条规则使得将一个有缓冲的 channel 作为一个计数信号(goroutine 池)成为可能：
在 channel 中的对象与池子中正在使用的 goroutine 相关，
channel 的 capacity 就是池子同时进行任务的最大值，
发送一个对象(启动一个 goroutine)就会占用一个空间，而从 channel 中接收一个值就会释放一个空间。

例子程序：

```Go
// the goroutines coordinate using the limit channel to ensure that at most three are running work functions at a time
var limit = make(chan int, 3)

func main() {
    for _, w := range work {
        go func(w func()) {
            limit <- 1
            w()
            <-limit
        }(w)
    }
    select{}
}
```

### Locks

对于任何 `sync.Mutex` or `sync.RWMutex` 变量 l, n < m, 调用第n个 `l.UnLock()` 会发生在第 m 个 `l.Lock()` 之前.例子程序：

```Go
var l sync.Mutex
var a string

func f() {
    a = "hello, world"
    l.Unlock()
}

func main() {
    l.Lock()
    go f()
    l.Lock()
    print(a)
}
```

先调用 f() 函数中的 l.UnLock() 然后再调用 main 中的第二个 l.Lock() 确保执行顺序正确。

对于 `sync.RWMutex` 类型的变量 l, 第 n 个 l.RLock()调用会在第 n 个 l.UnLock() 调用之后发生(return);
与之对应的 l.RUnlock() 会在 第 n + 1 个 l.Lock() 之前调用。

### Once

sync 包提供了一种在多个 goroutine 中的安全初始化机制，利用 Once 类型。
多个线程都可以执行 once.Do(f), 但是只有一个会运行 f() 函数，而其他的都会阻塞直到 f() 调用结束。

**单个由 Once 类型嗲用的 f() 函数发生在任何 `once.Do(f)` 返回之前**。

例子:

```Go
package main

import (
    "fmt"
    "sync"
)

var a string
var once sync.Once

func setup() {
    a = "hello, world"
    fmt.Println("init")
}

func doprint() {
    once.Do(setup)
    fmt.Println(a)
}

func twoprint() {
    go doprint()
    go doprint()
}

func main() {
    twoprint()
    select{}
}
```

在上面的程序中 twoprint 的两个线程同时调用 doprint，
但是 doprint 中的 once.Do(setup) 只会执行一次。
