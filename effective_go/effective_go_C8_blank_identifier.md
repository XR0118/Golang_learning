# The blank identifier

blank identifier 被声明为任何形式的变量，其作用类似于向 /dev/null 写内容，并且时一个只写的变量占位符。

## The blank identifier in multiple assignment

在多变量同时赋值的场景中， blank identifier 经常被用到。

返回多个变量的函数被调用时，如果其中的某个返回值在后面的程序中不会被用到，那么就可以用 blank identifier 来占据这个返回值的位置，以避免**创建和删除一个无用的变量**，例如：

```Go
if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```

**注意：** 不要用 blank identifier 来替代 err 的返回值，虽然这样做能够忽略 err， 但是对 err 的处理在程序中时非常主要的，**保证程序的鲁棒性**，反例:

```Go
// Bad! This code will crash if path does not exist.
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

## Unused imports and variables

在 Go 程序中，如果在定义变量或者引入包却没有用到，也是一种错误。主要的原因在于:
引入没有用到的包会导致**程序的膨胀和编译速度降低**；
而一个初始化之后未使用的变量会导致**计算资源的浪费并可能引发程序的 bug**。
然而，在程序的开发阶段通常会引入一些可能会使用到的包或者变量，
那么为了不让上述的错误发生，可以用 blank identifier 来解决这个问题。

举例说明：
以下写了一半的程序中，有两个未用到的包(fmt, io)和一个未用到的变量(fd)，因此在编译时会报错：

```Go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
}
```

为解决上述由于未用到的包和变量引起的编译问题，用 blank identifier 来饮用包中的对象或者变量就能够较好的解决这个问题, 同时通过注释的定位能够方便的找到没有使用的变量或者包, 例子如下:

```Go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // For debugging; delete when done.
var _ io.Reader    // For debugging; delete when done.

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

## Import for side effect

side effect 副作用？这一届不是特别懂，而且没有具体的例子

## Interface checks

某种 Type 不需要声明自己实现了哪个 interface, 只要实现了某个 interface 中的方法，那么该 Type 就实现了对应的 interface。
大多数 interface 转换都是静态的，因此在**编译时进行检查**。
举例来说，将 `*os.File` 传递给接受 `io.Reader` 变量类型的函数将不会编译，除非 `*os.File` 实现了 `io.Reader` 接口。

也有一些接口检查在程序运行时发生。
其中一个例子是关于 encoding/json 包中的 Marshaler interface。
当 JSON encoder 接收到一个实现了该接口的值，就会调用该值的 marshaling 方法来转换而不是标准方法。
接口检查的方法如下:

```Go
m, ok := val.(json.Marshaler)

// 如果只是检查是否能够进行接口转换，而不是实际要用实现这个接口的对象的话，可以用 blank identifier 来忽略类型转化的值

if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}

```

或者使用以下的方法进行静态编译时的检查:

```Go
var _ json.Marshaler = (*RawMessage)(nil)
```

上面的声明涉及到将  `*RawMessage` 转换为 `Marshaler` 接口，并且会在编译时就进行检查。**只有在代码中不存在静态转换时才会使用此类声明，这是一种少见的事件**。
