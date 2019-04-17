# Effective go

## Commentary

1. 每个 package 应该有自己对应的 package comment

## Names(命名规范)

### Package names

命名原则：short, concise, evocative；全小写，单个单词；

包的 Getter 和 Setter 命名： Getter 应该用首字母大写的包名(如包为 owner, 则 Getter 命名为 Owner)；Setter 则为 Set + 首字母大写的包名(如 SetOwner)

### interface name

接口命名一般在方法名后面加上 -er 后缀，如: Reader, Writer, Formatter, CloseNotifier

### MixedCaps

驼峰命名法在 Go 语言中更加常用

## Control structures(逻辑控制语句)

### if

在 if 语句中能够接受初始化语句声明，因此常见的用法为:

```Go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

if 的 body 中使用 `break, continue, goto, or return` 关键字能够省略不必要的 else 语句

```Go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

if 语句常用于错误处理，当一个连续的函数调用中任何地方发生了错误，那么后面的逻辑就没有必要继续执行

```Go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

### Redeclaration and reassignment(重新申请和重新分配)

`:=` 声明中即使已经声明了变量v，也可能出现，前提是：

* 该声明与v的现有声明的范围相同(如果在外层程序段中已经有 v 的声明，那么里面的这个声明会创建一个新的变量 v)

* 初始化中的相应值可分配给v，并且声明中**至少有一个其他变量被重新声明**

**重要**： 在Go中，**函数参数和返回值的范围与函数体相同**，即使它们在包围体的括号之外出现。

### for

for 循环的三种形式：

```Go
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

第一种方式能够轻松的访问带下标的变量 v:

```Go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

另外，访问可迭代的对象时可以用 `range` 关键字:

```Go
for key, value := range oldMap {
    newMap[key] = value
}

// 只需要用到 key

for key := range m {
    if key.expired() {
        delete(m, key)
    }
}

// 只用 value ，可以用 _ 做占位符

sum := 0
for _, value := range array {
    sum += value
}

```

对于一个 `string` 对象，`range` 关键字还能够解析UTF-8来分解单个 Unicode 编码。**错误的编码**会消耗一个字节并产生替换符**U+FFFD**。

Go 没有 `++` 和 `--` 运算符。在多个变量的 for 循环中，应该使用并行 `assignment`：

```Go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

### Switch

`switch` 的条件表达式不一定是 `int` 或者 常量，会从上到下扫描每一个条件直到找到匹配的条件

`default` 表示默认选项，另外 `switch` 中也可以用 `:=` 声明变量

```Go
func main() {  
    switch finger := 8; finger {//finger is declared in switch
    case 1:
        fmt.Println("Thumb")
    case 2:
        fmt.Println("Index")
    case 3:
        fmt.Println("Middle")
    case 4:
        fmt.Println("Ring")
    case 5:
        fmt.Println("Pinky")
    default: //default case
        fmt.Println("incorrect finger number")
    }
}
```

可以在一个 `case` 中包含多个表达式，每个表达式用逗号分隔。

```Go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

`break` 关键字能够提前终止 `switch` 的逻辑，同时如果想终止 `switch` 外层的 `loop` 循环，那么可以使用 `Loop` label 表示 `break` 的对象，例子：

```Go
Loop:
    for n := 0; n < len(src); n += size {
        switch {
        case src[n] < sizeOne:
            if validateOnly {
                break
            }
            size = 1
            update(src[n])

        case src[n] < sizeTwo:
            if n+1 >= len(src) {
                err = errShortInput
                break Loop
            }
            if validateOnly {
                break
            }
            size = 2
            update(src[n] + src[n+1]<<shift)
        }
    }
