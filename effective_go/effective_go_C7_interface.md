# Interface and other types

## Interfaces

Go 语言中 interface 用来指定一个对象的行为(方法): **如果某个类型具有指定的方法，那么就能够调用**。
之前的例子：一个对象有 String 方法就能够实现自定义 printer(data/print 章节中的例子)，
Fprintf 可以使用 Write 方法生成输出(method章节中的例子)

一个类型能够同时实现多个接口。例如一个 Sequence 类型，同时实现
`sort.Interface` (包含3个方法: `Len()`, `Less(i, j int) bool`, and `Swap(i, j int)`)
 和 自定义格式化接口:

 ```Go
type Sequence []int

// Methods required by sort.Interface.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Method for printing - sorts the elements before printing.
func (s Sequence) String() string {
    sort.Sort(s)
    str := "["
    for i, elem := range s {
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
 ```

## Conversions(类型转换)

针对上面的 Sequence 类型，能够用 `[]int()` 方法将其转换成 `int slice`:

```Go
func (s Sequence) String() string {
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

能够转换的原因是：
本质上其实 Sequence 和 []int 是**同一类型**的，这样在转换时不会重新创建一个值对象，而只是简单的转换了类型。
另外 int 也能够转换成 float，但是这样转换时会自动复制一个新的值。

Go程序中的一个习惯用法是转换表达式的类型以访问不同的方法集, 例如用 `sort.IntSlice` 来转换:

```Go
type Sequence []int

// Method for printing - sorts the elements before printing
func (s Sequence) String() string {
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

上面这个例子，利用了数据对象的多态性(`Sequence, sort.IntSlice and []int`)，省去了自己实现各种方法。

## Interface conversions and type assertions

`Type Switch` 是类型转换的一种形式: 对每一个 switch 条件，将对象的接口转换成那个条件中的类型。
下面是一个简略版的 `fmt.Printf` 的实现例子：

```Go
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

当对象类型本来就是 string 时则直接返回；
如果不是，那么就**将对象转换为 Stringer 接口**，并调用其 String 方法做返回

单个对象的类型转换可以用 `value.(typeName)` 完成,
**在转换时 `value` 对象必须是一个 `interface` 不能是具体的值类型(包括 `value.(type)`也是一样)**

例子: [Cannot type switch on non-interface value](https://stackoverflow.com/questions/23172219/cannot-type-switch-on-non-interface-value)

类型转换中，指定的类型名必须是 `interface` 中有的类型，或者可以转换为值类型的 `interface`

类型转换时能够**确保安全的做法**：

```Go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

上面的例子中，如果转换失败，那么 `str` 会是一个0值 string

## Generality(通用性)

如果某种 type 仅用于实现一个特定的 interface，并且永远不会有超出该 interface 的**其他方法**，则无需导出该类型本身。仅导出该接口能让我们更专注于其行为而非实现，其它属性不同的实现则能反映该原始类型的行为。 这也能够避免为每个通用接口的实例重复编写文档。

在这种情况下，构造函数应返回 interface 而不是具体的类型实现。例如，在散列库中，`crc32.NewIEEE` 和`adler32.New`都返回接口类型 `hash.Hash32`。 在Go程序中用 `Adler-32` 替换 `CRC-32` 算法只需要改变构造函数调用; 其余代码不受算法更改的影响。

类似地，在 `crytpo` 包中的  streaming cipher algorithms 能够与它们链接在一起的 block ciphers 分离。

`crypto/cipher` 包中的 `Block` 接口定义了 block ciphers 的行为(方法), 对单个 block data 提供了一种编码方法。 然后，通过类比 `bufio` 包，可以使用实现此接口的 `cipher` 包来构造 stream ciphers，由 `Stream` 接口表示，而不需知道 block ciphers 中的编码方法。

`crypto/cipher` 中的 interface 如下：

```Go
type Block interface {
    BlockSize() int
    Encrypt(src, dst []byte)
    Decrypt(src, dst []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}

// counter mode (CTR) stream 的定义如下，用来将 block cipher 转换为 streaming cipher

// NewCTR returns a Stream that encrypts/decrypts using the given Block in
// counter mode. The length of iv must be the same as the Block's block size.
func NewCTR(block Block, iv []byte) Stream
```

`NewCTR` 方法不单单只是用于指定的编码算法和数据源，而是对任何实现了 `Block` 和 `Stream` 接口的对象实例。因为 `NewCTR` 返回 `interface` 类型，所以将CTR加密替换为其他加密模式是**本地化的更改**。 必须编辑构造函数来实现，但由于周围的代码必须仅将结果视为Stream，因此不会注意到实现上的差异。

**该节的大意参照了[effective_go 翻译](https://github.com/bingohuang/effective-go-zh-en/blob/master/11_Interfaces_and_other_types.md)**

## interfaces and methods

因为任何的类型都能够附加方法，这也意味着任何的类型都能够满足一个 interface(实现interface 中的方法)。以 `http` 包中的 `Handler` 接口为例，任何实现了 `Handler` 接口的实现都能响应 Http requests:

```Go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

`ResponseWriter` 是一个实现了响应并返回客户端请求 methods 的 interface。
在这些 methods 中，也包含标准的 `Write` 方法，
因此在任何用到 `io.Writer` 的地方也能够使用 `http.ResponseWriter`。

以下是一个简单但完整的处理程序实现，用于计算访问页面的次数:

```Go
// Simple counter server.
// struct sample
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}

// int sample
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}

// channel sample
// A channel that sends a notification on each visit.
// (Probably want the channel to be buffered.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}

// func sample
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}

// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}

http.Handle("/args", http.HandlerFunc(ArgServer))
```

**注意:** struct sample 中，因为实现了 `Write` 方法，因此 `http.ResponseWriter` 能够在 `Fprintf` 中作为参数调用。

在本节中，用 struct，int，channel 和 func 创建了对应的HTTP服务器，因为 interface 只是与其对应的 methods，因此 interface 可以为（几乎）任何类型定义。
