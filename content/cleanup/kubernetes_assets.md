---
title: "Cleanup Kubernetes components"
date: 2018-08-07T12:37:34-07:00
weight: 35
---

* Delete the components from your EKS cluster:

```bash
# Delete the virtual service
kubectl delete virtualservice nodejs.appmeshworkshop.hosted.local -n appmesh-workshop-ns
# Delete the virtual node
kubectl delete virtualnode nodejs-app -n appmesh-workshop-ns
# Delete the deployment
kubectl delete deployment nodejs-app -n appmesh-workshop-ns
```

