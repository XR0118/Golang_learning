# Method

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