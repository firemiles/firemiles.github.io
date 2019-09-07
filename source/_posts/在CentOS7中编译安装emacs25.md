---
title: 在CentOS7中编译安装emacs25
description: CentOS7自带的emacs版本太低，想要更高版本只能编译安装。
date: 2018-01-06 00:24:46
tags: [emacs, tool]
category: tool
comments: true
---

## 讲在前面
目前使用spacemacs越来越顺手，每个环境我都要装一个，但是在CentOS7环境下默认的emacs版本是23，spacemacs只支持24以上版本，因此需要自行编译安装源代码，本文记录编译安装流程，方便后续查用。

<!--more-->

本文记录安装的包可能并不是最精简的，可以自行根据需要进行裁剪。

## 下载源码
```
wget http://mirrors.ustc.edu.cn/gnu/emacs/emacs-25.3.tar.gz 
tar xzvf emacs-25.3.tar.gz 
cd emacs-25.3
```

## 安装编译依赖
```
yum install -y gcc gtk2 gtk2-devel libXpm libXpm-devel libtiff libtiff-devel libjpeg-turbo  libjpeg-turbo-devel giflib giflib-devel ncurses-devel texinfo
```

## 编译&安装
```
./configure --prefix=/usr/local/emacs-25
make
make install
```

## 下载spacemacs和configure
```
git clone https://github.com/syl20bnr/spacemacs ~/.emacs.d/
git clone https://github.com/firemiles/configures.git ~/configure 
ln -s ~/Workspace/configures/comm/emacs/spacemacs/spacemacs-private/ ~/.spacemacs.d
```

至此，spacemacs编译安装完成，尽情享受spacemacs的美妙吧。


