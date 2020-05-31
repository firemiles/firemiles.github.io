# Kubernetes API 开发（五）


## 添加一个新 API Version

如果你要为一个 Group 添加一个新的 API Version，你可以从 `pkg/apis/<group>/<existing-version>` copy 到 `staging/src/k8s.io/api/<group>/<existing-version>` 目录中。

由于项目在飞速迭代，以下的内容有可能已经过时：

- 你可以通过修改[pkg/master/master.go.](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/master/master.go#L381)控制是否默认 API Version enable
- 你必须在[pkg/apis/group_name/install/install.go.](https://github.com/kubernetes/kubernetes/blob/v1.11.0/pkg/apis/apps/install/install.go)中添加新 version
- 你必须在[hack/lib/init.sh#KUBE_AVAILABLE_GROUP_VERSIONS.](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/hack/lib/init.sh#L53)中添加新version
- 你必须在[hack/update-generated-protobuf-dockerized.sh](https://github.com/kubernetes/kubernetes/blob/v1.8.2/hack/update-generated-protobuf-dockerized.sh#L44)中添加新version，帮助生成protobuf IDL和marshallers
- 你必须在[cmd/kube-apiserver/app#apiVersionPriorities](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/cmd/kube-apiserver/app/aggregator.go#L172)
- 你必须为新version安装storage [pkg/registry/group_name/rest](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/registry/authentication/rest/storage_authentication.go)

<!--more-->

## 添加一个新 API Group

你需要在 `pkg/apis` 和 `staging/src/k8s.io/api` 中为 Group 创建新的目录。复制已有的 API Group 的目录结构，例如：`pkg/apis/authentication` 和 `staging/src/k8s.io/api/authentication`；用自己的 group 名替换 authentication，用自己的 version 替换旧的 version。替换 [versioned](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/staging/src/k8s.io/api/authentication/v1/register.go#L47) 和 [internal](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/pkg/apis/authentication/register.go#L47) 中 register.go 的 API Kinds，还有 install.go 中的 kinds (1.11版本后 install.go 不再需要修改kind)

你需要在代码中一些未知加入你的 API Group/Verison，并且参考 [添加一个新 API Version](#添加一个新-API-Version) 段落，需要重新生成前面提到的代码。

## 更新 fuzzer

测试APIs的相互转换稳定性的一个方法是"fuzz"（填充随机值）到 API 对象中，并且把它们在不同版本的API间进行转换。这个一个很好的方法，能够暴露代码中丢失的信息和错误的假设前提。如果你添加了任意字段需要很好的格式化（测试不进行validation），或者你假设 slice 至少包含一个元素，你可能会得到 `serialization_test` 的错误返回。如果遇到了这类问题，你可以自定义 fuzz 方法满足要求。

fuzzer 可以在 `pkg/api/testing/fuzzer.go` 中找到。

## 更新语义比较

非常非常少见的情况，但是还是简单说明。某些更新了 boject 的内容，用了不同的表达方式，但是又想要比较函数在比较旧的表达方式和新的表达方式时，能够通过语义进行对比。例如二进制格式化的10和十进制格式化的2语义上时相同的。Go 唯一支持的 Deep-Equal 是一个字段一个字段进行比特比较。

首先你不应该这么做，如果不可避免，你需要了解 `apiequality.Semantic.DeepEqual` routine，它支持自定义指定类型——　可以在`pkg/api/helper/helpers.go` 中找

还有一件事你可能需要了解：`unexported field`. Go 的 `relect` 包支持 `unexported fields`。但是我们不支持——`apiequality.Semantic.DeepEqual`. 幸运的时大部分 API Object 导出了所有的 fields. 但是有时你想要不导出字段（time.Time包含未导出字段），你可能需要在 `apiequality.Semantic.DeepEqual` 自定义函数。

## 实现你的修改

到这里你已经完成了 API 的修改，接下来就是完成代码的实现。

## 编写 end-to-end tests

看以看这篇 [e2e doc](https://github.com/kubernetes/community/blob/master/contributors/devel/e2e-tests.md)
文档学习如何为你的特性编写 end-to-end test。

## Example 和 docs

在最后，你通过了单元测试、e2e测试，但是还没有结束。你只是修改了 API，如果你修改了已存在的 API， 你必须要确定所有的例子和文档也同时更新。这其实很困难，尤其时在JSON和YAML中会默默丢弃未知的字段。你可以使用 `grep` 和 `ack` 来辅助。

如果你添加了功能，你需要考虑添加文档和例子：

通过运行以下脚本确认你跟个新了 swagger 和 OpenAPI spec：

```sh
hack/update-swagger-spec.sh
hack/update-openapi-spec.sh
```

API spec 的修改 commit 应该和其他的修改分开。

## 参考

- https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#making-a-new-api-version
