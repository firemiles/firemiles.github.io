# Golang json 序列化


## 前言

这两天升级 client-go 版本，涉及 grpc proto 相关代码生成。升级完成后在调试的过程中，发现当数据存入etcd时，一个boolean类型的字段消失了，仔细研究了相关代码半天，最后才发现 go 在 Marshal 结构体时，如果字段是boolean类型，并且设置了omitempty，则会在false的事后作为empty删除。为了避免以后再次栽倒同样的坑里，特意写作本篇博客详细记录golang对json的处理。
<!--more-->

## golang  标签（tag）系统

在 golang 中，命名都推荐驼峰方式，并且首字母大小写有特殊语法含义：结构体中首字母小写外包无法引用。但是由于经常需要和其他系统进行数据交互，例如转换成 json 格式，存储到 mongodb 等等。这个时候如果用属性名作为键值可能不符合项目要求。

所以 golang 支持 \`\` 定义标签（tag），在转换成其他格式的时候，会使用其中特定的字段作为键值。例如：

```go
type User struct {
    UserId   int    `json:"user_id" bson:"user_id"`
    UserName string `json:"user_name" bson:"user_name"`
}

u := &User{UserId: 1, UserName: "tony"}
j, _ := json.Marshal(u)
fmt.Println(string(j))
// 输出内容：{"user_id":1,"user_name":"tony"}
```

如果没有定义标签，则输出：

```go
{"UserId":1,"UserName":"tony"}
```

可以看到直接使用了struct的属性名作为键值。

其中还有一个 bson 的声明，这个是用在数据存储到 mongodb 使用的。

### struct 成员变量标签 （Tag） 获取

开发者也可以通过 golang 提供的方法获取 Tag 内容，利用 Tag 进行开发。

如何获取呢，可以使用反射包（reflect）中的方法获取：

```go
t := reflect.TypeOf(u)
field := t.Elem().Field(0)
fmt.Println(field.Tag.Get("json"))
fmt.Println(field.Tag.Get("bson"))
```

json package 和 mongodb package 等包就是利用了标签系统完成数据交换。

## json 处理流程

### Encoding

编码json使用 Marshal 方法

```go
func Marshal(v interface{}) ([]byte, error)
```

Go 结构体 Message

```go
type Message struct {
    Name string
    Body string
    Time int64
}
```

实例

```go
m := Message{"Alice", "Hello", 1294706395881547000}
```

我们可以使用 json.Marshal 来获取 JSON-encoded data

```go
b, err := json.Marshal(m)
b == []byte(`{"Name":"Alice","Body":"Hello","Time":1294706395881547000}`)
```

只有能够被json正常表示的数据结构才能够完成编码：

- JSON 对象只支持 string 作为 key；因此支持的 Go map 类型必须是 map[string]T（T是json package支持的类型）
- Channel，complex，和 function 类型不支持编码
- **Pointer 会按照指向的值进行编码（如果pointer时nil，则编码成‘null’）**

json package 只编码 exported filed（也就是大写字母开头的字段）。因此只有导出的字段出现在JSON的输出中。

### Decoding

解析 JSON 数据使用 Unmarshal 方法：

```go
func Unmarshal(data []byte, v interface{}) error
```

我们需要创建一个变量存放解析后的数据

```go
var m Message
```

调用 Unmarshal

```go
err := json.Unmarshal(b, &m)
```

如果 b 包含有效的 JSON 数据，并且对应 m 的结构，方法返回 err 为 nil， 并且 b 中的数据解析后放入 m。

```go
m = Message{
    Name: "Alice",
    Body: "Hello",
    Time: 1294706395881547000,
}
```

Unmarshal 如何识别哪个字段存储解析的数据？如果解析获得一个JSON key “Foo”，`Unmarshall` 会在目标结构体中找合适的字段（以下条件按顺序查找）：

- 导出的字段，定义了 tag "Foo"（[Go Spec](https://golang.org/ref/spec#Struct_types))
- 导出的字段，名字是 "Foo"
- 导出的字段，名字是 "FoO" 或 "FOO" 获取其他在大小写不敏感情况下与 "Foo" 匹配

当 Json 数据中的结构和 Go type 不匹配时会发生什么？

```go
b := []byte(`{"Name":"Bob","Food":"Pickle"}`)
var m Message
err := json.Unmarshal(b, &m)
```

`Unmarshal` 会解析和目标结构体匹配的字段，忽略不认识的字段。这个特性在当你希望只获取大量json数据中的部分字段时十分有用，这也表示任何任意非导出的字段（小写字母开头的字段）不会被 `Unmarshal` 影响。

### 使用 Tag

一般在 golang 使用 json package，都会使用标签系统。

```go
// Field appears in JSON as key "myName".
Field int `json:"myName"`

