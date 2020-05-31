# iptables 快速入门


iptables 是 [netfilter](https://www.wikiwand.com/en/Netfilter) 项目的一部分，自1998年首发至今已经20年了，仍然是Linux中网络流量控制的重要软件。学习网络知识，iptables 是绕不过的一道坎。不论是配置网络防火墙，NAT，IP流量转发，使用iptables都可以实现。

介绍iptables的文章已经有很多，在这里我再炒一遍冷饭，为的是把已有的资料糅合在一起，方便能在一篇文章里找到需要的知识。

## 基本概念

iptables 可以检测、修改、转发、重定向和丢弃IPv4数据包。过滤IPv4数据包的代码已经内置在内核中，并且按照不同的目的被组织成 **表** 的集合。 **表** 由一组预先定义的 **链** 组成，**链** 包含便利顺序规则。每一条规则包含一个谓词的潜在匹配和相应的动作（称为 **目标**），如果谓词为真，该动作会被执行。这就是**条件匹配**。iptables 是用户工具，允许用户使用 **链** 和 **规则**。**链** 和 **规则** 能组合成非常复杂的Linux IP 路由规则，新手面对这些规则时可能会感到气馁，但是，实际上最常用的一些应用案例（NAT或者基本网络防火墙）并不是很复杂。

理解iptables如果工作的关键是下面这张图。

![iptables规则顺序](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2019-03-16-131721.jpg)

从任何网络端口进来的每一个IP数据包都要从上到下穿过这张图。一种常见的困扰是认为iptables对从内部端口进入的数据包和从面向互联网端口进入的数据包采用不同的处理方式，相反，iptables对从任何端口进入的数据包都会采用相同的处理方式。可以定义规则使iptables采取不同的方式对待从不同端口进入的数据包。

这张图里的顺序有点复杂，我把它拆成两组顺序关系：**表** 的顺序和 **链** 的顺序。

表的顺序：

```
raw -> mangle -> nat -> filter
```

链的顺序：

![链的顺序](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2019-03-16-131325.jpg)

### 表（Tables）

iptables 包含5张表

- `raw` 用于配置数据包，`raw` 中的数据包不会被系统跟踪
- `filter` 是用于存放所有与防火墙相关操作的默认表
- `nat` 用于网络地址转换
- `mangle` 用于对特定数据包的修改
- `security` 用于强制访问控制（例如：SELinux）

大部分情况仅需要使用 `filter` 和 `nat` 。其他表用于更复杂的情况——包括多路由和路由判定。

### 链（Chains）

表由链组成，链是一些安装顺序排列的规则和列表。默认的 `filter` 表包含 `INPUT`，`OUTPUT` 和 `FORWARD` 3 条内建链。 `nat` 表包含 `PREROUTING`, `POSTROUTING` 和 `OUTPUT` 链。

![四表五链关系](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2019-03-16-131401.jpg)

### 规则（Rules）

数据包的过滤基于 **规则**。**规则**由一个目标（数据包包匹配所有条件后的动作）和很多匹配（导致该规则可以应用的数据包所满足的条件）指定。一个规则的典型匹配事项是数据包进入的端口（例如： eth0或者eth1）、数据包的类型（ICMP，TCP或者UDP）和数据包的目的端口。

目标使用 `-j` 或者 `--jump` 选项指定。目标是可以用户定义的链、一个内置的特定目标或者是一个目标扩展。内置目标有 `ACCEPT`,`DROP`,`QUEUE`和`RETURN`,目标扩展是`REJECT`和`LOG`。如果目标是内置目标，数据包的命运立刻被决定并且在当前表的数据包处理过程会停止。如果目标用户定义的链，并且数据包成功穿过第二条链，目标将移动到原始链中的下一个规则。目标扩展可以被终止（像内置目标一样）或者不终止（像用户定义链一样）。

### 遍历链（Traversing Chains）

上述流程图描述了在任何接口上收到的网络数据包是按照怎样的顺序走过表的交通管制链。第一个路由策略包决定数据包的目的地址是本地主机（这时穿过 `INPUT` 链），还是其他主机（穿过`FORWARD`链）;中间的路由策略决定传出的包使用哪个源地址、分配哪个接口；最后一个路由策略存在是因为先前的 mangle 与 nat 链可能会改变数据包的路由信息。数据包通过路径上的每一条链时，链中的每一条规则按顺序匹配；无论何时匹配了一条规则，相应的 target/jump 动作将会执行。最常用的 3 个 taget 是 `ACCPET` , `DROP` 或者 jump 到自定义链。内置的链有默认的策略，但是用户自定义的链没有默认策略。在jump到的链中，若每条规则都不能完全匹配，那么数据包像下图一样返回到调用链。

![调用链返回](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2019-03-16-131434.jpg)

`DROP` target 的规则完全能匹配，那么被匹配的数据包会被丢弃。如果一个数据包在链中被 `ACCEPT`，那么他会被所有的父链 `ACCEPT`。并且不在便利其他父链。然而，要注意的是，数据包还会以正常的方式继续遍历其他表中的其他链。

`REJECT` 拦阻该数据包，并返回数据包通知对方，可以返回的数据包有几个选择：ICMP port-unreachable、ICMP echo-reply 或者是 tcp-reset（这个数据包会要求对方变比连接），进行完此处理动作后，不再对比其他规则，直接中断过滤程序。范例如下：

```sh
iptables -A  INPUT -p TCP --dport 22 -j REJECT --reject-with ICMP echo-reply
```

`REDIRECT` 将封包重新导向另一个端口（PNAT），进行完此动作后，将会继续对比其他规则。这个动作可以用来实现透明代理或者保护web服务器。例如：

```sh
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT--to-ports 8081
```

`MASQUERADE` 改写封包来源IP为防火墙的IP，可以指定port对应的范围，进行完此处理动作后，直接跳往下一个规则链(mangle:postrouting)。这个功能与 SNAT 略有不同，当进行IP伪装时，不需要指定为装成哪个IP，IP会从网卡直接读取，当使用拨接连线时，IP通常是由ISP公司的DHCP服务器指派的，这个时候 `MASQUERADE` 特别有用，范例如下:

```sh
iptables -t nat -A POSTROUTING -p TCP -j MASQUERADE --to-ports 21000-31000
```

`LOG` 将数据包相关信息记录在 /var/log 中，详细位置请查阅 /etc/syslog/conf 配置文件，进行完此处理动作后，将会继续对比其他规则。例如：

```sh
iptables -A INPUT -p tcp -j LOG --log-prefix "input packet"
```

`SNAT` 改写封包来源为某特定IP或IP范围，可以指定port对应的范围，进行完此处理动作后，将直接跳往下一规则链（mangle:postrouting）。范例如下：

```sh
iptables -t nat -A POSTROUTING -p tcp -o eth0 -j SNAT --to-source 192.168.10.15-192.168.10.160:2100-3200
```

`DNAT` 改写数据包包目的地IP为某特定IP或IP范围，可以指定port对应的范围，进行完此处理动作后，将会直接跳往下一规则链（filter:input或filter:forward)

