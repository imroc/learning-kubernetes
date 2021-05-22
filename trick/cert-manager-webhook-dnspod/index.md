---
title: "使用 cert-manager 为 dnspod 的域名签发免费证书"
type: book
date: "2021-05-11"
weight: 40
---

## 概述

如果你的域名使用 [DNSPod](https://docs.dnspod.cn/) 管理，想在 Kubernetes 上为域名自动签发免费证书，可以使用 cert-manager 来实现。

cert-manager 支持许多 dns provider，但不支持国内的 dnspod，不过 cert-manager 提供了 [Webhook](https://cert-manager.io/docs/concepts/webhook/) 机制来扩展 provider，社区也有 dnspod 的 provider 实现。本文将介绍如何结合 cert-manager 与 [cert-manager-webhook-dnspod](https://github.com/qqshfox/cert-manager-webhook-dnspod) 来实现为 dnspod 上的域名自动签发免费证书。

## 基础知识

推荐先阅读  [使用 cert-manager 签发免费证书](https://imroc.cc/k8s/trick/sign-free-certs-with-cert-manager/) 。

## 创建 DNSPod 密钥

在 DNSPod 控制台，在 [密钥管理](https://console.dnspod.cn/account/token) 中创建密钥，然后复制自动生成的 `ID` 和 `Token` 并保存下来，以备后面的步骤使用。

## 安装 cert-manager-webhook-dnspod

阅读了前面推荐的文章，假设集群中已经安装了 cert-manager，下面使用 helm 来安装下 cert-manager-webhook-dnspod 。

首先准备下 helm 配置文件 (`dnspod-webhook-values.yaml`):

```yaml
groupName: example.your.domain # 写一个标识 group 的名称，可以任意写

secrets: # 将前面生成的 id 和 token 粘贴到下面
  apiID: "<ID>"
  apiToken: "<Token>"

clusterIssuer:
  enabled: true # 自动创建出一个 ClusterIssuer
  email: your@email.com # 填写你的邮箱地址
```

> 完整配置见 [values.yaml](https://github.com/qqshfox/cert-manager-webhook-dnspod/blob/master/deploy/cert-manager-webhook-dnspod/values.yaml)

然后使用 helm 进行安装:

```bash
git clone --depth 1 https://github.com/qqshfox/cert-manager-webhook-dnspod.git
helm upgrade --install -n cert-manager -f dnspod-webhook-values.yaml cert-manager-webhook-dnspod ./cert-manager-webhook-dnspod/deploy/cert-manager-webhook-dnspod
```

## 创建证书

创建 `Certificate` 对象来签发免费证书:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-crt
  namespace: istio-system
spec:
  secretName: example-com-crt-secret # 证书保存在这个 secret 中
  issuerRef:
    name: cert-manager-webhook-dnspod-cluster-issuer # 这里使用自动生成出来的 ClusterIssuer
    kind: ClusterIssuer
    group: cert-manager.io
  dnsNames: # 填入需要签发证书的域名列表，确保域名是使用 dnspod 管理的
  - example.com
  - test.example.com
```

等待状态变成 Ready 表示签发成功:

```bash
$ kubectl -n istio-system get certificates.cert-manager.io
NAME              READY   SECRET                   AGE
example-com-crt   True    example-com-crt-secret   25d
```

若签发失败可 describe 一下看下原因:

```bash
kubectl -n istio-system describe certificates.cert-manager.io example-com-crt
```

## 使用证书

证书签发成功后会保存到我们指定的 secret 中，下面给出一些使用示例。

在 ingress 中使用:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web
          servicePort: 80
  tls:
    hosts:
    - test.example.com
    secretName: example-com-crt-secret # 引用证书 secret
```

在 istio 的 ingressgateway 中使用:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: example-gw
  namespace: istio-system
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: HTTP-80
      protocol: HTTP
    hosts:
    - example.com
    - test.example.com
    tls:
      httpsRedirect: true # http 重定向 https (强制 https)
  - port:
      number: 443
      name: HTTPS-443
      protocol: HTTPS
    hosts:
    - example.com
    - test.example.com
    tls:
      mode: SIMPLE
      credentialName: example-com-crt-secret # 引用证书 secret
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: example-vs
  namespace: test
spec:
  gateways:
  - istio-system/example-gw # 转发规则绑定到 ingressgateway，将服务暴露出去
  hosts:
  - 'test.example.com'
  http:
  - route:
    - destination:
        host: example
        port:
          number: 80
```