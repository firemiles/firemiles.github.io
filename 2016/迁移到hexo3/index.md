# 迁移到hexo3


好久没有更新 hexo，今天手贱把 hexo 和 node 都更新到了最新版本 hexo3 和 node6，然后就悲剧了，hexo 运行各种报错，各种百度后发现是 hexo3 改动比较大，很多功能都作为插件移出了 hexo，需要单独安装。在 hexo2 的基础上进行修改失败，最后决定重新构建一个 hexo3 工程进行迁移。

<!--more-->
## 安装

为了避免 node@6.x 的兼容问题，使用 nvm 安装并使用 node@5.8。

```shell
$ nvm install 5.8 && nvm use 5.8
$ npm install -g hexo-cli
$ hexo init blog && cd blog && npm i && hexo s
```

直接安装初始化默认版本的 hexo 并测试，hexo 默认会安装几个重要插件，包括 *hexo-server*。如果发现 `hexo server` 命令不能用，那么就有可能是 *_config.yml* 文件中的 *plugins* 配置有问题，尝试删除 *plugins* 配置再测试，我遇到的问题就是这个原因引起的。

> 一定要先测试默认配置，我直接用了 hexo2 的配置，然后打开服务器访问就是 404 错误.

## 迁移
安装并测试通过后，接下来就是迁移旧blog了。

更新 *_config.yml*，把 next 主题 copy 到 theme 目录下，顺便可以把 next 主题也一起更新了。迁移 *source* 下的文件夹到新的 *source* 目录下。

```shell
$ hexo g
$ hexo s
```

测试下迁移后的内容，没有问题迁移就成功了。

## 添加域名
阿里云搞活动买了个域名 firemiles.top ， 为了不浪费，给博客分配了个子域名 blog.firemiles.top。

在 *source* 下添加 CNAME 文件，内容就是 blog.firemiles.top。

## 参考链接

- [hexo](hexo.io)
- [hexo-issues](https://github.com/hexojs/hexo/issues)
- [Hexo搭建Github-Pages博客填坑教程](http://www.jianshu.com/p/35e197cb1273)
- [Hexo+Github域名和github绑定的问题](http://www.jianshu.com/p/1d427e888dda)

