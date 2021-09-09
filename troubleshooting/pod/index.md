---
title: "Pod 排障"
type: book
---

## Pod 一直处于 Pending 状态

Pending 状态说明 Pod 还没有被调度到某个节点上，需要看下 Pod 事件进一步判断原因，比如:

``` bash
$ kubectl describe pod tikv-0
...
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  3m (x106 over 33m)  default-scheduler  0/4 nodes are available: 1 node(s) had no available volume zone, 2 Insufficient cpu, 3 Insufficient memory.
```

下面列举下可能原因和解决方法。

### 节点资源不够

节点资源不够有以下几种情况:

* CPU 负载过高
* 剩余可以被分配的内存不够
* 剩余可用 GPU 数量不够 (通常在机器学习场景，GPU 集群环境)

如果判断某个 Node 资源是否足够？ 通过 `kubectl describe node <node-name>` 查看 node 资源情况，关注以下信息：

* `Allocatable`: 表示此节点能够申请的资源总和
* `Allocated resources`: 表示此节点已分配的资源 (Allocatable 减去节点上所有 Pod 总的 Request)

前者与后者相减，可得出剩余可申请的资源。如果这个值小于 Pod 的 request，就不满足 Pod 的资源要求，Scheduler 在 Predicates (预选) 阶段就会剔除掉这个 Node，也就不会调度上去。


### 不满足 nodeSelector 与 affinity

如果 Pod 包含 nodeSelector 指定了节点需要包含的 label，调度器将只会考虑将 Pod 调度到包含这些 label 的 Node 上，如果没有 Node 有这些 label 或者有这些 label 的 Node 其它条件不满足也将会无法调度。参考官方文档：https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector

如果 Pod 包含 affinity（亲和性）的配置，调度器根据调度算法也可能算出没有满足条件的 Node，从而无法调度。affinity 有以下几类:

* nodeAffinity: 节点亲和性，可以看成是增强版的 nodeSelector，用于限制 Pod 只允许被调度到某一部分 Node。
* podAffinity: Pod 亲和性，用于将一些有关联的 Pod 调度到同一个地方，同一个地方可以是指同一个节点或同一个可用区的节点等。
* podAntiAffinity: Pod 反亲和性，用于避免将某一类 Pod 调度到同一个地方避免单点故障，比如将集群 DNS 服务的 Pod 副本都调度到不同节点，避免一个节点挂了造成整个集群 DNS 解析失败，使得业务中断。

### Node 存在 Pod 没有容忍的污点

如果节点上存在污点 (Taints)，而 Pod 没有响应的容忍 (Tolerations)，Pod 也将不会调度上去。通过 describe node 可以看下 Node 有哪些 Taints:

``` bash
$ kubectl describe nodes host1
...
Taints:             special=true:NoSchedule
...
```

通常解决方法有两个:

1. 删除污点:

``` bash
kubectl taint nodes host1 special-
```

2. 给 Pod 加上这个污点的容忍:

