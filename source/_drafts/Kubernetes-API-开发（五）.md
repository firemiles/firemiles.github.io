---
title: Kubernetes API 开发（五）
photos:
  - image url
description: 你对本页的描述
date: 2018-12-20 23:15:12
tags:
category:
comments:
---

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

你需要在代码中一些未知加入你的 API Group/Verison，并且参考 [添加一个新 API Version](#添加一个新-api-version) 段落，需要重新生成前面提到的代码。

## 更新 fuzzer



