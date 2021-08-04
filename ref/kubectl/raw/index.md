---
title: "get --raw"
type: book
date: "2021-08-04"
---

## 获取节点 cadvisor 指标

```bash
kubectl get --raw=/api/v1/nodes/11.185.19.215/proxy/metrics/cadvisor
```

## 测试 Resource Metrics API

获取指定 namespace 下所有 pod 指标:

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/ns-prjzbsxs-1391012-production/pods/"
```

![](1.png)

获取指定 pod 的指标:

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/ns-prjzbsxs-1391012-production/pods/mixer-engine-0"
```

![](2.png)