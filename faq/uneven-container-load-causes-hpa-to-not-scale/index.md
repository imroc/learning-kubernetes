---
title: "container 负载不均导致 HPA 不扩容"
type: book
date: "2021-05-12"
weight: 10
---

## 问题描述

K8S HPA 算法是按照 Pod 中所有 container 的资源使用平均值来算的，如果 Pod 中有多个 container，它们的资源使用相差较大，可能导致某个 container 高负载了还不扩容。 

## 场景举例

istio 场景下，Pod 中有业务 container 和 sidecar container，如果业务 container 的 cpu 100%，而 sidecar container 的 CPU 0%，它们平均下来就是 50%，当 HPA 指定的是 CPU 60% 扩容，这种情况下不会扩容，导致业务高负载还不扩容，影响线上业务。

## 解决方案

1. 使用 VPA 根据 container 实际使用的资源大小来动态调整 container 的 request/limit，使得 HPA 计算出来的平均值能够比较客观反映 Pod 整体负载情况。
2. 使用 [Container HPA](https://imroc.cc/k8s/kep/container-hpa/) 特性，让 HPA 计算时更加 "聪明" 一点。