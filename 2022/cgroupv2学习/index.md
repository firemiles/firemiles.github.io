# cgroup v2学习


## 前言

这几年工作重心一直在容器网络方向，对虚拟化技术，容器隔离技术一直处于一知半解的状态，但是心里一直有着好奇，到底是怎样的技术，构建出了如今的容器化技术世界。项目中用到了cgroupV2，正好最近有时间，直接跳过cgroupV1学习了下cgroupV2的文档，这里分享内容主要来自[官方权威文档中文翻译](https://arthurchiao.art/blog/cgroupv2-zh/)。

## 什么是cgroup

`cgroup` 是 "control group" 的缩写，并且首字母永远不大写。`cgroup` 是一种以 hierarchical (树形层级)方式组织进程的机制，以及在层级中以受控和可配置的方式分发系统资源。

- 单数形式（cgroup）指这个特性，或用户 "cgroup controllers" 等术语中的修饰词
- 复数形式（cgroups） 显式地指多个cgroup

cgroup 是 Linux 内核的一个功能，用来限制、控制与分离一个进程组的资源（如CPU、内存、磁盘输入输出等）。它最早由Google的工程师（主要是Paul Menage和Rohit Seth）在2006年发起，最早名称为进程容器（process containers）。在2007年时，因为在Linux内核中，容器（container）这个名词有许多不同的意义，为避免混乱，被重新命名为cgroup，并且被合并到2.6.24版的内核中去。

cgroups的一个设计目标是为不同的应用情况提供统一的接口，从控制单一进程（像nice）到作业系统层虚拟化（像OpenVZ，Linux-VServer，LXC）。cgroups 提供：

- 资源限制：组可以被设置不超过设定的内存限制，包括虚拟内存。
- 优先级：一些组可能会得到大量的CPU或磁盘IO吞吐量。
- 结算：用来度量系统实际用了多少资源。
- 控制：冻结组或检查点和重启动。

cgroup 主要由两部分组成：
1. `核心（core）`：主要负责层级化地组织进程；
2. `控制器（controllers）`：大部分控制器负责cgroup层级中特定类型的系统资源分配，少部分utility控制器用于其他目的。

## 为什么会有cgroup

在 Linux 里，一直依赖就有对进程进行分组的概念和需求，比如 `session group`, `progress group` 等，后来随着人们对这方面的需求越来越多，比如需要追踪一组进程的内存和IO 使用情况等，于是出现了cgroup，用来统一将进程进行分组，并在分组的基础上对进程进行监控和资源控制管理等。

在 `CentOS 7` 中，通过将cgroup层级系统与systemd单位树捆绑， 可以把资源管理设置从进程级别移至应用程序级别。默认情况下，systemd 会自动创建 `slice`、`scope`和`service`单位的层级，来为 cgroup 树提供统一结构，可以通过 `systemctl` 命令创建自定义slice进一步修改结构。

![systemd cgroup slice](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/images/20220522144509.png)

如上图所示，系统默认创建了3个顶级`slice`(`System`,`User`和`Machine`)。每个slice都会获得相同CPU使用时间，如果`user.slice`想要获得`100%`的CPU使用时间，而此时CPU比较空闲，那么`user.slice`就能如愿以偿。

## 为什么会有cgroupV2

cgroup v1 支持多个 hierarchy，每个hierarchy可以启用任意数量的controller。这种方式看上去高度灵活，但实际中并不是很有用。例如：
1. utility 类型的 controller（例如freezer）本可用与多个hierarchy，而由于v1中每个controller只有一个实例，utility controller的作用就大打折扣；而hierarchy 一旦被populated之后，控制器就不能移动到其他hierarchy的事实，更是加剧了这个问题。
2. 另一个问题是，关联到某个hierarchy的所有控制器，只能拥有相同的hierarchy视图。无法在controller粒度改变这种视图。

![cgroup v1](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/images/20220522144949.png)

在实际中，大部分hierarhchy都启动了所有控制器，而实际上只有联系非常紧密的控制器——例如cpu和cpuacct放到同一个hierarchy中才有意义。最终结果是：
1. 用户控件最后管理这多个非常类似的hierarchy
2. 在执行hierarchy管理操作是，每个hierarchy上重复着相同的操作。

### 其他 cgroup 接口相关问题

v1的设计并没有前瞻性，因此后面引入了大量怪异特性和不一致性。

## cgroupV2的功能

cgroup v2将多个hierarchy的方式变成了 unified hierarchy，并将所有的controller挂载到一个unified hierarchy。

当前kernel没有一处 cgroup v1版本，允许cgroup v1和v2共存，但是相同的controller不能同时mount到这两个不同的cgroup版本中。

> 进程能否是同时属于cgroup v1和cgroup v2？可以的，查看 `/proc/$PID/cgroup` 会返现同时包含v1和v2 membership

Linux 社区 cgroup 分享：

{{<youtube z7mgaWqiV90>}}

### 进程/线程与 cgroup 关系

所有cgroup组成一个树形结构。

- 系统中的每个进程都属于且只属于某一个cgroup；
- 一个进程的所有线程属于同一个cgroup；
- 创建子进程时，继承其父进程的cgroup；
- 一个进程可以迁移到其他cgroup；
- 迁移一个进程时，子进程（后台进程）不会自动跟着一起迁移；

### 控制器

可以选择性地针对一个cgroup启用或禁用某些控制器；

控制器的所有行为是hierarchical的。
- 如果一个cgroup启用了某个控制器，它的sub-hierarchy中所有进程都会受控制
- 如果在更接近的root的节点上设置了资源限制，那下面的sub-hierarchy是无法覆盖也，也就是sub-hierarchy受parent限制。

### 基础操作
#### 挂载

```shell
mount -t cgroup2 none $MOUNT_POINT
```

cgroupv2 文件系统的magic number是`0x63677270`（"cgrp"）

- 所有支持v2且未绑定到v1的控制器，会被自动绑定到v2的hierarchy，出现在root层级中。
- v2中未使用的控制器也可以绑定到其他的hierarchies。

这说明我们能以完全向后兼容的方式混用v2和v1 hierarchy。

### 组织进程和线程
#### 进程：创建/删除/移动/查看 cgroup
初始状态下，只有root cgroup，所有进程都属于这个cgroup。

- 创建sub-cgroup：只需要创建一个子目录
  ```shell
  mkdir $CGROUP_NAME
  ```

+ 将进程移动到指定cgroup：将PID写入到相应cgroup的cgroup.procs文件即可。
  - 每次`write(2)`只能迁移一个进程
  - 如果进程有多个线程，那将任意线程PID写入到文件，都会将该进程的所有线程迁移到相应的cgroup。
  - 如果进程fork出一个子进程，那子进程属于执行fork操作时父进程所属的cgroup。
  - 进程退出（exit）后，仍然留在退出时它所属的cgroup，直到这个进程被收割（reap）；
  - 僵尸进程不会出现在cgroup.procs中，因此无法对僵尸进程执行迁移操作。

+ 删除cgroup/sub-cgroup
  - 如果一个cgroup 已经没有任何children或活进程，那直接删除对应的文件夹就删除该cgroup了
  - 如果一个cgroup已经没有children，但是还有僵尸进程，也认为这个cgroup是空的，可以直接删除

+ 查看进程的cgroup信息：`cat /proc/$PID/cgroup` 会列出该进程的 cgroup membership。如果启用了v1，这个文件可能会包含多行，每个hierarchy一行。v2对应的行永远是0::$PATH格式：
  ```shell
  $ cat /proc/$$/cgroup # ubuntu 20.04 上的输出，$$ 是当前 shell 的进程 ID
  12:devices:/user.slice
  11:freezer:/
  10:memory:/user.slice/user-1000.slice/session-1.scope
  9:hugetlb:/
  8:cpuset:/
  7:perf_event:/
  6:rdma:/
  5:pids:/user.slice/user-1000.slice/session-1.scope
  4:cpu,cpuacct:/user.slice
  3:blkio:/user.slice
  2:net_cls,net_prio:/
  1:name=systemd:/user.slice/user-1000.slice/session-1.scope
  0::/user.slice/user-1000.slice/session-1.scope  # v2
  ```

#### 线程
- cgroup v2 的一部分控制器支持线程粒度的资源控制，这种控制器称为threaded controllers。
  - 默认情况下，一个进程的所有线程属于同一个cgroup，
  - 线程模型使我们能将不同线程放到subtree的不同位置，同时还能保持二者在同一个资源域（resource domain）内。
- 不支持线程模型的控制器称为domain controllers。

将cgroup改成threaded模式（单向/不可逆操作）
cgroup 创建之后都是domain cgroup，可以通过下面的命令将其改成threaded模式
```shell
echo threaded > cgroup.type
```

### 控制器
- CPU
- Memory
- IO
- PID
- Cpuset
- Device Controller（基于cgroup BPF）
- RDMA
- HugeTLB
- Misc
  - perf_event

### cgroup 命名控件（cgroupns）
容器环境中用cgroup和其他一些namespace来隔离进程，但是`/proc/$PID/cgroup` 文件可能会泄露潜在的系统层信息。例如：
```shell
cat /proc/self/cgroup
0::/batchjobs/container_id1 # <-- cgroup 的绝对路径，属于系统层信息，不希望暴露给隔离的进程
```

因此引入了cgroup namespace。

## cgroupV2的实现
### cgroup 控制接口
![vfs cgroup](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/images/20220522152440.png)

Linux 使用了多种数据结构在内核中实现了cgroups的配置，关联了进程和cgroups节点，那么Linux又是如何让用户态进程使用cgroups的功能呢？Linux内核有一个很大的模块叫VFS（Virtual File System）。VFS能够把具体的文件系统细节隐藏起来，给用户态进程提供一个同一个为文件系统API接口。cgrpus与CFS之间衔接的部分称之为cgroup文件系统。

## cgroupV2的典型应用
听说cgroup是从docker开始，也是容器技术带火了cgroup技术，这里简单介绍下容器如何使用cgroup。

![kubernetes cgroup](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/images/20220522155818.png)

![kubernetes pod cgroup](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/images/20220522160937.png)

![kubernetes pod cgroup controller](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/images/20220522161034.png)

{{<youtube u8h0e84HxcE>}}

## 参考资料

- https://www.wikiwand.com/zh-hans/Cgroups
- https://arthurchiao.art/blog/cgroupv2-zh/
- https://segmentfault.com/a/1190000040980305
- https://juejin.cn/post/6844904110639054861
