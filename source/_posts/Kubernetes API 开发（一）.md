---
title: Kubernetes API 开发 （一）
photos:
  - http://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-12-09-140421.jpg
description: 本系列文章主要介绍开发者如何对 Kuberentes 的 API 进行修改和生成。 
date: 2018-12-09 17:26:26
tags: [kubernetes, API, client-go]
category: kubernetes
comments: true
---
本文宏观介绍 **kubernetes** 的 **API** 各版本之间的联系以及 **API** 开发的原则。
<!--more-->

## API Overview

### API 版本转换

Kubernetes 的 API 分为 **Internal** 和 **External** 版本。apiserver 同时支持多个API Version，在为开发提供了很高的自由度的同时也对API版本间的转换提出了更高的鲁棒性要求。

多个 external 版本间的转换， internal 版本处于中心位置，所有版本的 API 都能够转换成 internal 版本，但是其他版本之间能相互转换。所有的 kubernetes 代码 internal 版本结构中进行操作，并在存入*etcd* 前转换成 external 版本。客户端必须使用 external 版本操作。

以下是一个假设的操作例子：

1. 用户 **POST** *Pod* 到 */api/v7beta1/...*
2. **JSON** unmarshall 为 *v7beta1.Pod* 
3. 为 *v7beta1.Pod* 补充默认值
4. *v7beta1.Pod* 转换成 *api.Pod*
5. 校验 *api.Pod* 并返回错误
6. *api.Pod* 转换成 *v6.Pod* (稳定版本)
7. *v6.Pod* marshall 成 **JSON** 并写入 *etcd*

### 版本兼容

Kubernets 把 API 的版本前向和后向兼容作为最高优先级进行考虑。
想要做到兼容很难，以下几点是实现 API 兼容需要考虑的：

- 如果添加一个新功能，一定要是非必须的，要让旧版本也能正常工作
- 不要修改已有接口的定义，包括：

  - 默认值和默认行为，以及其包含的语义
  - 已有 API 类型的字段和值的解释
  - 字段是否是必须字段
  - 可修改字段不要变为非可修改字段
  - 非法字段不要变成合法字段
  
换个说法：

1. 所有修改前成功调用的 API， 修改后也要能成功调用。
2. 所有不涉及修改的 API 调用，行为和修改前一致。
3. 使用修改后的 API，在和未修改的 apiserver 通信时，不会导致问题。
4. API 修改支持在不同版本间转换，并不丢失信息。
5. 旧版本的 client 不感知 server 端修改，仍然正常工作。
6. 支持回退到之前不包含修改的 API 版本， 并且不影响没有使用修改部分代码的 client。所有被修改影响到的 API 对象能够隐式回退。

## 参考
- https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#making-a-new-api-version