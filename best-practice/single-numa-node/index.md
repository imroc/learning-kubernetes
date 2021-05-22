---
title: "Pod 绑定 NUMA 亲和性"
type: book
date: "2021-05-08"
weight: 100
draft: true
---

## 前提条件
* 内核启用 NUMA: 确保 `/etc/default/grub` 中没有 `numa=off`，若有就改为 `numa=on`
* k8s 1.18 版本以上

## 参考资料
* https://docs.qq.com/doc/DSkNYQWt4bHhva0F6
* https://blog.csdn.net/nicekwell/article/details/9368307
* [为什么 NUMA 会影响程序的延迟](https://draveness.me/whys-the-design-numa-performance/)
