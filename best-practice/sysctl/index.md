---
title: "内核参数优化"
type: book
---

## Node 调参汇总

```bash
# conntrack优化
net.netfilter.nf_conntrack_tcp_be_liberal = 1 # 容器环境下，开启这个参数可以避免 NAT 过的 TCP 连接 带宽上不去。
net.netfilter.nf_conntrack_tcp_loose = 1 
net.netfilter.nf_conntrack_max = 3200000
net.netfilter.nf_conntrack_buckets = 1600512
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30

# 以下三个参数是 arp 缓存的 gc 阀值，相比默认值提高了，避免在某些场景下arp缓存溢出导致网络超时，参考：https://k8s.imroc.io/troubleshooting/cases/arp-cache-overflow-causes-healthcheck-failed
net.ipv4.neigh.default.gc_thresh1="2048"
net.ipv4.neigh.default.gc_thresh2="4096"
net.ipv4.neigh.default.gc_thresh3="8192"

net.ipv4.tcp_max_orphans="32768"
vm.max_map_count="262144"
kernel.threads-max="30058"
net.ipv4.ip_forward="1"

# 磁盘 IO 优化: https://www.cnblogs.com/276815076/p/5687814.html
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 5
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 50
vm.dirty_ratio = 10
vm.dirty_writeback_centisecs = 50
vm.dirtytime_expire_seconds = 43200

```

### nf_conntrack_tcp_be_liberal

容器环境下，不开启这个参数可能造成 NAT 过的 TCP 连接带宽上不去或经常断连。

现象是有一点时延的 TCP 单流速度慢或经常断连，比如:
1. 跨地域专线挂载 nfs ，时延 5ms，下载速度就上不去，只能到 12Mbps 左右。
2. 经过公网上传文件经常失败。

原因是如果流量存在一定时延时，有些包就可能 out of window 了，netfilter 会将 out of window 的包置为 INVALID，如果是 INVALID 状态的包，netfilter 不会对其做 IP 和端口的 NAT 转换，这样协议栈再去根据 ip + 端口去找这个包的连接时，就会找不到，这个时候就会回复一个 RST，但这个 RST 是直接宿主机发出，容器内不知道，导致容器内还以为连接没断不停重试。 所以如果数据包对应的 TCP 连接做过 NAT，在 conntrack 记录了地址转换信息，也有可能部分包因 out of window 不走 conntrack 转换地址，造成一些混乱导致流量速度慢或卡住的现象。

## Pod 调参汇总

```bash
# socket buffer优化
net.ipv4.tcp_wmem = 4096        16384   4194304
net.ipv4.tcp_rmem = 4096        87380   6291456
net.ipv4.tcp_mem = 381462       508616  762924
net.core.rmem_default = 8388608
net.core.rmem_max = 26214400 # 读写 buffer 调到 25M 避免大流量时导致 buffer 满而丢包 "netstat -s" 可以看到 receive buffer errors 或 send buffer errors
net.core.wmem_max = 26214400
 
# timewait相关优化
net.ipv4.tcp_max_tw_buckets = 131072 # 这个优化意义不大
net.ipv4.tcp_timestamps = 1  # 通常默认本身是开启的
net.ipv4.tcp_tw_reuse = 1 # 仅对客户端有效果，对于高并发客户端，可以复用TIME_WAIT连接端口，避免源端口耗尽建连失败
net.ipv4.ip_local_port_range="1024 65535" # 对于高并发客户端，加大源端口范围，避免源端口耗尽建连失败（确保容器内不会监听源端口范围的端口)
net.ipv4.tcp_fin_timeout=30 # 缩短TIME_WAIT时间,加速端口回收
 
# 握手队列相关优化
net.ipv4.tcp_max_syn_backlog = 10240 # 没有启用syncookies的情况下，syn queue(半连接队列)大小除了受somaxconn限制外，也受这个参数的限制，默认1024，优化到8096，避免在高并发场景下丢包
net.core.somaxconn = 65535 # 表示socket监听(listen)的backlog上限，也就是就是socket的监听队列(accept queue)，当一个tcp连接尚未被处理或建立时(半连接状态)，会保存在这个监听队列，默认为 128，在高并发场景下偏小，优化到 32768。参考 https://imroc.io/posts/kubernetes-overflow-and-drop/
net.ipv4.tcp_syncookies = 1

# fd优化
fs.file-max=1048576 # 提升文件句柄上限，像 nginx 这种代理，每个连接实际分别会对 downstream 和 upstream 占用一个句柄，连接量大的情况下句柄消耗就大。
fs.inotify.max_user_instances="8192" # 表示同一用户同时最大可以拥有的 inotify 实例 (每个实例可以有很多 watch)
fs.inotify.max_user_watches="524288" # 表示同一用户同时可以添加的watch数目（watch一般是针对目录，决定了同时同一用户可以监控的目录数量) 默认值 8192 在容器场景下偏小，在某些情况下可能会导致 inotify watch 数量耗尽，使得创建 Pod 不成功或者 kubelet 无法启动成功，将其优化到 524288
```