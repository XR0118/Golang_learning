# Data Declear and Define

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
