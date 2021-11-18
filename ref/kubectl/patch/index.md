---
title: kubectl patch
type: book
date: "2021-11-18"
---

## 清理 finalizers

```bash
kubectl get federatedtypeconfigs.core.kubefed.io | grep -v NAME | awk '{print $1}' | xargs -I {} kubectl patch federatedtypeconfigs.core.kubefed.io {} --type=json -p='[{"op": "remove", "path": "/metadata/finalizers"}]'
```