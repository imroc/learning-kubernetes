---
title: "实用 PromQL"
type: book
date: "2021-11-18"
weight: 5
---

## 查看内存占 limit 比例

可用于定位内存泄露，如果有 container 内存持续上涨，涨到 100% 然后掉下来，如此循环，一般是内存泄露:

```txt
sum by (pod, container)(
  container_memory_usage_bytes{container!~"POD|sandbox|", image!="", cluster="$cluster", namespace="$namespace", pod=~"nginx-.+"}
) /
sum by (pod, container)(
  kube_pod_container_resource_limits_memory_bytes{cluster="$cluster", namespace="$namespace", pod=~"nginx-.+"}
)
```


## 查看 CPU 限流比例

```txt
sum by (namespace, pod)(
    irate(container_cpu_cfs_throttled_periods_total{container!~"POD|sandbox|", cluster="$cluster", namespace=~"$namespace", pod=~"nginx-.+"}[5m])
) /
sum by (namespace, pod)(
    irate(container_cpu_cfs_periods_total{container!~"POD|sandbox|", cluster="$cluster", namespace=~"$namespace", pod=~"nginx-.+"}[5m])
)
```