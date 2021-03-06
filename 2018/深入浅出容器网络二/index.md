# 深入浅出容器网络（二）


<!--more-->

# 容器网络组网类型
将容器介入容器网络前，要先搞清楚接入的容器网络到底使用了什么组网类型。常见的组网类型有**underlay l2**, **overlay l2** 和 **overlay l3**，还有一种是**Host**网络，也就是容器网络直接接入主机网络也就不存在所谓的容器网络，本文主要介绍前三种容器网络组网类型。

## Underlay 网络
underlay网络一般理解是底层网络，传统的网络组网就是underlay类型，区别于如今流行的隧道技术。

### Underlay L2
underlay l2网络就是2层（链路层）互通的底层网络，传统网络大多数属于这种类型。容器网络使用这种组网，常使用的技术就有IPVLAN L2和MACVLAN。

#### MACVLAN
MACVLAN是linux提供的一种简单的网络子接口技术，它允许你在主机的一个网络接口上配置多个虚拟的网络接口，这些网络interface拥有自己独立的mac地址，也可以配置上独立的ip地址进行通信。macvlan下的容器和主机在同一网段中，共享同一广播域。macvlan和bridge比较类似，但是它省去了bridge的存在，网络效率更高，除此之外，macvlan也完美支持VLAN技术。

如果希望容器放在主机相同的网络中，享受已经存在的网络栈的各种优势，可以考虑macvlan。

- https://docs.docker.com/network/macvlan/
- http://cizixs.com/2017/02/14/network-virtualization-macvlan

#### IPVLAN
IPVLAN和MACVLAN类似，都是从一个主机接口虚拟出多个虚拟网络接口。一个重要区别是所有的虚拟接口都有相同的mac地址，而拥有不同的ip地址。因为所有的虚拟接口要共享mac地址，因此DNCP不能使用mac地址分配ip。

IPVLAN支持两种模式L2和L3，在Underlay L2中，我们使用L2模式。L2模式相比L3模式，父接口工作在类似交换机的模式，子接口可以收到并处理L2报文，相比L3有着更高的处理效率。

- https://people.netfilter.org/pablo/netdev0.1/papers/IPVLAN-The-beginning.pdf
- http://cizixs.com/2017/02/14/network-virtualization-macvlan

### Underlay L3
在Underlay L3组网中，可以选择使用IPVLAN的L3模式，该模式下ipvlan 有点像路由器的功能，它在各个虚拟网络和主机网络之间进行不同网络报文的路由转发工作。只要父接口相同，即使虚拟机/容器不在同一个网络，也可以互相 ping 通对方，因为 ipvlan 会在中间做报文的转发工作。

flannel 的 host-gw 组网，calico 的 BGP 组网方式都是 Underlay L3。

![host-gw](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2019-05-15-130628.jpg)
<center> flannel网络架构（图片来自openshift）</center>

## Overlay 网络
Overlay是在传统网络上虚拟出一个虚拟网络来，传统网络不再需要做任何适配，这样物理网络只对物理层的计算（物理机、虚拟化层管理网），虚拟网络只对应虚拟计算（虚拟机的业务IP）。

### Overlay L2
![overlay-l2](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-09-02-071823.jpg)
<center> 基于 VXLAN 的 Overlay L2 组网 </center>

传统的二层网络范围有限，Overlay l2网络是构建在传统网络上的L2网络，相较于传统l2网络，Overlay l2网络可以跨越多个数据中心，提供了大二层网络。构建在大二层网络上的VM或者容器在动态迁移具有很大的范围和很高的灵活性。

Overlay技术主要用VXLAN、NVGRE、GENEVE等，无论哪种协议都要求在发送方向对报文进行封装外层头，接收方向剥离外层头。VXLAN作为Overlay技术一个代表，是云计算的核心技术之一。

上图的容器Overlay L2网络就由VXLAN实现，通过在UDP包中封装L2报文，实现了容器跨主机进行L2通信。

- https://feisky.gitbooks.io/sdn/basic/overlay.html
- http://techblog.d2-si.eu/2017/05/09/deep-dive-into-docker-overlay-networks-part-2.html

### Overlay L3
Overlay L3组网类似Overlay L2，但是节点上会增加一个网关。每个节点上的容器都在同一个子网内，可以直接进行二层通信，但是跨节点的容器间通信只能走L3，都会经过网关转发，性能相比于Overlay L2较弱。但是牺牲的性能获得了更高的灵活性，跨节点的容器可以存在于不同的网段中。

flannel 的最新版本中，VXLAN 模式采用了 Overlay L3 模型。

![overlay-l3](http://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2019-05-15-131415.jpg)
<center> flannel 基于 VXLAN 的 Overlay L3 组网 </center>

## 参考

- https://www.sdnlab.com/21143.html
