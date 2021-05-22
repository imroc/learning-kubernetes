---
title: Bound Service Account Tokens
type: book
date: "2021-05-07"
weight: 10
draft: true
---

## 背景

Kubernetes 会自动为 Service Account 创建 JWT token，但存在一些问题:
1. token 中没有指定 `audience`，任何人拿到 token 后都可以伪装。
2. 每个 token 都会存到一个 `secret` 中，会占用 etcd 存储，大规模场景下可消耗大量 etcd 存储空间。
3. 只要有 `secret` 读权限，就可以拿到其它 service account 的 `token`，从而实现提权，存在安全隐患。
4. token 不会过期，只要 service account 还在就一直有效，当泄露后存在长期的安全隐患。

所以，提出了 [Bound Service Account Tokens](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/1205-bound-service-account-tokens) ，提供一种更安全的机制来管理 service account token，其核心是一个 `TokenRequest API`，在 KEP issue [#542](https://github.com/kubernetes/enhancements/issues/542) 中整体跟进。

## 设计


## 参考资料
* issue: https://github.com/kubernetes/enhancements/issues/542
* KEP: https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/1205-bound-service-account-tokens
* https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume
* https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection
