---
title: Load Balance 访问 Kubernetes Pod 源 IP 保留方案探索
date: 2019-09-04 22:21:59
tags: [kubernetes, network]
category: kubernetes
comments: true
---

Kuberentes 已经成为云计算事实上的基础设施，各大云服务厂商都推出了自己的Kubernets 集群托管服务。处于信息安全考虑，一些企业不允许自己的核心资产暴露在公共服务器上，公有云并不适合他们。一些云服务厂商嗅到了其中的商机，国外大厂OpenShift，国内云厂商阿里云、华为云、青云等纷纷顺势推出了基于Kubernetes的私有集群以及混合云的解决方案。

企业在选型方案是，网络能力往往是考量各解决方案优劣的一个重要指标，但是本文并不打算分析各厂商的网络方案（工作量太大，计划后续补上），而是主要聚焦 Kubernetes 集群中的一个小的网络特性，即 Kubernetes Load Balance Service 如何保留客户端源IP。

<!--more-->

## 现状

Kuberntes 的 Service Type 分为三种[^1]：

1. Cluster IP
2. NodePort
3. LoadBalancer

以下我们基于 kube-proxy 的 iptables 模式或 ipvs 模式进行分析，不考虑 userspace 模式。

`Type=ClusterIP` 的 Service 一般只支持集群内部访问，通过 iptables 规则或者 ipvs 规则完成 ClusterIP 到 Pod IP的转换，Kubernetes只对访问ClusterIP的流量进行DNAT，并不会进行SNAT，但是实际集群是否会进行 SNAT 还要看CNI网络插件实现，像calico、flannel等，流量必须经过主机网络空间的容器网络实现，并不需要进行SNAT，而一些基于IPVlan，VLAN，将网络直接接入容器的实现，则需要进行SNAT，保证回程流量能够经过主机网络空间，实现NAT转换。大致流量如图：

![cluster-ip-service](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20190907142942.png)

`Type=NodePort` 的 Service 支持集群外访问，客户端通过访问节点的端口，在节点内通
过iptables 或ipvs 规则转发流量实现负载均衡，该模式下Kubernetes除了会对流量进行DNAT，还会进行SNAT，这是为了保证负载均衡到其他节点的流量能够回到该节点，进行NAT解除操作。可能有的用户不想要在节点上再进行一次负载均衡，并且想要保留 client ip，Kubernetes 很贴心的支持了这个选项，Service  `service.spec.externalTrafficPolicy` 字段设置成 `Local` 后，访问NodePort的流量就不再进行SNAT和负载均衡了。

![nodeport-service](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20190907144052.png) `Type=LoadBalance` 的 Service 默认开启 SNAT，因为所有的 `Ready` 节点都是 LB 的后端，如果数据报文到了一个没有 endpoint 的节点，系统需要将该报文转发到其他节点。

如果是在 Google Kubernetes Engine/GCE，同样的设置`service.spec.externalTrafficPolicy` 为 `Local` 可以让没有endpoint 的节点 service 健康检查失败，移除LB后端，kubernetes不会将流量转发到其他节点，也就不再需要进行 SNAT，保留了源 IP。

```shell
                      client
                        |
                      lb VIP
                     / ^
                    v /
health check --->   node 1   node 2 <--- health check
        200  <---   ^ |             ---> 500
                    | V
                 endpoint
```

## LoadBalancer 保留源 IP

LoadBalancer的工作模式有两种：

1. 作为代理，接收来自客户端的请求，同时建立新的连接请求，和后端服务器建立连接。这种情况下，后端服务器看到的源IP是LB的IP。
2. 作为数据包转发器，客户端访问 LB 的 VIP，最终保持源IP和后端服务器建立连接。

第一种模式的LB需要依赖协议通知客户端真实的源IP，例如 HTTP 的 X-FORWARD-FOR，或者proxy protocol。第二种模式天然保留了源IP，但是在kubernetes中，需要配合关闭SNAT，保留源IP进入Pod，同时`service.spec.healthCheckNodePort`可以帮助LB检测节点上是否存在endpoint，剔除不合格的node。

### Direct Route

Direct Route模式即LB作为路由器角色，将客户端的流量直接负载均衡，路由到后端节点，也就是上面提到的工作模式2.

DR 模式有几个限制[^2]：

1. VIP 的端口必须和RS服务的端口一致（DR模式只修改包的 mac 地址，不会修改IP及上层的内容）
2. RS 必须对 arp 做相关设置，lo 接口需要绑定VIP（使用iptables进行DNAT处理也是可行的[^3]）
3. VIP 和 RIP 不需要在同一个网段，但是 Director 要有一个网口和 RS 是再同一个物理网络下

支持该模式的开源方案有LVS和HAPROXY。

#### LVS

![lvs-dr](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20190907151443.png)

#### Haproxy

![haproxy-dr](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20190907152339.png)

#### kubernetes

Kubernetes 可以利用这两个组件实现 DR 模式源IP保留。

![kubernets-dr](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20190907152530.png)

### PROXY Protocol

PROXY Protocol 是HAProxy作者提出的一个协议[^4]，通过在数据流前端加入一小段协议报文，实现源IP透传。该协议要求有 sender 和 receiver，两者需要匹配 proxy protocol，否则数据流解析就会有问题。目前已经有很多开源软件加入支持该协议，因此通过部署支持该协议的LB，如F5，以及在容器中加入支持proxy protocol的sidecar，例如nginx，envoy，可以实现源IP透传。

![lb-proxy-protocol](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20190907153305.png)

### TOA

TOA是一个内核模块，通过在TCP包中添加一个OPTION来传递客户端的源IP[^5]。LB需要支持在TCP报文中插入该OPTION，节点通过安装TOA模块，修改getpeername()系统调用，让Pod内应用通过getpeername()获取到真实的源IP。

[^1]: https://kubernetes.io/docs/tutorials/services/source-ip/
[^2]: http://linbo.github.io/2017/08/20/lvs-dr
[^3]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/load_balancer_administration/s1-lvs-direct-vsa
[^4]: https://www.haproxy.com/blog/haproxy/proxy-protocol/
[^5]: http://www.just4coding.com/blog/2015/11/16/toa/
