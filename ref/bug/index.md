---
title: "BUG 踩坑"
type: book
date: "2021-07-26"
weight: 4
---

## kubelet 与 apiserver 断连后仍然使用旧连接导致连接失败 (1.16-1.18)

* 现象: kubelet 连不上 apiserver，日志报错 `use of closed network connection`
* 相关 issue: [#87615](https://github.com/kubernetes/kubernetes/issues/87615)
* 原因: golang http 包的 bug，参考 [#34978](https://github.com/golang/go/issues/34978) 和 [#40213](https://github.com/golang/go/issues/40213)
* 解决方案:
  * 临时: 重启 kubelet
  * 彻底: 升级 K8S 版本到 1.19+

## 内核 bug 导致大量不必要的 cpu 限流
* 现象: cpu 使用远远没达到 limit，但仍然频繁被 throttle。
* 相关 issue: [#67577](https://github.com/kubernetes/kubernetes/issues/67577)
* 原因: 内核调度器的 bug 导致。
* KubeCon 分享: https://sched.co/Uae1
* KubeCon 分享视频: https://www.youtube.com/watch?v=UE7QX98-kO0
* 修复 commit: [de53fd7a](https://github.com/torvalds/linux/commit/de53fd7aedb100f03e5d2231cfce0e4993282425)
* 修复版本:
  * linux 官方稳定内核版本: 4.14.154+, 4.19.84+, 5.3.9+, 5.4+
  * 发行版内核: ubuntu 4.15.0-67+ 和 5.3.0-24+，RHEL7 kernel-3.10.0-1062.8.1.el7，TencentOS 4.14.105-19-0013+ ([commit](https://github.com/Tencent/TencentOS-kernel/commit/77d6fbc204337530f9030a4d700e00cd13618368))