```

`switch` 中的表达式是可选的，可以省略。如果省略表达式，则相当于 `switch true`，这种情况下会将每一个 `case` 的表达式的求值结果与 `true` 做比较，如果相等，则执行相应的代码。

```Go
func main() {  
    num := 75
    switch { // expression is omitted
    case num >= 0 && num <= 50:
        fmt.Println("num is greater than 0 and less than 50")
    case num >= 51 && num <= 100:
        fmt.Println("num is greater than 51 and less than 100")
    case num >= 101:
        fmt.Println("num is greater than 100")
    }
}
```

在上面的程序中，`switch` 后面没有表达式因此被认为是 `switch true` 并对每一个 `case` 表达式的求值结果与 `true` 做比较。`case num >= 51 && num <= 100:` 的求值结果为 `true`，因此程序输出：`num is greater than 51 and less than 100` 。**这种类型的 switch 语句可以替代多重 if else 子句**。

`fallthrough` 关键字： Go 中默认每次只执行一个 case， 如果需要执行多个 case 的话，在每个 case 的最后一行加上 `fallthrough` 关键字能够继续执行下一个 `case` 语句。

### Type switch

`switch` 能够用来查看 interface 变量的类型。例子中 `switch

```Go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

补充资料: [Golang教程：（十）switch 语句](https://studygolang.com/articles/10856)

## Functions

### Multiple return values(多返回值)

### Named result parameters(返回值命名)

函数的返回值能够像传入参数一样直接在函数声明时直接命名。命名之后，在函数被调用时返回值会被默认初始化；如果函数执行不带参数的 return 语句，那么当前的结果值就会被返回。

### Defer

`defer` 关键字修饰的方法会在函数调用返回之前立即执行，能够有效的释放已经打开的资源，如文件、连接等，例子：

```Go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

defer 的优势：

1. 保证不会忘记关闭已经打开的资源句柄

2. close 调用可以紧挨着 open 调用

defer 函数的参数会在调用时计算，会在 **defer** 时计算，而不是 call 时。除了参数变化之外，还需要注意 defer 调用可能会延迟多个其他的函数调用， defer 函数调用时 LIFO 的(后进先出)。例子:

```Go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}

// 函数输出：4 3 2 1 0
// 第二个例子
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}

// result
// entering: b
// in b
// entering: a
// in a
// leaving: a
// leaving: b

```

defer 调用并不是基于 block(阻塞机制) 而是基于 function(函数调用机制的)

## Data

### Allocation with new

`new` 关键字 会申请内存空间，并且将这片空间初始化为0值(zeros)。因为这个特性因此不需要**初始化对象或者默认构造函数**。

`new(T)` 为类型 `T` 申请一个新的 zeroed storage(0值)，并返回其地址(*T)。其中 **T 为类型**。例子：

```Go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}

// 这两个变量声明完就能直接使用，不需要初始化
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

### Constructors and composite literals(构造函数和复合文字)

能够使用自己定义的构造函数代替0值初始化, `NewFile` 对象的构造函数例子:

```Go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    return &File{fd, name, nil, 0}
}
```

**在 `Go` 语言中函数体内获取的地址能够返回到函数体外**

或者可以用**键值对**的方式对一个对象初始化，例如 `&File{fd: fd, name: name}` 键值对中没有出现的属性会自动初始化为0值

当花括号初始化，括号中没有值时，与 new 表达式完全等价，如 `new(File)` 和 `&File{}`

花括号初始化也能够用于 slice, array 和 map, 其中 filed 域必须是唯一且符合对象初始化要求的。例子：

```Go
// 其中前 array 和 slice 的 field 域 应该是 int 类型
// map 花括号初始化时 field 域应该与 var 声明时一致
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

**总结**: `var` 关键字只是声明了变量，但是并不会为该变量申请内存空间；
而 `new` 关键字在声明的同时会为变量申请内存空间并创建0值对象(或者按构造函数初始化)，并返回改内存空间的地址

### Allocation with make

`make` 关键字用来创建 slices, maps 和 channels，并且返回一个初始化后的 `T` 对象(不是 `*T`)。
主要原因在于：这三种类型表示对使用前必须初始化的数据结构的引用。
因此，对于这三个数据结构，`make` 关键字初始化了它们的数据结构和值。例如:

