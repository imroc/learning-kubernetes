---
title: TKE 相关
type: book
date: "2021-04-29"
weight: 100
---

## Service 相关

### 复用已有 CLB

```yaml
apiVersion: v1
kind: Service
metadata:
 annotations:
   service.kubernetes.io/tke-existed-lbid: lb-6swtxxxx # 替换为要复用的 CLB 的 ID
 name: nginx-service
```

> 参考 [官方文档](https://cloud.tencent.com/document/product/457/45491)

### 限制 CLB 只绑定部分节点 NodePort

避免绑太多机器导致 [CLB 健康检查探测频率过高](https://cloud.tencent.com/document/product/214/3394#.E5.81.A5.E5.BA.B7.E6.A3.80.E6.9F.A5.E6.8E.A2.E6.B5.8B.E9.A2.91.E7.8E.87.E8.BF.87.E9.AB.98):

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-backends-label: "group=access-layer"
  name: nginx-service
```

> 参考 [官方文档](https://cloud.tencent.com/document/product/457/45492#.E7.A4.BA.E4.BE.8B)

### Local 模式配置优化

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/local-svc-only-bind-node-with-pod: "true" # 只绑有 service 对应的 endpoint 的节点
    service.cloud.tencent.com/local-svc-weighted-balance: "true" # LB 转发加权平衡，endpoint 多的节点被转发几率就大，避免负载不均
  name: nginx-service
spec:
  externalTrafficPolicy: Local # Local 模式
  type: LoadBalancer
```

> 参考 [官方文档](https://cloud.tencent.com/document/product/457/45492#local-.E9.BB.98.E8.AE.A4.E5.90.8E.E7.AB.AF.E9.80.89.E6.8B.A9)

### CLB 直通 Pod

```yaml
kind: Service
apiVersion: v1
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true" ## 开启直连 Pod 模式
  name: my-service
```

> 参考 [官方文档](https://cloud.tencent.com/document/product/457/41897#yaml-.E6.93.8D.E4.BD.9C.E6.8C.87.E5.BC.95)

**注意:** 开启 CLB 直通 Pod 需集群网络支持 VPC-CNI 模式，有两种情况:
1. 集群网络模式只支持 VPC-CNI。
2. 集群网络模式同时支持 VPC-CNI 与 Global Router (两种混用)。

对于第一种，不需要做其它配置，对于第二种，要确保 Pod 使用了弹性网卡，在创建时需为 pod 指定 `tke.cloud.tencent.com/networks: "tke-route-eni"` 这个 annotation 并配置 `tke.cloud.tencent.com/eni-ip: "1"` 这个 request 和 limit，以下是一个 Deployment 示例:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        tke.cloud.tencent.com/networks: "tke-route-eni"
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          resources:
            requests:
              tke.cloud.tencent.com/eni-ip: "1"
            limits:
              tke.cloud.tencent.com/eni-ip: "1"
```
