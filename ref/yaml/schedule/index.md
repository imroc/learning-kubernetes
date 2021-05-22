---
title: 调度相关
type: book
date: "2021-04-29"
weight: 10
---

## 尽量调度到不同节点

``` yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: k8s-app
            operator: In
            values:
            - kube-dns
      topologyKey: kubernetes.io/hostname
```

## 必须调度到不同节点

``` yaml
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - weight: 100
     labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname
```

## 只调度到有指定 label 的节点

``` yaml
affinity:
 nodeAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
     nodeSelectorTerms:
       matchExpressions:
       - key: ingress
         operator: In
         values:
         - true
```