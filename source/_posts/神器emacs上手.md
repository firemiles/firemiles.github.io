---
title: 神器Emacs上手
category: tool
date: 2017-02-03 20:46:44
tags: [emacs, tool]
photos:
comments: true
---

[<img src="https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2017-08-27-132249.jpg" width="20%" height="20">](http://spacemacs.org)

## 为什么是Emacs
vi（及其衍生软件）和Emacs的编辑器之战由来以久，vi被称为编辑器之神，而Emacs则是神的编辑器。两款软件有着自己的哲学，vi党认为vi专注编辑，通过扩展能够适应各种编辑需求，是最好的编辑器，继承了Unix小而美的传统；Emacs党认为Emacs不仅仅是编辑器，而是一款操作系统，通过使用Emacs，很多工作都可以在Emacs中完成，这种All-in-one的哲学也有很多人推崇，认为Emacs是一种信仰。

我曾经是坚定的vim党，认为编辑器就应该做编辑器应该做的事，直到遇上`Org-mode`。`Org-mode`是Emacs中的一个主模式，相信很多人使用Emacs就是为了使用`Org-mode`。相比简单的Markdown，Org-mode拥有更加丰富的语法功能，和Emacs的良好配合让写文档变成了一种享受，通过使用Emacs写Org的过程中接触了Emacs的其他功能，被它良好的文档和帮助系统，插件扩展系统深深的吸引，让我明白了，这就是我想要一直使用的工具，vim只是我临时使用会打开的编辑器，而Emacs是可以让我一直在里面工作的编辑器。

配置Emacs的过程中发现了[Spacemacs](https://github.com/syl20bnr/spacemacs)，这是一个民间的Emacs配置方案，使用layers机制将不同类型的功能进行良好的封装，由社区进行开发维护，开箱即用的特性让Emacs新手也能够很好的使用Emacs。安装了Spacemacs后发现Emacs的使用变得更加简单了，良好的交互式补全功能让Emacs的使用曲线不再那么陡峭。

![](https://firemiles-blog.oss-cn-shanghai.aliyuncs.com/2017-02-03-122036.jpg)

<!--more-->
## Spacemacs的安装
Spacemacs的安装十分简单，先安装Emacs软件，然后下载Spacemacs配置，根据官方仓库的说明操作。

```sh
$ git clone https://github.com/syl20bnr/spacemacs ~/.emacs.d
```

使用Spacemacs的配置前记得备份原先的配置。

为了方便Spacemacs升级，不要修改 `~/.emacs.d` 中的文件，新建文件夹`~/.spacemacs.d`，并将`.spacemacs`移动到`~/.spacemacs.d/init.el`，然后进行修改。Spacemacs会加载`~/.spacemacs.d`目录优先作为用户配置。


## Spacemacs 的简单配置
首先在`init.el`中添加进几个常用的layers。

```elisp
dotspacemacs-configuration-layers
'(
  html
  ;; ----------------------------------------------------------------
  ;; Example of useful layers you may want to use right away.
  ;; Uncomment some layer names and press <SPC f e R> (Vim style) or
  ;; <M-m f e R> (Emacs style) to install them.
  ;; ----------------------------------------------------------------
  osx
  chinese            ;; pinyin
  auto-completion
  better-defaults
  emacs-lisp
  git
  markdown
  org
  python
  ;; (shell :variables
  ;;        shell-default-height 30
  ;;        shell-default-position 'bottom)
  ;; spell-checking
  ;; syntax-checking
  ;; version-control
  )
```

为了方便管理配置文件，在`.spacemacs.d`中创建目录`custom`，作为用户配置目录。在`init.el`中的`user-config`函数中添加以下代码：

```elisp
(defun dotspacemacs/user-config ()
  "Configuration function for user code.
This function is called at the very end of Spacemacs initialization after
layers configuration.
This is the place where most of your configurations should be done. Unless it is
explicitly specified that a variable should be set before a package is loaded,
you should place your code here."
  ;; import user configurations
  (add-to-list 'load-path (expand-file-name "custom" dotspacemacs-directory))
  (require 'init-packages)
  (require 'init-ui)
  (require 'init-better-defaults)
  (require 'init-keybindings)
  (require 'init-org)
  (setq custom-file (expand-file-name "custom/custom.el" dotspacemacs-directory))
  (load-file custom-file)
  )
```

首先使用`add-to-list`将`custom`目录添加到`load-path`变量中，让Emacs可以从用户自定义目录中加载设置，然后是一系列的`require`命令，该命令会从`features`变量中寻找相同的名字，如果找到了相应的feature，则使用`load`函数进行加载，`load`会使用`load-file`函数加载相应的文件。如何将自定义的配置添加到`features`变量中呢，Emacs提供了一个`Provide`函数通过在配置文件中调用`(provide 'init-packages)`可以将该配置文件添加到`features`中，并命名为`init-packages`。

Emacs默认会将用户自定义自动生成在`init.el` 文件末尾。将自动生成的代码和手写的代码放在一起不是一个好的选择，因此将自动生成代码分离到`custom.el`中：

```elisp
(setq custom-file (expand-file-name "custom/custom.el" dotspacemacs-directory))
(load-file custom-file)
```

以上是Spacemacs的简单配置思路。

## Spacemacs 的缺点
如果说Spacemacs有什么缺点的话，就是太容易卡住了，常常需要使用命令：

```shell
$ pkill -SIGUSR2 -i emacs
```

该命令可以让Emacs停止耗时的操作，恢复响应。

Spacemacs还提供了精简版本，可以最大限度的提升速度，减少Bug发生，如果发现完整版本使用经常出现Bug，可以考虑使用精简版本。

```elisp
dotspacemacs-distribution 'spacemacs-base
```

更多Spacemacs的使用经验后续再继续更新。
