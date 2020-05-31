# Kubernetes API 开发（三）


## 修改 Internal structures (unversioned API)

现在是时候开始介绍如何修改内部结构体，这样你修改的 versioned API 才能正常使用。
<!--more-->

### Edit types.go

和 versioned APIs 修改类似，内部结构体定义在 `pkg/apis/<group>/types.go` 中。想要修改什么内容就在这里完成，主要内部结构体必须兼容所有的 versioned APIs。

当然，你也必须添加 `+k8s:deepcopy-gen` tag 来告诉工具需要生成 DeepCopyObject 方法。

### Edit validation.go

大部分情况下修改了内部结构体后，需要一些内部输入校验。校验代码放在 `pkg/apis/<group>/validation/validation.go` 。 这个校验对用户体验十分重要，提供良好的错误信息，保证用户的输入正确性。当错误发生的时候，告诉用户为什么以及如何修复。

当然，测试代码也是必须的—— `pkg/apis/<group>/validation/validation_test.go`

### Edit version conversions

看到这里的时候，你已经完成了 versioned APIs 和 内部结构的修改。如果某些字段名，类型在内部结构和 versioned APIs中存在变化，你必须添加一些从内部结构转换成 versioned APIs的逻辑。如果你在执行 `serialization_test` 是发现了错误，它可能是在告诉你需要显示指定转换方法。

转换方法的性能会影响 apiserver 的流畅性。因此我们采用自动生成的代码而不是使用通用的转换代码（通用代码使用反射会有很高代价）

转换代码在每个 versionded API 下都有一份，有两个文件：

- `pkg/apis/<group>/<version>/conversion.go` 包含手写的转换函数
- `pkg/apis/<group>/<version>/zz_generated.conversion.go ` 包含自动生成的转换函数

因为自动生成的转换代码会使用手写的转换代码，因此对手写的转换代码的命名有要求。例如，pkg `a` 中的类型 `X` 转换成 pkg `b` 中的类型 `Y`， 需要命名成: `convert_a_x_To_b_Y`

你可以手写转换代码的时候同样可以引用自动生产的转换代码（为了提高效率也应该这样）。

手动添加的转换代码同样要求你添加 test `pkg/apis/<group>/<version>/conversion_test.go`

当手动添加完必须的转换方法后，你需要重新生成 auto-generated 的代码。执行：

```shell
make clean && make generaged_files
```

`make clean` 很重要，否则生成的文件会是旧的版本，因为构建系统会使用用户的缓存文件。

`make all` 也会调用 `make generated_files`。

`make generated_files` 也会重新生成 `zz_generated.deepcopy.go`, `zz_generated.defaults.go` 和 `api/openapi-spec/swagger.json`。

如果生成中偶尔出现了编译错误，最简单的解决方法就是删除引起错误的文件重新生成（真暴力）

## 参考

- https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#making-a-new-api-version
