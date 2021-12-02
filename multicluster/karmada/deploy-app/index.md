---
title: 部署应用
type: book
weight: 30
---

## 准备应用 yaml

这里我们用一个简单的 `nginx.yaml` 作为示例:

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
      containers:
      - name: nginx
        image: nginx
```

将 yaml 应用到 karmada 控制面中:

```bash
kubectl create ns test-nginx
kubectl -n test-nginx apply -f nginx.yaml
```

## 应用多集群部署

准备 `policy.yaml`:

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-policy
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: nginx
  placement:
    clusterAffinity:
      clusterNames:
      - cd
      - eks
```

> `clusterNames` 指定工作负载需要被部署到的集群

将 yaml 应用到 karmada 控制面中:

```bash
kubectl -n test-nginx apply -f policy.yaml
```

查看部署