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
certs:
  mode: auto
  auto:
    expiry: 43800h
    hosts: [ # 罗列 karmada 控制面可能会被访问的域名，这些会被放入 karmada-apiserver 的证书中，kubectl 或 agent 访问 karmada 控制面时需要校验证书
        "*.imroc.cc",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster.local",
        "*.karmada-system",
        "*.karmada-system.svc",
        "*.karmada-system.svc.cluster.local",
        "localhost",
        "127.0.0.1"
    ]
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
```

使用 helm 安装:

```bash
helm -n karmada-system upgrade --install -f values.yaml karmada roc/karmada
```

## 暴露 karmada-apiserver

karmada 部署好后，我们最好将它的 apiserver 给暴露出来，这样我们就可以在本地使用 kubectl 操作 karmada 了。

最简单的暴露方式就是直接修改 `karmada-apiserver` service 的 type 为 LoadBalancer，自动创建的 LB，使用 LB 暴露出来。

我这里使用 istio 的 ingressgateway 暴露，示例:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: karmada
  namespace: karmada-system
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 5443
      name: Karmada
      protocol: TCP
    hosts:
    - 'karmada.imroc.cc'

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: karmada-apiserver
  namespace: karmada-system
spec:
  hosts:
  - "karmada.imroc.cc"
  gateways:
  - karmada
  tcp:
  - route:
    - destination:
        host: karmada-apiserver
        port:
          number: 5443
```


## 获取 karmada 控制面 kubeconfig

karmada 部署好后会将控制面的 kubeconfig 保存到 host 集群的 secret 中，我们将其导出来:

```bash
kubectl -n karmada-system get secret karmada-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 -d > karmada
```

修改一下 server 地址为暴露出来后实际可以访问的地址; 如果有证书校验问题，我们可以将 cluster 中的 `certificate-authority-data` 字段删掉，并加上 `insecure-skip-tls-verify: false`，就可以忽略证书校验。

最后，可以使用 [kubecm](https://imroc.cc/k8s/trick/kubecm/) 将其合并到我们默认使用的 kubeconfig 中:

```bash
kubecm add -f karmada
```

然后我们就可以直接使用本地 kubectl 操作 karmada 了:

```bash
$ kubectl ctx karmada
Switched to context "karmada".
$ kubectl get ns
NAME              STATUS   AGE
default           Active   4d12h
karmada-cluster   Active   4d12h
karmada-system    Active   4d12h
kube-node-lease   Active   4d12h
kube-public       Active   4d12h
kube-system       Active   4d12h
```

## 安装 kubectl-karmada

Karmada 提供了 kubectl 插件，可以使用 [krew](https://krew.sigs.k8s.io/) 安装:

```bash
kubectl krew install karmada
```

krew 更新可能没那么及时，要安装最新版也可以直接源码编译安装:

```bash
git clone https://github.com/karmada-io/karmada.git
cd cmd/kubectl-karmada
go install
```

实际也可以完全不作为 kubectl 插件用，直接安装独立的命令 karmadactl:

```bash
git clone https://github.com/karmada-io/karmada.git
cd cmd/karmadactl
go install
```