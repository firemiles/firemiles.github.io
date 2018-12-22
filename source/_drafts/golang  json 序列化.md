---
title: golang
tags:
---

## 前言

这两天升级 client-go 版本，涉及 grpc proto 相关代码生成。升级完成后在调试的过程中，发现当数据存入etcd时，一个boolean类型的字段消失了，仔细研究了相关代码半天，最后才发现 go 在 marshall 结构体时，如果字段是boolean类型，并且设置了omitempty，则会在false的事后作为empty删除。为了避免以后再次栽倒同样的坑里，特意写作本篇博客详细记录golang对json的处理。

## golang tag 系统

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

- 导出的字段，定义了 tag "Foo"（[Go Spec](https://golang.org/ref/spec#Struct_types）)
- 导出的字段，名字是 "Foo"
- 导出的字段，名字是 "FoO" 或 "FOO" 获取其他在大小写不敏感情况下与 "Foo" 匹配

当 Json 数据中的结构和 Go type 不匹配时会发生什么？

```go
b := []byte(`{"Name":"Bob","Food":"Pickle"}`)
var m Message
err := json.Unmarshal(b, &m)
```

`Unmarshal` 会解析和目标结构体匹配的字段，忽略不认识的字段。这个特性在当你希望只获取大量json数据中的部分字段时十分有用，这也表示任何任意非导出的字段（小写字母开头的字段）不会被 `Unmarshal` 影响。

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

## 常见错误

## 参考

- http://docs.studygolang.com/pkg/encoding/json/
- https://blog.golang.org/json-and-go
- https://ethancai.github.io/2016/06/23/bad-parts-about-json-serialization-in-Golang/