``` yaml
tolerations:
- key: "special"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

我们通常使用后者的方法来解决。污点既可以是手动添加也可以是被自动添加，下面来深入分析一下。

#### 手动添加的污点

通过类似以下方式可以给节点添加污点:

``` bash
$ kubectl taint node host1 special=true:NoSchedule
node "host1" tainted
```

另外，有些场景下希望新加的节点默认不调度 Pod，直到调整完节点上某些配置才允许调度，就给新加的节点都加上 `node.kubernetes.io/unschedulable` 这个污点。

#### 自动添加的污点

如果节点运行状态不正常，污点也可以被自动添加，从 v1.12 开始，`TaintNodesByCondition` 特性进入 Beta 默认开启，controller manager 会检查 Node 的 Condition，如果命中条件就自动为 Node 加上相应的污点，这些 Condition 与 Taints 的对应关系如下:

``` txt
Conditon               Value       Taints
--------               -----       ------
OutOfDisk              True        node.kubernetes.io/out-of-disk
Ready                  False       node.kubernetes.io/not-ready
Ready                  Unknown     node.kubernetes.io/unreachable
MemoryPressure         True        node.kubernetes.io/memory-pressure
PIDPressure            True        node.kubernetes.io/pid-pressure
DiskPressure           True        node.kubernetes.io/disk-pressure
NetworkUnavailable     True        node.kubernetes.io/network-unavailable
```

解释下上面各种条件的意思:

* OutOfDisk 为 True 表示节点磁盘空间不够了
* Ready 为 False 表示节点不健康
* Ready 为 Unknown 表示节点失联，在 `node-monitor-grace-period` 这么长的时间内没有上报状态 controller-manager 就会将 Node 状态置为 Unknown (默认 40s)
* MemoryPressure 为 True 表示节点内存压力大，实际可用内存很少
* PIDPressure 为 True 表示节点上运行了太多进程，PID 数量不够用了
* DiskPressure 为 True 表示节点上的磁盘可用空间太少了
* NetworkUnavailable 为 True 表示节点上的网络没有正确配置，无法跟其它 Pod 正常通信

另外，在云环境下，比如腾讯云 TKE，添加新节点会先给这个 Node 加上 `node.cloudprovider.kubernetes.io/uninitialized` 的污点，等 Node 初始化成功后才自动移除这个污点，避免 Pod 被调度到没初始化好的 Node 上。

### 低版本 kube-scheduler 的 bug

可能是低版本 `kube-scheduler` 的 bug, 可以升级下调度器版本。

### kube-scheduler 没有正常运行

检查 maser 上的 `kube-scheduler` 是否运行正常，异常的话可以尝试重启临时恢复。

### 驱逐后其它可用节点与当前节点有状态应用不在同一个可用区

有时候服务部署成功运行过，但在某个时候节点突然挂了，此时就会触发驱逐，创建新的副本调度到其它节点上，对于已经挂载了磁盘的 Pod，它通常需要被调度到跟当前节点和磁盘在同一个可用区，如果集群中同一个可用区的节点不满足调度条件，即使其它可用区节点各种条件都满足，但不跟当前节点在同一个可用区，也是不会调度的。为什么需要限制挂载了磁盘的 Pod 不能漂移到其它可用区的节点？试想一下，云上的磁盘虽然可以被动态挂载到不同机器，但也只是相对同一个数据中心，通常不允许跨数据中心挂载磁盘设备，因为网络时延会极大的降低 IO 速率。

## Pod 一直处于 Terminating 状态
### 磁盘爆满

如果 docker 的数据目录所在磁盘被写满，docker 无法正常运行，无法进行删除和创建操作，所以 kubelet 调用 docker 删除容器没反应，看 event 类似这样：

```bash
Normal  Killing  39s (x735 over 15h)  kubelet, 10.179.80.31  Killing container with id docker://apigateway:Need to kill Pod
```

### 存在 "i" 文件属性

如果容器的镜像本身或者容器启动后写入的文件存在 "i" 文件属性，此文件就无法被修改删除，而删除 Pod 时会清理容器目录，但里面包含有不可删除的文件，就一直删不了，Pod 状态也将一直保持 Terminating，kubelet 报错:

``` log
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.922965   14109 remote_runtime.go:250] RemoveContainer "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257" from runtime service failed: rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.923027   14109 kuberuntime_gc.go:126] Failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
```

通过 `man chattr` 查看 "i" 文件属性描述:

``` txt
       A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no
link can be created to this file and no data can be written to the file.  Only the superuser
or a process possessing the CAP_LINUX_IMMUTABLE capability can set or clear this attribute.
```

彻底解决当然是不要在容器镜像中或启动后的容器设置 "i" 文件属性，临时恢复方法： 复制 kubelet 日志报错提示的文件路径，然后执行 `chattr -i <file>`:

``` bash
chattr -i /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash
```

执行完后等待 kubelet 自动重试，Pod 就可以被自动删除了。

### docker 17 的 bug

docker hang 住，没有任何响应，看 event:

```bash
Warning FailedSync 3m (x408 over 1h) kubelet, 10.179.80.31 error determining status: rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

怀疑是17版本dockerd的BUG。可通过 `kubectl -n cn-staging delete pod apigateway-6dc48bf8b6-clcwk --force --grace-period=0` 强制删除pod，但 `docker ps` 仍看得到这个容器

处置建议：

* 升级到docker 18. 该版本使用了新的 containerd，针对很多bug进行了修复。
* 如果出现terminating状态的话，可以提供让容器专家进行排查，不建议直接强行删除，会可能导致一些业务上问题。

