---
title: "Cleanup Kubernetes components"
date: 2018-08-07T12:37:34-07:00
weight: 15
---

Delete the app namespace along with its AWS resources

```bash
# Delete the virtual service
kubectl delete ns appmesh-workshop-ns
```

Delete the cluster

```bash
eksctl delete cluster appmesh-workshop
```
