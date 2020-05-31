# golang的reflect使用


## reflect 介绍

计算机科学中，反射(reflect) 指计算机程序在运行时（runtime）可以访问、检测和修改它本身状态或行为的一种能力。用比喻来说，反射就是程序在运行时能够“观察”并且修改自己的行为。

反射和内省（type introspection）不同，内省机制仅指程序在运行时对自身信息（称为元数据）的检测；反射机制不仅包括要能运行在对程序自身信息进行，还要能进一步根据这些信息改变程序状态或结构。

早期语言汇编包含反射能力，动态修改指令或对它们进行分析等等反射功能时很平常的。编程发展到如C语言等高抽象层次语言时，这种实践消失了，带有反射特性的高级编程语言要更晚出现。

golang 作为一个诞生较晚的现代语言，自然也支持反射能力。通过标准库 `reflect` 包我们可以使用 golang 提供的这个能力。

<!--more-->

## Golang reflect 原则

### interface 类型

reflect 建立在类型系统上，我们先从 Go 的类型系统开始讲起。Go 是静态类型语言，所有的变量都有对应的静态类型，因此所有变量的类型在编译时就已经确定：`int`,`float32`,`*MyType`,`[]byte`等等。

如果我们定义

```go
type MyInt int
var i int
var j MyInt
```

那么 `i` 是 `int` 类型，而 `j` 是 `MyInt` 类型。两个变量拥有不同的静态类型，尽管它们的底层类型是一样的，但是它们之间的赋值必须通过显式的类型转换完成。

interface type 是一类非常重要的类型，它代表一些方法的集合。interface 变量能够存储所有实现了 interface 中的方法的变量。一个广为人知的例子是 `io.Reader` 和 `io.Writer`。

```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任何类型实现上述定义的 Read 或 Write 方法，我们认为该类型实现了 Reader 或 Writer 接口。

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

很重要的一点是，无论 `r` 的值是什么，`r` 的类型永远是 `io.Reader`，go 是静态类型，而 `r` 的静态类型是 `io.Reader` 。

空接口类型是一个十分重要的类型。

```go
interface{}
```

它代表空方法的集合，它可以接收任意值，包括零值或有方法的值。

有些人可能认为Go的interface是动态类型，其实这是误解，它们是静态类型：interface 类型的变量永远是相同的静态类型，但是存储在interface中的值在运行时可以修改类型，该值要求实现interface定义的方法集合。

### interface 的细节

interface 类型的变量存储了一个对值：赋值给该interface的值；以及该值的类型。

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

`r` 存储了(value, type)对：(tty, *os.File)。需要注意 `*os.File` 实现了比 `Read` 更多的方法，但是使用 `Read` interface只能访问`Read` 中的方法。

```go
var w io.Writer
w = r.(io.Writer)
```

通过赋值给 `Write` 接口，我们可以访问 tty 中实现的 `Write` 中的方法。

我们还可以这样

```go
var empty interface{}
empty = w
```

空的interface仍然包含了相同的pair, (tty, *os.File)。它的用处是：空的interface可以接收任意值并保留该值的所有信息。

这里不需要进行类型断言，因为 `Write` 接口满足空interface类型要求，而 `Read` 接口和 `Wirte` 接口的方法集并不匹配，我们需要显式进行类型断言，告诉编译器我们知道它们存储的值是满足接口的。

还有很重要的一点要记住：interface中存储的pair是（value， concrete type），并不是 (value, interface type)。所以interface不能保存interface 值。

### 第一个法则： 可以从 interface 值获取反射对象

先说基本功能，反射是测试interface中存储的值和类型的机制。我们需要了解 [package reflect](https://golang.org/pkg/reflect/) 中的 `Type` 和 `Value`。这两个类型可以访问interface中的内容。`reflect.TypeOf` 和 `reflect.ValueOf` 用来从interface中获取 `reflect.Type` 和 `reflect.Value` 。

```go
package main
import (
    "fmt"
    "reflect"
)
func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
// type: flot64
```

反射对象的第二个属性是 `Kind`， 它是底层类型，非静态类型，例如反射对象包含用户自定义类型的值：

```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
// v.Kind() == reflect.Int
```

`v` 的 `Kind` 是 `reflect.Int`，但是 x 的静态类型是 MyInt，不是int。

### 第二个法则： 可以从反射对象获取 interface 值

获得一个`reflect.Value` 后，我们可以使用 `Interface()` 方法恢复 interface 值。该方法将type和value打包放入interface表达式中，并将它返回。

```go
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}
```

之后你可以这么处理

```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

我们还可以简化，`fmt.Println` 的参数是空 interface，他们在方法内部会做解包操作，因此我们只需要直接将interface传入。

```go
fmt.Println(v.Interface())
```

### 第三个法则：想要修改反射对象，对象的指必须是settable的

第三法则比较难懂，要结合法则一进行理解。

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```

结果会 panic 并输出信息。

```txt
panic: reflect.Value.SetFloat using unaddressable value
```

错误的原因是 v 不是 addressable 的，Settable 是 reflect.Value 的一个属性，不是所有的 reflect.Value 都有该属性。

`CanSet` 方法输出 settable 属性

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
// settability of v: false
```

Settablity 是一个 bit 类似 addressability，但是更严格。这个属性表述反射对象可以修改创建该反射对象的原始变量的值。Settability 由反射对象是否持有原始对象决定。

当我们使用

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
```

反射对象只持有了 x 的 copy，并不是原始对象，因此

```go
v.SetFloat(7.1)
```

如果允许运行成功，那么它修改的也不是x，而是反射对象内部的一个值，这会让开发者产生疑惑。所以这并不被允许，settability 属性就是用来避免该问题。

如果我们想要修改原始对象，那么我们可以使用指针：

```go
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
// type of p: *float64
// settability of p: false
```

这里 p 也是不可设置的，但是我们并不想修改 p，我们想要修改 p 指向的对象。我们可以使用 `Elem` 方法获取 p 指向的对象，该方法获取指针指向的元素，并保存成 `reflect.Value` 类型返回。

```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
// settability of v: true
```

在这里我们可以修改 x 了。

```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
// 7.1
// 7.1
```

### 结构体

在我们前面的例子中，`v` 并不是指针本身，而是它派生出来的对象。这种方法的一个使用场景是修改结构体的字段。只有我们拥有结构体的地址，我们才可以修改它的字段。

下面是一个分析结构体值的例子。我们使用结构体变量的地址构建一个反射对象，这样我们可以通过该对象对它进行修改。

```go
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
// 0: A int = 23
// 1: B string = skidoo
```

因为 `s` 是 settable 的，所以我们可以用它修改结构体字段。

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
// t is now {77 Sunset Strip}
```

### 总结

再重复一遍三法则：

1. 可以从interface值获取反射对象
2. 可以从反射对象获取interface值
3. 想要修改反射对象，对像必须是settable

一旦你理解这三个反射原则，那么Go就变得更容易使用了。反射是一个需要小心使用的强大工具，可以帮助你的代码减少不必要的重复。

下一次我将介绍 `encoding/json` 的实现，真正深入学习反射的使用。

## 参考

- https://www.wikiwand.com/zh-hans/%E5%8F%8D%E5%B0%84_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6)
- https://golang.org/pkg/reflect/
- https://blog.golang.org/laws-of-reflection
- https://colobu.com/2018/02/27/go-addressable/
- https://studygolang.com/articles/938
- [Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
- https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/
