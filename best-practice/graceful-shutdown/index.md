---
title: "优雅终止"
type: book
date: "2021-05-29"
weight: 2
---

## 概述

Pod 销毁时，会停止容器内的进程，通常在停止的过程中我们需要执行一些善后逻辑，比如等待存量请求处理完以避免连接中断，或通知相关依赖进行清理等，从而实现优雅终止目的。本文介绍在 Kubernetes 场景下，实现容器优雅终止的最佳实践。

## 容器终止流程

我们先了解下容器在 Kubernetes 环境中的终止流程:

1. Pod 被删除，状态置为 `Terminating`。
2. kube-proxy 更新转发规则，将 Pod 从 service 的 endpoint 列表中摘除掉，新的流量不再转发到该 Pod。
3. 如果 Pod 配置了 [preStop Hook](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) ，将会执行。
4. kubelet 对 Pod 中各个 container 发送 `SIGTERM` 信号以通知容器进程开始优雅停止。
5. 等待容器进程完全停止，如果在 `terminationGracePeriodSeconds` 内 (默认 30s) 还未完全停止，就发送 `SIGKILL` 信号强制杀死进程。
6. 所有容器进程终止，清理 Pod 资源。

## 业务代码处理 SIGTERM 信号

要实现优雅终止，务必在业务代码里面处理下 `SIGTERM` 信号，参考 [处理 SIGTERM 代码示例](https://imroc.cc/k8s/ref/code-example-of-handle-sigterm/) 。

## 别让 shell 导致收不到 SIGTERM 信号

如果容器启动入口使用了脚本 (如 `CMD ["/start.sh"]`)，业务进程就成了 shell 的子进程，在 Pod 停止时业务进程可能收不到 `SIGTERM` 信号，因为 shell 不会自动传递信号给子进程。更详细解释请参考 [为什么我的容器收不到 SIGTERM 信号 ?](https://imroc.cc/k8s/faq/why-cannot-receive-sigterm/)

如果解决？请参考 [实用技巧: 在 SHELL 中传递信号](https://imroc.cc/k8s/trick/propagating-signals-in-shell/) 。

## 合理使用 preStop Hook

若你的业务代码中没有处理 `SIGTERM` 信号，或者你无法控制使用的第三方库或系统来增加优雅终止的逻辑，也可以尝试为 Pod 配置下 preStop，在这里面实现优雅终止的逻辑，示例:

```yaml
        lifecycle:
          preStop:
            exec:
              command:
                - /clean.sh
```

> 参考 [Kubernetes API 文档](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#lifecycle-1)

## 调整优雅时长

如果需要的优雅终止时间比较长 (preStop + 业务进程停止可能超过 30s)，可根据实际情况自定义 `terminationGracePeriodSeconds`，避免过早的被 `SIGKILL` 杀死，示例:

![](1.png)