// Field appears in JSON as key "myName" and
// the field is omitted from the object if its value is empty,
// as defined above.
Field int `json:"myName,omitempty"`

// Field appears in JSON as key "Field" (the default), but
// the field is skipped if empty.
// Note the leading comma.
Field int `json:",omitempty"`

// Field is ignored by this package.
Field int `json:"-"`

// Field appears in JSON as key "-".
Field int `json:"-,"`

// Field appears in JSON as string type.
Field int `json:,string,omitempty`
Field int `json:,omitempty,string`
```

通过标签系统可以对转换后的 JSON 数据进行自定义：

- `json:"myName"` 定义 JSON key 为 myName
- `json:"myName,omitempty"` 定义 JSON key 为 myName，并且当 empty 的时候删除该字段（**empty的定义为：0，false和nil 指针，nil interface和空的slice、map、array和string**）
- `json:"-"` 在 JSON 中忽略该字段
- `json:"-,"` 在 JSON 中输出 key 值为 "-"
- `json:,string,omitempty` 和 `json:,omitempty,string` 定义该字段在 JSON 中按 string 类型存储，并且 omitempty。（只有 string、float、integer 和 boolean 字段支持这么定义）

## 高级用法

### 使用 interface{} 解析未知 JSON

json pacakge 使用 map[string]interface 和 []interface{} 存储各种 JSON 对象和数组。任意有效的JSON数据类型都能轻松存储到 interface{} 中，默认的具体 Go types 是：

- bool 对应 JSON boolean
- float64 对应 JSON numbers（精度上会有损失）
- string 对应 JSON strings
- nil 对应 JSON null

一个例子：

```go
b := []byte(`{"Name":"Wednesday","Age":6,"Parents":["Gomez","Morticia"]}`)
var f interface{}
err := json.Unmarshal(b, &f)
f = map[string]interface{}{
    "Name": "Wednesday",
    "Age":  6,
    "Parents": []interface{}{
        "Gomez",
        "Morticia",
    },
}
m := f.(map[string]interface{})
for k, v := range m {
    switch vv := v.(type) {
    case string:
        fmt.Println(k, "is string", vv)
    case float64:
        fmt.Println(k, "is float64", vv)
    case []interface{}:
        fmt.Println(k, "is an array:")
        for i, u := range vv {
            fmt.Println(i, u)
        }
    default:
        fmt.Println(k, "is of a type I don't know how to handle")
    }
}
```

### Unmarshal 为引用类型申请内存

我们定义一个类型包含引用

```go
type FamilyMember struct {
    Name    string
    Age     int
    Parents []string
}

    var m FamilyMember
    err := json.Unmarshal(b, &m)
```

Unmarshalling JSON object 到 FamilyMember 里，相比之前例子， Parents 是引用类型，默认值是nil。如果 JSON object有值写入Parents，`Unmarshal` 会为 Parents 申请内存。这个是 Unmarshal 对于引用类型（pointers,slicemaps）的典型工作场景。

```go
type Foo struct {
    Bar *Bar
}
```

如果 JSON object 中包含 Bar 字段，Unmarshal 会为 Bar 申请内存，反之则保持 Bar 为 nil。

### 支持 Streaming

json package 支持对 Stream 进行 Decoder 和 Encoder。使用 NewDecoder 和 NewEncoder 装饰 io.Reader 和 io.Writer inteface typs.

```go
func NewDecoder(r io.Reader) *Decoder
func NewEncoder(w io.Writer) *Encodergo
```

一个例子：

```go
package main

import (
    "encoding/json"
    "log"
    "os"
)

