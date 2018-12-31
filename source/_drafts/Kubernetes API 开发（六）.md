---
title: Kubernetes API 开发（六）
tags:
---

## Alpha，Beta 以及稳定版本

新特性的开发需要经历逐渐成熟的几个阶段：

- Development level
  - Object Versioning：不需要conversion
  - Availablity：没有commit到kubernetes代码主分支，因此在official release中也无法访问。
  - Audience：其他开发着合作开发feature或者概念验证。
