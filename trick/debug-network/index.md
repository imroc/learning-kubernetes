---
title: 网络调试
type: book
date: "2021-05-12"
weigth: 10
draft: true
---

## 进入容器 netns

参考 [这里](https://imroc.cc/k8s/trick/enter-container-netns) 。

## 使用 kubectl 插件快捷抓包

参考 [使用 ksniff 抓包](https://imroc.cc/k8s/trick/summary/debug-network/) 。

## 不断尝试建立 TCP 连接测试网络连通性

``` bash
while true; do echo "" | telnet 10.0.0.3 443; sleep 0.1; done
```

* `ctrl+c` 终止测试
* 替换 `10.0.0.3` 与 `443` 为需要测试的 IP/域名 和端口

## 查看当前各种状态的 TCP 连接数

```bash
$ netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 18
TIME_WAIT 457
```

## 使用 wireshark 分析 dns

过滤没有收到响应的 dns 请求:

```txt
dns && (dns.flags.response == 0) && ! dns.response_in
```