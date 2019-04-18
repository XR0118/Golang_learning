# Function

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