```Go
make([]int, 10, 100)
```

申请了一个长度为 100 的 int 数组，然后创建一个长度为 10 容量为 100 的 `slice`，指向前10个数组元素。
相对的，如果用 `new([]int)` 的话只是返回一个指向新申请的、0长度的 `slice` 的指针，其实是一个指向空 `slice` 的指针(`&[]`)
例子解释:

```Go
var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

// Unnecessarily complex:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic:
v := make([]int, 100)
```

总结： `make` 关键字只用来创建 maps, slices 和 channels 对象，并且**不返回对象指针**。
可以在创建之后使用**取地址符** `&` 来显示获得对象的地址。

### Arrays

array 定义例子  `var a [10]int 或 primes := [6]int{2, 3, 5, 7, 11, 13}`
前者默认初始化0值，后者直接花括号初始化

Go 中的 array：

* array 是值类型，将 array 赋值给其他 array 会拷贝整个 array

* 函数传递时会将整个 array 复制，而不是传递指针

* array 的长度是 array 这个类型的一部分， 因此 `[10]int` 和 `[20]int` 是不同的。

值类型是方便，但是也是开销昂贵的(函数传递时会完全复制),使用 array 的例子:

```Go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
// pass address instead of copy array
x := Sum(&array)  // Note the explicit address-of operator
```

### Slices

Slice 有一个对底层 array 的引用，如果将一个 slice 赋值给另一个， 那么这两个 slice 将指向同样的 array。
如果函数将 slice 作为参数，将会传递指针，并且对元素的改变会在函数外层可见。以 `Read` 函数为例：

```Go
// func (f *File) Read(buf []byte) (n int, err error) Read函数定义
n, err := f.Read(buf[0:32])
```

只要切片仍然适合底层阵列的限制，就可以改变切片的长度。
slice 的 capacity 是指 slice 的最大长度，能够用 `cap()` 函数获得。
一个动态改变 slice capacity 的例子：

```Go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // Allocate double what's needed, for future growth.
        newSlice := make([]byte, (l+len(data))*2)
        // The copy function is predeclared and works for any slice type.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```

这个例子中，一定要返回 slice 变量， 因为 slice 是通过值传递而不是引用传递的

### Two-dimensional slices

必须自己定义二维数组类型，如:

```Go
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.
```

申请 2D slice 的两种办法：1.逐行申请，然后放到一个 2D slice 里面；
2.申请一个大 slice 存放所有的元素，然后再用循环分成 2D slice
一次一行的例子:

```Go
// Allocate the top-level slice.
picture := make([][]uint8, YSize) // One row per unit of y.
// Loop over the rows, allocating the slice for each row.
for i := range picture {
    picture[i] = make([]uint8, XSize)
}
```

一次性放到一个数组中，然后再切片:

```Go
/ Allocate the top-level slice, the same as before.
picture := make([][]uint8, YSize) // One row per unit of y.
// Allocate one large slice to hold all the pixels.
pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
// Loop over the rows, slicing each row from the front of the remaining pixels slice.
for i := range picture {
    picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

### Maps

Map 的 键值必须有**相等判断**的具体实现，与 slice 一样如果将 map 传入一个函数，那么对改map
的变化在函数的调用者的作用域中仍然存在。

Map 的初始化方式:

```Go
// 花括号初始化
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}

// make 初始化
timeZone := make(map[string]int)
```

尝试拿 map 中不存在的 key 时，会返回 value 类型的0值。
可以通过建立一个 value 为 bool 类型的 map 来实现 set 的功能。例子：

```Go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // will be false if person is not in the map
    fmt.Println(person, "was at the meeting")
}
```

可以通过 `seconds, ok = timeZone[tz]` 来判断 map 中是否含有指定的 key, 完整例子:

```Go
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

删除 map 中的键值对可以用 `delete` 函数，传入参数为 map 和想删除的 key。
**即使 map 中没有该 key, delete 操作也是安全的**

### Printing

几个 Print 函数的例子：

