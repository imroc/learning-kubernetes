---
title: "排查节点高负载"
type: book
date: "2021-07-14"
---

## 概述

Kubernetes 节点高负载如何排查？本文来盘一盘。

## 排查思路

通常不是因为内核 bug 导致的高负载，在卡死之前从监控一般能看出一些问题，可以观察下各项监控指标，如果没有相关监控或监控维度较少不足以查出问题，就尝试登录节点抓现场分析。

## 线程数量过多

如果 load 高但 CPU 利用率不高，通常是进程/线程数过多，排队等 CPU 切换的进程/线程较多。

通过以下命令统计 PID 数量:

```bash
ps -eLf | wc -l
```

如果数量过多，可以大致扫下有哪些进程，如果有大量重复启动命令的进程，就可能是这个进程对应程序的 bug 导致。

还可以通过以下命令统计线程数排名:

```bash
printf "NUM\tPID\tCOMMAND\n" && ps -eLf | awk '{$1=null;$3=null;$4=null;$5=null;$6=null;$7=null;$8=null;$9=null;print}' | sort |uniq -c |sort -rn | head -10
```

找出线程数量较多的进程，可能就是某个容器的线程泄漏，导致 PID 耗尽。

随便取其中一个 PID，用 nsenter 进入进程 netns:

```bash
nsenter -n --target <PID>
```

然后执行 `ip a` 看下 IP 地址，如果不是节点 IP，通常就是 Pod IP，可以通过 `kubectl get pod -o wide -A | grep <IP>` 来反查进程来自哪个 Pod。