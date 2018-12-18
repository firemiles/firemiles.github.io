---
title: Kubernetes API 开发（四）
photos:
  - http://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-12-17-143714.jpg
date: 2018-12-16 20:21:32
tags: [kubernetes, API, client-go]
category: kubernetes
comments: true
---
## 生成代码

除了 `default-gen`, `deepcopy-gen`, `conversion-gen`, 和 `openapi-gen` 外，还有其他的生成器：

- `go-to-protobuf`
- `client-gen`
- `lister-gen`
- `informer-gen`
- `codecgen` (使用 ugorji codec快速串行化json)

大部分生成器基于 [gengo](https://github.com/kubernetes/gengo) ，拥有相同的命令行选项。 `--verify-only` 会检查磁盘上的将要生成的文件。

生成器生成代码时有个 `--go-header-file` 选项，该选项指定包含生成代码头部内容的文件。头部内容一般是版权信息，放在生成代码的最前面，并且会被稍后的 [repo-infra/verify/verify-boilerplane.sh ](https://git.k8s.io/repo-infra/verify/verify-boilerplate.sh) 脚本检查。

执行 `make update` 会调用一系列[脚本](https://github.com/kubernetes/kubernetes/blob/v1.8.0-alpha.2/hack/update-all.sh#L63-L78) 包括以上的生成器。请继续阅读以下段落，因为某些生成器的使用需要一些先决条件，而且执行 `make update` 耗时太久，下面还会介绍如何单独调用生成器。
<!--more-->

### Generate protobuf objects

对于任意 core API object ，我们需要生成 Protobuf IDL 和 marshallers。通过调用以下脚本生成：

```sh
hack/update-generated-protobuf.sh
```

大部分 object 对于转换成 protobuf 不需要太多的考虑，但是我们需要注意哪些依赖 golang 标准库类型的的 object，可能需要提供一些额外的工作，实践上我们一般使用自定义的 JONST 串行化。`pkg/api/serialization_test.go` 会测试你的 protobuf 串行化，多运行几次确保没有非兼容的字段。

### Generate Clientset

`client-gen` 是为顶层 API object 生成 clientsets 的工具。

`client-gen` 要求每个想要导出的类型前添加 `// +genclient` 注释，包括 internal type `pkg/apis/<group>/types.go` 和 versioned type `staging/src/k8s.io/api/<group>/<version>/types.go` 。
