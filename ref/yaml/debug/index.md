---
title: 测试与调试相关
type: book
date: "2021-04-29"
weight: 20
---

## 最简单的 nginx 测试服务

``` yaml
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
      containers:
      - name: nginx
        image: nginx

---

apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: nginx
```

## 起一个带有调试工具的 Deployment

用 Deployment，不直接起裸 Pod，方便修改配置后滚动更新:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug
spec:
  replicas: 1
  selector:
    matchLabels:
      app: debug
  template:
    metadata:
      labels:
        app: debug
    spec:
      # nodeName: 10.10.10.10 # 指定调度到某个节点上
      containers:
      - name: debug
        image: cr.imroc.cc/library/net-tools:latest
```