### 存在 Finalizers

k8s 资源的 metadata 里如果存在 `finalizers`，那么该资源一般是由某程序创建的，并且在其创建的资源的 metadata 里的 `finalizers` 加了一个它的标识，这意味着这个资源被删除时需要由创建资源的程序来做删除前的清理，清理完了它需要将标识从该资源的 `finalizers` 中移除，然后才会最终彻底删除资源。比如 Rancher 创建的一些资源就会写入 `finalizers` 标识。

处理建议：`kubectl edit` 手动编辑资源定义，删掉 `finalizers`，这时再看下资源，就会发现已经删掉了

### 低版本 kubelet list-watch 的 bug

之前遇到过使用 v1.8.13 版本的 k8s，kubelet 有时 list-watch 出问题，删除 pod 后 kubelet 没收到事件，导致 kubelet 一直没做删除操作，所以 pod 状态一直是 Terminating

### dockerd 与 containerd 的状态不同步

判断 dockerd 与 containerd 某个容器的状态不同步的方法：

* describe pod 拿到容器 id
* docker ps 查看的容器状态是 dockerd 中保存的状态
* 通过 docker-container-ctr 查看容器在 containerd 中的状态，比如:
  ``` bash
  $ docker-container-ctr --namespace moby --address /var/run/docker/containerd/docker-containerd.sock task ls |grep a9a1785b81343c3ad2093ad973f4f8e52dbf54823b8bb089886c8356d4036fe0
  a9a1785b81343c3ad2093ad973f4f8e52dbf54823b8bb089886c8356d4036fe0    30639    STOPPED
  ```

containerd 看容器状态是 stopped 或者已经没有记录，而 docker 看容器状态却是 runing，说明 dockerd 与 containerd 之间容器状态同步有问题，目前发现了 docker 在 aufs 存储驱动下如果磁盘爆满可能发生内核 panic :

``` txt
aufs au_opts_verify:1597:dockerd[5347]: dirperm1 breaks the protection by the permission bits on the lower branch
```

如果磁盘爆满过，dockerd 一般会有下面类似的日志:

``` log
Sep 18 10:19:49 VM-1-33-ubuntu dockerd[4822]: time="2019-09-18T10:19:49.903943652+08:00" level=error msg="Failed to log msg \"\" for logger json-file: write /opt/docker/containers/54922ec8b1863bcc504f6dac41e40139047f7a84ff09175d2800100aaccbad1f/54922ec8b1863bcc504f6dac41e40139047f7a84ff09175d2800100aaccbad1f-json.log: no space left on device"
```

随后可能发生状态不同步，已提issue:  https://github.com/docker/for-linux/issues/779

* 临时恢复: 执行 `docker container prune` 或重启 dockerd
* 长期方案: 运行时推荐直接使用 containerd，绕过 dockerd 避免 docker 本身的各种 BUG

### Daemonset Controller 的 BUG

有个 k8s 的 bug 会导致 daemonset pod 无限 terminating，1.10 和 1.11 版本受影响，原因是 daemonset controller 复用 scheduler 的 predicates 逻辑，里面将 nodeAffinity 的 nodeSelector 数组做了排序（传的指针），spec 就会跟 apiserver 中的不一致，daemonset controller 又会为 rollingUpdate类型计算 hash (会用到spec)，用于版本控制，造成不一致从而无限启动和停止的循环。

* issue: https://github.com/kubernetes/kubernetes/issues/66298
* 修复的PR: https://github.com/kubernetes/kubernetes/pull/66480

升级集群版本可以彻底解决，临时规避可以给 rollingUpdate 类型 daemonset 不使用 nodeAffinity，改用 nodeSelector。

### mount 的目录被其它进程占用

dockerd 报错 `device or resource busy`:

``` bash
May 09 09:55:12 VM_0_21_centos dockerd[6540]: time="2020-05-09T09:55:12.774467604+08:00" level=error msg="Handler for DELETE /v1.38/containers/b62c3796ea2ed5a0bd0eeed0e8f041d12e430a99469dd2ced6f94df911e35905 returned error: container b62c3796ea2ed5a0bd0eeed0e8f041d12e430a99469dd2ced6f94df911e35905: driver \"overlay2\" failed to remove root filesystem: remove /data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged: device or resource busy"
```

