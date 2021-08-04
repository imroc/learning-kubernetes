---
title: "排查网络丢包"
type: book
date: "2021-08-02"
---

## 检查连接队列是否满了

```bash
cat /proc/net/netstat | awk '/TcpExt/ { print $21,$22 }'
```

![](1.png)

> 不同内核版本的列号可能有差别
