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
