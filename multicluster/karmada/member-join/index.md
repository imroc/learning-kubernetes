---
title: 添加成员集群
type: book
weight: 20
---

## 概述

本文记录如何给 Karmada 添加成员集群。

## 成员集群模式

Karmada 的成员集群支持 push 和 pull 两种模式：
1. push 模式: Karmada 控制面将应用推送到成员集群。
2. pull 模式: 成员集群中安装 Karmada Agent，Agent 会将应用从控制面中下拉到本地。

![1](1.jpg)

## 添加 push 模式成员集群

这种模式最简单也最常用，直接用 `kubectl karmada join` 或 `karmadactl join` 就行了：
1. 拿到成员集群的 kubeconfig，确保该 kubeconfig 中的 server 地址能够被 karmada 控制面所访问到。
2. 确保 kubectl 当前 context 是 karmada 控制面 (如 `kubectl ctx karmada`)。
3. 添加集群:
   ```bash
   kubectl karmada join prod --cluster-kubeconfig=prod --cluster-context=prod
   ```
## 添加 pull 模式成员集群

这种模式需要在成员中安装 agent，准备 `values.yaml`:

```yaml
installMode: "agent"

agent:
  image: # 默认是 IfNotPresent，tag 又是 latest，这里改成 Always 方便重建 pod 即可升级 karmada 组件
    pullPolicy: Always
  clusterName: "prod"
  kubeconfig:  # 从 karmada 的 kubeconfig 中提取相关证书与密钥
    caCrt: |
      -----BEGIN CERTIFICATE-----
      XXXXXXXXXXXXXXXXXXXXXXXXXXX
      -----END CERTIFICATE-----
    crt: |
      -----BEGIN CERTIFICATE-----
      XXXXXXXXXXXXXXXXXXXXXXXXXXX
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      XXXXXXXXXXXXXXXXXXXXXXXXXXX
      -----END RSA PRIVATE KEY-----
    server: "https://karmada.imroc.cc:5443" # 务必用成员集群能够访问到的 karmada-apiserver 地址
```

使用 helm 安装 agent 到成员集群:

```bash
kubectl create ns karmada-system
helm -n karmada-system upgrade --install -f values.yaml karmada-agent roc/karmada
```

## 如何选择？

我建议尽量使用 push 模式，相比 pull 模式的优势:
1. 注册与注销的方法非常简单，也无需在成员集群安装 agent，易于维护。
2. 支持 `karmadactl get` 操作，方便在 karmada 控制集群就能看到工作负载在各个集群的部署情况。

只有控制集群无法主动访问到成员集群 apiserver 地址的情况下，才选择 pull 模式。