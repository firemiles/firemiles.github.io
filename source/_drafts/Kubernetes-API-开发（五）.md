---
title: Kubernetes API 开发（五）
photos:
  - image url
description: 你对本页的描述
date: 2018-12-20 23:15:12
tags:
category:
comments:
---

## 添加一个API Version

如果你要为一个 Group 添加一个新的 API Version，你可以从 `pkg/apis/<group>/<existing-version>` copy 到 `staging/src/k8s.io/api/<group>/<existing-version>` 目录中。

<!--more-->
