# Kubernetes API 开发（二）

## 修改 External API（版本API）

修改版本API是所有修改中最简单的，只需要开发者保持修改后的API相互兼容，比从头写要一个新的 rest API 更容易。
<!--more-->

### Edit types.go

每个 API 结构体的定义放在 `staging/src/k8s.io/api/<group>/<version>/types.go` 中。修改这些文件可以实现 API 结构体的修改。

版本 API 内的所有类型和非内嵌字段需要在之前加上描述性注释——用于生成文档。类型注释不要包含类型名。这些注释生成的文档供用户查看，不应该向用户包含类型名。

如果类型需要生成 [DeepCopyObject](https://github.com/kubernetes/kubernetes/commit/8dd0989b395b29b872e1f5e06934721863e4a210#diff-6318847735efb6fae447e7dbf198c8b2R3767) 方法，一般只在顶层类型上添加，例 `Pod` ，在类型前添加一行注释（[example](https://github.com/kubernetes/kubernetes/commit/39d95b9b065fffebe5b6f233d978fe1723722085#diff-ab819c2e7a94a3521aecf6b477f9b2a7R30)）

```go
  // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

Optional 字段必须包含 `,omitempty` json tag;

### Edit defaults.go

如果你在 type 中添加了新的字段，那么你可能需要修改默认值，需要在 `pkg/apis/<group>/<version>/defaults.go` 中添加 case。

>*注意：* 当在给新字段添加默认值的时候，别忘记给所有版本都添加。[#66135](https://github.com/kubernetes/kubernetes/issues/66135)

老版本的 core ve API 比较特殊， 它的 `default.go` 过去放在 `pkg/api/v1/defaults.go`。如果你看到有引用这个路径，那可以确定那个代码过时了。现在 core v1 API 放在了 `pkg/apis/core/v1/defaults.go`

当你添加完代码，最好再加上一个 test ：`pkg/apis/<group>/<version>/defaults_test.go`

当你要区分变量是 `unset` 还是 `automatic zero` 时，最好使用指针。例如 `PodSpec.TerminationGracePeriodSeconds` 定义成 `*int64`。0值表示0秒，nil表示unset。

别忘了 **run the tests**

### Edit conversion.go

当前还没有修改 internal 结构体，一般没有必要修改 conversion.go。我们会在后面的修改internal api 的章节说明。

当你进行了非兼容的修改后，以下的文件可能需要进行修改：
`pkg/apis/<group>/<version>/conversion.go` 
`pkg/apis/<group>/<version>/conversion_test.go`

注意 conversion machinery 并不能通用的处理各种类型的字段和API常量。[The client library](https://github.com/kubernetes/client-go/blob/v4.0.0-beta.0/rest/request.go#L352) 自定义了字段引用的转换代码。你还需要为你的 scheme 的 `AddFieldLabelConversionFunc` 添加调用，帮助它支持字段转换，就像[这行代码](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/api/v1/conversion.go#L165)

## 参考

- https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#making-a-new-api-version
