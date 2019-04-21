# Embedding

Go 通过在 struct 或 interface 中嵌入 types，来"借用"其他类型的实现。

简单的例子： `io.Reader`, `io.Writer` 和 `io.ReadWriter` 的定义

```Go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ReadWriter is the interface that combines the Reader and Writer interfaces.
type ReadWriter interface {
    Reader
    Writer
}
```

通过在 ReadWriter 同时嵌入 Reader 和 Writer interface 同时实现读和写。

**注意:**

1. 嵌入的 interface 中，各个 interface 的 methods 不能有交集

2. interface 中只能嵌入 interface

同样以上的“嵌入”的概念也适用于 struct，而且约束更少。
例子：bufio 包中的 `bufio.Reader` and `bufio.Writer`，这两个对象实现了 buffered reader/writer, 通过"嵌入"可以得到 ReadWriter：

```Go
type ReadWriter struct {
    reader *Reader // *bufio.Reader
    writer *Writer // *bufio.Writer
}
```

通过这样的嵌入之后，不需要在新的 struct 上绑定对应 interface 的方法.
这意味着: ReadWriter(新的结构体) 不仅有子结构体的方法,也同时满足了三种 interface —— `io.Reader, io.Writer, and io.ReadWriter`

**嵌入和子类之间的区别**：

在嵌入一个类型时，被嵌入的类型的方法会变成其外层类型的方法，但是，当调用这个方法时，真正的 method receiver 还是被嵌入的这个类型，而不是外层类型。

此示例显示嵌入字段以及常规命名字段：

```Go
type Job struct {
    Command string
    *log.Logger
}

// 通过嵌入 Job 类型获得了 *log.Logger 的 Log, Logf等方法
job.Log("starting now...")

// 同时也能够定义构造函数
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}

// 或者通过花括号直接初始化
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}

// 如果我们需要直接引用嵌入字段，则忽略 package name 字段的类型名称将用作字段名称, 在本例中如果要获取 job 对象中的 *log.Logger字段，可以直接写 job.Logger
func (job *Job) Logf(format string, args ...interface{}) {
    job.Logger.Logf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

嵌入类型会导致**命名冲突**的问题，其解决的方案为：

1. 外层的属性或者方法占主导地位

2. 如果在同一层级有命名冲突，一般会报错。但是当这样同级重复命名没有在外层的程序中使用过时，就不会出错。这样做是为了保护外部类型嵌入导致的更改；一个会导致冲突的字段加入到了某个类型中，如果这两个冲突的字段都没有被用到，那么就不会发生错误
