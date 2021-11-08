---
title: "服务超时"
type: book
---

## cpu 被限制了 (throttled)

如果 Pod 使用的 CPU 超过了 limit，会触发 cgroup 的 CPU throttle，限制 Pod 使用 CPU，自然也就可能导致服务超时 (进程分配的 CPU 时间片少了，处理就变慢了，就容易发生超时)。

如果确认？可以查 Promehtues 监控，PromQL 查询语句:

1. cpu 被限制比例:
```txt
sum by (namespace, pod)(
    irate(container_cpu_cfs_throttled_periods_total{container!="POD", container!=""}[5m])
) /
sum by (namespace, pod)(
    irate(container_cpu_cfs_periods_total{container!="POD", container!=""}[5m])
)
```

2. cpu 被限制核数:
```txt
sum by (namespace, pod)(
    irate(container_cpu_cfs_throttled_periods_total{container!="POD", container!="", cluster="$cluster"}[5m])
)
```

## 节点高负载

如果节点高负载了，进程分配的 CPU 不够用，也会导致进程处理慢，从而超时，详见 [排查节点高负载](https://imroc.cc/k8s/troubleshooting/high-load/)