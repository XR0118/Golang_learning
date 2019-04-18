# Initialize

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
