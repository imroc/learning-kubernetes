---
title: "Pod 打散调度"
type: book
date: "2021-06-17"
weight: 100
---

## 概述

将 Pod 打散调度到不同地方，可避免因软硬件故障、光纤故障、断电或自然灾害等因素导致服务不可用，以实现服务的高可用部署。

Kubernetes 支持两种方式将 Pod 打散调度:
* Pod 反亲和 (Pod Anti-Affinity)
* Pod 拓扑分布约束 (Pod Topology Spread Constraints)

本文介绍两种方式的使用方法以及对比。

## 使用 podAntiAffinity 打散 Pod

在 Pod Spec 的 `affinity` 下指定 `podAntiAffinity` 即可配置 Pod 反亲和，下面给出示例。

**将 Pod 强制打散调度到不同节点上(强反亲和)，以避免单点故障**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: nginx
      containers:
      - name: nginx
        image: nginx
```

* `matchLabels` 替换成 Pod 实际使用的 label。
* 若不希望用强制，可以使用弱反亲和，让 Pod 尽量调度到不同节点:
  ```yaml
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
              - podAffinityTerm:
                  topologyKey: kubernetes.io/hostname
                weight: 100
  ```

**将 Pod 强制打散调度到不同可用区(机房)，以实现容灾**:

将 `kubernetes.io/hostname` 换成 `topology.kubernetes.io/zone`，其余同上。

## 使用 topologySpreadConstraints 打散 Pod

TODO

## 参考资料

* [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/)