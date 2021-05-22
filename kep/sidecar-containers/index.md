---
title: Sidecar Containers
type: book
date: "2021-05-02"
weight: 1
---

## 背景

K8S 中 Pod 如果有多个 container，正常情况会同时启动或销毁，但有些场景对容器启动或销毁顺序有依赖，就可能存在一些问题，比如在 istio 场景中:
* Pod 启动时: 业务容器比 istio-proxy 先 ready。容器化过渡的应用，业务容器启动时需要调用其它服务(比如从配置中心拉取配置)，如果失败就退出，没有重试逻辑，而当 envoy 启动更慢时，业务容器调用其它服务失败，导致 pod 启动失败，如此循环 (参考 k8s issue [#65502](https://github.com/kubernetes/kubernetes/issues/65502) ，解决方案参考 [istio常见问题: Sidecar 启动顺序问题](https://imroc.cc/istio/faq/sidecar-startup-order/)) 。
* Pod 销毁时: 业务容器和 envoy 同时收到 SIGTERM，envoy 不再处理增量连接，但业务容器在 graceful shutdown 过程中可能需要调用另外的服务（比如通知其它清理进行清理操作)，这时候 envoy 就拒绝掉新的请求，导致调用失败 (参考 istio issue [#7136](https://github.com/istio/istio/issues/7136) ，解决方案参考 [istio常见问题: Sidecar 停止顺序问题](https://imroc.cc/istio/faq/sidecar-shutdown-order/)) 。

## 发起提案
社区很多人也都遇到了类似的问题，开始有人提出 Proposal 来解决:
* 在 2018-05， [Joseph Irving](https://github.com/Joseph-Irving) 发起 Sidecar Containers 的 [KEP](https://github.com/kubernetes/community/pull/2148)
* 随后在 2018-11 KEP 被[接受](https://github.com/kubernetes/community/pull/2148#issuecomment-442991599) 。
* 接着在 2019-01 作者又新开了 issue [#753](https://github.com/kubernetes/enhancements/issues/753) 来跟进这个特性的进展。

## 提案被废弃
经过两年的设计与开发，在 2020-10 社区意见出现分歧，最终宣布该 KEP 被废弃，见作者的 [评论](https://github.com/kubernetes/enhancements/issues/753#issuecomment-713471597) 。

还有文章闹过乌龙，称 1.18 会支持 sidecar 特性: [Sidecar container lifecycle changes in Kubernetes 1.18
](https://banzaicloud.com/blog/k8s-sidecars/) ，但事实证明最终没有，并且还被废弃了。

## 原因总结

总结一下原因就是，很多相关问题都是与 pod 生命周期管理有关，涉及很多场景，不仅仅是局限于一两个场景。 我们不能给每种场景都搞一个特性去解决，而是需要由一个能够从更高的高度解决所有问题的新提案来解决。

## 讨论新提案

随后，社区发起了 sidecar 相关场景与要求的搜集 [Sidecar use cases/requirements](https://docs.google.com/document/d/1Drw9C_Ljpcr4X9UPLvms1fn8uMRnTfJLb-xipgX4C1M/edit#heading=h.1kqwby7migh2) ，我印象比较深刻的有:
* Job 运行完毕退出，但 istio sidecar 不会退出，导致 Job 永不退出 (Job 需要等所有 container 停止才算退出)
* 升级 sidecar 版本会重启所有 Pod，对大集群不友好，能够支持单个 container 升级就好了

然后在 2020-11，Tim Hockin (K8S首席) 发起新 [Proposal](https://docs.google.com/document/d/1Q3685Ic2WV7jPo9vpmirZL1zLVJU91zd3_p_aFDPcS0) 草稿。

## 最新进展

然后就没有然后了，最近也没发现什么跟这个特性相关的动静，可能是要覆盖众多场景，就需要更复杂的设计，就没那么快能想好...
