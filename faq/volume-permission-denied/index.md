---
title: "容器启动报错: Permission denied"
type: book
date: "2021-05-11"
weight: 20
---

## 背景

我们在 Kubernetes 中挂载 volume 后，有时会遇到容器启动报错 `Permission denied`，然后 Crash 并重启，一般是文件系统权限问题，本文介绍如何解决。

## 修改 NFS 目录权限

如果挂载的 volume 存储是 nfs ，在不改动 Kubernetes 部署的情况下，我们可以将挂载的 nfs 目录权限修改成应用启动时使用的 uid 和 gid:

```bash
sudo mount -t nfs -o vers=4.0,noresvport 10.10.0.15:/ /localfolder # 挂载 nfs 根目录到本地目录 (如/localfolder)
sudo chown -R 1001:1001 /localfolder/mariadb # 修改挂载 volume 中使用的目录权限为指定的 uid:gid
```

## 使用 initContainers 修改目录权限

为 Pod 加入 initContainers 来执行 `chown` 修改容器内挂载点目录的权限:

```yaml
  initContainers:
  - name: chown
     image: debian:stable-slim
     command:
      - /bin/sh
      - -c 
      - "chown -R 1001:1001 /data/mnt"
```