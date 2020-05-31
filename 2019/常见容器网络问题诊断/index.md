# 常见容器网络问题诊断


Kubernetes 的容器网络插件家族越来越庞大，各种模式层出不穷，但是万变不离其宗，overlay 和 underlay。overlay 常使用 vxlan 或者 ipip，underlay 使用 host-gw、bgp，或者结合云厂商提供的底层路由，如：vpc router，virtual switch；还有一类underlay 是目前各大云厂商主推的 eni（Elastic Network Interface）模式，将本来虚拟机使用的网卡插入到容器使用，使容器网络和虚拟机网络一样成为一等公民，拥有相同的网络访问权限，这个场景下容器网络被极度简化，大部分功能都由公有云虚拟机网络完成。

容器网络发展到现在，基本的组网方式已经十分成熟，抛开负载的性能优化问题不谈，网络功能问题的根因定位已经没多少新花样了，下面我就介绍下我在工作中使用的三板斧。

## 系统配置

系统配置被修改导致网络行为改变，容器网络故障是最常出现的。

### sysctl

- net.ipv4.ip_forward（需要打开）
- net.ipv4.conf.{all,interface}.rp_filter（非对称路由环境设置为0或2）
- net.ipv4.tcp_tw_recycle（NAT环境关闭）
- net.ipv4.tcp_tw_reuse

`net.ipv4.ip_forward` 控制主机是否允许转发数据报文，该值一般是 1，为0时常常导致容器访问service失败。

`net.ipv4.conf.all.rp_filter` 控制系统是否开启对数据包源地址的校验。先看下文档中的介绍：

```txt
rp_filter - INTEGER
0 - No source validation.
1 - Strict mode as defined in RFC3704 Strict Reverse Path
    Each incoming packet is tested against the FIB and if the interface
    is not the best reverse path the packet check will fail.
    By default failed packets are discarded.
2 - Loose mode as defined in RFC3704 Loose Reverse Path
    Each incoming packet's source address is also tested against the FIB
    and if the source address is not reachable via any interface
    the packet check will fail.

　　Current recommended practice in RFC3704 is to enable strict mode
to prevent IP spoofing from DDos attacks. If using asymmetric routing
or other complicated routing, then loose mode is recommended.
　　The max value from conf/{all,interface}/rp_filter is used
when doing source validation on the {interface}.
　　Default value is 0. Note that some distributions enable itin startup scripts.
```

即rp_filter参数有三个值，0，1，2，具体含义：

- 0: 不开启源地址校验
- 1: 开启严格的反向路径校验，对每个进来的包，校验其反向路径是否最佳。如果不是最佳，则丢弃该数据包
- 2: 开启松散的反向路径校验。对每个进来的包，校验其反向源地址是否可达（通过任意网口），如果不可达，则丢弃该数据包。

容器网络假如使用了非对称路由，rp_filter 一定要设置成2或者0.

`net.ipv4.tcp_tw_recycle` 参数是为了服务端快速回收 `TIME_WAIT` 状态的 sockets 设置的，一些网络调优的文档可能会提到设置这个参数，但是该参数并不建议在 NAT(Network Address Translation) 环境中使用。

TCP 为了避免序列号反转的问题，使用时间戳（非标准时间，只是一个计数器）识别过时的报文并丢弃，即 PAWS([PROTECT AGAINST WRAPPED SEQUENCE NUMBERS](https://www.freesoft.org/CIE/RFC/1323/13.htm))。默认PAWS是针对单个连接的，如果开启了 `net.ipv4.tcp_tw_recycle` ，为了解决重用 `TIME_WAIT` 导致的前一个连接发来的数据包在新连接中被当成有效数据包处理，Per-Connection 被升级成了 Per-Host。在SNAT环境中，来自不同HOST的连接被当成来自同一个HOST，导致时间戳数字小的友军会被系统误认为是过时报文丢弃。
> 该参数在 Linux 4.1 内核中已经移除

`net.ipv4.tcp_tw_reuse` 和上一个 `net.ipv4.tcp_tw_recycle` 功能类似，主要用于客户端快速回收 `TIME_WAIT` 状态的 port 重用。该参数当前还没有发现明显的问题，打不开开都可以。

### iptables

- iptalbes --policy FORWARD ACCEPT
- DROP
- SNAT

数据包经过主机时第二道关口一般是 iptables，常常 tcpdump 抓到了数据包，但是应用就是没有收到，这个时候很可能是 iptables 把数据包过滤了。

大多数容器网络需要iptables默认接受数据包转发，即设置：`iptables --policy FORWARD ACCEPT`。

这个可以用 iptables -nvL FORWARD 查看当前默认规则。

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20191124181730.png)

iptables 还有其他规则可能导致丢包，使用 `iptables -nvL ｜ grep DROP` 查看丢包统计。

除了丢包，错误的处理 SNAT 也可能导致容器网络访问外网失败，可以查看 iptables 的 SNAT 规则，并在主机上抓包确认。

### route

- default route
- ip rule

如果容器访问某个ip报错 `no route to host`，这个时候自然要查看下路由了。不光要排查容器内部，还要排查报文经过路径上的路由，并且需要注意 iptables 的 DNAT 规则改写目的ip，以及 `ip rule` 的 route table 查找规则。

## 定位工具

当我们排查了一遍系统参数还没有头绪的时候，这个时候就该网络调试工具上场了。

- tcpdump
- wireshark
- netstat/ss
- iptables
- lsof – p PID

使用 `tcpdump` 抓包是网络问题排查的常见手段。

```sh
tcpdump -i eth0 tcp host 192.168.4.3 and port 4789 -ne
```

也可以将抓到的包信息写入文件放到 wireshark 中分析，可以得到更全面的信息。

```sh
tcpdump -i eth0 tcp host 192.168.4.3 -w packet.pcap
```

`netstat` 或 `ss` 命令用来查看当前连接统计信息，可以分析连接失败的原因。

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20191124182528.png)

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/20191124182228.png)

`iptables` 需要排查规则是否符合期望，具体使用可以参见 [iptables快速入门](https://blog.firemiles.top/2019/03/16/iptables%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8/) 

`lsof` 可以用来查看进程占用的句柄数，有时候新的连接无法建立，可能是程序句柄泄露或者最大值设置太小（`ulimits -a`查看）

## 配置扫描脚本


```sh
## green to echo 
function green(){
    echo -e "\033[32m[ $1 ]\033[0m"
}

containerID=$1
green kernel: `uname -r`
green cni.conf
cat /etc/cni/net.d/cni.conf
green IP in container:
docker exec -ti $containerID ip a s
green Route in container:
docker exec -ti $containerID ip r s
green rp_filter in container:
docker exec -ti $containerID sysctl -a|grep rp_filter
green route in host:
ip r s
green ip rule in host:
ip rule
green ip forward in host:
sysctl sysctl net.ipv4.ip_forward
iptables -nvL|grep FORWARD
green rp_filter in host:
sysctl -a|grep rp_filter
green tcp_tw_recycle in host:
sysctl net.ipv4.tcp_tw_recycle
```

## 参考资料

- https://computingforgeeks.com/netstat-vs-ss-usage-guide-linux/





