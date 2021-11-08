---
title: "偶现超时"
type: book
---

## cpu 被限制了 (throttled)

如果 Pod 使用的 CPU 超过了 limit，或者容器内线程数太多，发生 CPU 争抢，会触发 cgroup 的 CPU throttle，限制 Pod 使用 CPU，自然也就可能导致服务超时 (进程分配的 CPU 时间片少了，处理就变慢了，就容易发生超时)。

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

## MTU 不一致

如果容器内网卡的 MTU 大于容器外的网卡 (veth 或主网卡)，就可能存在部分容器内发出的包，到节点后发现数据包过大，可能不会像交换机那样严谨会分片，直接就丢掉了； tcp 协商 mss 的时候，主要看的是进程通信两端网卡的 MTU，所以如果容器内网卡的 MTU 大于容器外的网卡，还可能存在数据包到了节点后，发现大小超过了节点网卡的 MTU，直接丢弃了。

MTU 大小可以通过 `ip address show` 或 `ifconfig` 来确认。