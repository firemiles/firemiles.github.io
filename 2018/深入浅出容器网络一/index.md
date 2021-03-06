# 深入浅出容器网络（一）


作为系列第一篇，先开个头，介绍模型和框架，后续再新增实战系列，理论是实战结合才能更好的理解容器网络的设计和实现原理。
## 模型

容器技术已经火遍全球，业界迫切需要一个统一的容器网络模型，此时Docker提出了Container
Network Model（CNM），Kubernetes提出了Container Network Interface（CNI）。

### CNM

Libnetwork是CNM的实现。它为Docker daemon和网络驱动程序之间提供了接口。网络控
制器负责将驱动和一个网络进行对接。每个驱动程序负责管理它所拥有的网络，以及为
该网络提供各种服务，例如IPAM负责ip分配。支持多个驱动程序多个网络同时共存。

网络驱动可以按提供方划分为原生驱动（libnetwork内置的或者Docker支持的）或者远
程驱动（第三方插件）。原生驱动包括none，bridge，overlay一粒macvlan。驱动也可
以按照适用范围划分为本地（单主机）的和全局的（多主机）。

#### 架构

![CNM架构](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-05-26-175114.jpg)

1. Network Sandbox

    包含了一个容器的网络栈。包括了管理容器的网卡，路由表以及DNS设置。一种
    Sandbox实现是通过linux的网络命名空间，一个FreeBSD Jail或者其他类似的概念。
    一个Sandbox可以包含多个endpoints。

2. Endpoint

    一个endpoint将Sandbox连接到network上。一个endpint的实现可以通过veth pair，
    Open vSwitch internal port或者其他的方式。一个endpint只能属于一个network，
    也只能属于一个sandbox。

3. Network

    一个network是一组可以互相通信的endpoints组成。一个network的实现可以是linux
    bridge，vlan或者其他方式。一个网络中可以包含多个endpoints。

#### 接口

![CNM接口](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-05-26-180222.jpg)

CNM的接口对比CNI模型较为复杂。提供了remote plugin方式，进行插件化开发。
remote plugin相较与CNI的命令行，更加友好一些，是通过http请求进行的。remote
plugin监听一个指定的端口，docker daemon直接通过这个端口和remote plugin进行
操作。

##### 调用过程

###### Create Network

这一系列调用发生再调用docker network create过程中。

1. /IpamDriver.RequestPool: 创建subnetpool用于分配IP
2. /IpamDriver.RequestAddress: 为gateway获取IP
3. /NetworkDriver.CreateNetwork: 创建neutron network和subnet

###### Create Container

这一系列调用发生再适用docker run，创建一个container过程中，也可以通过
`docker network connect` 触发。

1. /IpamDriver.RequestAddress: 为容器获取IP
2. /NetworkDriver.CreateEndpiont: 创建neutron port
3. /NetworkDriver.Join: 为容器和port绑定
4. /NetworkDriver.ProgramExternalConnectivity:
5. /NetworkDriver.EndpointOperInfo

###### Delete Network

这一系列调用发生再使用 `docker network delete` 的过程中。

1. /NetworkDriver.DeleteNetwork: 删除network
2. /IpamDriver.ReleaseAddress: 释放gateway的IP
3. /IpamDriver.ReleasePool: 删除subnetpool

### CNI

CNI是由CoreOS提出的一种容器网络规范。已采纳该规范的包括Apache Mesos， Cloud
Foundry， Kubernetes，Kurma和rkt。另外Calico 和Weave这些项目也为CNI提供插件。

![CNI驱动组织](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-05-26-180604.jpg)

CNI的规范比较小巧。它规定了一个容器runtime和网络插件之间的简单的契约。这个契
约通过JSON的语法定义了CNI插件所需要提供的输入和输出。

一个容器可以被加入到不同插件所驱动的多个网络之中。一个网络有自己对应的插件和
唯一的名称。CNI插件需要提供两个命令：一个用来将网络接口加入到指定网络，另一
个用来将其移除。这两个接口分别再容器被创建和销毁时调用。

#### CNI工作流程

docker 已经支持安装CNI插件。<https://docs.docker.com/ee/ucp/kubernetes/install-cni-plugin/>

容器runtime(e.g. kubernetes)首先需要分配一个网络命名空间以及一个容器ID。然后连同一些CNI配置
参数传给网络驱动。接着网络驱动会将容器间接入网络，并将分配的IP地址以JSON格
式返回给容器runtime。

目前，CNI的功能涵盖了IPAM, L2 和 L3。端口映射(L4)则由容器runtime自己负责。
CNI也没有规定端口映射的规则。

CNI支持与第三方IPAM的集成，可以用于任何容器runtime。CNM从设计上就仅仅支持
Docker。由于CNI简单的设计，许多人认为编写CNI插件会比编写CNM插件来得简单。

### CNI和CNM的转化

CNI和CNM并非完全不可调和的两个模型。两者概念上有很多相似点，因此可以相互转化，
比如calico项目就支持两种接口模型。

CNI中的container与CNM的sandbox概念一致，CNI中的network与CNM中的network一致。
再CNI中，CNM中的endpoint被隐含在了ADD/DELETE操作中。CNI接口更加简洁，把更多
的工作托管给了容器的管理者和网络的管理者。从这个角度来说，CNI的ADD/DELETE接
口只实现了 `docker network connect` 和 `docker network disconnet` 两个命令。

kubernetes/contrib项目提供了一种从CNI向CNM转化的过程。其中原理很简单，就是直
接通过shell脚本执行 `docker network connect` 和 `docker network disconnect`
命令，来是心啊从CNI到CNM的转化。

## Reference

- <https://github.com/containernetworking/cni>
- <https://thenewstack.io/container-networking-landscape-cni-coreos-cnm-docker/>
- <http://dockone.io/article/1974>
- <http://www.cnblogs.com/xuxinkun/p/5707687.html>