```Go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

其中 Fprint 的第一个参数需要一个实现了 io.Writer 接口的对象
Sprint 的功能是可以连接字符串并返回新的字符串

与 C 格式化输出的不同点：

1. 不需要用标志位或者输出格式大小标志，而是直接使用类型输出，例子：

```Go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))

// 结果：
// 18446744073709551615 ffffffffffffffff; -1 -1
```

使用 `%v` 的格式，能够打印初和 `Print` `Println` 一样的结果，且可以打印任何值

```Go
fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
// result: map[CST:-21600 PST:-28800 EST:-18000 UTC:0 MST:-25200]
```

当打印结构体的时候， **`%+v` 能够打印结构体的属性名和属性值**,
**`%#v` 则能够打印完整的 Go 完整格式化信息(包括类型名)**。

另外， `%T` 能够打印变量的类型

对自己的定制类进行格式化输出，需要在定制类中定义 String() string 签名函数。例如：

```Go
type T struct {
    a int
    b float64
    c string
}

func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

**如果需要同时打印 T 的值和指向 T 的指针，那么必须用 值传递 `func (t *T) String() string`**

另外，自定义的 String 方法也能够调用 Sprintf 来格式化字符串，但是需要注意的是:
**在 String 方法中调用 Sprintf 是不要造成循环调用。错误的例子如下：

```Go
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
    // correct method:
    // return fmt.Sprintf("MyString=%s", string(m))
}
```

### Append

append 的函数声明类似于：

```Go
func append(slice []T, elements ...T) []T
```

append 的简单例子：

```Go
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)

x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

`...` 占位符的**作用总结**:

* 在函数变量定义时 ：`... Type` 占位符意思是将该**变量视为一个 list**: 如 `a ...int` 其中 `a` 为一个 int array

* 在函数调用时，可以在 slice 后面加上 `...`, 这样表示将 slice 中的所有元素视为一个list变量(即一连串变量而不是一个 slice)

## Initialization(初始化)

### Constants

常量在编译的时候就直接创建，**即便是在函数中作为局部变量**,
并且只能是 numbers, characters (runes), strings or booleans。

定义常量的表达式必须时**常量表达式**，因为在编译的时候不涉及程序运行、函数调用

用 `iota` 创建常量和应用方式：

```Go
type ByteSize float64

const (
    _           = iota // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)

func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

**注意**：在 `ByteSize` 的 `String` 函数中调用 `Sprintf` 时安全的。
因为格式化字符为 `%f` 并不会发生 string 的转换

### Variables

变量初始化发生在程序运行时，定义方式：

var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)

### init function

每个源文件都可以定义自己的 init 函数来设置所需的状态。(也可以包含多个 init 函数)
init 函数是在所有的 packages 初始化完成后，并**对其中所有的变量声明初始化求值完成后**才调用的。

init函数的常见用途是在实际执行开始之前验证或修复程序状态的正确性。例子：

```Go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

## Methods

### Pointers vs. Values

任意一个命名的类型(除了 指针和 interface)都可以定义方法；
这个方法的接受者(也就是指定的类型)不一定是一个结构体。例子：

```Go
// 首先声明需要绑定方法的类型
type ByteSlice []byte

// 然后将这个方法绑定到指定的类型上(将该方法的接收器设置为指定类型)
// 下面这个是值类型传递的方法，为此需要将改变后的值返回
func (slice ByteSlice) Append(data []byte) []byte {
    // Body exactly the same as the Append function defined above.
}

// 通过指针传递的接收器可以避免返回值，直接改变原对象的值
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // Body as above, without the return.
    *p = slice
}
```

值传递 和 引用传递的规则为：值传递方法既可以接受指针，也可以是值；而引用方法只能时指针
原因在于：指针方法能够直接改变其接收器，如果用值传递来触发的话会导致这个改变的特性失效。
但是有一个**特例**：当传入的值是一个可寻址的对象时，那么编译器会自动取这个对象的地址，调用引用传递。

## interface and other types