```sh
iptables -t nat -A PREROUTING -p tcp -d 15.45.23.67 --dport 80 -j DNAT --to-destination 192.168.10.1-192.168.10.10:80-100
```

`MIRROR` 镜像数据包，也就是将来源IP与目的IP对调后，将数据包返回，进行完此处理动作后将会中断过滤程序。

`QUEUE` 中断过滤程序，将封包放入队列，交给其他程序处理。透过自行开发的处理程序，可以进行其他应用，例如：计算联机费用……等。

`RETURN` 结束在目前规则链中的处理程序，返回父规则链中继续过滤。

`MARK` 将封包打上某个代号，以便提供作为后续过滤的条件判断依据，尽享完此处理动作后，将会继续对比其他规则，范例如下：

```sh
iptables -t mangle -A PREROUTING -p tcp --dport 22 -j MARK --set-mark 22
```

### 模块

有许多模块可以用来扩展 iptables，例如 connlimit， conntrack，limit和recent。这些模块增添了功能，可以进行更复杂的过滤。

## 配置并运行 iptables

iptables 是一个 systemd 服务，因此可以这样启动：

```sh
# systemctl start iptables
```

但是，除非有 `/etc/iptables/iptables.rules` 否则服务不会启动。因此第一次启动服务时如果没有该文件，使用以下命令创建：

```sh
# touch /etc/iptables/iptables.rules
```

### 命令行

**显示当前规则**

```sh
# iptables -nvL
-----------------------------------------------------
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination   
     
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination    
    
Chain OUTPUT (policy ACCEPT 0K packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

添加 `--line-numbers` 选项可以显示行号，插入规则是有用。

**重置规则**

使用这些命令刷新和重置iptalbes到默认状态：

```sh
# iptables -F
# iptables -X
# iptables -t nat -F
# iptables -t nat -X
# iptables -t mangle -F
# iptables -t mangle -X
# iptables -t raw -F
# iptables -t raw -X
# iptables -t security -F
# iptables -t security -X
# iptables -P INPUT ACCEPT
# iptables -P FORWARD ACCEPT
# iptables -P OUTPUT ACCEPT
```
没有任何参数的 `-F` 命令在当前表中刷新所有链。同样的, `-X` 命令删除表中所有非默认链。

**编辑规则**

有两种添加规则的方式，一种是在链上附加规则，另一种是将规则插入到链上的某个特定位置。

让计算机禁止转发数据包，将 `FORWARD` 链默认的路由规则由 `ACCEPT` 改成 `DROP`。

```sh
# iptables -P FORWARD DROP
```

Dropbox 的局域网同步特性，每30s广播数据包到所有可视的计算机，如果我们碰巧在一个拥有Dropbox客户端的局域网中，但是我们不想使用这个特性，那么我们希望拒绝这些数据包。

```sh
# iptables -A INPUT -p tcp --dport 17500 -j REJECT --reject-with icmp-port-unreachable
```

```sh
# iptables -nvL --line-numbers
------------------------------------------------------------------
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 REJECT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:17500 reject-with icmp-port-unreachable

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

