# 用docker快速完成翻墙服务搭建


## 开始

起因是banwogon的ip又被墙了，由于bandwagon是包年的，换ip并不容易，因此决定使用别家的服务。

[Vultr](https://www.vultr.com/?ref=6905056) 相比 linode 等老牌厂商，界面简单清爽，提供了月付2.5刀，随时可以删除换ip的VPS，非常适合作为扶墙主机。

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-05-30-151301.png)

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2018-05-30-151558.png)

## 搭建

使用paypal完成付款，创建完一台 **1U/512M** 的VPC后，ssh登录机器，开始 **ssr** 的搭建。这里我推荐使用 **CentOS7** 镜像，稳定够用。

## 安装 docker

```
yum install docker
systemctl start docker
```

## 安装docker-ssr

```
docker run --restart always --privileged -d -p 4567:4567/tcp -p 4567:4567/udp --name ssr-bbr-docker letssudormrf/ssr-bbr-docker -p 4567 -k YOU-PASSWORD -m aes-128-ctr -O auth_aes128_sha1 -o http_post
```

自从docker出现后，很多事情变得简单。

一条命令就完成了ssr服务端的下载和启动，并且是支持TCP BBR的。启动命令中的具体参数的定义可以查看附录中的ssr-bbr-docker，这里就不多做解释了。命令中的aes-128-ctr、auth_aes128_sha1和http_post作为ssr客户端中的协议类型和混淆协议类型进行配置，端口推荐使用自定义端口，不要用常用的443，容易被封。

想了解大名鼎鼎的BBR的请点[这里](https://medium.com/google-cloud/tcp-bbr-magic-dust-for-network-performance-57a5f1ccf437)

## 配置 ssr 客户端

ssr各平台客户端的下载地址，具体配置可以参考客户端帮助文档，这里就不赘述了。

- [Windows](https://github.com/shadowsocks/shadowsocks-windows)
- [Mac](https://github.com/shadowsocks/shadowsocks-iOS/releases)
- [Android](https://github.com/shadowsocks/shadowsocks-android)
- [IOS](https://github.com/shadowsocks/shadowsocks-iOS/wiki/Help)
- [OpenWRT](https://github.com/shadowsocks/openwrt-shadowsocks)

## 附录
- [Vultr](https://www.vultr.com/?ref=6905056)
- [ssr-bbr-docker](https://github.com/work-on-docker/ssr-bbr-docker)
