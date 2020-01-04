---
title: openvswitch 使用速查
tags:
  - tool
  - openvswitch
photos:
  - https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20200101155834.png
description: openvswitch 已经成为 SDN 网络的一块基石，是网络开发者无法绕过的一道关口。本文总结 openvswitch 的模块结构以及命令行工具，方便日常开发查询。2020年的第一篇博客，我们就来谈谈openvswitch。
date: 2020-01-01 15:47:09
category: openvswitch 
comments: true
---

## OVS 架构

>本文主要针对内核态 openvsiwth，用户态 openvswitch 的使用和调试不在本文的讨论范围内。

![ovs架构](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/C369CE5D-F9F8-4ECE-A01F-3C12115E74A8.jpg)

`ovs` 架构如上图所示，主要由内核 `datapath` 和用户空间的 `vswitchd`、 `ovsdb` 组成。

![openvswitch internals](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/C718D270-8E15-4943-8A8E-A54C85EAF075.jpg)

### 主要模块

- `datapath` 是负责数据交换的内核模块，从网口读取数据，并快速匹配Flowtable中的流表项，成功的直接转发，失败的 upcall 到 `vswitchd` 处理。它在初始化`port binding` 的时候注册钩子函数，把端口的报文处理接收到 ovs 内核模块。
- `vswitchd` 是一个守护进程，是ovs的管理和控制服务，通过unix socket将配置信息保存到 `ovsdb` ，并通过 `netlink` 和内核模块交互。
- `ovsdb` 则是 ovs 的数据库，保存 ovs 的配置信息。

### client与组件对应关系

|组件|功能|
|---------|----------------------------------------------|
|ovs-dpctl|datapath控制器,可以创建删除DP,控制DP中的FlowTables,最常使用show命令，其他很少手动操作|
|ovs-ofctl|流表控制器，控制bridge上的流表，查看端口统计信息等|
|ovsdb-tool|专门管理ovsdb的client|
|ovs-vsctl|最常用的命令,通过操作ovsdb去管理相关的bridge,ports什么的|
|ovs-appctl|这个可以直接与openvswitch daemon进行交互|

### 常用子命令说明

- ovs-dpctl **show -s**
- ovs-ofctl **show, dump-ports, dump-flows, add-flow, mod-flows, del-flows**
- ovsdb-tools **show-log -m**
- ovs-vsctl
  - show 显示数据库内容
  - 关于桥的操作 **add-br, list-br, del-br, br-exits**
  - 关于port的操作 **list-ports, add-port, del-port, add-bond, port-to-br**
  - 关于interface 的操作 **list-ifaces, iface-to-br**
  - **ovs-vsctl list/set/get/add/remove/clear/destroy table record column [value]** 常见的表有bridge,controller,interface,mirror,netflow,open_vswitch,port,qos,queue,ssl,sflow.
- ovs-appctl **list-commands, fdb/show, qos/show**

## 日常操作

### 查看有哪些桥，桥中有哪些ports，哪些interfaces

```sh
root@l-network-1:~# ovs-vsctl list-br
br-ex
br-int
br-tun

root@l-network-1:~# ovs-vsctl list-ports br-tun
patch-int
vxlan-ac1c0509
vxlan-ac1c050d
vxlan-ac1c051c
vxlan-ac1c053f

root@l-network-1:~# ovs-vsctl list-ifaces br-tun
patch-int
vxlan-ac1c0509
vxlan-ac1c050d
vxlan-ac1c051c
vxlan-ac1c053f
# iface与ports同名.
```

### 查看 port，interface属于那个bridge， xxx-to-br 

```sh
root@l-network-1:~# ovs-vsctl port-to-br vxlan-ac1c0509
br-tun
root@l-network-1:~# ovs-vsctl iface-to-br vxlan-ac1c0509
br-tun
```

### 查看隐藏的流表规则，很少使用

```sh
root@l-network-1:# ovs-appctl bridge/dump-flows br-ex
duration=6067771s, n_packets=1313936898, n_bytes=881574100116, priority=0,actions=NORMAL
table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=2,recirc_id=0,actions=drop
table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=0,reg0=0x1,actions=controller(reason=no_match)
table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=0,reg0=0x2,actions=drop
table_id=254, duration=6067771s, n_packets=0, n_bytes=0, priority=0,reg0=0x3,actions=drop
```

### 查看一些有用的统计信息

查看datapath统计信息.主要关注lost数值.hit表示datapath命中数,missed未命中，lost表示没有传递到用户空间就丢弃了. 主要关注lost值是否上升，如果上升说明存在问题了.该命令可以使用-s选项，会将每个port的统计信息也显示出来。