现在我们希望安装Dropbox，也希望局域网同步，但是我们的网络只有一个特定的IP地址，因此需要使用 `-R` 参数来替换旧规则，只接收特定IP地址的广播包，`10.0.0.85`是我们另一个IP地址。

```sh
# iptables -R INPUT 1 -p tcp --dport 17500 ! -s 10.0.0.85 -j REJECT --reject-with icmp-port-unreachable
```

```sh
# iptables -nvL --line-numbers
-------------------------------------------------------------------
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 REJECT     tcp  --  *      *      !10.0.0.85            0.0.0.0/0            tcp dpt:17500 reject-with icmp-port-unreachable

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

我们现在允许 `10.0.0.85` 访问计算机 `17500` 端口，但是这条规则不可升级，如果有友好的Dropbox用户想要访问设备上的 '17500' 端口，我们应该马上允许，而不是在测试过所有防火墙规则后再允许。

因此我们在旧规则前插入一条新规则：

```sh
# iptables -I INPUT -p tcp --dport 17500 -s 10.0.0.85 -j ACCEPT -m comment --comment "Friendly Dropbox"
```

```sh
# iptables -nvL --line-numbers
-------------------------------------------------------------------
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 ACCEPT     tcp  --  *      *       10.0.0.85            0.0.0.0/0            tcp dpt:17500 /* Friendly Dropbox */
2        0     0 REJECT     tcp  --  *      *      !10.0.0.85            0.0.0.0/0            tcp dpt:17500 reject-with icmp-port-unreachable

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

改变第二条规则，使其拒绝`17500`端口的任何数据包：

```sh
# iptables -R INPUT 2 -p tcp --dport 17500 -j REJECT --reject-with icmp-port-unreachable
```

```sh
# iptables -nvL --line-numbers
-------------------------------------------------------------------
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 ACCEPT     tcp  --  *      *       10.0.0.85            0.0.0.0/0            tcp dpt:17500 /* Friendly Dropbox */
2        0     0 REJECT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:17500 reject-with icmp-port-unreachable

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

**保存规则**
通过命令行添加规则，规则文件不会自动改变，必须手动保存：

```sh
# iptables-save > /etc/iptables/iptables.rules
```

修改配置文件后，需要重新加载服务：

```sh
# systemctl reload iptables
```

或者通过iptables直接加载：

```sh
# iptables-restore < /etc/iptables/iptables.rules
```

## 日志

LOG 目标可以来记录匹配某个规则的数据包。和ACCEPT或DROP规则不同，进入LOG目标之后数据包会继续沿着链往下走。所以要记录所有丢弃的数据包，只需要在DROP规则前加上相应的LOG规则。但是这样会比较复杂，影响效率，所以应该创建一个`logdrop`链。

```sh
# iptables -N logdrop
# iptables -A logdrop -m limit --limit 5/m --limit-burst 10 -j LOG
# iptables -A logdrop -j DROP
```

现在任何时候想要丢弃数据包并且记录该事件，只要跳转到`logdrop`链，例如：

```sh
# iptables -A INPUT -m conntrack --ctstate INVALID -j logdrop
```

### 限制日志级别

上述 'logdrop' 链使用限制(limit)模式来防止iptables日志过大造成不必要的硬盘读写。

限制模式使用 `-m limit`，可以使用 `--limit` 来设置平均速率或者使用 `--limit-brust` 来设置起始触发速率。在上述 `logdrop` 里，添加一条记录所有通过其数据包的规则，开始的连续10个数据包将被记录，之后的每分钟只会记录5个数据包。

### 查看记录的数据包

记录的数据包作为内核信息，可以在 systemd journal看到，使用以下命令查看所有最近一次启动后所记录的数据包：

```sh
# journalctl -k | grep "IN=.*OUT=.*" | less
```

### ulogd

`ulogd` 是专门用于netfilter的日志工具，可以替代默认的LOG目标。

## FAQ

### tcpdump 可以抓到包，但是程序接收不到

tcpdump在进入iptables之前抓包，所有有可能数据包被iptables丢弃了。

## 参考资料

- https://wiki.archlinux.org/index.php/Iptables_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
- https://www.jianshu.com/p/6d77d73325bf
