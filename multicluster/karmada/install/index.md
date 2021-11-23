---
title: 安装 Karmada
type: book
weight: 10
---

## 概述

本文记录如何将 Karmada 部署到 K8S 集群里 (host 集群)。推荐安装到 `karmada-system` 命名空间下， 务必提前创建好命名空间:

```bash
kubectl create ns karmada-system
```

## 安装 etcd

karmada 依赖 etcd，需要先部署 etcd，这里推荐不使用 karmada 的 chart 包中自带的 etcd，不支持挂载 pvc，不利于数据的持久化。

部署前先生成相关证书 (参考 [使用 cfssl 生成证书](https://imroc.cc/k8s/trick/generate-certs-with-cfssl/)):

```bash
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "SiChuan",
      "L": "ChengDu",
      "O": "Kubernetes",
      "OU": "CA"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF

cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "*.etcd-headless",
    "*.etcd-headless.karmada-system",
    "*.etcd-headless.karmada-system.karmada",
    "*.etcd-headless.karmada-system.karmada.svc",
    "*.etcd-headless.karmada-system.karmada.svc.cluster",
    "*.etcd-headless.karmada-system.karmada.svc.cluster.local",
    "*.karmada-system",
    "*.karmada-system.svc",
    "*.karmada-system.svc.cluster",
    "*.karmada-system.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "SiChuan",
      "L": "Chengdu",
      "O": "etcd",
      "OU": "etcd"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd
```

生成关键文件:
* `ca.pem`: CA 证书。
* `etcd-key.pem`: etcd 密钥。
* `etcd.pem`: etcd 证书。

> `etcd-csr.json` 中的域名是 etcd 可能会被访问的 service 域名，这里假设 etcd 部署到 karmada-system 命名空间，如果要部署到其它命名空间，替换下即可。

将生成的证书保存到 secret 中:

```bash
kubectl -n karmada-system create secret generic etcd-certs --from-file=cert.pem=etcd.pem --from-file=key.pem=etcd-key.pem --from-file=ca.crt=ca.pem
```

添加 helm repo:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

通过以下命令可将 etcd 完整配置项导出到 `etcd-fullvalues.yaml`:
```bash
helm show values bitnami/etcd --version 6.10.0 > etcd-fullvalues.yaml
```

我们只需要设置需要的一些配置项，准备 `values.yaml`:

```yaml
auth:
  rbac:
    enabled: false
    allowNoneAuthentication: true
  client:
    secureTransport: true # 启动 https，karmada 只通过 https 连接 etcd
    enableAuthentication: true
    useAutoTLS: false
    existingSecret: etcd-certs # 我们创建的 etcd 相关证书保存在这个 secret 中
  peer:
    secureTransport: true
    useAutoTLS: false
    existingSecret: etcd-certs
    enableAuthentication: true
replicaCount: 1
resources:
  requests:
    cpu: 100m
    memory: 512Mi
persistence:
  enabled: true # 启用数据持久化
  storageClass: "csi-cbs" # 自动创建 pvc
  size: 10Gi
metrics:
  enabled: false
readinessProbe:
  enabled: false
livenessProbe:
  enabled: false
```

使用 helm 部署 etcd:

```bash
helm -n karmada-system upgrade --install --version 6.10.0 -f values.yaml etcd bitnami/etcd
```

## 部署 Karmada

添加 helm repo (官方没有，我这里提供了一个):

```bash
helm repo add roc https://charts.imroc.cc
helm repo update
```

通过以下命令可将 etcd 完整配置项导出到 `karmada-fullvalues.yaml`:

```bash
helm show values roc/karmada > karmada-fullvalues.yaml
```

我们只需要设置需要的一些配置项，准备 `values.yaml`:
```yaml
installMode: "host"
etcd:
  mode: "external" # 使用我们自己部署的 etcd
  external:
    servers: "https://etcd.karmada-system.svc.cluster.local:2379" # etcd 访问地址
    registryPrefix: "/registry/karmada"
    certs: # 替换 ca 证书，etcd 证书和密钥的内容
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
scheduler:
  image:
    pullPolicy: Always # 默认是 IfNotPresent，tag 又是 latest，这里改成 Always 方便重建 pod 即可升级 karmada 组件
webhook:
  image:
    pullPolicy: Always
controllerManager:
  image:
    pullPolicy: Always
schedulerEstimator:
  image:
    pullPolicy: Always
agent:
  image:
    pullPolicy: Always
```

使用 helm 安装:

```bash
helm -n karmada-system upgrade --install -f values.yaml karmada roc/karmada
```