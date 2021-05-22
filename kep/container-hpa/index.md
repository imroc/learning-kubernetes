---
title: Container HPA
type: book
date: "2021-02-22"
weight: 2
---

## 背景

k8s HPA 按照 Pod 中所有 container 的平均资源占用来判断是否扩缩容，当 Pod 中多个 container 中有 1 个占用非常高，但低于 HPA 定义的平均占用时，不会触发扩容，但实际上很需要扩容。Container HPA 就是为了解决这类问题，将 HPA 资源占用判断能细化到 container 级别，而不是 pod 级别。

## 历史

* 2019-12-17: [Arjun Naik](https://github.com/arjunrn) 提出 issue [#86349](https://github.com/kubernetes/kubernetes/issues/86349) 。
* 2020-03-11: 提出 KEP [PR #1609](https://github.com/kubernetes/enhancements/pull/1609) ；发起 issue [#1610](https://github.com/kubernetes/enhancements/issues/1610) 来跟进此特性。
* 2020-04-03: KEP [#1609](https://github.com/kubernetes/enhancements/pull/1609) 被接受。

## 进展

* 规划在 1.19 进入 alpha，但最终未能实现完，还没合入，见 [comment](https://github.com/kubernetes/enhancements/issues/1610#issuecomment-631216704)
* 有望在 1.20 合入，进入 alpha，见 [comment](https://github.com/kubernetes/enhancements/issues/1610#issuecomment-691667871)

## 参考资料

* 特性跟踪 issue: https://github.com/kubernetes/enhancements/issues/1610
* KEP 文档: https://github.com/kubernetes/enhancements/tree/master/keps/sig-autoscaling/1610-container-resource-autoscaling