查找还有谁在"霸占"此目录:

``` bash
$ grep 8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59 /proc/*/mountinfo
/proc/27187/mountinfo:4500 4415 0:898 / /var/lib/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged rw,relatime - overlay overlay rw,lowerdir=/data/docker/overlay2/l/DNQH6VPJHFFANI36UDKS262BZK:/data/docker/overlay2/l/OAYZKUKWNH7GPT4K5MFI6B7OE5:/data/docker/overlay2/l/ANQD5O27DRMTZJG7CBHWUA65YT:/data/docker/overlay2/l/G4HYAKVIRVUXB6YOXRTBYUDVB3:/data/docker/overlay2/l/IRGHNAKBHJUOKGLQBFBQTYFCFU:/data/docker/overlay2/l/6QG67JLGKMFXGVB5VCBG2VYWPI:/data/docker/overlay2/l/O3X5VFRX2AO4USEP2ZOVNLL4ZK:/data/docker/overlay2/l/H5Q5QE6DMWWI75ALCIHARBA5CD:/data/docker/overlay2/l/LFISJNWBKSRTYBVBPU6PH3YAAZ:/data/docker/overlay2/l/JSF6H5MHJEC4VVAYOF5PYIMIBQ:/data/docker/overlay2/l/7D2F45I5MF2EHDOARROYPXCWHZ:/data/docker/overlay2/l/OUJDAGNIZXVBKBWNYCAUI5YSGG:/data/docker/overlay2/l/KZLUO6P3DBNHNUH2SNKPTFZOL7:/data/docker/overlay2/l/O2BPSFNCVXTE4ZIWGYSRPKAGU4,upperdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/diff,workdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/work
/proc/27187/mountinfo:4688 4562 0:898 / /var/lib/docker/overlay2/81c322896bb06149c16786dc33c83108c871bb368691f741a1e3a9bfc0a56ab2/merged/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged rw,relatime - overlay overlay rw,lowerdir=/data/docker/overlay2/l/DNQH6VPJHFFANI36UDKS262BZK:/data/docker/overlay2/l/OAYZKUKWNH7GPT4K5MFI6B7OE5:/data/docker/overlay2/l/ANQD5O27DRMTZJG7CBHWUA65YT:/data/docker/overlay2/l/G4HYAKVIRVUXB6YOXRTBYUDVB3:/data/docker/overlay2/l/IRGHNAKBHJUOKGLQBFBQTYFCFU:/data/docker/overlay2/l/6QG67JLGKMFXGVB5VCBG2VYWPI:/data/docker/overlay2/l/O3X5VFRX2AO4USEP2ZOVNLL4ZK:/data/docker/overlay2/l/H5Q5QE6DMWWI75ALCIHARBA5CD:/data/docker/overlay2/l/LFISJNWBKSRTYBVBPU6PH3YAAZ:/data/docker/overlay2/l/JSF6H5MHJEC4VVAYOF5PYIMIBQ:/data/docker/overlay2/l/7D2F45I5MF2EHDOARROYPXCWHZ:/data/docker/overlay2/l/OUJDAGNIZXVBKBWNYCAUI5YSGG:/data/docker/overlay2/l/KZLUO6P3DBNHNUH2SNKPTFZOL7:/data/docker/overlay2/l/O2BPSFNCVXTE4ZIWGYSRPKAGU4,upperdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/diff,workdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/work
```

> 自行替换容器 id

找到进程号后查看此进程更多详细信息:

``` bash
ps -f 27187
```

## Pod Terminating 慢

### 可能原因

