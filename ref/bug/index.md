---
title: "BUG 踩坑"
type: book
date: "2021-07-26"
---

## kubelet 与 apiserver 断连后仍然使用旧连接导致连接失败 (1.16-1.18)

* 相关 issue: [#87615](https://github.com/kubernetes/kubernetes/issues/87615)
* 原因: golang http 包的 bug，参考 [#34978](https://github.com/golang/go/issues/34978) 和 [#40213](https://github.com/golang/go/issues/40213)
* 解决方案:
  * 临时: 重启 kubelet
  * 彻底: 升级 K8S 版本到 1.19+