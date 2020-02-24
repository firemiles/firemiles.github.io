---
title: Kubernetes-ListWatch代码分析（客户端）
photos:
  - image url
description: Kubernetes 的 ListWatch 机制支撑起了整个消息通知系统，保证系统的正常运转。本文分析 ListWatch 在客户端的实现，通过学习它的设计思想，可以帮助我们设计其他类似的消息系统。
date: 2020-01-12 11:20:53
tags: [kubernetes,listwatch]
category: kubernetes
comments: true
---

## kube-proxy 获取 Apiserver 的数据

Kubernetes 提供了 `client-go` 来访问 Apiserver，我选取 kube-proxy 中的代码作为入口来进行分析。

```go
informerFactory := informers.NewSharedInformerFactory(s.Client, s.ConfigSyncPeriod)

// Create configs (i.e. Watches for Services and Endpoints)
// Note: RegisterHandler() calls need to happen before creation of Sources because sources
// only notify on changes, and the initial update (on process start) may be lost if no handlers
// are registered yet.
serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
serviceConfig.RegisterEventHandler(s.ServiceEventHandler)
go serviceConfig.Run(wait.NeverStop)

endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().V1().Endpoints(), s.ConfigSyncPeriod)
endpointsConfig.RegisterEventHandler(s.EndpointsEventHandler)
go endpointsConfig.Run(wait.NeverStop)

// This has to start after the calls to NewServiceConfig and NewEndpointsConfig because those
// functions must configure their shared informer event handlers first.
go informerFactory.Start(wait.NeverStop)
```

上面这段是 kube-proxy 中 `ProxyServer.Run()` 的启动 ListWatch 的代码，内容十分简洁清晰，启动 ListWatch 分成了两个部分：`InformerFactory` 和 `ServiceConfig`。

`client-go` 由很多包组成，我们从 `Informer` 开始逐一介绍。

![client-go组成](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200112173121.png)

### Informer

`Informer` 是一个依赖 `Kubernetes List/Watch API`、`可监听事件并触发回调函数`的`二级缓存` 工具包。

#### List/Get 缓存

使用 `Informer` 实例的 `Lister()` 方法获取 Kubernetes Object 时，`Informer` 不会直接请求 `Kubernetes API`，而是直接查找缓存在本地内存中的数据（这份数据由`Informer`自己维护）。通过这种方式，`Informer`可以更快返回结果，又能减少对`Kubernetes API` 的直接调用。

#### 依赖 List/Watch API

`Informer` 只会调用 List 和 Watch 两种类型的 API。`Informer` 在初始化的时候，先调用 `List API` 获得某种 resource 的全部 object，缓存在内存中，然后调用 `Watch API` 去 watch 这种 resource，去维护这份缓存，最后，`Informer` 就不再调用 Kubernetes 的任何 API。

用 `List/Watch` 去维护缓存、保持一致性是非常典型的做法。`Informer` 只在初始化时调用一次 `List API` ，之后完全依赖 `Watch API`，没有任何 `resync` 机制，因为目前 `Watch API` 的可靠性非常高。

#### 可监听事件并触发回调函数

`Informer` 通过 `Watch API` 监听某种 `resource` 下的所有事件。而且 `Informer` 可以添加自定义的回调函数，这个回调函数实例（`ResourceEventHandler`）只需要实现 `OnAdd`、`OnUpdate` 和 `OnDelete` 三个方法。

#### 二级缓存

二级缓存属于 `Informer` 底层缓存机制，这两级缓存分别是 `DeltaFIFO` 和 `LocalStore`。