func main() {
    dec := json.NewDecoder(os.Stdin)
    enc := json.NewEncoder(os.Stdout)
    for {
        var v map[string]interface{}
        if err := dec.Decode(&v); err != nil {
            log.Println(err)
            return
        }
        for k := range v {
            if k != "Name" {
                delete(v, k)
            }
        }
        if err := enc.Encode(&v); err != nil {
            log.Println(err)
        }
    }
}
```

对于更普遍的Reader和Writer，Encoder和Decoder类型可以使用在更广泛的场景中。例如对HTTP连接，WebSockets和文件进行读写。

## 需要注意的问题

### omitempty

omitempty 中的 empty 并不仅仅指 empty， 还包括 0，false和nil 指针，nil interface和空的slice、map、array和string。 定义了该 tag 后，boolean、integer、float32、string等属性为默认值时，输出的 JSON 数据就会移除该属性。

在一些需要区分未初始化和0值的场景下，omitempty的使用会造成混淆，这个时候可以使用指针来作为属性。未初始化状态下指针为 nil，输出 JSON 时会被 omit；初始化后 0 值和非 0 值都会正常输出，这样属性的三个状态：未初始化、0值和非0值就可以正常区分了。

### JSON反序列化成interface{}对Number的处理

JSON 规范中，对于数字类型，并不区分整型还是浮点型。

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-12-23-090254.jpg)

对于如下 JSON 文本：

```go
{
    "name": "ethancai",
    "fansCount": 9223372036854775807
}
```

如果反序列化不指定结构体类型或者变量类型，则JSON中的数字类型，默认被反序列化成 `float64` 类型

```go
package main

import (
    "encoding/json"
    "fmt"
    "reflect"
)

func main() {
    const jsonStream = `
        {"name":"ethancai", "fansCount": 9223372036854775807}
    `
    var user interface{}  // 不指定反序列化的类型
    err := json.Unmarshal([]byte(jsonStream), &user)
    if err != nil {
        fmt.Println("error:", err)
    }
    m := user.(map[string]interface{})

    fansCount := m["fansCount"]

    fmt.Printf("%+v \n", reflect.TypeOf(fansCount).Name())
    fmt.Printf("%+v \n", fansCount.(float64))
}

// Output:
// 	float64
//  	9.223372036854776e+18
```

如果 `fansCount` 精度比较高，反序列化成 `float64` 类型的数值时存在丢失精度的问题。

如何解决这个问题，看下面的程序：

```go
package main

import (
    "encoding/json"
    "fmt"
    "reflect"
    "strings"
)

func main() {
    const jsonStream = `
        {"name":"ethancai", "fansCount": 9223372036854775807}
    `

    decoder := json.NewDecoder(strings.NewReader(jsonStream))
    decoder.UseNumber()    // UseNumber causes the Decoder to unmarshal a number into an interface{} as a Number instead of as a float64.

    var user interface{}
    if err := decoder.Decode(&user); err != nil {
        fmt.Println("error:", err)
            return
        }

    m := user.(map[string]interface{})
    fansCount := m["fansCount"]
    fmt.Printf("%+v \n", reflect.TypeOf(fansCount).PkgPath() + "." + reflect.TypeOf(fansCount).Name())

     v, err := fansCount.(json.Number).Int64()
    if err != nil {
        fmt.Println("error:", err)
            return
    }
    fmt.Printf("%+v \n", v)
}

// Output:
// 	encoding/json.Number
// 	9223372036854775807
```

使用 json.NewDecoder 可以将 JSON 数字转换成 json.Number。json.Number 内部实现机制：

```go
// A Number represents a JSON number literal.
type Number string

// String returns the literal text of the number.
func (n Number) String() string { return string(n) }

// Float64 returns the number as a float64.
func (n Number) Float64() (float64, error) {
    return strconv.ParseFloat(string(n), 64)
}

// Int64 returns the number as an int64.
func (n Number) Int64() (int64, error) {
    return strconv.ParseInt(string(n), 10, 64)
}
```

其实就是延迟处理，保存了 JSON 数字的原始字符串，使用的时候根据需要进行转换。

## 总结

其实 Golang 对于 json 的支持已经十分完善，通过和标签系统配合，序列化和反序列化 json 字符串只要几行代码，只是在使用过程中要清楚 go 提供的默认机制，避免踩坑。

## 参考

- http://docs.studygolang.com/pkg/encoding/json/
- https://blog.golang.org/json-and-go
- http://docs.studygolang.com/pkg/encoding/json/#Marshal
- https://ethancai.github.io/2016/06/23/bad-parts-about-json-serialization-in-Golang/
