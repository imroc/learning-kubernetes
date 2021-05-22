---
title: "调试集群组件"
type: book
date: "2021-05-12"
weight: 20
id: "ed3704134f35724853d61de708d2a4e6"
---

## 概述

本文介绍一些常用的 Kubernetes 集群相关组件的调试方法。

## 动态调整 kubelet 日志级别

```bash
curl -k -X PUT --header "Authorization: Bearer <token>" https://127.0.0.1:10250/debug/flags/v -d "4"
```

## 获取 kubelet 堆栈日志

```bash
curl -k --header "Authorization: Bearer <token>" "https://127.0.0.1:10250/debug/pprof/goroutine?debug=2" > stack.txt
```

## 动态调整 scheduler 日志级别

```bash
# 默认日志级别为2，这里调为4。别忘记最后调回去。
curl -X PUT http://localhost:10251/debug/flags/v -d "4"
```

## 获取 dockerd 堆栈日志

```bash
kill -SIGUSR1 `$(pidof dockerd)`
```

## 查看 containerd task/container 状态

```bash
# 查看task 
docker-container-ctr -a /run/docker/containerd/docker-containerd.sock -n moby t list
# 查看容器
docker-container-ctr -a /run/docker/containerd/docker-containerd.sock -n moby c list
```

## 获取 containerd 堆栈日志

```bash
curl --unix-socket /var/run/docker/containerd/docker-containerd-debug.sock http://./debug/pprof/goroutine?debug=2
```

## 模拟 containerd 投递事件

```bash
contianerd  -a /run/docker/containerd/docker-containerd.sock -n moby publish 
```