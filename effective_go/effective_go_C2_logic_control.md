# Logic Control

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

`:=` 声明中即使已经声明了变量v，也能够再次出现，前提是：

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
