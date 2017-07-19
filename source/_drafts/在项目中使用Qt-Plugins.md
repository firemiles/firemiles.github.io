---
title: 在项目中使用Qt Plugins
tags:
---

Qt 是一个很好的 C++ 开源库，提供了包罗万象的功能，我写项目常常愿意加入Qt，STL标准库那东西虽然也很强，但是提供的API还是不如Qt人性化。最近的一个项目我用到了Qt的Plugin机制，在这里进行下总结记录。

Qt支持两种类型的插件API，一种是称为 The High-Level API: Writing Qt Extensions，另一种是The Low-Level API: Extending Qt Applications。我使用的是第二种，第一种是为Qt本身添加插件而设计的API。



## 参考资料

1. [How to Create Qt Plugins](http://doc.qt.io/qt-5/plugins-howto.html)