```sh
root@l-network-1:# ovs-dpctl show
system@ovs-system:
        lookups: hit:2596765540 missed:6438005 lost:0
        flows: 39
        masks: hit:19416324706 total:10 hit/pkt:7.46
        port 0: ovs-system (internal)
        port 1: tapd9a71635-53 (internal)
        port 2: tap82059645-8e (internal)
        port 3: tap3f0507b3-70 (internal)

root@l-network-1:# ovs-dpctl show -s
system@ovs-system:
        lookups: hit:2596836956 missed:6438236 lost:0
        flows: 42
        masks: hit:19416902233 total:10 hit/pkt:7.46
        port 0: ovs-system (internal)
                RX packets:0 errors:0 dropped:0 overruns:0 frame:0
                TX packets:0 errors:0 dropped:0 aborted:0 carrier:0
                collisions:0
                RX bytes:0  TX bytes:0
        port 1: tapd9a71635-53 (internal)
                RX packets:4170 errors:0 dropped:0 overruns:0 frame:0
                TX packets:1091973 errors:0 dropped:0 aborted:0 carrier:0
                collisions:0
                RX bytes:906944 (885.7 KiB)  TX bytes:48392325 (46.2 MiB)
        port 2: tap82059645-8e (internal)
                RX packets:7130 errors:0 dropped:0 overruns:0 frame:0
                TX packets:7379 errors:0 dropped:0 aborted:0 carrier:0
                collisions:0
                RX bytes:641578 (626.5 KiB)  TX bytes:650492 (635.2 KiB)
        port 3: tap3f0507b3-70 (internal)
                RX packets:5505 errors:0 dropped:0 overruns:0 frame:0
                TX packets:7549 errors:0 dropped:0 aborted:0 carrier:0
```

ovs-ofctl dump-ports br [port]也可以查看port的统计信息.这个命令优势是可以指定port.

```sh
root@l-network-1:# ovs-ofctl dump-ports br-ex
OFPST_PORT reply (xid=0x2): 19 ports
  port LOCAL: rx pkts=78668524, bytes=34522064349, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=78827893, bytes=5746347600, drop=0, errs=0, coll=0 port 10: rx pkts=14386485, bytes=10843773932, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=15666927, bytes=10659584526, drop=0, errs=0, coll=0 port 26: rx pkts=3830, bytes=161388, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=3025213, bytes=244511981, drop=0, errs=0, coll=0 port 36: rx pkts=16, bytes=1200, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=898463, bytes=68248745, drop=0, errs=0, coll=0 port 25: rx pkts=4693, bytes=206958, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=3256605, bytes=264430395, drop=0, errs=0, coll=0 port 35: rx pkts=229447, bytes=21077911, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=1905393, bytes=758729599, drop=0, errs=0, coll=0 port 28: rx pkts=13, bytes=1074, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=2266685, bytes=179166266, drop=0, errs=0, coll=0 port 23: rx pkts=10345, bytes=636977, drop=0, errs=0, frame=0, over=0, crc=0 tx pkts=3345437, bytes=272099392, drop=0, errs=0, coll=0
```

### 查看bridge的转发表

```sh
root@l-network-1:# ovs-appctl fdb/show br-ex
 port  VLAN  MAC                Age
   10     0  fa:16:3e:f8:28:9f    9
   11     0  fa:16:3e:a7:d2:f5    8
   34     0  fa:16:3e:2f:b2:71    8
    4     0  2c:44:fd:89:cf:3a    6
    6     0  fa:16:3e:45:c7:c5    5
    4     0  fa:16:3e:44:15:eb    5
    4     0  2c:44:fd:8a:78:06    4
    1     0  fa:16:3e:be:08:35    2
    3     0  fa:16:3e:2f:dd:71    1
    2     0  fa:16:3e:d7:f0:c1    0
LOCAL     0  2c:44:fd:8a:32:ce    0
    4     0  20:0b:c7:37:d0:05    0
```

## 设置qos

```sh
ovs-vsctl set Interface tap0 ingress_policing_rate=100000 ovs-vsctl set Intervace tap ingress_policing_burst=10000 ovs-appctl qos/show <iface>
```

## 调试技巧

### 查看流表匹配规则

```sh
watch -d -n 1 "ovs-ofctl dump-flows <bridge>"
```

### 查看统计信息

```sh
ovs-dpctl show -s
ovs-ofctl dump-ports <br> [port]
```

### 使用tcpdump抓包,需要设置端口镜像

```sh
ip link add name snooper0 type dummy
ip link set dev snooper0 up
ovs-vsctl add-port br-int snooper0

ovs-vsctl -- set Bridge br-int mirrors=@m -- --id=@snooper0 \
  get Port snooper0  -- --id=@patch-tun get Port patch-tun \
  -- --id=@m create Mirror name=mymirror select-dst-port=@patch-tun \
  select-src-port=@patch-tun output-port=@snooper0 select_all=1

tcpdump -i snooper0

ovs-vsctl clear Bridge br-int mirrors
ovs-vsctl del-port br-int snooper0
ip link delete dev snooper0
```

### 查看日志

```sh
cat /var/log/openvswitch/*.log
```

### 使用ovs-appctl ofproto/trace {bridge} k1=v1,k2=v2测试流匹配

```sh
root@l-network-1:~# ovs-appctl ofproto/trace br-ex in_port=10,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8 -generate
Bridge: br-ex
Flow: in_port=10,vlan_tci=0x0000,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8,dl_type=0x0000

Rule: table=0 cookie=0 priority=0
OpenFlow actions=NORMAL
no learned MAC for destination, flooding

Final flow: in_port=10,vlan_tci=0x0000,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8,dl_type=0x0000
Megaflow: in_port=10,vlan_tci=0x0000/0x1fff,dl_src=66:4e:cc:ae:4d:20,dl_dst=46:54:8a:95:dd:f8,dl_type=0x0000
Datapath actions: 27,30,12,14,15,17,20,37,9,28,47,48,54,57,29,25,61,21
```

## 参考

- https://docs.openvswitch.org/en/latest/
- http://fishcried.com/2016-02-09/openvswitch-ops-guide/