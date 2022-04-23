# WSL2安装内核头文件


## 前言

一般 Linux 的发行版直接使用软件源就可以安装内核模块和内核头文件，但是WSL2作为一个特殊的版本，
大部分内核模块无法安装，也无法直接安装内核头文件，还有部分依赖内核头文件的工具也无法运行，本文主要介绍内核头文件的安装。

## 安装

- 第一步先去github上下载WSL2内核源码，仓库在 https://github.com/microsoft/WSL2-Linux-Kernel 。下载当前运行的内核版本。这里以`4.19.121`为例

- 开始安装依赖

```sh
apt install libelf-dev build-essential pkg-config
apt install bison build-essential flex libssl-dev libelf-dev bc
```

- 编译

```sh
tar -zvxf 4.19.121-microsoft-standard.tar.gz
cd WSL2-Linux-Kernel-4.19.121-microsoft-standard.tar.gz
zcat /proc/config.gz > .config
make -j $(nproc)
make -j $(nproc) modules_install
```

> 编译过程中可能会出现一些报错，一般是还缺少部分依赖，通过Google可以找到解决办法。

编译完成后，在 /lib/modules/ 会出现 4.19.121-microsoft-standard 目录，头文件就可以使用了，借助头文件，其他的内核模块也可以自行编译安装。

## 参考资料

- [WSL2 下的 kernel header 的安装](https://mashiro01.github.io/2020/08/24/wsl2%E4%B8%8B%E5%86%85%E6%A0%B8%E5%A4%B4%E6%96%87%E4%BB%B6/)

