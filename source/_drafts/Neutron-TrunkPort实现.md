---
title: Neutron-TrunkPort实现
tags:
---

## Neutron 整体框架

Neutron 是 OpenStack 中负责管理虚拟网络的模块，Neutron 架构主要有几部分：

- Neutron Server
- Neutron Core Plugin
- Neutron Service Plugin
- MessageQueue（AMQP）
- Agent

下面的架构图很好地展示了Neutron整体架构。

![Neutron Frame](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200209104715.png)

除了组件，Neutron将API资源分为两类：

- `Core API`
- `Extension API`

`Core API` 由 `Core Plugin:ML2` 进行处理，其余的都归为 `Extension API`，由 `Service Plugin` 处理。

`Core API` 包含三种资源:

- `Network`
- `Subnet`
- `Port`

目前已有的 `Extension API` 有：

- `Router`
- `FWaaS`
- `LBaaS`
- `QoS`
- `Trunk`
- ……

![API与Plugin](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200209134109.png)

## TrunkPort 设计

TrunkPort的提出主要是为了解决以下几个用户问题：

1. 一台虚拟如果要加入100个的network，需要添加100个VIF.
2. 云计算场景下的负载网络变化十分常见，虚拟机中添加和删除VLAN相比添加和删除网卡更有效率。
3. 虚拟机从一个network迁移到另一个network，无需删除网卡。
4. 当虚拟机中运行大量容器（Kubernetes），不同容器需要连接不同的网络，设置VLAN相比为每个容器申请一张vNIC更有效率。
5. 一些传统的应用期望使用VLAN来连接不同的网络，Neutron需要提供该使用模式的支持。

使用Trunk前虚拟机加入多个network的网络模型如下。

![legacy model](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200209160637.png)

大量的 vNICs 需要占用大量资源，频繁的在虚拟机上添加和删除网卡效率也十分低下，无法满足当前云计算的快速变化的需求。

虚拟机使用Trunk后，只需要添加一次网卡，之后通过创建 VLAN 子接口实现网卡添加，相对于添加vNIC，效率提升明显。

![trunk model](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200209160741.png)

> 如何区分 parent port 和 subport，subport 和 parent port 如何连接。我们会在后文分析。

### 资源更新

我们接下来分析API 资源对象的变化，先是更新前 Port 资源对象的关系。

![legacy API](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200209161926.png)

下面是加入 `Trunk` 对象后的资源关系。

![Trunk API](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200209161952.png)

新的API中添加了扩展资源 `Trunk` 和 `SubPort`，在不改动已有 Port 的情况下，描述了Port的拓扑关系，实现 Trunk Port 效果。

### 约束

- parent port 必须对应创建一个 trunk 并引用 *port_id*
- 在同一个parent port下的subport必须有唯一的(segmentation_type, segmentation_id)。否则会无法区分逻辑源端口和目标端口
- parent port 会接收 "untagged" 流量，还会接收所有subport无法匹配的报文。如果不想要这个流量，可以设置安全组规则drop所有流量，或者把parent port添加到dummy network 中。
- port 继承 parent port 的 binding:host_id
- 尝试更新 subport 的 binding:host_id(例如绑定一个subport UUID启动VM)必须返回error
- 普通用户只能连接自己的subport和parent port。Admin用户可以连接不同用户的subport和parent port。
- 在OVS支持QinQ之前，使用OVS的Neturon不支持在"vlan_transparent"network中使用subport。因为这样会要求使用 netsted VLANs，目前还不支持

还有以下删除约束：

- 删除 trunk 自动删除它的所有 subports
- 删除一个被 subport 引用的 (child) port 是禁止的。必须先删除 subport
- 删除一个被 trunk 引用的 (parent) port 是禁止的。必须先删除 trunk。

### 数据表影响

Neutron Port: 没有变化
Neutron DB table: trunks

- id: integer primary key
- uuid
- name
- tenant_id
- port_id: foreign key to ports.id

Neutron DB table: subports

- id
- port_id: foreign key to ports.id
- trunk_id: foreign key to trunks.id
- segmentation_type
- segmentation_id

Neutron DB 约束:

- (segmentation_type, segmentation_id, trunk_id) 在 subports 表中必须唯一
- Port 如果被 trunks.port 引用，就不能被 subports.port_id 再引用，反之亦然。

### REST API 影响

**trunk resource**:

- id
- name
- tenant_id
- port_id

```shell
POST /v2.0/trunks
     ...
PUT /v2.0/trunks/TRUNK-ID/add_subports
PUT /v2.0/trunks/TRUNK-ID/delete_subports
     {'subports':
         [{'port_id': PORT_ID,
           'segmentation_type': SEGMENTATION_TYPE,
           'segmentation_id': SEGMENTATION_ID},
          ...
         ]}}
GET /v2.0/trunks/TRUNK-ID/subports
```

## TrunkPort 实现

我们先查看 `setup.cfg` 文件的配置，确认trunk相关的对象。

```ini
[entry_points]
neutron.core_plugins =
    ml2 = neutron.plugins.ml2.plugin:Ml2Plugin
neutron.service_plugins =
    trunk = neutron.services.trunk.plugin:TrunkPlugin
    ...
neutron.objects =
    SubPort = neutron.objects.trunk:SubPort
    Trunk = neturon.objects.trunk:Trunk
    ...
```

TrunkPort 特性增加了一个 service plugin `neutron.services.trunk.plugin:TrunkPlugin`。我们从这个 Plugin 入手开始分析 TrunkPort 实现。



## 参考资料

- [Neutron Server启动](http://xiehongfeng100.github.io/2017/11/09/openstack-neutron-server-startup/)
- [Neutron/TrunkPort](https://wiki.openstack.org/wiki/Neutron/TrunkPort)
- [specs/newton/vlan-aware-vms](http://specs.openstack.org/openstack/neutron-specs/specs/newton/vlan-aware-vms.html)