* 进程通过 bash -c 启动导致 kill 信号无法透传给业务进程，参考 [在 SHELL 中传递信号](https://imroc.cc/k8s/trick/propagating-signals-in-shell/)

## Pod 一直处于 Unknown 状态

通常是节点失联，没有上报状态给 apiserver，到达阀值后 controller-manager 认为节点失联并将其状态置为 `Unknown`。

可能原因:

* 节点高负载导致无法上报
* 节点宕机
* 节点被关机
* 网络不通

## Pod 一直处于 Pending 状态
Pending 状态说明 Pod 还没有被调度到某个节点上，需要看下 Pod 事件进一步判断原因，比如:

``` bash
$ kubectl describe pod tikv-0
...
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  3m (x106 over 33m)  default-scheduler  0/4 nodes are available: 1 node(s) had no available volume zone, 2 Insufficient cpu, 3 Insufficient memory.
```

下面列举下可能原因和解决方法。

### 节点资源不够

节点资源不够有以下几种情况:

* CPU 负载过高
* 剩余可以被分配的内存不够
* 剩余可用 GPU 数量不够 (通常在机器学习场景，GPU 集群环境)

如果判断某个 Node 资源是否足够？ 通过 `kubectl describe node <node-name>` 查看 node 资源情况，关注以下信息：

* `Allocatable`: 表示此节点能够申请的资源总和
* `Allocated resources`: 表示此节点已分配的资源 (Allocatable 减去节点上所有 Pod 总的 Request)

前者与后者相减，可得出剩余可申请的资源。如果这个值小于 Pod 的 request，就不满足 Pod 的资源要求，Scheduler 在 Predicates (预选) 阶段就会剔除掉这个 Node，也就不会调度上去。


### 不满足 nodeSelector 与 affinity

如果 Pod 包含 nodeSelector 指定了节点需要包含的 label，调度器将只会考虑将 Pod 调度到包含这些 label 的 Node 上，如果没有 Node 有这些 label 或者有这些 label 的 Node 其它条件不满足也将会无法调度。参考官方文档：https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector

如果 Pod 包含 affinity（亲和性）的配置，调度器根据调度算法也可能算出没有满足条件的 Node，从而无法调度。affinity 有以下几类:

* nodeAffinity: 节点亲和性，可以看成是增强版的 nodeSelector，用于限制 Pod 只允许被调度到某一部分 Node。
* podAffinity: Pod 亲和性，用于将一些有关联的 Pod 调度到同一个地方，同一个地方可以是指同一个节点或同一个可用区的节点等。
* podAntiAffinity: Pod 反亲和性，用于避免将某一类 Pod 调度到同一个地方避免单点故障，比如将集群 DNS 服务的 Pod 副本都调度到不同节点，避免一个节点挂了造成整个集群 DNS 解析失败，使得业务中断。

### Node 存在 Pod 没有容忍的污点

如果节点上存在污点 (Taints)，而 Pod 没有响应的容忍 (Tolerations)，Pod 也将不会调度上去。通过 describe node 可以看下 Node 有哪些 Taints:

``` bash
$ kubectl describe nodes host1
...
Taints:             special=true:NoSchedule
...
```

通常解决方法有两个:

1. 删除污点:

``` bash
kubectl taint nodes host1 special-
```

2. 给 Pod 加上这个污点的容忍:

``` yaml
tolerations:
- key: "special"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

我们通常使用后者的方法来解决。污点既可以是手动添加也可以是被自动添加，下面来深入分析一下。

#### 手动添加的污点

通过类似以下方式可以给节点添加污点:

``` bash
$ kubectl taint node host1 special=true:NoSchedule
node "host1" tainted
```

另外，有些场景下希望新加的节点默认不调度 Pod，直到调整完节点上某些配置才允许调度，就给新加的节点都加上 `node.kubernetes.io/unschedulable` 这个污点。

#### 自动添加的污点

如果节点运行状态不正常，污点也可以被自动添加，从 v1.12 开始，`TaintNodesByCondition` 特性进入 Beta 默认开启，controller manager 会检查 Node 的 Condition，如果命中条件就自动为 Node 加上相应的污点，这些 Condition 与 Taints 的对应关系如下:

``` txt
Conditon               Value       Taints
--------               -----       ------
OutOfDisk              True        node.kubernetes.io/out-of-disk
Ready                  False       node.kubernetes.io/not-ready
Ready                  Unknown     node.kubernetes.io/unreachable
MemoryPressure         True        node.kubernetes.io/memory-pressure
PIDPressure            True        node.kubernetes.io/pid-pressure
DiskPressure           True        node.kubernetes.io/disk-pressure
NetworkUnavailable     True        node.kubernetes.io/network-unavailable
```

解释下上面各种条件的意思:

* OutOfDisk 为 True 表示节点磁盘空间不够了
* Ready 为 False 表示节点不健康
* Ready 为 Unknown 表示节点失联，在 `node-monitor-grace-period` 这么长的时间内没有上报状态 controller-manager 就会将 Node 状态置为 Unknown (默认 40s)
* MemoryPressure 为 True 表示节点内存压力大，实际可用内存很少
* PIDPressure 为 True 表示节点上运行了太多进程，PID 数量不够用了
* DiskPressure 为 True 表示节点上的磁盘可用空间太少了
* NetworkUnavailable 为 True 表示节点上的网络没有正确配置，无法跟其它 Pod 正常通信

另外，在云环境下，比如腾讯云 TKE，添加新节点会先给这个 Node 加上 `node.cloudprovider.kubernetes.io/uninitialized` 的污点，等 Node 初始化成功后才自动移除这个污点，避免 Pod 被调度到没初始化好的 Node 上。

### 低版本 kube-scheduler 的 bug

可能是低版本 `kube-scheduler` 的 bug, 可以升级下调度器版本。

### kube-scheduler 没有正常运行

检查 maser 上的 `kube-scheduler` 是否运行正常，异常的话可以尝试重启临时恢复。

### 驱逐后其它可用节点与当前节点有状态应用不在同一个可用区

有时候服务部署成功运行过，但在某个时候节点突然挂了，此时就会触发驱逐，创建新的副本调度到其它节点上，对于已经挂载了磁盘的 Pod，它通常需要被调度到跟当前节点和磁盘在同一个可用区，如果集群中同一个可用区的节点不满足调度条件，即使其它可用区节点各种条件都满足，但不跟当前节点在同一个可用区，也是不会调度的。为什么需要限制挂载了磁盘的 Pod 不能漂移到其它可用区的节点？试想一下，云上的磁盘虽然可以被动态挂载到不同机器，但也只是相对同一个数据中心，通常不允许跨数据中心挂载磁盘设备，因为网络时延会极大的降低 IO 速率。

## Pod 处于 CrashLoopBackOff 状态

Pod 如果处于 `CrashLoopBackOff` 状态说明之前是启动了，只是又异常退出了，只要 Pod 的 [restartPolicy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 不是 Never 就可能被重启拉起，此时 Pod 的 `RestartCounts` 通常是大于 0 的，可以先看下容器进程的退出状态码来缩小问题范围，参考 [分析 ExitCode 定位 Pod 异常退出原因](https://imroc.cc/k8s/troubleshooting/analysis-exitcode/)

### 容器进程主动退出

如果是容器进程主动退出，退出状态码一般在 0-128 之间，除了可能是业务程序 BUG，还有其它许多可能原因。

### 系统 OOM

如果发生系统 OOM，可以看到 Pod 中容器退出状态码是 137，表示被 `SIGKILL` 信号杀死，同时内核会报错: `Out of memory: Kill process ...`。大概率是节点上部署了其它非 K8S 管理的进程消耗了比较多的内存，或者 kubelet 的 `--kube-reserved` 和 `--system-reserved` 配的比较小，没有预留足够的空间给其它非容器进程，节点上所有 Pod 的实际内存占用总量不会超过 `/sys/fs/cgroup/memory/kubepods` 这里 cgroup 的限制，这个限制等于 `capacity - "kube-reserved" - "system-reserved"`，如果预留空间设置合理，节点上其它非容器进程（kubelet, dockerd, kube-proxy, sshd 等) 内存占用没有超过 kubelet 配置的预留空间是不会发生系统 OOM 的，可以根据实际需求做合理的调整。

### cgroup OOM

如果是 cgrou OOM 杀掉的进程，从 Pod 事件的下 `Reason` 可以看到是 `OOMKilled`，说明容器实际占用的内存超过 limit 了，同时内核日志会报: `Memory cgroup out of memory`。 可以根据需求调整下 limit。

### 节点内存碎片化

如果节点上内存碎片化严重，缺少大页内存，会导致即使总的剩余内存较多，但还是会申请内存失败，参考 [内存碎片化](https://imroc.cc/k8s/troubleshooting/memory-fragmentation/)

### 健康检查失败

参考 [Pod 健康检查失败](https://imroc.cc/k8s/troubleshooting/healthcheck-failed/) 进一步定位。
