---
title: "排查节点高负载"
type: book
date: "2021-07-14"
---

## 概述

Kubernetes 节点高负载如何排查？本文来盘一盘。

## 排查思路

通常不是因为内核 bug 导致的高负载，在卡死之前从监控一般能看出一些问题，可以观察下各项监控指标。

## 线程数量过多

通过以下命令统计 PID 数量:

```bash
ps -eLf | wc -l
```

如果数量过多，可以通过以下命令统计线程数排名:

```bash
printf "NUM\tPID\tCOMMAND\n" && ps -eLf | awk '{$1=null;$3=null;$4=null;$5=null;$6=null;$7=null;$8=null;$9=null;print}' | sort |uniq -c |sort -rn | head -10
```

找出线程数量较多的进程，可能就是某个容器的线程泄漏，导致 PID 耗尽。